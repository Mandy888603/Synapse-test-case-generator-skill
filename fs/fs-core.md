# FS/TS文档核心提取

**本章职责：从Functional Specification/Technical Specification文档提取业务规则、字段定义、cutoff日期等关键信息。**

---

## 2.2 docx结构化提取（章节完整读取）

**安装**：`python-docx`库（必须）

**提取目标**：将`<TBD_*>`占位符替换为真实值

**提取方法**：
1. **定位章节**：找到目标章节的起止段落（如"4.3.3 Core KPI logic"到下一个标题）
2. **完整读取**：提取该区间内的所有段落和表格
3. **结构验证**：检查是否遗漏表格/字段（统计表格数量、字段数量）
4. **⚠️ 阻断点**：**向用户展示提取清单，等待确认后再继续**

---

## 提取要素清单

| 要素 | 提取规则 | 提取验证方法 |
|------|---------|----------|
| `itcm_number` | ITCM编号 | `ITCM(\d+)` |
| `project_name` | 项目名称 | 文档标题或Project字段 |
| `pm_name` | 项目经理 | PM/项目经理字段 |
| `table_names` | 全限定表名 | `[gold\|silver\|prepared\|staging\|ingress].dbo.[\w_]+` |
| `field_names` | 新增/修改字段 | 列出所有提取的字段，按表格分组展示 |
| `business_rules` | 业务逻辑 | 列出所有业务规则（如"Current Backlog: 3-condition filter"），标注所在表格编号 |
| `cutoff_dates` | 生效日期 | `\d{6}`（如202603）在比较运算符后 |
| `data_sources` | 源系统 | SAP BW4 OData, CRM等标识 |

---

## 提取代码示例

```python
from docx import Document
import re

def extract_fs_doc(docx_path):
    """提取FS/TS文档关键信息"""
    doc = Document(docx_path)
    
    extraction = {
        'itcm_number': None,
        'project_name': None,
        'pm_name': None,
        'tables': {},
        'cutoff_dates': [],
        'business_rules': []
    }
    
    # 遍历所有段落
    for para in doc.paragraphs:
        text = para.text.strip()
        
        # 提取ITCM编号
        if not extraction['itcm_number']:
            match = re.search(r'ITCM(\d+)', text)
            if match:
                extraction['itcm_number'] = match.group(1)
        
        # 提取项目名称
        if 'Project' in text and ':' in text:
            extraction['project_name'] = text.split(':')[-1].strip()
            
        # 提取PM
        if 'PM' in text or 'Project Manager' in text:
            extraction['pm_name'] = text.split(':')[-1].strip()
        
        # 提取cutoff日期 (格式: 202603)
        dates = re.findall(r'>=?\s*(\d{6})', text)
        extraction['cutoff_dates'].extend(dates)
        
        # 提取表名
        tables = re.findall(r'(gold|silver|prepared|staging|ingress)\.dbo\.(\w+)', text)
        for db, table in tables:
            full_name = f"{db}.dbo.{table}"
            if full_name not in extraction['tables']:
                extraction['tables'][full_name] = {
                    'fields': [],
                    'rules': []
                }
    
    # 遍历所有表格
    for table in doc.tables:
        # 提取表格中的字段定义
        fields = extract_fields_from_table(table)
        
        # 提取业务规则
        rules = extract_business_rules(table)
        
    return extraction

def extract_fields_from_table(table):
    """从表格提取字段定义"""
    fields = []
    
    # 假设表格结构：Field Name | Data Type | Description
    for row in table.rows[1:]:  # 跳过表头
        cells = row.cells
        if len(cells) >= 3:
            field = {
                'name': cells[0].text.strip(),
                'type': cells[1].text.strip(),
                'description': cells[2].text.strip()
            }
            fields.append(field)
    
    return fields

def extract_business_rules(table):
    """从表格提取业务规则"""
    rules = []
    
    # 查找包含"Rule", "Logic", "Condition"的单元格
    for row in table.rows:
        for cell in row.cells:
            text = cell.text.strip()
            if any(kw in text.lower() for kw in ['rule', 'logic', 'condition', 'filter']):
                rules.append(text)
    
    return rules
```

---

## 常见文档结构模式

### 模式1：标准FS文档结构
```
1. Introduction
2. Business Requirements
3. Data Model
   3.1 Source Systems
   3.2 Target Tables
       3.2.1 Table: dm_job_costing
             - Field Definitions (表格)
             - Business Rules
4. Data Mapping
   4.1 Field Mapping Details (表格)
5. Business Logic
   5.1 Calculation Rules
   5.2 Filter Conditions
6. Cutoff Date
   6.1 Effective Date: 202603
```

