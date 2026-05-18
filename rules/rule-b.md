# 2. Input Layer - 输入提取（Extraction）

**本章职责：从输入文件提取所有原始数据，不做任何业务逻辑判断。**

---

## 2.0 自适应提取原则

| 输入源 | 优先级 | 触发条件 | 用途 |
|-------|-------|---------|------|
| FS/TS 文档 | P1 | **{IF: FS_TS_DOCX_EXISTS}** → 激活2.2/2.2.1 | 业务规则提取 |
| Old Notebook | P2 | **{IF: IS_ENHANCEMENT}** → 激活2.5.0.1 | 变更对比基准 |
| New Notebook | P2 | **{IF: NOTEBOOK_EXISTS}** → 激活2.3b/2.5/2.6 | Schema提取 |
| Excel 模板 | P3 | 强制（无标记） | 格式基准 |

**占位符规范**:
- 表名未知 → `<TBD_TABLE>`
- 字段名未知 → `<TBD_FIELD>`
- 主键未知 → `<TBD_PK>`
- 日期未知 → `<TBD_CUTOFF_DATE>`
- 逻辑未知 → `<TBD_LOGIC>`

**非阻断承诺**: 无论缺失多少输入，AI 必须生成可交付的测试用例框架（含占位符），供用户后续填充。

---

## 2.1 文件扫描与项目类型判定

**文件路径**：`openspec/changes/[change-id]/`

**必需文件**：
- `TKE_IT_Template_Functional_Requirement_Specification_*_ITCM*.docx` → 业务规则、cutoff日期
- `TKE IT Project SIT Test Case - template_ITCMXXXXX.xlsx` → 模板

**条件文件**（决定项目类型）：
- `[notebook_name]_new.ipynb` → 新逻辑（New Dev）或增强后逻辑（Enhancement）
- `[notebook_name]_old.ipynb` → **存在**=Enhancement项目；**不存在**=New Development项目

**扫描流程**：
```python
# 强制判定逻辑 - 绝对准确
if old_notebook.exists():
    project_type = "Enhancement"
    analysis_method = "Code Diff Only"  # 强制：对比before/after代码
else:
    project_type = "New Development" 
    analysis_method = "Full Extraction"  # 全量提取new代码
```

---

## 2.1.5 范围澄清（阻断点）

**触发条件**：
- 用户请求中包含可能多义的词（如"XXX项目中的XXX部分"）
- 文档中发现多个同名/相似的章节或页面

**强制流程**：
1. 列出所有可能的理解
2. 向用户明确询问：
```
请确认测试范围：
- 仅XXX页面（约Y个字段）
- 整个XXX项目（约Z个页面）
- 其他：[请说明]
```
3. ❌ **禁止擅自假设**：未确认前，不得进入2.2提取阶段

---

## 2.2.5 DOCX提取完整性验证（阻断点）

**强制执行**：完成2.2 docx提取后，生成任何测试用例前，必须：

1. **生成提取清单**：
```
已从 [文档名] 提取以下内容：

【章节X - YYY】
✓ Table A: 字段1, 字段2, ...（共N个字段）
- 关键逻辑：字段1的3-condition filter
- 关键逻辑：字段2的除零保护
✓ Table B: 字段3, 字段4, ...（共M个字段）

总计：K个表格，L个字段，P条业务规则
```

2. **向用户确认**：
```
请确认：
✅ 上述内容完整？
❌ 遗漏了哪些逻辑/字段？（请指出具体章节/表格）
```

3. **等待确认**：
- 用户回复"确认" → 进入2.3主键识别
- 用户指出遗漏 → 返回2.2补充提取

---

## 2.3 主键识别（Key Identification）

**本章职责**：从输入文件提取主键信息，供Analysis Layer使用。

**识别优先级**（按可靠性排序）：

### 1. 显式约束（最高优先级）
- 从`CREATE TABLE`语句提取：`PRIMARY KEY (field1, field2)`
- 从`ALTER TABLE`提取：`ADD CONSTRAINT PK_name PRIMARY KEY (field)`

### 2. 注释标记（次优先级）
- 匹配：`-- PK: (\w+)`或`/* Primary Key: (\w+) */`
- 匹配：`-- PRIMARY KEY: field1, field2`

### 3. 业务推断（最低优先级，需标记不确定性）
- 表名`dm_sap_job_costing` → 推断`repair_tender_id`（或组合`repair_tender_id + company_code`）
- 表名含`opportunity` → 推断`opportunity_id`
- 通用：`{table_name}_id`字段存在时视为主键
- 标准字段：`id`, `code`, `key`等唯一标识

