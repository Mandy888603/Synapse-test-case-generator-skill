# 文档地图 / 决策树（Document Map & Decision Tree）

本文档提供完整的执行导航和条件决策树。

---

## 条件标记注册表（Condition Registry）

```yaml
condition_registry:
  # === 输入层条件 ===
  HAS_FS_TS: 
    type: binary
    detector: "file_exists('*.docx') in openspec/changes/[change-id]/"
    
  HAS_OLD_NB: 
    type: binary
    detector: "file_exists('*_old.ipynb')"
    
  HAS_NEW_NB: 
    type: binary
    detector: "file_exists('*_new.ipynb')"
  
  # === 项目类型派生条件（互斥组）===
  IS_ENHANCEMENT: 
    type: derived
    expression: "HAS_OLD_NB AND HAS_NEW_NB"
    used_in: [step_5_change_points]
    
  IS_NEW_DEV: 
    type: derived
    expression: "NOT HAS_OLD_NB AND HAS_NEW_NB"
    mutex_with: IS_ENHANCEMENT
    
  IS_CODELESS:
    type: derived
    expression: "NOT HAS_OLD_NB AND NOT HAS_NEW_NB"
    triggers: [step_8_output_placeholder_mode]
  
  # === 文档特征条件 ===
  IS_LARGE_DOC:
    type: binary
    detector: "paragraph_count > 50 OR file_size > 524288"
    triggers: [2.2.1_large_doc_handling]
  
  # === 阻断点状态条件 ===
  SCOPE_CONFIRMED:
    type: state
    set_by: "user_input_at_2.1.5"
    required_for: [step_3_docx_extraction_continue]
    
  VERSION_SELECTED:
    type: enum
    values: [1, 2, 3]
    set_by: "user_input_at_2.4"
    
  CHANGE_POINTS_FROZEN:
    type: state
    set_by: "user_input_at_2.5.2"
    required_for: [step_7_analysis]
  
  # === 分析层条件 ===
  DM_EXISTS:
    type: inferred
    detector: "从notebook SQL推断存在gold.dbo.dm_*表"
    
  LOGIC_DIFFERENT:
    type: derived
    detector: "diff_result中标记为logic_changed的字段存在"
    
  INTEGRATION_REQUIRED:
    type: binary
    detector: "notebook.contains('mssparkutils.notebook.run') OR landing_type == 2"

mutex_groups:
  project_type: [IS_ENHANCEMENT, IS_NEW_DEV, IS_CODELESS]
  doc_processing: [IS_LARGE_DOC, IS_NORMAL_DOC]
```

---

## 九步执行流程（完整映射）

