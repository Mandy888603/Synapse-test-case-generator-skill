# 代码对比规则 - Enhancement项目

**适用场景**：Enhancement项目（同时存在old和new notebook），代码对比流程

---

## 2.5.0.1 Enhancement项目（有old notebook）

### 分析前提

已确认`[name]_old.ipynb`与`[name]_new.ipynb`同时存在。

---

## 阶段1：提取与对比（必须先完成）

| 步骤 | 操作对象 | 提取方法 | 输出 |
|------|---------|---------|------|
| 1.1 | Old notebook | 提取所有SQL code cells的**最终SELECT结构** | `old_schema = {字段名: 生成逻辑SQL片段}` |
| 1.2 | New notebook | 提取所有SQL code cells的**最终SELECT结构** | `new_schema = {字段名: 生成逻辑SQL片段}` |
| 1.3 | 差异计算 | 对比`old_schema` vs `new_schema` | `diff_result = {added[], removed[], logic_changed[]}` |

### 提取方法详解

```python
# 伪代码：提取notebook的schema结构
def extract_notebook_schema(notebook_path):
    schema = {}
    
    with open(notebook_path, 'r') as f:
        nb = json.load(f)
    
    for cell in nb['cells']:
        if cell['cell_type'] == 'code':
            source = ''.join(cell['source'])
            
            # 提取所有SELECT语句
            selects = re.findall(r'SELECT\s+(.+?)\s+FROM\s+(\w+\.\w+\.\w+)', 
                                source, re.DOTALL | re.IGNORECASE)
            
            for select_cols, table in selects:
                # 解析字段和别名
                fields = parse_select_columns(select_cols)
                for field_name, field_sql in fields.items():
                    schema[field_name] = {
                        'sql': field_sql,
                        'source_table': table
                    }
    
    return schema
```

---

## 阶段2：测试范围确定（基于diff_result ONLY）

### 变更分类规则

| 变更类型 | 判定条件 | 测试策略 | 优先级 |
|---------|---------|---------|--------|
| **Added** | old中不存在，new中存在 | **必须测试**（新增逻辑验证） | P0 |
| **Removed** | old中存在，new中不存在 | **必须测试**（回归验证/下线确认） | P1 |
| **Logic Changed** | 字段名相同，但生成SQL不同 | **必须测试**（逻辑变更验证） | P0 |
| **Unchanged** | old与new逻辑完全一致 | **免测**（3.2.1规则） | - |

### Logic Changed判定标准

两个SQL被视为"逻辑不同"的条件（满足任一）：

```python
def is_logic_different(old_sql, new_sql):
    """判定两个SQL是否逻辑不同"""
    
    # 1. 标准化处理（去除空格、换行、大小写）
    old_norm = normalize_sql(old_sql)
    new_norm = normalize_sql(new_sql)
    
    # 2. 直接文本对比
    if old_norm == new_norm:
        return False  # 完全相同
    
    # 3. 结构对比（以下差异视为logic changed）
    diff_signals = [
        'CASE WHEN' in new_norm.upper() and 'CASE WHEN' not in old_norm.upper(),
        'COALESCE' in new_norm.upper() and 'COALESCE' not in old_norm.upper(),
        'WHERE' in new_norm.upper() and 'WHERE' not in old_norm.upper(),
        'JOIN' in new_norm.upper() and 'JOIN' not in old_norm.upper(),
        # 表源变更
        extract_source_table(old_sql) != extract_source_table(new_sql),
        # 计算表达式变更
        has_calculation_changed(old_sql, new_sql)
    ]
    
    return any(diff_signals)
```

### 常见Logic Changed模式

| 模式 | Old SQL | New SQL | Change Type |
|------|---------|---------|-------------|
| NULL处理增强 | `COALESCE(col, 'N/A')` | `COALESCE(col, 'Unknown', 'N/A')` | NULL_handling |
| Cutoff日期变更 | `WHERE date >= 202601` | `WHERE date >= 202603` | Filter_condition |
| 表源变更 | `FROM mr_table` | `FROM rr_table` | Source_change |
| 新增条件分支 | `CASE WHEN a THEN 1 END` | `CASE WHEN a THEN 1 ELSE 0 END` | Case_branch |
| 计算逻辑变更 | `amount * 1.1` | `amount * 1.15` | Calculation |
| 格式转换 | `CAST(date AS VARCHAR)` | `FORMAT(date, 'yyyy-MM')` | Format_change |

---

## 禁止行为（红色警示）

> **严禁**通过分析cell注释（如`# added 2025-08`）、markdown描述、change log段落来判定变更范围。
> 
> **任何未在`diff_result`中出现的字段，即使change log声称"recently changed"，也视为未变更。**

---

## diff_result输出格式