### 4. 复合主键处理
- 发现多个候选字段 → 组合为`CONCAT(pk1, '-', pk2)`
- 在[Cardinality]段使用复合键去重验证

**输出**：主键信息传递给Analysis Layer，用于生成`[Cardinality]`段的主键唯一性SQL。

---

## 2.4 版本确认（阻断点 - Blocking Point）

**在分析开始前必须确认**：

> **Agent**: "已提取项目信息，准备生成测试用例。请选择版本：1)仅英文 2)仅中文 3)中英文？"

**阻断规则（强制）**：
- ❌ **禁止假设默认值**：用户必须明确回复"1"、"2"或"3"（或对应文字）
- ❌ **禁止自动继续**：在用户明确选择前，**不得**进入下一步（2.5 Change Points提取）
- ✅ **有效确认**：用户回复"1/2/3"或"英文/中文/中英文"后，锁定版本选择，方可进入2.5

**错误处理**：
- 用户回复模糊（如"都可以"）→ 重复提问，要求明确选择1/2/3
- 用户未回复 → 等待，每30秒重复提问一次，不得擅自使用"默认"

---

## 2.5 Change Points确认（阻断点）

**执行时机**：
- Enhancement项目：完成`old.ipynb` vs `new.ipynb`代码对比后
- New Development项目：完成`new.ipynb`全量Schema提取后

**阻断性质**：**必须等待用户明确确认后，方可进入Analysis Layer（第3章）生成测试用例。**

---

## 2.5.1 Change Points提取与结构化展示

### Enhancement project格式
```yaml
project_type: Enhancement
comparison_basis: "[notebook_name]_old.ipynb vs [notebook_name]_new.ipynb"
change_points:
  added_fields:  # 新增字段（old无，new有）
    - field_name: "new_field_1"
      location: "gold.dbo.dm_table"
      logic_snippet: "CAST(source_col AS VARCHAR(100)) as new_field_1"
      detected_by: "schema_diff"
      
  removed_fields:  # 删除字段（old有，new无）
    - field_name: "deprecated_field"
      old_location: "gold.dbo.dm_table"
      removal_impact: "需确认是否影响下游消费"
      
  logic_changed_fields:  # 逻辑变更（同名但SQL不同）
    - field_name: "existing_field_1"
      location: "gold.dbo.dm_table"
      change_type: "NULL_handling"
      old_logic: "COALESCE(old_col, 'N/A')"
      new_logic: "COALESCE(old_col, 'Unknown', 'N/A')"
      test_implication: "需验证新增'Unknown'分支"
      
total_change_points: 5
test_scope_recommendation: "基于上述变更，建议生成X个测试场景"
```

### New Development project格式
```yaml
project_type: New Development
analysis_basis: "[notebook_name]_new.ipynb"
change_points:
  new_tables:
    - table_name: "gold.dbo.dm_new_table"
      source: "silver.dbo.dwt_source"
      fields_count: 15
      key_fields: ["primary_key_id", "business_date"]
      
  key_business_rules_detected:
    - rule_type: "Cutoff_date"
      field: "calendar_year_month"
      condition: ">= 202603"
      
  data_lineage:
    - path: "ingress → staging → silver → gold"
      key_transformation: "staging到silver有字段合并逻辑"
      
total_new_components: "2 tables, 23 fields"
test_scope_recommendation: "建议生成全量覆盖测试，约Y个场景"
```

---

## 2.5.2 用户确认流程（强制性）

**展示方式**：以结构化Markdown表格/YAML形式向用户展示上述Change Points清单。

**确认话术模板**：
```
Agent: "已完成代码分析，识别以下关键变更点（基于代码对比，未参考change log）：
[展示Change Points清单]

请确认：
1. 上述变更点是否完整准确？是否有遗漏或误判？
2. 是否有特定字段需要额外关注或排除？

确认无误后，我将基于这些变更点生成测试用例。"

选项：
- 回复"确认" / "Proceed" → 进入第3章Analysis Layer生成测试用例
- 回复"补充：[具体字段/变更]" → 更新Change Points清单，再次展示确认
- 回复"排除：[具体字段]" → 从清单移除，说明原因后再次确认
- 回复"停止" / "Stop" → 暂停生成，等待用户补充信息
```

**阻断规则**：
- 在收到用户明确确认（如"确认"、"Proceed"、"Go"）前，禁止进入第3章生成测试用例
- 若用户提出修正，更新Change Points后需再次展示完整清单供二次确认
- 所有确认记录需在Change History中备注用户反馈摘要

---

## 2.5.3 确认后锁定（Freeze）

