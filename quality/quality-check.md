# 5. Quality Control - 质量控制

**本章职责：交付前的全面质量检查，确保测试用例符合所有规范。**

---

## 5.1 反幻觉规则（Anti-Hallucination）

**必须使用占位符`<TBD_*>`**（禁止发明）：

| 缺失信息 | 占位符 | 标记方式 |
|---------|--------|----------|
| 表名未在输入中出现 | `<TBD_TABLE>` | Sql的[Conclusion]段添加`-- TODO: Confirm [TBD_TABLE]` |
| 字段名未定义 | `<TBD_FIELD>` | Sql的[Conclusion]段添加`-- TODO: Confirm [TBD_FIELD]` |
| 主键无法推断 | `<TBD_PK>` | Sql的[Conclusion]段添加`-- TODO: Confirm [TBD_PK]` |
| 源系统路径不确定 | `<TBD_SOURCE>` | Sql的[Conclusion]段添加`-- TODO: Confirm [TBD_SOURCE]` |
| Cutoff日期未提取 | `<TBD_CUTOFF_DATE>` | Sql的[Conclusion]段添加`-- TODO: Confirm [TBD_CUTOFF_DATE]` |

---

## 5.2 交付前检查清单

### 通用检查

- [ ] **分层策略应用**：仅测试最高可用层（DM>DWT>Staging>Ingress），逻辑差异例外
- [ ] **字段场景数合规**：单字段≤2场景，简单字段=1场景，总场景≤15（超需确认）
- [ ] **主键验证存在**：所有DM/DWT表包含[Cardinality]段的主键唯一性检查（使用2.3提取的PK）
- [ ] **Procedure≤3步**：步骤已合并，无冗长描述
- [ ] **SQL四段式**：[Coverage]→[Cardinality]→[Value Diff]→[Conclusion]
- [ ] **表名规范**：全部`<database>.dbo.<table>`格式
- [ ] **跨层/源系统场景**：如存在多层或源系统对账需求，已生成对应场景

### Excel特定检查

- [ ] **写入位置**：SIT Case Sheet（非Change History）
- [ ] **表头已填**：行3-6列E（Project/ITCM/PM/Env）
- [ ] **ID格式**：`TC.{L1}.{L2}`（自然数）
- [ ] **行高字体**：行9+行高80pt，TKE Type字体
- [ ] **Change History**：行2已追加变更记录（格式正确）
- [ ] **Level 1 No.** 为连续自然数（1,2,3...）无跳跃，无前导零（❌ 01, 001）
- [ ] **Level 2 No.** 在每个 Level 1 组内为连续自然数（1,2,3...），无前导零（❌ 01, 001）
- [ ] **ID** 格式严格为 TC.{L1}.{L2}（如 TC.1.1, TC.1.2, TC.2.1），无重复 ID

### 中英文一致性检查

- [ ] **场景数量**：EN与CN版本测试用例数完全一致
- [ ] **SQL一致**：CN版本的Sql列与EN完全英文一致
- [ ] **术语保留**：CN版本业务术语未翻译（opportunity/tender等）

---

## 5.2.1 补充检查项

### 场景完整性检查

- [ ] **增量细分覆盖**：landing_type=2时，是否包含DEL-01至DEL-04（冷启动、纯追加、纯更新、零变更）
- [ ] **防御性测试存在**：每个关键配置字段（key_columns/partition_column/source_query）是否有NULL/空值边界TC
- [ ] **类型溢出验证**：Decimal/Integer/String类型映射是否有精度或转义验证（DEF-02）
- [ ] **空源表验证**：是否有TC验证源表0条记录时的处理逻辑（DEF-03）

### 集成验证检查

- [ ] **前置依赖容错**：是否有INT-01验证ingress表不存在时的错误信息可读性
- [ ] **Schema漂移检测**：是否有INT-02验证mapping配置与源表结构一致性
- [ ] **Next Step验证**：如有next_step配置，是否有INT-03/INT-04验证链式调用

### SQL质量检查

- [ ] **无占位符**：全文搜索"SELECT 0"或"SELECT 1"，确认无无意义填充
- [ ] **子查询优化**：确认无重复子查询（同一SELECT出现>1次），必须使用CTE
- [ ] **显式列名**：确认无"SELECT *"，必须列出具体字段

### 业务逻辑深度

- [ ] **零变更不刷新**：landing_type=2时，是否有验证数据无变化时dw_update_time不应刷新的场景（DEL-04）
- [ ] **复合分区删除**：partition_process_type=1且多列分区时，是否验证AND条件拼接正确（如year=2026 AND month=03）

---

## 5.2.2 交付物完整性检查（强制）