```yaml
step_1_scan:
  action: 扫描目录
  detect: [FS/TS.docx, old.ipynb, new.ipynb, template.xlsx]
  next: step_2

step_2_template:
  trigger: 强制
  blocking: true
  action: 加载template.xlsx
  load_section: [rules/rule-a.md 4.2.0, rules/rule-a.md 4.2.1]
  next: step_3

step_3_docx_extraction:
  trigger: "{IF: FS/TS.docx EXISTS}"
  condition_branch:
    normal_doc: 
      condition: "para<=50 AND size<=500KB"
      load: [fs/fs-core.md 2.1, fs/fs-core.md 2.2]
    large_doc:
      condition: "para>50 OR size>500KB"
      load: [fs/fs-core.md 2.1, fs/fs-core.md 2.2,fs/fs-edge.md 2.2.1]
  blocking_sub_step:
    - 2.1.5 范围澄清（多义时必须确认）
    - 2.2.5 DOCX提取完整性验证（展示清单等待确认）
  next: step_3b

step_3b_primary_key:
  trigger: "{IF: notebook EXISTS OR docx有DDL}"
  action: 主键识别
  load_section: [rules/rule-b.md 2.3]
  next: step_4_pre_check

step_4_pre_check:  
  condition: "{IF: NOT VERSION_SELECTED}"
  if_true: step_4_version_gate  # 进入阻断
  if_false: step_5_pre_check    # 已确认则跳过

step_4_version_gate:
  trigger: "{IF: INPUT_EXTRACTION_COMPLETE AND NOT VERSION_SELECTED}"
  blocking: true
  action: 询问版本1/2/3
  set_state: VERSION_SELECTED
  next: step_5_pre_check

step_5_pre_check: 
  conditions:
    - "{IF: NOT HAS_NEW_NB}" → 跳转 step_8_output_placeholder_mode
    - "{IF: CHANGE_POINTS_FROZEN}" → 跳转 step_6_inventory  # 已确认则跳过
    - "{IF: HAS_NEW_NB AND NOT CHANGE_POINTS_FROZEN}" → step_5_change_points

step_5_change_points:  # 对应2.5, 2.5.0-2.5.3
  trigger: "{IF: VERSION_SELECTED AND HAS_NEW_NB AND NOT CHANGE_POINTS_FROZEN}"
  load_sections:
    - [notebooks/nb-diff.md 2.5.0.1] (IF:old+new, Enhancement)
    - [notebooks/nb-diff.md 2.5.0.2] (IF:new_only, New Development)
    - [rules/rule-b.md 2.5.1] (展示清单)
    - [rules/rule-b.md 2.5.2] (用户确认流程 - 阻断)
    - [rules/rule-b.md 2.5.3] (确认后锁定)
  blocking: true  # 2.5.2阻断点
  output: confirmed_change_points
  next: step_6

step_6_inventory:
  trigger: "{IF: notebook EXISTS}"
  load_section: [rules/rule-b.md 2.6]
  output: field_inventory.yaml
  next: step_7

step_7_analysis:
  trigger: "{IF: NOT IS_CODELESS}"
  load_sections:
    entry_gate: [rules/rule-c.md 3.0]
    strategy:
      - [rules/rule-c.md 3.1.1] (IF: Enhancement)
      - [rules/rule-c.md 3.1.2] (IF: New Development)
    core_logic: ["rules/rule-c.md 3.2", "rules/rule-c.md 3.3", "rules/rule-c.md 3.4", "rules/rule-c.md 3.5"]
    integration: [rules/rule-c.md 3.6] (IF: INTEGRATION_REQUIRED)
  next: step_8

step_8_output:
  trigger: Always
  branches:
    - condition: "{IF: IS_CODELESS}"
      load: [output/output-excel.md 4.0_placeholder_mode]
    - condition: "{IF: NOT IS_CODELESS}"
      parallel_output:
        excel: [output/output-excel.md 4.2]
        sql: [output/output-sql.md 4.1]
        markdown: [output/output-excel.md 4.3]
  next: step_9

step_9_quality_gate:
  trigger: Always
  load_sections:
    - quality/quality-check.md 5.1, quality/quality-check.md 5.2, quality/quality-check.md 5.2.1
  final_action: "更新manifest + 生成P0自检报告"
```

---

## 动态加载决策树（可视化）

