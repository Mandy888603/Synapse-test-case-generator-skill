# 3. Analysis Layer - 分析决策（Decision Making）

**本章职责：所有业务逻辑判断，包括分层决策、场景设计、优先级应用。**

---

## 3.0 准入检查（Entry Gate）

**准入条件**：必须已获取`field_inventory.yaml`（即2.6节）

**检查逻辑**：
```python
if not confirmed_change_points.exists():
    raise BlockingError("未完成Change Points确认。请先执行2.5节用户确认流程。")
    
if current_change_points != confirmed_change_points:
    raise ConsistencyError("当前分析范围与用户确认范围不一致，禁止擅自扩展。")
```

```yaml
3.0_global_condition_setup:
  - condition_name: "INTEGRATION_REQUIRED"
    set_by: "分析准入时预检测"
    detection_rules:
      - "notebook分析：contains('mssparkutils.notebook.run')"
      - "配置检查：landing_type == 2"
      - "风险评估：上游表不稳定性"
```

---

## 3.1 项目类型分析策略

### 3.1.1 Enhancement项目（有old notebook）

**准入条件**: 必须已获取 `field_inventory.yaml`（2.6 节输出）

**分析输入**: 
- 读取 `field_inventory.sources.notebook_old` 和 `notebook_new`（追溯来源）
- 遍历 `field_inventory.fields`，基于 `change_status` 和 `primary_key` 决策

**测试范围决策逻辑**:

| Inventory 状态 | 判定规则 | 测试策略 | 输出标记 |
|----------------|---------|---------|---------|
| `change_status: "Added"` | 字段仅存在于new | **必须测试**（新增逻辑验证） | `scope: "Full"` |
| `change_status: "LogicChanged"` | 字段存在但`code_logic.current`≠`baseline` | **必须测试**（新旧逻辑对比） | `scope: "Regression+New"` |
| `change_status: "Removed"` | 字段仅存在于old | **下线验证**（3.6集成测试处理） | `scope: "Deprecation"` |
| `change_status: "Unchanged"` + `primary_key: true` | 主键未变更 | **抽样验证**（关键回归） | `scope: "Sample"` |
| `change_status: "Unchanged"` + `primary_key: false` | 普通字段未变更 | **免测**（3.2.1规则） | `scope: "Skip"` |

**关键逻辑**:
```python
# 从2.6 Inventory读取，不再直接对比代码
for field in field_inventory.fields:
    if field.change_status in ["Added", "LogicChanged"]:
        # 标记为必须测试，传递详细逻辑给3.3
        test_scope.append({
            "field": field.field_name,
            "table": field.table_name,
            "status": field.change_status,
            "old_logic": field.code_logic.baseline.derivation_sql if field.code_logic.baseline else None,
            "new_logic": field.code_logic.current.derivation_sql,
            "fs_rules": field.fs_definition.business_rules,
            "priority": "P0" if field.primary_key else "P1"
        })
    elif field.change_status == "Removed":
        # 移至3.6集成验证处理下线逻辑
        deprecation_list.append(field)
    # Unchanged非主键：明确排除
```

### 3.1.2 New Development项目（无old notebook）

**准入条件**: 必须已获取 `field_inventory.yaml`（2.6 节输出）

**分析输入**:
- 读取 `field_inventory.sources.notebook_new`（追溯来源）
- 遍历所有 `field_inventory.fields`（因无old版本，所有字段`change_status`应为"New"）

**测试范围决策逻辑**:

| 字段属性 | 判定来源 | 测试策略 | 场景分配 |
|---------|---------|---------|---------|
| 所有字段 | `change_status: "New"` | **全量覆盖** | 每字段1-2场景 |
| `primary_key: true` | 2.3主键识别 | **强制验证**（唯一性） | [Cardinality]必含 |
| `complexity_score: 1` | 2.6计算（透传） | 简单验证 | 1场景（存在性） |
| `complexity_score: 2/3` | 2.6计算（条件/计算） | 深度验证 | 2场景（主路径+边界） |
| `implementation_gap: true` | FS有但Code无 | **阻断标记** | 移至3.6待处理清单 |

**关键逻辑**:
```python
# 全量读取，基于复杂度分层测试
for field in field_inventory.fields:
    # New Development所有字段都是"New"状态
    test_item = {
        "field": field.field_name,
        "table": field.table_name,
        "logic": field.code_logic.current.derivation_sql,
        "complexity": field.test_metadata.complexity_score,  # 1/2/3
        "pk_flag": field.primary_key,
        "fs_requirements": field.fs_definition.business_rules,
        "cutoff": field.fs_definition.cutoff_date
    }
    
    # 复杂度决定场景数
    if field.test_metadata.complexity_score == 1:
        test_item["scenarios"] = ["Existence"]
    else:
        test_item["scenarios"] = ["HappyPath", "EdgeCase"]
        
    test_scope.append(test_item)
```

---

## 3.2 分层测试优先级策略（Cascade Strategy）

