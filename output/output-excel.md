# 4.2 Excel生成规范

**本章职责：标准化Excel输出格式，确保符合TKE模板要求。**

---

## 4.2.0 模板格式保留原则（强制执行）

**核心禁令**：**禁止**创建新的Excel工作簿（`Workbook()`），**必须**加载提供的模板文件。

### 强制性加载流程

```python
# 正确做法 - 必须遵循
template_path = "openspec/changes/[change-id]/TKE IT Project SIT Test Case - template_ITCMXXXXX.xlsx"
wb = load_workbook(template_path)  # 必须加载模板，禁止创建新文件
ws = wb['SIT Case']  # 必须定位到特定Sheet，禁止wb.active

# 错误做法 - 严格禁止
wb = Workbook()  # ❌ 禁止创建新工作簿，这会丢失所有模板格式
ws = wb.active   # ❌ 禁止使用active sheet，可能写到Change History
```

### 格式修改规则

| 场景 | 策略 | 代码示例 |
|------|------|----------|
| **行3-6表头信息** | 仅修改值，保留格式 | `ws.cell(row=3, column=5).value = project_name` |
| **行8列名** | **仅修改值**，保留模板格式 | `ws.cell(row=8, column=i).value = header` |
| **行9+测试数据** | 可设置格式（新数据） | `cell.font = Font(...); cell.alignment = Alignment(...)` |
| **Change History** | 追加新行，可设置格式 | `ch_ws.cell(next_row, 1).value = ...` |

---

## 4.2.1 Sheet与定位（强制）

### 必须显式指定Sheet

```python
# 伪代码（工具无关，逻辑必须）
ws = wb['SIT Case']  # 禁止 wb.active
ws.cell(row=3, column=5).value = project_name  # E3: Project
ws.cell(row=4, column=5).value = itcm_number   # E4: ITCM No.
ws.cell(row=5, column=5).value = pm_name       # E5: PM
ws.cell(row=6, column=5).value = "QA"          # E6: Environment
```

### Change History Sheet

- 仅用于追加变更记录，**不得写入测试用例或项目信息**
- 格式：`Change ID | Change Point | Change Date | Comments`
- 示例：`ITCM1812420 | Added 12 test cases | 2026-03-27 | Enhancement: supervisor logic`
- 从行2开始动态追加（不覆盖历史记录，找到第一个空白行继续写入）

```python
# Change History写入示例
ch_ws = wb['Change History']

# 找到第一个空白行
next_row = 2
while ch_ws.cell(next_row, 1).value:
    next_row += 1

# 写入变更记录
ch_ws.cell(next_row, 1).value = itcm_number
ch_ws.cell(next_row, 2).value = f"Added {test_case_count} test cases"
ch_ws.cell(next_row, 3).value = datetime.now().strftime('%Y-%m-%d')
ch_ws.cell(next_row, 4).value = f"{project_type}: {change_summary}"
```

---

## 4.2.2 内容写入（行9起）

### 列映射（13列）

| 列 | 字段名 | 格式/规则 |
|----|-------|----------|
| A | ID | 公式：`TC.{Level 1 No.}.{Level 2 No.}` |
| B | Level 1 No. | 自然数（1,2,3...） |
| C | Level1 Scenario | 英文（Schema Validation/Data Lineage/Business Logic） |
| D | Level 2 No. | 自然数（1,2,3...） |
| E | Level2 Scenario | 具体验证点（英文） |
| F | Prerequisites/test data | 具体数据条件（英文） |
| G | Procedure | ≤3步骤，编号1. 2. 3. （英文） |
| H | Sql | 四段式，含[Coverage]等标记（英文） |
| I | Expected result | 技术+业务双轨描述 |
| J-M | Actual result/Tested By/Test Date/Pass(Y/N) | 空白（测试执行后填写） |

### 格式要求

- **行高**：行9及以后 **80 points**
- **字体**：**TKE Type**（所有内容），size=10
- **文本控制**：开启Wrap Text

```python
from openpyxl.styles import Font, Alignment

# 设置行高
ws.row_dimensions[row].height = 80

# 设置字体
cell.font = Font(name='TKE Type', size=10)

# 设置自动换行
cell.alignment = Alignment(wrap_text=True, vertical='top')
```

### 序列号强制规则

```python
# 伪代码（工具无关）
current_l1 = 1
current_l2 = 1

for scenario in scenarios:
    ws.cell(row, col_B).value = current_l1  # Level 1 No.
    ws.cell(row, col_D).value = current_l2  # Level 2 No.
    ws.cell(row, col_A).value = f"TC.{current_l1}.{current_l2}"  # ID
    
    # 同组递进L2，换组重置L2递增L1
    if scenario.is_last_in_group:
        current_l1 += 1
        current_l2 = 1
    else:
        current_l2 += 1
```

### Excel文件命名规范

- **EN**: `TKE IT Project SIT Test Case_{Project}_ITCM{number}.xlsx`
- **CN**: `TKE IT Project SIT Test Case _{Project}_ITCM{number}_CN.xlsx`

---

## 4.3 Markdown生成规范

### 文件命名

- **EN**: `TKE_IT_Project_SIT_Test_Case_{Project}_ITCM{number}.md`
- **CN**: `TKE_IT_Project_SIT_Test_Case_{Project}_ITCM{number}_CN.md`

### 内容规则

- **列名**：英文（与Excel一致）
- **SQL**：完全英文（包括注释）
- **业务术语**：英文（opportunity/tender/supervisor/contract等）
- **描述文本**：CN版本翻译为中文，EN版本保持英文