```
Start
 │
 ▼
Step 1: 扫描 ──┬─► HAS_FS_TS=True ──┬─► IS_LARGE_DOC? ──YES──► 2.2.1分页
               │                    │      NO ──► 2.2标准提取
               │                    ▼
               │              2.1.5范围确认(阻断)
               │                    ▼
               │              2.2.5完整性确认(阻断) ──► Step 3b
               │                    
               └─► HAS_FS_TS=False ────────────┘
 │
 ▼
Step 3b: PK识别 ──┬─► 条件满足 ──► 2.3 PK识别 ──► Step 4
                  │
                  └─► 条件不满足 ──► <TBD_PK>占位 ──► Step 4
 │
 ▼
Step 4: 版本确认(阻断) ──► VERSION_SELECTED ∈ {1,2,3}
 │
 ├─► IS_CODELESS ────────────────────────────────┐
 │                                               ▼
 │                                         Step 8: 占位符模式
 │                                         (生成<TBD_*>框架)
 │                                               │
 │                                               ▼
 └─► NOT IS_CODELESS ──► Step 5 ──► Step 6 ──► Step 9
                                                          │
Step 5: ──┬─► IS_ENHANCEMENT ──► 2.5.0.1 Diff对比         │
          └─► IS_NEW_DEV ────► 2.5.0.2 全量提取            │
                                                          ▼
Step 6: 2.6 Inventory整合 (IF: HAS_NEW_NB)               │
                                                          ▼
Step 7: 3.0准入检查 ──► 3.1.1/3.1.2 ──► 3.2决策           │
       (设置INTEGRATION_REQUIRED) ──► 3.6(IF条件触发)    │
                                                          ▼
Step 8: 4.0输出层 ──┬─► 4.1 SQL生成                       │
                    ├─► 4.2 Excel (EN/CN/双语)           │
                    └─► 4.3 Markdown (条件触发)           │
                                                          │
                                                          ▼
                                                  Step 9: 质量门禁
                                                  (强制：Always)
```

---

## 分层测试决策树

```
START: 字段X需要测试
    │
    ▼
存在DM层(gold.dbo.dm_*)? ──YES──► 测试DM层
    │                               │
    NO                              ▼
    │                        DWT与DM逻辑不一致?
    ▼                               │
存在DWT层(silver/prepared.dbo.dwt_*)? ──YES──► 测试DWT层  ◄──YES──┘
    │                               │              (补充测试DWT)
    NO                              │
    │                               ▼
    ▼                         逻辑一致(仅透传)
存在Staging层(staging.dbo.*)? ──YES──► 测试Staging层 ──► 结束(DWT覆盖)
    │
    NO
    │
    ▼
存在Ingress层(ingress.dbo.*)? ──YES──► 测试Ingress层
    │
    NO
    │
    ▼
    错误：无可用数据层
```

---

## 文档依赖关系图

```
SKILL.md (入口)
    │
    ├──► index.md (本文档 - 决策树)
    │
    ├──► rules/
    │       ├── rule-a.md (核心原则) ──► 所有步骤引用
    │       ├── rule-b.md (输入提取) ──► Step 3-6
    │       └── rule-c.md (分析决策) ──► Step 7
    │
    ├──► fs/
    │       ├── fs-core.md ──► Step 3 (标准文档)
    │       └── fs-edge.md ──► Step 3 (大文档)
    │
    ├──► notebooks/
    │       ├── nb-extraction.md ──► Step 5-6 (New Dev)
    │       └── nb-diff.md ──► Step 5 (Enhancement)
    │
    ├──► output/
    │       ├── output-sql.md ──► Step 8 (SQL生成)
    │       └── output-excel.md ──► Step 8 (Excel/Markdown)
    │
    ├──► quality/
    │       └── quality-check.md ──► Step 9 (检查清单)
    │
    └──► examples/
            └── examples.md ──► 参考示例
```

---

## 快速查找表

| 问题 | 参考文档 | 章节 |
|------|----------|------|
| ID格式是什么？ | rule-a.md | 1.1 |
| 表名怎么写？ | rule-a.md | 1.1 |
| 如何提取FS文档？ | fs-core.md | 2.2 |
| 大文档怎么处理？ | fs-edge.md | 2.2.1 |
| 如何识别主键？ | rule-b.md | 2.3 |
| Enhancement怎么对比代码？ | nb-diff.md | 2.5.0.1 |
| New Dev怎么提取全量？ | nb-extraction.md | 2.5.0.2 |
| Change Points怎么确认？ | rule-b.md | 2.5.2 |
| 分层测试优先级？ | rule-c.md | 3.2 |
| 字段场景怎么设计？ | rule-c.md | 3.3 |
| SQL四段式怎么写？ | output-sql.md | 4.1 |
| Excel怎么生成？ | output-excel.md | 4.2 |
| 交付前检查什么？ | quality-check.md | 5.2 |