**核心原则**：**最高可用层优先，逻辑差异例外。**

### 3.2.1 决策树（Testing Layer Decision Tree）

**流向**：单向流动，从原始到消费  
`ingress`（原始增量层）→ `staging`（Bronze）→ `prepared`（Silver）→ `curated`（Gold，按需创建）

**层级映射与测试优先级**：

| Synapse DB | 标准层级 | 数据流位置 | 测试优先级 | 存在性 |
|-----------|---------|-----------|-----------|-------|
| `ingress` | 原始增量层 | 第1层（源头） | 最后 | 始终存在 |
| `staging` | Bronze | 第2层 | 条件测试 | 始终存在 |
| `prepared` | Silver | 第3层（整合层） | **高** | 始终存在 |
| `curated` | Gold | 第4层（语义层） | **最高**（如果存在） | **按需创建** |

**关键规则**：
1. **curated 条件优先**：如果 `curated`/`gold` 数据库存在且包含目标表，**如存在必须测试 curated**（最高层）
2. **prepared 作为最高层**：如果 `curated` 不存在，`prepared`/`silver` 成为最高消费层**如存在必须测试prepared**
3. **staging 免测规则**：由于 `prepared` 整合 `staging` 数据，**默认免测 staging**（除非业务要求验证原始Bronze层）
4. **ingress 免测规则**：作为原始增量接入层，通常不直接用于业务验证（除非数据接入问题）

### 3.2_cascade_strategy.yaml

```yaml
priority_order: [DM, DWT, Staging, Ingress]

decision_tree:
  start: "字段X需要测试"
  steps:
    - condition: "**{IF: DM_EXISTS}**"
      if_yes:
        action: "测试DM层"
        sub_check: "**{IF: LOGIC_DIFFERENT}**"
        if_sub_yes: "补充测试DWT（DM+DWT双测）"
        if_sub_no: "免测DWT（透传逻辑）"
      if_no:
        next_step: "检查DWT层"
        
    - condition: "存在DWT层(silver/prepared.dbo.dwt_*)?"
      if_yes:
        action: "测试DWT层（最高消费层）"
        sub_check: "Staging有复杂转换?"
        if_sub_yes: "补充测试Staging"
        if_sub_no: "免测Staging"
      if_no:
        next_step: "检查Staging层"
        
    - condition: "存在Staging层?"
      if_yes: "测试Staging层（最高层，字段存在性必测）"
      if_no: "检查Ingress层"
      
    - condition: "存在Ingress层?"
      if_yes: "测试Ingress层"
      if_no: "ERROR: 无可用数据层"

logic_diff_rules:
  consistent: "简单透传（SELECT field FROM DWT）→ 免双测"
  inconsistent_signals:  # 出现以下任一即视为不一致
    - "CASE WHEN"
    - "COALESCE"
    - "FORMAT"
    - "计算表达式（field_a + field_b）"
    - "重命名改变业务含义"
  inconsistent_action: "DM+DWT双测"
```

### 3.2.2 分层测试规则详解

| 场景 | 存在层级 | 测试策略 | 字段存在性测试 |
|------|---------|---------|--------------|
| **标准DM** | DM + DWT + Staging | **仅测DM**<br>DWT为透传逻辑则免测 | 测DM，免测DWT/Staging |
| **DM+DWT双测** | DM + DWT（逻辑不同） | **DM + DWT双测**<br>DM验证转换，DWT验证原始映射 | 双测（各自验证存在性） |
| **DWT封顶** | DWT + Staging（无DM） | **仅测DWT**<br>DWT成为最高消费层 | 测DWT，免测Staging |
| **Staging封顶** | Staging + Ingress（无DM/DWT） | **仅测Staging** | **必须测Staging**（最高层） |
| **原始层** | 仅Ingress | **仅测Ingress** | 测Ingress |

### 3.2.3 逻辑差异判定标准

**一致（免双测）**：DM直接`SELECT field_name FROM DWT`（简单透传，无函数包裹）

**不一致（需双测）**：DM对字段有任何处理：
- `CASE WHEN field IS NULL THEN 'Unknown' ELSE field END`
- `COALESCE(field, 'N/A')`
- `FORMAT(field, 'yyyy-MM-dd')`
- 计算：`field_a + field_b`
- 重命名/别名改变业务含义

---

## 3.3 字段驱动测试设计（Field-Driven Design）

**禁止**：按`if/else`分支拆分场景（导致场景过多）。  
**采用**：按字段聚合，单字段≤2场景。

