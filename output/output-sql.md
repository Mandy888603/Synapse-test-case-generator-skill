# 4.1 SQL编写规范

**本章职责：标准化SQL格式，确保所有测试用例SQL符合Synapse Serverless SQL Pool约束。**

---

## 4.1.1 表名与主键验证

### 表名格式（强制）

```sql
-- 正确
SELECT field FROM gold.dbo.dm_table
SELECT field FROM silver.dbo.dwt_table
SELECT field FROM staging.dbo.staging_table

-- 错误（禁止）
SELECT field FROM gold.table          -- 缺少schema dbo
SELECT field FROM dbo.table           -- 缺少database
SELECT field FROM gold.schema.table   -- schema必须是dbo
```

### 主键唯一性验证SQL模板（必须包含在[Cardinality]段）

使用**2.3 Input Layer提取的主键**生成：

```sql
[Coverage]
SELECT 
    COUNT(*) as total_records,
    COUNT(DISTINCT primary_key_field) as distinct_keys
FROM gold.dbo.dm_table
WHERE business_date >= 202603;

[Cardinality]
-- 主键唯一性验证：检查重复记录
SELECT COUNT(*) as duplicate_count
FROM (
    SELECT primary_key_field
    FROM gold.dbo.dm_table
    WHERE business_date >= 202603
    GROUP BY primary_key_field
    HAVING COUNT(*) > 1
) t;

-- 若复合主键（使用2.3提取的复合键）：
SELECT COUNT(*) as duplicate_count
FROM (
    SELECT CONCAT(pk1, '-', pk2) as composite_key
    FROM gold.dbo.dm_table
    GROUP BY CONCAT(pk1, '-', pk2)
    HAVING COUNT(*) > 1
) t;

[Value Diff]
-- 与上层或源系统比对（如适用）
SELECT COUNT(*) as mismatch_count
FROM gold.dbo.dm_table dm
LEFT JOIN silver.dbo.dwt_table dwt ON dm.pk = dwt.pk
WHERE dm.field != dwt.field;

[Conclusion]
total_records = distinct_keys 且 duplicate_count = 0，表示主键唯一性符合预期，无重复记录；与上层比对mismatch=0。
```

---

## 4.1.2 四段式SQL结构

| 段落 | 职责 | 内容示例 |
|------|------|----------|
| **[Coverage]** | 覆盖度检查 | 总记录数、目标字段非空数、业务条件命中数 |
| **[Cardinality]** | 主键唯一性、重复值、枚举值分布 | 见4.1.1模板 |
| **[Value Diff]** | 数值差异比对（Before/After, Source/Target, Layer vs Layer） | `EXCEPT`, `FULL OUTER JOIN` |
| **[Conclusion]** | 管理可读结论 | 纯文本总结，如"主键唯一，与源差异0.5%在阈值内" |

### Serverless SQL约束

- ❌ 禁止：`#temp`, `CREATE TABLE`, `INSERT`, `UPDATE`, `DELETE`, cursor, stored procedure
- ✅ 允许：CTE(`WITH`), `FULL OUTER JOIN`, `EXCEPT`, `NOT EXISTS`, window functions, `INFORMATION_SCHEMA`

---

## 4.1.3 SQL工程质量规范

```yaml
sql_quality_gates:
  forbidden_patterns:
    - pattern: "SELECT 0 as diff"
      reason: "无意义占位符，必须用实际业务验证替代"
      replacement: "使用具体字段抽样或EXCEPT比对"
      
    - pattern: "SELECT 1 as dummy"
      reason: "无意义填充，违反Coverage职责"
      replacement: "删除该段或补充实际统计"
      
    - pattern: "ABS((SELECT...) - (SELECT...)) as calc_diff"
      reason: "重复子查询，性能差"
      replacement: "使用CTE（WITH子句）预先计算"
      
    - pattern: "SELECT *"
      reason: "显式列名优于*，避免结构变更影响"
      replacement: "列出具体业务字段"

  mandatory_rules:
    - rule: "重复子查询消除"
      example:
        bad: |
          SELECT 
              (SELECT COUNT(*) FROM A) as cnt_a,
              (SELECT COUNT(*) FROM B) as cnt_b,
              ABS((SELECT COUNT(*) FROM A) - (SELECT COUNT(*) FROM B)) as diff
        good: |
          WITH counts AS (
              SELECT (SELECT COUNT(*) FROM A) as cnt_a,
                     (SELECT COUNT(*) FROM B) as cnt_b
          )
          SELECT cnt_a, cnt_b, ABS(cnt_a - cnt_b) as diff FROM counts
          
    - rule: "Value Diff必须有实际比对意义"
      criteria: "至少包含一个业务字段的数值对比，或明确的记录级差异统计"
```