```python
# 动态生成检查清单（基于2.4节确认的版本）
if version_choice == "1":  # 仅英文
    required_deliverables = [
        f"TKE IT Project SIT Test Case_{project}_ITCM{number}.xlsx",
        f"TKE_IT_Project_SIT_Test_Case_{project}_ITCM{number}.md"
    ]
elif version_choice == "2":  # 仅中文
    required_deliverables = [
        f"TKE IT Project SIT Test Case _{project}_ITCM{number}_CN.xlsx",
        f"TKE_IT_Project_SIT_Test_Case_{project}_ITCM{number}_CN.md"
    ]
else:  # 3 或默认 中英文
    required_deliverables = [
        f"TKE IT Project SIT Test Case_{project}_ITCM{number}.xlsx",
        f"TKE IT Project SIT Test Case _{project}_ITCM{number}_CN.xlsx",
        f"TKE_IT_Project_SIT_Test_Case_{project}_ITCM{number}.md",
        f"TKE_IT_Project_SIT_Test_Case_{project}_ITCM{number}_CN.md"
    ]
```

---

## Expected Result业务可读性检查

- [ ] **双轨制结构**：每个Expected Result必须包含"技术层面："和"业务层面："（或同等清晰的分隔）
- [ ] **业务术语翻译**：技术指标（如duplicate_count）后必须紧跟业务解释（如"即无重复记录"）
- [ ] **验收标准明确**：业务层面描述应让非技术PM能判断"这个数据是否符合业务预期"
- [ ] **无纯缩写**：禁止单独使用"PK=0"，必须写为"主键重复数PK=0"或"主键唯一性PK无重复"
- [ ] **价值阐述**：业务描述应说明"这个数据质量对下游业务的影响"（如对账准确性、报表完整性、合规要求）
- [ ] **Expected Result双轨制**：列I必须同时包含技术指标（如count=100）和业务解释（如无丢失），仅有一项视为不完整，禁止交付

---

## P0自检报告模板

```yaml
quality_gate_report:
  generated_at: "2026-04-03T10:30:00Z"
  project: "Repair Costing Enhancement"
  itcm_number: "1812420"
  
  checks:
    - category: "格式规范"
      items:
        - check: "ID格式"
          status: "PASS"
          details: "所有ID符合TC.{L1}.{L2}格式，无重复"
        - check: "表名规范"
          status: "PASS"
          details: "所有表名使用database.dbo.table格式"
        - check: "SQL四段式"
          status: "PASS"
          details: "所有SQL包含Coverage/Cardinality/Value Diff/Conclusion"
          
    - category: "内容完整性"
      items:
        - check: "主键验证"
          status: "PASS"
          details: "所有DM/DWT表包含主键唯一性验证"
        - check: "分层策略"
          status: "PASS"
          details: "按DM>DWT>Staging优先级测试"
        - check: "场景数量"
          status: "PASS"
          details: "单字段≤2场景，总计12个场景"
          
    - category: "SQL质量"
      items:
        - check: "无占位符SQL"
          status: "PASS"
          details: "无SELECT 0/1占位符"
        - check: "CTE优化"
          status: "PASS"
          details: "重复子查询已用CTE优化"
        - check: "显式列名"
          status: "PASS"
          details: "无SELECT *"
          
    - category: "Expected Result"
      items:
        - check: "双轨制"
          status: "PASS"
          details: "所有Expected Result包含技术+业务描述"
        - check: "无缩写"
          status: "PASS"
          details: "无单独PK=0等缩写"
          
    - category: "交付物"
      items:
        - check: "Excel文件"
          status: "PASS"
          details: "TKE IT Project SIT Test Case_Repair Costing_ITCM1812420.xlsx"
        - check: "Markdown文件"
          status: "PASS"
          details: "TKE_IT_Project_SIT_Test_Case_Repair_Costing_ITCM1812420.md"
          
  summary:
    total_checks: 15
    passed: 15
    failed: 0
    warnings: 0
    
  conclusion: "所有质量检查通过，测试用例符合交付标准。"
```

---

## 占位符模式特殊检查

当生成占位符模式测试用例时：

- [ ] 检查所有`<TBD_*>`标记已正确插入
- [ ] 生成待人工填充项清单
- [ ] 标记为DRAFT状态（非最终交付）

### 待人工填充项清单模板

```markdown
## 待人工填充项清单

| 占位符 | 位置 | 说明 | 建议来源 |
|--------|------|------|----------|
| <TBD_TABLE> | TC.1.1 Sql | 目标表名 | FS文档Section 3.2 |
| <TBD_PK> | TC.1.1 Sql | 主键字段 | Notebook CREATE TABLE |
| <TBD_CUTOFF_DATE> | TC.1.1, TC.2.1 Sql | 生效日期 | FS文档Section 6.1 |
| <TBD_LOGIC> | TC.3.1 Sql | supervisor生成逻辑 | Notebook SELECT语句 |

## 填充后操作
1. 替换所有<TBD_*>为实际值
2. 重新执行SQL验证
3. 更新Expected Result数值
4. 移除-- TODO注释
5. 将状态从DRAFT改为FINAL
```

---

## 质量门禁失败处理

```
质量检查失败
    │
    ├──► 格式错误（ID/表名/SQL结构）
    │       └──► 自动修复或标记为BLOCKER
    │
    ├──► 内容缺失（主键验证/分层策略）
    │       └──► 返回对应章节补充生成
    │
    ├──► SQL质量问题（占位符/重复子查询）
    │       └──► 返回4.1节重新生成SQL
    │
    └──► Expected Result不完整
            └──► 补充业务解释或技术指标
```