```yaml
# 3.3_field_driven_design.yaml
field_classification:
  simple_field:
    definition: "直接映射，无转换（SELECT field FROM source）"
    scenarios: 1
    test_focus: ["存在性", "基础值验证"]
    
  complex_field:
    definition: "含CASE WHEN/计算/JOIN/多条件判断"
    max_scenarios: 2
    scenario_split:
      A_happy_path: "主逻辑路径（最高频业务场景）"
      B_edge_case: "边界/异常（NULL/ELSE/极值）"
    restriction: "单字段禁止因多分支产生3个及以上场景"

scenario_selection_priority:
  if_exceeds_2_branches:
    select:
      - "最高频路径（Happy Path）"
      - "风险最高路径（Edge Case）"
    defer_others: "在Notes标注Deferred"

hard_limits:
  max_scenarios_per_field: 2
  max_total_scenarios: 15
  procedure_max_steps: 3
```

### 3.3.1 字段分类与场景数

| 字段类型 | 定义 | 场景数 | 测试重点 |
|---------|------|-------|----------|
| **简单字段** | 直接映射，无转换（`SELECT field FROM source`） | **1个** | 存在性、基础值验证 |
| **复杂字段** | 含CASE WHEN、计算、JOIN关联、多条件判断 | **≤2个** | 场景A：主逻辑路径<br>场景B：边界/异常路径 |

### 3.3.2 复杂字段的场景拆分原则

**场景A（Happy Path - 主逻辑）**：
- 覆盖最频繁的业务路径
- 例：`converted_opportunity_id IS NOT NULL`时，`is_converted=1`

**场景B（Edge Case - 边界/异常）**：
- 覆盖NULL处理、ELSE分支、极值（Max/Min）、空字符串、数据边界
- 例：`converted_opportunity_id IS NULL`时，`is_converted=0`

**硬性限制**：单个字段不得因多分支逻辑产生3个及以上场景。若业务规则复杂（3+分支），选择**最高频路径**和**风险最高路径**各1个，其余在Notes标注"Deferred"。

---

## 3.4 数据血缘与一致性场景

**独立场景类型**，不占用字段测试配额。

### 3.4.1 跨层数据一致性（Ingress→Staging→DWT→DM）

**场景名称**：`{Table} Cross-Layer Data Consistency`  
**Purpose**：验证ETL链条中数据无丢失、无异常截断、关键字段值一致。

**触发条件**：存在多层（至少3层）且业务关注数据完整性时创建。

**SQL结构要点**：
```sql
[Coverage]
-- 各层记录数快照（基于同一逻辑日期/批次）
-- 日期参数应从FS/TS文档"Data Cutoff Date"章节提取

[Cardinality]
-- 关键业务键在各层的distinct count（检查提前聚合导致的重复）

[Value Diff]
-- 抽样比对：选取10个business_key，对比各层关键字段值

[Conclusion]
各层记录数差异符合预期（考虑过滤逻辑），关键字段distinct count合理，抽样值一致。
```

### 3.4.2 源系统数据一致性（Source System Reconciliation）

**场景名称**：`{Table} Source System Reconciliation`  
**Prerequisites**：源系统（SAP BW4等）提供总计数或抽样数据。

**触发条件**：DM/DWT为最终消费层，且业务要求与源系统对账。

---

## 3.5 Procedure简化规范（测试步骤）

**硬性限制**：**最多3步骤**，建议1-3步。

**步骤合并规则**：
- 合并"准备环境"与"加载数据" → "准备测试数据并加载目标表"
- 合并"检查存在性"与"验证类型" → "验证目标字段存在性及数据类型"
- 合并"执行查询"与"检查主键" → "执行验证查询并检查主键唯一性"

**标准模板**：
```
1. 准备测试数据：[具体条件，如CALMONTH>=202603且converted_opportunity_id非空]
2. 执行验证查询：运行Sql列[Coverage]查总数，[Cardinality]查主键唯一性
3. 比对预期结果：确认[Value Diff]返回0差异，且结果符合Expected result列
```

---

## 3.6 依赖与集成验证（条件触发）

**准入条件**：**{IF: INTEGRATION_TEST_REQUIRED}**

**触发判定**（满足任一即true）：
1. **Next Step链式调用**：`mssparkutils.notebook.run`存在且`next_step`参数非空
2. **Schema漂移风险**：`landing_type = 2`且`config_etl_task_queue_mapping`存在字段映射
3. **依赖缺失风险**：上游表(ingress)可能在ETL执行时不存在

**{IF: NOT INTEGRATION_TEST_REQUIRED}** → 跳过3.6，进入4.x输出层

```yaml
integration_test_scenarios:
  - category: "前置依赖"
    cases:
      - id: "INT-01"
        name: "Ingress_Table_Not_Exist"
        description: "验证ingress表不存在时的错误处理"
        expected: "在DROP/CREATE/REFRESH阶段明确报错TableNotFoundException"
        
      - id: "INT-02"
        name: "Schema_Drift_Detection"
        description: "验证mapping配置字段与源表结构一致性"
        
  - category: "Next Step链式调用"
    cases:
      - id: "INT-03"
        name: "Next_Step_Chain_Execution"
        description: "验证mssparkutils.notebook.run的链式调用"
        
      - id: "INT-04"
        name: "Next_Step_Failure_Propagation"
        description: "验证子任务失败时父任务状态"
```
