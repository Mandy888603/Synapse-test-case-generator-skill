# Notebook提取规则 - New Development项目

**适用场景**：New Development项目（无old notebook），全量提取新代码Schema

---

## 2.5.0.2 New Development项目（无old notebook）

### 分析重点

- **Full Schema**：提取 new notebook 的全量 Schema → `full_schema`
- **Data Lineage**：从源头到目标的数据流验证
- **Business Rules**：所有业务规则覆盖（cutoff、mapping、calculation）
- **Data Quality**：主键唯一性、非空约束、值域验证

### 提取内容

包含：所有字段名、生成逻辑 SQL、主键信息、数据类型

**输出格式**：
```yaml
full_schema:
  table_name: "gold.dbo.dm_new_table"
  source_table: "silver.dbo.dwt_source"
  
  fields:
    - name: "field_1"
      sql_logic: "CAST(source_col AS VARCHAR(100))"
      type: "VARCHAR(100)"
      nullable: true
      
    - name: "field_2"  
      sql_logic: "CASE WHEN condition THEN 'A' ELSE 'B' END"
      type: "VARCHAR(50)"
      nullable: false
      
    - name: "field_3"
      sql_logic: "COALESCE(source_field, 'Unknown')"
      type: "VARCHAR(100)"
      nullable: false
      
  primary_keys:
    - "field_1"
    
  business_rules_detected:
    - type: "cutoff_date"
      field: "calendar_year_month"
      condition: ">= 202603"
    - type: "null_handling"
      field: "field_3"
      logic: "COALESCE(source_field, 'Unknown')"
```

### 测试范围

**全量功能覆盖**：所有字段均需测试

---

## Notebook代码提取方法

### 提取目标

从`.ipynb`文件中提取：
1. **CREATE TABLE语句** → 表结构、字段名、数据类型
2. **SELECT语句** → 字段生成逻辑
3. **注释标记** → 主键、业务规则标记
4. **mssparkutils调用** → 集成测试触发检测

### 提取步骤

```python
# 伪代码
import json

with open('notebook_new.ipynb', 'r') as f:
    nb = json.load(f)

schema = {}
for cell in nb['cells']:
    if cell['cell_type'] == 'code':
        source = ''.join(cell['source'])
        
        # 提取CREATE TABLE
        if 'CREATE TABLE' in source:
            table_info = extract_create_table(source)
            schema['table_name'] = table_info['table_name']
            schema['fields'] = table_info['fields']
            
        # 提取SELECT逻辑
        if 'SELECT' in source and 'FROM' in source:
            field_logic = extract_select_logic(source)
            merge_field_logic(schema, field_logic)
            
        # 检测mssparkutils链式调用
        if 'mssparkutils.notebook.run' in source:
            schema['has_chain_call'] = True
            schema['next_steps'] = extract_next_steps(source)
```

### 关键SQL提取模式

| 模式 | 正则/关键词 | 用途 |
|------|------------|------|
| 表名 | `CREATE TABLE\s+(\w+\.\w+\.\w+)` | 提取目标表名 |
| 字段定义 | `(\w+)\s+(\w+\(?\d*\)?)` | 字段名+数据类型 |
| SELECT逻辑 | `SELECT\s+(.+?)\s+FROM` | 字段生成SQL |
| 主键注释 | `--\s*PK:\s*(\w+)` | 主键识别 |
| Cutoff日期 | `>=\s*(\d{6})` | 业务日期条件 |
| COALESCE | `COALESCE\s*\((.+?)\)` | NULL处理逻辑 |
| CASE WHEN | `CASE\s+WHEN` | 条件分支逻辑 |

---

## 复杂度评分规则

用于`field_inventory.test_metadata.complexity_score`：

| 分数 | 定义 | 判定规则 |
|------|------|----------|
| **1** | 简单透传 | `SELECT field FROM source`（无函数包裹） |
| **2** | 条件判断 | 含`CASE WHEN`、`COALESCE`、`NULLIF` |
| **3** | 复杂计算 | 含计算表达式、多表JOIN、窗口函数、字符串操作 |

### 自动评分逻辑

```python
def calculate_complexity(sql_logic):
    score = 1  # 基础分
    
    # 条件判断 +1
    if any(kw in sql_logic.upper() for kw in ['CASE', 'COALESCE', 'NULLIF']):
        score += 1
        
    # 复杂计算 +1
    if any(kw in sql_logic.upper() for kw in ['JOIN', 'OVER', 'ROW_NUMBER', 'FORMAT', 'SUBSTRING']):
        score += 1
        
    # 计算表达式 +1
    if any(op in sql_logic for op in [' + ', ' - ', ' * ', ' / ']):
        score += 1
        
    return min(score, 3)  # 最高3分
```

---

## 数据血缘提取

### 自动推断血缘关系

```yaml
data_lineage_extraction:
  method: "从SQL语句解析FROM/JOIN子句"
  
  rules:
    - pattern: "FROM\s+(\w+\.\w+\.\w+)"
      extract: "source_table"
    - pattern: "JOIN\s+(\w+\.\w+\.\w+)"
      extract: "join_tables"
    - pattern: "INSERT\s+INTO\s+(\w+\.\w+\.\w+)"
      extract: "target_table"
      
  output_format:
    lineage_path: "ingress.dbo.source → staging.dbo.stg → silver.dbo.dwt → gold.dbo.dm"
    transformations:
      - layer: "staging→silver"
        type: "field_merge"
        description: "多个staging字段合并为一个silver字段"
      - layer: "silver→gold"
        type: "aggregation"
        description: "按业务键聚合"
```

---

## 常见Notebook模式识别

### 模式1：标准ETL流程
```python
# 典型结构
# Cell 1: 配置参数
config = {...}

# Cell 2: 读取源数据
df = spark.read.table("ingress.dbo.source")

# Cell 3: 转换逻辑
df_transformed = df.select(...)

# Cell 4: 写入目标
df_transformed.write.mode("overwrite").saveAsTable("gold.dbo.dm_target")
```

### 模式2：带Cutoff日期的增量加载
```python
# 识别cutoff日期
cutoff_date = "202603"  # 从SQL中提取

sql = """
SELECT * FROM source
WHERE calendar_year_month >= 202603  -- 提取此日期
"""
```

### 模式3：链式Notebook调用
```python
# 触发INTEGRATION_REQUIRED = true
mssparkutils.notebook.run("Child_Notebook_1", timeout=3600)
mssparkutils.notebook.run("Child_Notebook_2", timeout=3600)
```

---

## 提取质量检查

### 必填字段检查
- [ ] `table_name` 已提取且格式正确（database.dbo.table）
- [ ] `fields` 列表非空
- [ ] 每个字段有 `name` 和 `type`
- [ ] 至少一个字段有 `sql_logic`

### 完整性检查
- [ ] CREATE TABLE字段数 == SELECT逻辑字段数
- [ ] 主键已识别（显式或推断）
- [ ] Cutoff日期已提取（如存在业务日期过滤）

### 异常处理
- 字段类型无法识别 → 标记`<TBD_TYPE>`
- SQL逻辑过于复杂 → 标记`<REVIEW_REQUIRED>`
- 多表JOIN无法解析血缘 → 标记`<TBD_LINEAGE>`
