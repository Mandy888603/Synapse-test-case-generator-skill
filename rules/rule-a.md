# 1. 核心原则与硬性约束（Hard Constraints）

**本章仅包含不可协商的格式规范，不含业务逻辑决策。**

---

## 1.1 硬性约束清单

| 维度 | 规范 | 错误示例 |
|------|------|----------|
| **ID格式** | `TC.{L1}.{L2}`（自然数，如TC.1.1, TC.2.3） | ❌ TC.1.01, TC.1.1.1, TC.01.1 |
| **表名规范** | `<database>.dbo.<table>` | ❌ `gold.table`, `dbo.table` |
| **数据库名** | `gold`, `silver`, `prepared`, `staging`, `ingress`, `curated` | ❌ 视为schema |
| **SQL结构** | 四段式：[Coverage]→[Cardinality]→[Value Diff]→[Conclusion] | ❌ 缺少段落标记 |
| **SQL平台** | Synapse Serverless SQL Pool | ❌ Temp table, SP, cursor |
| **Excel写入** | **SIT Case** Sheet（非默认Change History） | ❌ 使用`wb.active` |
| **Excel表头** | 行3-6列E（Project/ITCM/PM/Env） | ❌ 写在Change History |
| **格式要求** | 行高80pt，TKE Type字体（行9+） | ❌ 默认格式 |
| **Procedure** | ≤3步骤，合并准备与验证 | ❌ 冗长步骤 |
| **测试范围** | 按DM→DWT→Staging优先级 | - |

---

## 1.1_hard_constraints.yaml

```yaml
# AI执行时必须遵守的格式约束（不可协商）
id_format:
  pattern: "TC.{L1}.{L2}"
  rules:
    - "L1和L2必须为自然数（1,2,3...）"
    - "禁止前导零（❌ TC.1.01）"
    - "禁止三级编号（❌ TC.1.1.1）"

table_naming:
  format: "<database>.dbo.<table>"
  valid_databases: [gold, silver, prepared, staging, ingress, curated]
  schema: "dbo"  # 强制，不可变更
  forbidden_patterns:
    - "gold.table"      # 缺少schema
    - "dbo.table"       # 缺少database
    - "gold.custom.table"  # 错误schema

sql_structure:
  required_sections: [Coverage, Cardinality, "Value Diff", Conclusion]
  platform: "Synapse Serverless SQL Pool"
  forbidden_operations:
    - "temp tables (#temp)"
    - "CREATE TABLE"
    - "INSERT/UPDATE/DELETE"
    - "cursor"
    - "stored procedure"

excel_requirements:
  target_sheet: "SIT Case"  # 禁止用wb.active
  header_rows:
    row_3: "Project (列E)"
    row_4: "ITCM No. (列E)"
    row_5: "PM (列E)"
    row_6: "Environment (列E)"
  data_start_row: 9
  formatting:
    row_height: "80pt (行9+)"
    font: "TKE Type"
    wrap_text: true

procedure_limits:
  max_steps: 3
  merge_rules:
    - "准备环境+加载数据 → 准备测试数据并加载"
    - "检查存在性+验证类型 → 验证字段存在性及类型"
```

---

## 1.2 交付物清单

根据2.4节版本确认结果判断：

### Excel交付物
- **EN**: `TKE IT Project SIT Test Case_{Project}_ITCM{number}.xlsx`
- **CN**: `TKE IT Project SIT Test Case _{Project}_ITCM{number}_CN.xlsx`

### Markdown交付物
- **EN**: `TKE_IT_Project_SIT_Test_Case_{Project}_ITCM{number}.md`
- **CN**: `TKE_IT_Project_SIT_Test_Case_{Project}_ITCM{number}_CN.md`

### 三不变原则（中英文版本）
1. **列名保持英文**
2. **SQL保持英文**（含注释）
3. **业务术语保持英文**（opportunity/tender/supervisor等）

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
与Excel SIT Case Sheet完全一致（Markdown表格）。