### 结构

与Excel SIT Case Sheet完全一致（Markdown表格）：

```markdown
| ID | Level 1 No. | Level1 Scenario | Level 2 No. | Level2 Scenario | Prerequisites/test data | Procedure | Sql | Expected result |
|----|-------------|-----------------|-------------|-----------------|------------------------|-----------|-----|-----------------|
| TC.1.1 | 1 | Schema Validation | 1 | repair_tender_id PK uniqueness | ... | ... | ... | ... |
```

---

## 4.0 占位符模式（Placeholder Mode）

**触发条件**：`{IF: IS_CODELESS}`（无notebook输入）

当无法获取代码逻辑时，生成含占位符的测试用例框架：

```yaml
placeholder_mode:
  trigger: "NOT HAS_OLD_NB AND NOT HAS_NEW_NB"
  
  content_rules:
    table_name: "<TBD_TABLE>"
    field_names: "<TBD_FIELD>"
    primary_key: "<TBD_PK>"
    cutoff_date: "<TBD_CUTOFF_DATE>"
    sql_coverage: "<TBD_LOGIC>"
    
  output:
    - "TKE IT Project SIT Test Case_{Project}_ITCM{number}.xlsx (框架版)"
    - "TKE IT Project SIT Test Case_{Project}_ITCM{number}.md (框架版)"
    
  notes: |
    占位符模式生成的测试用例框架包含所有必要结构，
    但关键值使用<TBD_*>标记，需人工根据FS文档和代码补充。
    
  status: "DRAFT（非最终交付）"
```

### 占位符模式SQL示例

```sql
[Coverage]
-- TODO: Replace <TBD_TABLE> with actual table name from FS doc
-- TODO: Replace <TBD_CUTOFF_DATE> with actual cutoff date
SELECT 
    COUNT(*) as total_records,
    COUNT(<TBD_PK>) as non_null_pk
FROM <TBD_TABLE>
WHERE calendar_year_month >= <TBD_CUTOFF_DATE>;

[Cardinality]
-- TODO: Replace <TBD_PK> with actual primary key
SELECT COUNT(*) as duplicate_pk
FROM (
    SELECT <TBD_PK>
    FROM <TBD_TABLE>
    GROUP BY <TBD_PK>
    HAVING COUNT(*) > 1
) t;

[Value Diff]
-- TODO: Implement value comparison logic based on FS doc
<TBD_LOGIC>

[Conclusion]
-- TODO: Update conclusion after filling placeholders
<TBD_CONCLUSION>
```

---

## 完整生成代码示例

```python
from openpyxl import load_workbook
from openpyxl.styles import Font, Alignment
from datetime import datetime

def generate_excel(test_cases, template_path, output_path, project_info):
    """生成Excel测试用例"""
    
    # 1. 加载模板
    wb = load_workbook(template_path)
    ws = wb['SIT Case']
    
    # 2. 写入表头信息（行3-6）
    ws.cell(row=3, column=5).value = project_info['project_name']
    ws.cell(row=4, column=5).value = project_info['itcm_number']
    ws.cell(row=5, column=5).value = project_info['pm_name']
    ws.cell(row=6, column=5).value = 'QA'
    
    # 3. 写入测试用例（行9起）
    start_row = 9
    
    for i, tc in enumerate(test_cases):
        row = start_row + i
        
        # 写入各列
        ws.cell(row, 1).value = tc['id']                    # A: ID
        ws.cell(row, 2).value = tc['level1_no']             # B: Level 1 No.
        ws.cell(row, 3).value = tc['level1_scenario']       # C: Level1 Scenario
        ws.cell(row, 4).value = tc['level2_no']             # D: Level 2 No.
        ws.cell(row, 5).value = tc['level2_scenario']       # E: Level2 Scenario
        ws.cell(row, 6).value = tc['prerequisites']         # F: Prerequisites
        ws.cell(row, 7).value = tc['procedure']             # G: Procedure
        ws.cell(row, 8).value = tc['sql']                   # H: Sql
        ws.cell(row, 9).value = tc['expected_result']       # I: Expected result
        
        # 设置格式
        ws.row_dimensions[row].height = 80
        for col in range(1, 14):
            cell = ws.cell(row, col)
            cell.font = Font(name='TKE Type', size=10)
            cell.alignment = Alignment(wrap_text=True, vertical='top')
    
    # 4. 更新Change History
    ch_ws = wb['Change History']
    next_row = 2
    while ch_ws.cell(next_row, 1).value:
        next_row += 1
    
    ch_ws.cell(next_row, 1).value = project_info['itcm_number']
    ch_ws.cell(next_row, 2).value = f"Added {len(test_cases)} test cases"
    ch_ws.cell(next_row, 3).value = datetime.now().strftime('%Y-%m-%d')
    ch_ws.cell(next_row, 4).value = f"Generated by AI Skill v2.6"
    
    # 5. 保存
    wb.save(output_path)
    
    return output_path
```

---

## Excel生成检查清单

- [ ] 使用模板加载（非创建新文件）
- [ ] 写入'SIT Case' Sheet（非active）
- [ ] 行3-6表头信息已填写
- [ ] ID格式为TC.{L1}.{L2}（自然数，无前导零）
- [ ] Level 1 No.连续无跳跃
- [ ] Level 2 No.在每个Level 1组内连续
- [ ] 行9+行高80pt
- [ ] 字体为TKE Type，size=10
- [ ] 开启Wrap Text
- [ ] Change History已追加记录
- [ ] 文件命名符合规范
