# 大文档处理与边界情况

**适用场景**：大文档（段落数>50或文件大小>500KB）或用户指定特定章节提取

---

## 2.2.1 大文档分页与特定章节提取策略

### 准入条件

当检测到 docx 段落数 > 50、文件大小 > 500KB，或用户明确要求特定章节时，必须执行本子流程。

```yaml
large_document_threshold:
  token_limit: 6000
  trigger_condition: "docx段落数>50 或 文件大小>500KB 或 用户指定章节名"
```

---

## 内容定位规则

### 规则1: 全文段落扫描（替代Heading样式匹配）

```yaml
paragraph_scanner:
  required: true
  method: "全文段落扫描 + 关键词匹配"
  strategy: |
    1. 加载python-docx，遍历doc.paragraphs获取所有段落文本
    2. 不检查paragraph.style.name（因为可能无样式）
    3. 仅检查paragraph.text内容是否包含目标关键词
```

### 规则2: 章节边界识别（不依赖Heading样式）

```yaml
boundary_detection:
  primary_method: "关键词锚点 + 启发式边界"
  steps:
    - "步骤1: 全文搜索关键词（如'Repair Backlog'），记录所有匹配段落的索引（如match_idx=[45, 120, 200]）"
    - "步骤2: 确定章节起始=start_idx=第一个匹配段落"
    - "步骤3: 确定章节结束=扫描后续段落，遇到以下信号即停止：
        - 信号A: 出现下一个编号段落（如'4. Data Mapping'、'4.1 XXX'）
        - 信号B: 连续2个空行（段落文本为空或仅空格）
        - 信号C: 段落字体大小突变（如从10pt变为14pt且加粗）- 通过runs检测
        - 信号D: 出现新表格（<w:tbl>标签开始）
        - 信号E: 达到上下文窗口上限（4000 tokens）"
    - "步骤4: 如无法识别边界，默认提取从start_idx开始的连续20个段落或3000字符（取先到者）"
    
  placeholder_if_not_found: "<TBD_KEYWORD_NOT_FOUND>"
```

### 规则3: 硬性规则（修正版）

```yaml
mandatory_rules:
  - "禁止顺序读取前N个段落（如只读前50段）"
  - "必须基于关键词全文搜索定位，而非文档物理顺序"
  - "提取范围由段落索引（paragraph index）决定，而非Heading层级"
  - "未找到关键词时标记<TBD_KEYWORD_NOT_FOUND>，禁止退化为读取文档开头"
```

---

## 分页提取策略

```yaml
pagination_strategy:
  enabled: true
  unit: "段落数"  # 改为按段落计数，而非token（更鲁棒）
  chunk_size: 15  # 每次提取15个段落（约3000-4000 tokens）
  overlap: 2      # 与前一块重叠2个段落，确保连续性
  continuation_marker: "<CONTINUE_FROM_PARA_X>"  # 标记下一块起始段落索引
  
  sequence: |
    1. 先提取文档元数据（固定前10段，通常包含Project/ITCM/PM）
    2. 再基于关键词定位提取指定业务章节（如Repair Backlog）
    3. 如该章节>15段，分块提取并标记<CONTINUE>，等待用户确认后再提取下一块
```

---

## 实现代码示例

```python
from docx import Document
import re

def extract_large_doc_by_keyword(docx_path, keyword, chunk_size=15, overlap=2):
    """大文档按关键词分页提取"""
    
    doc = Document(docx_path)
    paragraphs = doc.paragraphs
    
    # 步骤1: 全文搜索关键词
    match_indices = []
    for i, para in enumerate(paragraphs):
        if keyword.lower() in para.text.lower():
            match_indices.append(i)
    
    if not match_indices:
        return {
            'status': 'NOT_FOUND',
            'content': f'<TBD_KEYWORD_NOT_FOUND: {keyword}>'
        }
    
    # 步骤2: 确定起始位置
    start_idx = match_indices[0]
    
    # 步骤3: 确定结束位置（启发式边界检测）
    end_idx = find_section_boundary(paragraphs, start_idx)
    
    # 步骤4: 分页提取
    total_paras = end_idx - start_idx + 1
    
    if total_paras <= chunk_size:
        # 小章节，一次性提取
        content = extract_paragraphs(paragraphs, start_idx, end_idx)
        return {
            'status': 'COMPLETE',
            'content': content,
            'range': (start_idx, end_idx)
        }
    else:
        # 大章节，分页提取
        chunks = []
        current = start_idx
        
        while current <= end_idx:
            chunk_end = min(current + chunk_size - 1, end_idx)
            chunk_content = extract_paragraphs(paragraphs, current, chunk_end)
            
            chunks.append({
                'chunk_id': len(chunks) + 1,
                'range': (current, chunk_end),
                'content': chunk_content,
                'has_more': chunk_end < end_idx
            })
            
            # 下一块起始（带重叠）
            current = chunk_end - overlap + 1
        
        return {
            'status': 'PAGINATED',
            'total_chunks': len(chunks),
            'chunks': chunks,
            'first_chunk': chunks[0]
        }

def find_section_boundary(paragraphs, start_idx):
    """启发式边界检测"""
    
    for i in range(start_idx + 1, len(paragraphs)):
        para = paragraphs[i]
        text = para.text.strip()
        
        # 信号A: 下一个编号段落
        if re.match(r'^\d+\.\s+\w+', text):
            return i - 1
        
        # 信号B: 连续2个空行
        if i > start_idx + 1:
            prev_text = paragraphs[i-1].text.strip()
            if not text and not prev_text:
                return i - 2
        
        # 信号C: 字体大小突变（简化版）
        if para.runs:
            current_size = para.runs[0].font.size
            if current_size and current_size.pt > 14:
                return i - 1
        
        # 信号E: 达到最大扫描范围
        if i - start_idx > 100:  # 最多扫描100段
            return i
    
    return len(paragraphs) - 1

def extract_paragraphs(paragraphs, start, end):
    """提取指定范围的段落文本"""
    content = []
    for i in range(start, end + 1):
        text = paragraphs[i].text.strip()
        if text:
            content.append(text)
    return '\n\n'.join(content)
```