```yaml
diff_result:
  comparison_metadata:
    old_notebook: "repair_costing_old.ipynb"
    new_notebook: "repair_costing_new.ipynb"
    comparison_timestamp: "2026-04-03T10:30:00Z"
    
  added_fields:
    - field_name: "new_supervisor_code"
      location: "gold.dbo.dm_job_costing"
      sql_logic: "CAST(supervisor_id AS VARCHAR(20))"
      detected_by: "schema_diff"
      test_priority: "P1"
      
    - field_name: "cost_center_name"
      location: "gold.dbo.dm_job_costing"
      sql_logic: |
        CASE 
          WHEN cost_center = 'CC001' THEN 'Main Plant'
          WHEN cost_center = 'CC002' THEN 'Branch A'
          ELSE 'Unknown'
        END
      detected_by: "schema_diff"
      test_priority: "P0"  # 复杂字段
      
  removed_fields:
    - field_name: "legacy_cost_code"
      old_location: "gold.dbo.dm_job_costing"
      old_sql_logic: "SELECT cost_code FROM legacy_table"
      removal_impact: "需确认下游报表是否引用此字段"
      
  logic_changed_fields:
    - field_name: "supervisor"
      location: "gold.dbo.dm_job_costing"
      change_type: "NULL_handling"
      old_logic: "COALESCE(sv.name, 'N/A')"
      new_logic: "COALESCE(sv.name, 'Unknown', 'N/A')"
      test_implication: "需验证新增'Unknown'分支的处理"
      scenarios_needed: 2
      
    - field_name: "total_cost"
      location: "gold.dbo.dm_job_costing"
      change_type: "Calculation"
      old_logic: "labor_cost + material_cost"
      new_logic: "labor_cost + material_cost + overhead_cost"
      test_implication: "需验证overhead_cost加入后的计算正确性"
      scenarios_needed: 2
      
    - field_name: "active_records"
      location: "silver.dbo.dwt_job_costing"
      change_type: "Filter_condition"
      old_logic: "WHERE status != 'Deleted'"
      new_logic: "WHERE status IN ('Active', 'Pending') AND is_deleted = 0"
      test_implication: "过滤条件变更，需验证边界情况"
      scenarios_needed: 2
      
  unchanged_fields_with_dependency:
    - field_name: "dependent_cost"
      location: "gold.dbo.dm_job_costing"
      note: "逻辑未变，但依赖字段total_cost发生logic_changed，建议抽样验证"
      
  summary:
    total_fields_old: 25
    total_fields_new: 27
    added_count: 2
    removed_count: 1
    logic_changed_count: 3
    unchanged_count: 22
    test_scope_recommendation: "建议生成8个测试场景（Added×2 + LogicChanged×2 + Removed×1 + Sample×1）"
```

---

## 对比算法实现

### 字段级对比

```python
def compare_field_level(old_schema, new_schema):
    """字段级对比，生成diff_result"""
    
    old_fields = set(old_schema.keys())
    new_fields = set(new_schema.keys())
    
    result = {
        'added_fields': [],
        'removed_fields': [],
        'logic_changed_fields': [],
        'unchanged_fields': []
    }
    
    # Added: new中有，old中无
    for field in new_fields - old_fields:
        result['added_fields'].append({
            'field_name': field,
            'new_logic': new_schema[field]
        })
    
    # Removed: old中有，new中无
    for field in old_fields - new_fields:
        result['removed_fields'].append({
            'field_name': field,
            'old_logic': old_schema[field]
        })
    
    # Logic Changed: 两者都有，但逻辑不同
    for field in old_fields & new_fields:
        if is_logic_different(old_schema[field], new_schema[field]):
            result['logic_changed_fields'].append({
                'field_name': field,
                'old_logic': old_schema[field],
                'new_logic': new_schema[field]
            })
        else:
            result['unchanged_fields'].append({
                'field_name': field,
                'logic': new_schema[field]
            })
    
    return result
```

---

## 测试范围决策

基于`diff_result`的测试范围确定：

```python
def determine_test_scope(diff_result, field_inventory):
    """确定测试范围"""
    
    test_scope = []
    
    # 1. Added字段 - 全测
    for field in diff_result['added_fields']:
        test_scope.append({
            'field': field['field_name'],
            'test_type': 'Full',
            'scenarios': 2,
            'priority': 'P1'
        })
    
    # 2. Logic Changed字段 - 全测
    for field in diff_result['logic_changed_fields']:
        test_scope.append({
            'field': field['field_name'],
            'test_type': 'Regression+New',
            'scenarios': 2,
            'priority': 'P0',
            'old_logic': field['old_logic'],
            'new_logic': field['new_logic']
        })
    
    # 3. Removed字段 - 下线验证（移至3.6集成测试）
    for field in diff_result['removed_fields']:
        test_scope.append({
            'field': field['field_name'],
            'test_type': 'Deprecation',
            'scenarios': 1,
            'priority': 'P2'
        })
    
    # 4. Unchanged字段 - 仅主键抽样
    for field in diff_result['unchanged_fields']:
        is_pk = check_is_primary_key(field['field_name'], field_inventory)
        if is_pk:
            test_scope.append({
                'field': field['field_name'],
                'test_type': 'Sample',
                'scenarios': 1,
                'priority': 'P2'
            })
    
    return test_scope
```

---

## 常见陷阱与处理

### 陷阱1：字段重命名
```python
# Old: employee_id
# New: staff_id

# 处理：通过SQL逻辑相似度判断是否为同一字段
if similarity(old_sql, new_sql) > 0.8:
    # 视为重命名，标记为LogicChanged（重命名类型）
    change_type = "Renamed"
```

### 陷阱2：SQL格式化差异
```python
# Old: SELECT a FROM t
# New: SELECT  a  FROM  t  -- 只是空格不同

# 处理：标准化后再对比
old_norm = ' '.join(old_sql.split())
new_norm = ' '.join(new_sql.split())
```

### 陷阱3：别名变更
```python
# Old: SELECT col AS old_name
# New: SELECT col AS new_name

# 处理：提取实际字段名对比
old_field = extract_actual_field(old_sql)
new_field = extract_actual_field(new_sql)
```

---

## 质量门禁

对比完成后必须检查：

- [ ] `diff_result`非空（至少有一个变更）
- [ ] 所有Added字段有完整的`new_logic`
- [ ] 所有Logic Changed字段有`old_logic`和`new_logic`
- [ ] Removed字段已标记`removal_impact`
- [ ] 总变更数与预期一致（与change log交叉验证，但**不以change log为准**）