用户确认后：
- 将确认后的Change Points清单保存为`confirmed_change_points.yaml`（逻辑文件，非实际交付物）
- 锁定测试范围：后续第3章Analysis Layer仅允许基于已确认的Change Points生成测试场景
- 禁止回溯：进入第3章后，不得因"发现新变更"而扩大范围；若确有遗漏，需返回2.5重新确认

---

## 2.6 数据资产整合 - Field Inventory

**本章职责**: 将多源数据整合为统一的 Field Inventory，供第 3 章使用。

**整合原则**: 
- **FS文档为语义基准**: 业务含义、cutoff日期
- **代码为技术实现**: 生成逻辑、SQL表达式
- **变更标记**: 叠加 2.5 的变更检测结果

### 2.6.0 准入条件
必须已完成 2.2、2.3、2.5 节。

### 2.6.1 整合输入源
| 输入源 | 章节 | 数据 | 用途 |
|--------|------|------|------|
| FS/TS文档 | 2.2 | 业务定义 | fs_definition |
| 主键识别 | 2.3 | PK信息 | primary_key标记 |
| 代码提取 | 2.5 | diff_result/full_schema | change_status/code_logic |

### 2.6.2 Field Inventory结构
```yaml
field_inventory:
  version: "v1.0"
  generated_at: "<timestamp>"
  sources:
    fs_doc: "<filename>.docx"
    notebook_old: "<name>_old.ipynb"  # Enhancement only
    notebook_new: "<name>_new.ipynb"
  
  fields:
    - field_id: "F001"
      field_name: "repair_tender_id"
      table_name: "gold.dbo.dm_sap_job_costing"
      
      # 来自FS文档(2.2)
      fs_definition:
        business_name: "维修投标ID"
        description: "关联到维修模块的唯一标识"
        data_type: "INTEGER"
        nullable: false
        business_rules: ["必须关联到有效投标"]
        cutoff_date: "202603"
      
      # 来自代码(2.5提取)
      code_logic:
        current:
          derivation_sql: "SELECT tender_id FROM repair_table"
        baseline:  # Enhancement项目存在
          derivation_sql: "SELECT mr.tender_id FROM mr_table mr"
      
      # 来自2.3主键识别
      primary_key: true
      pk_composite_part: false
      
      # 来自2.5代码对比
      change_status: "LogicChanged"  # Added/Removed/LogicChanged/Unchanged/New
      change_detail: "从mr表改为rr表取数"
      
      # 计算字段（供3.3使用）
      test_metadata:
        complexity_score: 2  # 1=简单透传, 2=条件判断, 3=复杂计算
        scenarios_needed: 2
        test_priority: "P1"   # P0=关键业务键, P1=变更字段, P2=未变更字段
```

### 2.6.3 整合规则（数据对齐策略）

**场景 A: FS有定义，Code有实现**（正常匹配）
- 策略：合并为一条记录，FS提供业务语义，Code提供技术实现
- test_priority: 基于 change_status 判定

**场景 B: FS有定义，Code无实现**（实现遗漏）
- 标记：`implementation_gap: true`
- code_logic: 填充 `<TBD_LOGIC>`
- test_priority: 强制 P0（需验证是否遗漏开发）
- 备注："FS要求但代码未实现，需人工确认是否为待开发项"

**场景 C: FS无定义，Code有实现**（未文档化字段）
- 从 Code 提取 field_name，fs_definition 填充 `<TBD_FROM_FS>`
- 标记：`undocumented: true` 
- 备注："代码中存在但FS未定义，已假设业务含义为..."

**场景 D: 变更状态判定**（基于 2.5 输出）
- added → change_status: "Added", scenarios_needed: 2
- removed → change_status: "Removed", scenarios_needed: 1  
- logic_changed → change_status: "LogicChanged", scenarios_needed: 2
- 无变更 → change_status: "Unchanged", scenarios_needed: 0/1（仅PK或关键字段）

### 2.6.4 输出交付
**输出文件**：`field_inventory.yaml`（逻辑文件，非客户交付物）

**向后传递**：
- 第 3 章 3.0 节：读取 `field_inventory` 作为准入检查
- 第 3 章 3.3 节：遍历 `fields` 列表生成测试场景
- 第 4 章 4.1 节：使用 `code_logic.derivation_sql` 构建 SQL

**质量门禁**（2.6 内部检查）：
- [ ] 所有变更字段（Added/LogicChanged）必须有 code_logic
- [ ] 所有主键字段（primary_key=true）必须有 PK 验证标记
- [ ] cutoff_date 为 `<TBD>` 的字段 >20% 时，警告"FS文档信息不完整"