---

## 4.1.4 Expected Result双轨制

```yaml
expected_result_format:
  structure: "技术断言 | 业务含义解释"
  separator: "分号或换行分隔，业务描述用括号或'即'字引导"
  
  positioning: "承接4.1.3的SQL质量要求，将工程化SQL的执行结果转化为可验收的业务证据"

  mandatory_elements:
    technical_assertion: "可量化的SQL指标（COUNT/SUM/DISTINCT结果），必须符合4.1.3的CTE优化规范"
    business_interpretation: "该指标代表的业务状态（对终端用户意味着什么）"
    
  tiered_audience:
    etl_developer: "验证技术断言是否精准反映代码逻辑（如landing_type分支）"
    tester: "执行SQL并核对技术指标数值"
    pm_ba: "查看业务含义确认验收标准，无需理解SQL语法"
    auditor: "两者结合验证合规性（技术实现+业务目标）"

  forbidden_patterns:
    - "纯技术指标堆砌（如a=1;b=2;c=3无解释）"
    - "模糊业务描述（如'数据正确'无4.1.3规范的技术指标支撑）"
    - "英文缩写无注释（如PK=0，应写为主键重复数PK=0）"

examples:
  bad_single_technical: 
    content: "duplicate_count=0; mismatch=0"
    problem: "业务人员不知道这两个指标代表什么，且未应用4.1.3的CTE优化"
    
  good_dual_track:
    technical: "主键重复数duplicate_count=0（基于4.1.1主键验证），字段差异数mismatch=0（基于4.1.3的EXCEPT比对）"
    business: "即数据加载无重复记录，关键字段值与源系统完全一致，满足财务对账一致性要求"
    
  bad_vague_business:
    content: "数据正确加载"
    problem: "无法量化验证，测试人员不知如何执行，违反4.1.3的可审计要求"
    
  good_precise:
    technical: "目标表记录数=源表记录数（100条），差异数diff=0（基于CTE预计算）"
    business: "即全量同步完成，无数据丢失，业务对象100%完整迁移"

field_specific_templates:
  landing_type_1_full_load:
    technical: "目标表记录数={source_count}（基于4.1.3的CTE），差异数diff=0，重复数duplicate=0"
    business: "即全量数据成功替换，历史数据已清理，无重复业务对象"
    
  landing_type_2_delta:
    technical: "MATCHED更新数={update_count}，NOT MATCHED插入数={insert_count}，零变更数={no_change_count}（dw时间戳未刷新）"
    business: "即{update_count}条记录业务属性变更已更新，{insert_count}条新业务对象已创建，未变更记录时间戳未刷新以节省系统资源"
    
  landing_type_3_append:
    technical: "去重后新增数={new_count}，目标表总记录数=原记录数+{new_count}"
    business: "即仅追加全新业务对象，已有数据未被覆盖，历史记录完整性保持"
    
  partition_delete:
    technical: "删除分区数={partition_count}（符合4.1.3的多列AND条件），该分区残留记录数=0"
    business: "即指定时间周期（如2026年3月）的历史数据已完全清理，满足GDPR/数据保留策略要求"
```

---

## SQL模板库

### 模板1：字段存在性验证

```sql
[Coverage]
-- 验证目标字段存在且有值
SELECT 
    COUNT(*) as total_records,
    COUNT(target_field) as non_null_count,
    COUNT(*) - COUNT(target_field) as null_count
FROM gold.dbo.dm_table
WHERE business_date >= <TBD_CUTOFF_DATE>;

[Cardinality]
-- 主键唯一性验证
SELECT COUNT(*) as duplicate_pk
FROM (
    SELECT primary_key_field
    FROM gold.dbo.dm_table
    WHERE business_date >= <TBD_CUTOFF_DATE>
    GROUP BY primary_key_field
    HAVING COUNT(*) > 1
) t;

[Value Diff]
-- 与源表比对（如适用）
SELECT COUNT(*) as mismatch
FROM gold.dbo.dm_table dm
FULL OUTER JOIN silver.dbo.dwt_table dwt 
    ON dm.pk = dwt.pk
WHERE dm.target_field != dwt.source_field
   OR (dm.target_field IS NULL AND dwt.source_field IS NOT NULL)
   OR (dm.target_field IS NOT NULL AND dwt.source_field IS NULL);

[Conclusion]
字段存在性验证通过：total_records > 0，null_count在预期范围内，主键唯一（duplicate_pk=0），与源表比对一致（mismatch=0）。
```