### 模式2：Technical Specification结构
```
1. Overview
2. Architecture
3. ETL Design
   3.1 Data Flow
   3.2 Transformations
4. Table Definitions
   4.1 CREATE TABLE Statements
   4.2 Field Descriptions (表格)
5. Business Logic Implementation
   5.1 SQL Logic
   5.2 Cutoff Date Handling
```

---

## 关键信息定位策略

### 定位Cutoff日期
```python
def find_cutoff_date(doc):
    """查找cutoff日期"""
    patterns = [
        r'cutoff\s*date\s*[:=]\s*(\d{6})',
        r'effective\s*date\s*[:=]\s*(\d{6})',
        r'>=\s*(\d{6})',
        r'from\s+(\d{6})',
        r'start\s*date\s*[:=]\s*(\d{6})'
    ]
    
    for para in doc.paragraphs:
        text = para.text
        for pattern in patterns:
            match = re.search(pattern, text, re.IGNORECASE)
            if match:
                return match.group(1)
    
    return '<TBD_CUTOFF_DATE>'
```

### 定位业务规则
```python
def find_business_rules(doc):
    """查找业务规则"""
    rules = []
    
    keywords = [
        'condition', 'filter', 'rule', 'logic',
        'when', 'if', 'case', 'criteria'
    ]
    
    for para in doc.paragraphs:
        text = para.text.strip()
        if any(kw in text.lower() for kw in keywords):
            # 提取规则描述（通常包含冒号或破折号）
            if ':' in text or '-' in text:
                rules.append(text)
    
    return rules
```

### 定位表名
```python
def find_table_names(doc):
    """查找全限定表名"""
    tables = set()
    
    # 模式1: database.dbo.table
    pattern1 = r'(gold|silver|prepared|staging|ingress)\.dbo\.(\w+)'
    
    # 模式2: CREATE TABLE语句
    pattern2 = r'CREATE\s+TABLE\s+(\w+\.\w+\.\w+)'
    
    for para in doc.paragraphs:
        text = para.text
        
        matches = re.findall(pattern1, text, re.IGNORECASE)
        for db, table in matches:
            tables.add(f"{db}.dbo.{table}")
        
        matches = re.findall(pattern2, text, re.IGNORECASE)
        for full_name in matches:
            tables.add(full_name.lower())
    
    return list(tables)
```

---

## 错误处理

| 错误类型 | 处理策略 |
|---------|----------|
| 文件被Word锁定 | 提示关闭文件，等待OneDrive同步 |
| 包未安装 | 提示`install_python_packages(["python-docx"])` |
| 解析失败 | 保留`<TBD_*>`占位符，标记TODO |
| 章节未找到 | 标记`<TBD_KEYWORD_NOT_FOUND>` |
| 表格格式异常 | 记录警告，尝试模糊匹配 |

---

## 提取完整性验证

提取完成后必须验证：

```python
def validate_extraction(extraction):
    """验证提取结果完整性"""
    issues = []
    
    # 必填项检查
    if not extraction['itcm_number']:
        issues.append("ITCM编号未提取")
    
    if not extraction['project_name']:
        issues.append("项目名称未提取")
        
    if not extraction['tables']:
        issues.append("未提取到任何表名")
    
    # 字段检查
    for table_name, table_info in extraction['tables'].items():
        if not table_info['fields']:
            issues.append(f"表{table_name}未提取到字段")
    
    # Cutoff日期检查
    if not extraction['cutoff_dates']:
        issues.append("Cutoff日期未提取，将使用<TBD_CUTOFF_DATE>")
    
    return issues
```

---

## 输出格式

提取结果输出为结构化YAML：

```yaml
fs_extraction:
  metadata:
    source_file: "FS_Repair_Costing_ITCM1812420.docx"
    extraction_timestamp: "2026-04-03T10:30:00Z"
    
  project_info:
    itcm_number: "1812420"
    project_name: "Repair Costing Enhancement"
    pm_name: "John Smith"
    
  tables:
    gold.dbo.dm_job_costing:
      fields:
        - name: "repair_tender_id"
          type: "INTEGER"
          description: "维修投标唯一标识"
          nullable: false
          
        - name: "supervisor"
          type: "VARCHAR(100)"
          description: "主管名称"
          nullable: true
          
      business_rules:
        - "supervisor: COALESCE(supervisor, 'Unknown')"
        - "cutoff_date: calendar_year_month >= 202603"
        
    silver.dbo.dwt_job_costing:
      fields:
        - name: "repair_tender_id"
          type: "INTEGER"
          
  cutoff_dates:
    - "202603"
    
  data_sources:
    - "SAP BW4 OData"
    - "CRM System"
```