---

## 完整性验证阻断点

```yaml
integrity_check:
  blocking: true
  validation_items:
    - "检查章节末尾是否有截断标记（段落是否以句号/完整句子结束）"
    - "检查是否遗漏用户明确要求的章节（关键词未找到时标记<TBD>）"
    - "检查关键业务要素是否为空（cutoff_date/business_rules字段发现数=0）"
    - "检查表格完整性（如提取范围内有表格开始标记<w:tbl>，确认有结束标记</w:tbl>）"
    
  failure_action: |
    标记<TBD_INCOMPLETE>，暂停进入2.3节，提示用户：
    "已提取 Repair Backlog 部分内容（段落X-Y），但检测到：
     [ ] 段落Y可能不是完整结束（无句号）
     [ ] 发现未闭合表格
     请选择：1) 继续提取下一块 2) 手动补充 3) 跳过"
```

---

## 5.2.3 文档解析完整性检查

```yaml
document_parse_integrity:
  - check: "Content_Continuity"
    description: "验证提取内容在段落层面是否连续（无跳跃，如从para[45]跳到para[50]而遗漏中间段落）"
    method: "检查提取的段落索引序列是否连续（45,46,47...而非45,50,51）"
    
  - check: "Table_Closure"
    description: "验证Word中的表格是否完整提取（无断行）"
    
  - check: "User_Specified_Chapters"
    description: "验证用户明确要求的章节（如Repair Backlog）是否已通过关键词定位提取"
    validation_method: "确认无<TBD_KEYWORD_NOT_FOUND>标记，且提取内容包含关键词"
    failure_critical: true
    
  - check: "Truncation_Markers"
    description: "全文搜索<TBD_TRUNCATED>、<TBD_INCOMPLETE>、<CONTINUE_FROM_PARA_X>，发现则标记为解析未完成"
    
  - check: "Keyword_Relevance"
    description: "验证提取内容确实与目标章节相关（包含业务关键词如tender_id/opportunity等）"
    threshold: "至少发现1个业务字段名或SQL关键词（如CASE WHEN, SELECT）"
```

---

## 边界情况处理

### 情况1: 关键词在多个位置出现
```python
# 策略：让用户选择或取第一个匹配
if len(match_indices) > 1:
    # 向用户展示所有匹配位置
    options = []
    for idx in match_indices:
        context = paragraphs[idx].text[:50]
        options.append(f"位置{idx}: {context}...")
    
    # 等待用户选择
    selected = ask_user_to_select(options)
    start_idx = match_indices[selected]
```

### 情况2: 章节边界模糊
```python
# 策略：提取固定范围+标记待确认
if boundary_uncertain:
    content = extract_paragraphs(paragraphs, start_idx, start_idx + 20)
    return {
        'status': 'BOUNDARY_UNCERTAIN',
        'content': content,
        'note': '<TBD_BOUNDARY: 章节边界不确定，请人工确认是否继续提取>'
    }
```

### 情况3: 表格跨段落
```python
# 策略：检测表格边界并完整提取
def extract_table_complete(doc, table_index):
    """完整提取表格，包括跨段落情况"""
    table = doc.tables[table_index]
    
    rows = []
    for row in table.rows:
        cells = [cell.text.strip() for cell in row.cells]
        rows.append(cells)
    
    return rows
```

---

## 用户交互流程

```
检测到文档>500KB或段落>50
    │
    ▼
询问用户：
"文档较大，请选择处理方式：
 1) 提取整个文档（可能分多次）
 2) 指定特定章节（输入章节名/关键词）
 3) 提取前N个字段（输入数量）"
    │
    ├──► 选项1 ──► 分页提取，每15段确认一次
    │
    ├──► 选项2 ──► 关键词定位提取
    │
    └──► 选项3 ──► 提取指定数量字段后停止
```