### 模板2：NULL处理验证

```sql
[Coverage]
-- 统计源表NULL值和DM层映射结果
SELECT 
    COUNT(*) as total_records,
    SUM(CASE WHEN source_field IS NULL THEN 1 ELSE 0 END) as source_null_count,
    SUM(CASE WHEN dm.mapped_field = 'Unknown' THEN 1 ELSE 0 END) as mapped_unknown_count,
    SUM(CASE WHEN dm.mapped_field != 'Unknown' AND source_field IS NULL THEN 1 ELSE 0 END) as incorrect_mapping
FROM gold.dbo.dm_table dm
JOIN silver.dbo.dwt_table dwt ON dm.pk = dwt.pk
WHERE dm.business_date >= <TBD_CUTOFF_DATE>;

[Cardinality]
-- 主键唯一性（同上模板）

[Value Diff]
-- 验证NULL值是否正确映射为'Unknown'
SELECT COUNT(*) as null_not_mapped
FROM gold.dbo.dm_table dm
JOIN silver.dbo.dwt_table dwt ON dm.pk = dwt.pk
WHERE dwt.source_field IS NULL 
  AND dm.mapped_field != 'Unknown';

[Conclusion]
NULL处理验证通过：所有源表NULL值已正确映射为'Unknown'（null_not_mapped=0），非NULL值保持原样。
```

### 模板3：跨层数据一致性

```sql
[Coverage]
-- 各层记录数快照
WITH date_params AS (
    SELECT CAST('<TBD_CUTOFF_DATE>' AS DATE) as load_date_filter
)
SELECT 'ingress' as layer, COUNT(*) as record_count 
FROM ingress.dbo.ingress_table t
CROSS JOIN date_params d
WHERE t.load_date = d.load_date_filter

UNION ALL

SELECT 'staging', COUNT(*) 
FROM staging.dbo.staging_table t
CROSS JOIN date_params d
WHERE t.load_date = d.load_date_filter

UNION ALL

SELECT 'gold', COUNT(*) 
FROM gold.dbo.dm_table t
CROSS JOIN date_params d
WHERE t.calendar_year_month = CAST(FORMAT(d.load_date_filter, 'yyyyMM') AS INT);

[Cardinality]
-- 各层主键去重数
WITH date_param AS (
    SELECT CAST('<TBD_CUTOFF_DATE>' AS DATE) as load_date_filter
)
SELECT 
    'staging' as layer, 
    COUNT(DISTINCT business_key) as distinct_keys
FROM staging.dbo.staging_table
WHERE load_date = (SELECT load_date_filter FROM date_param)

UNION ALL

SELECT 
    'gold', 
    COUNT(DISTINCT business_key)
FROM gold.dbo.dm_table
WHERE calendar_year_month = CAST(FORMAT((SELECT load_date_filter FROM date_param), 'yyyyMM') AS INT);

[Value Diff]
-- 抽样比对
SELECT COUNT(*) as diff_count 
FROM (
    SELECT business_key, amount 
    FROM silver.dbo.dwt_table
    WHERE business_key IN ('KEY001','KEY002','KEY003')
    
    EXCEPT
    
    SELECT business_key, amount 
    FROM gold.dbo.dm_table
    WHERE business_key IN ('KEY001','KEY002','KEY003')
) t;

[Conclusion]
跨层数据一致性验证通过：各层记录数差异符合预期过滤逻辑，主键去重数一致，抽样比对无差异（diff_count=0）。
```

---

## SQL反模式检查清单

生成SQL后必须检查：

- [ ] 无`SELECT 0`或`SELECT 1`占位符
- [ ] 无`SELECT *`，必须显式列名
- [ ] 重复子查询已用CTE优化
- [ ] 表名格式正确（database.dbo.table）
- [ ] 包含[Coverage]→[Cardinality]→[Value Diff]→[Conclusion]四段
- [ ] [Cardinality]段包含主键唯一性验证
- [ ] 日期参数使用<TBD_CUTOFF_DATE>占位符（待FS文档确认后替换）
