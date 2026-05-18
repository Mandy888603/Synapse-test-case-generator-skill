---
name: openspec-generate-testcase-document
description: TKE Synapse Serverless SIT/UAT 自动生成符合template格式的专业/高质量测试用例.
license: MIT
compatibility: Requires openspec CLI.  
user:
  invocable: true 
metadata:
  author: DA IT Team
  version: "1.0"
  generatedBy: "1.2.0"

---

# Synapse SIT Test Case Generator - AI Skill Guide

**版本**：v2.6  
**适用**：TKE Synapse Serverless SIT/UAT 自动生成符合template格式的专业/高质量测试用例  
**架构**：判断流程走向 → 原则约束 → 输入提取 → 分析决策 → 输出生成 → 检查后交付

---

## 快速开始（Quick Start）

### 1. 前置检查
确保工作目录包含以下文件：
- `TKE_IT_Template_Functional_Requirement_Specification_*_ITCM*.docx` → 业务规则
- `TKE IT Project SIT Test Case - template_ITCMXXXXX.xlsx` → 模板
- `[notebook_name]_new.ipynb` → 新逻辑代码
- `[notebook_name]_old.ipynb` → 旧逻辑代码（Enhancement项目）

### 2. 执行流程（九步导航）

```
Step 1: 扫描目录 → 检测文件存在性
    │
Step 2: 加载模板 → 强制加载template.xlsx
    │
Step 3: 提取FS/TS → 业务规则提取（大文档触发分页策略）
    │
Step 3b: 主键识别 → 从代码/注释推断PK
    │
Step 4: 版本确认 → 阻断点：用户选择1/2/3（EN/CN/双语）
    │
Step 5: Change Points → 代码对比或全量提取(阻断点确认测试范围)
    │
Step 6: Field Inventory → 整合多源数据
    │
Step 7: 分析决策 → 分层策略 + 场景设计
    │
Step 8: 输出生成 → Excel + Markdown（条件触发）
    │
Step 9: 质量门禁 → 强制检查清单
```

### 3. 项目类型判定（强制）

```python
if old_notebook.exists():
    project_type = "Enhancement"      # 代码对比流程
else:
    project_type = "New Development"  # 全量提取流程
```

### 4. 核心决策树

```
存在DM层(gold.dbo.dm_*)? ──YES──► 测试DM层
    │                               │
    NO                              ▼
    │                        DWT与DM逻辑不一致?
    ▼                               │
存在DWT层? ──YES──► 测试DWT层 ◄──YES──┘
    │
    ▼
存在Staging层? ──YES──► 测试Staging层
    │
    ▼
存在Ingress层? ──YES──► 测试Ingress层
```

---

## 文档导航

| 文件 | 用途 | 何时加载 |
|------|------|----------|
| [index.md](./index.md) | 文档地图/完整决策树 | 始终 |
| [rules/rule-a.md](./rules/rule-a.md) | 核心原则与硬性约束 | Step 1 |
| [rules/rule-b.md](./rules/rule-b.md) | 输入提取规则 | Step 3-6 |
| [rules/rule-c.md](./rules/rule-c.md) | 分析决策规则 | Step 7 |
| [fs/fs-core.md](./fs/fs-core.md) | FS/TS文档提取核心 | Step 3 |
| [fs/fs-edge.md](./fs/fs-edge.md) | 大文档处理策略 | Step 3 (大文档) |
| [notebooks/nb-extraction.md](./notebooks/nb-extraction.md) | Notebook提取规则 | Step 5-6 |
| [notebooks/nb-diff.md](./notebooks/nb-diff.md) | 代码对比规则 | Step 5 (Enhancement) |
| [output/output-sql.md](./output/output-sql.md) | SQL编写规范 | Step 8 |
| [output/output-excel.md](./output/output-excel.md) | Excel生成规范 | Step 8 |
| [quality/quality-check.md](./quality/quality-check.md) | 质量控制清单 | Step 9 |
| [examples/examples.md](./examples/examples.md) | 示例用例 | 参考 |

---

## 硬性约束速查（不可协商）

| 维度 | 规范 |
|------|------|
| **ID格式** | `TC.{L1}.{L2}`（自然数，如TC.1.1） |
| **表名规范** | `<database>.dbo.<table>`（如gold.dbo.dm_table） |
| **SQL结构** | 四段式：[Coverage]→[Cardinality]→[Value Diff]→[Conclusion] |
| **SQL平台** | Synapse Serverless SQL Pool（禁止temp table/SP） |
| **Excel写入** | **SIT Case** Sheet（非Change History） |
| **Procedure** | ≤3步骤 |
| **字段场景** | 单字段≤2场景，简单字段=1场景 |

---

## 占位符规范

当信息缺失时，必须使用占位符：

| 缺失信息 | 占位符 |
|----------|--------|
| 表名未知 | `<TBD_TABLE>` |
| 字段名未知 | `<TBD_FIELD>` |
| 主键未知 | `<TBD_PK>` |
| 日期未知 | `<TBD_CUTOFF_DATE>` |
| 逻辑未知 | `<TBD_LOGIC>` |

---

## 版本选择（Step 4 阻断点）

> **Agent**: "已提取项目信息，准备生成测试用例。请选择版本：
> 1) 仅英文  2) 仅中文  3) 中英文？"

**阻断规则**：
- ❌ 禁止假设默认值
- ❌ 禁止自动继续
- ✅ 必须等待用户明确回复"1/2/3"或"英文/中文/中英文"


#### 用户确认变更点清单（Step 5 阻断点）

展示方式：以结构化Markdown表格/YAML形式向用户展示上述Change Points清单。
确认话术模板：
Agent: "已完成代码分析，识别以下关键变更点（基于代码对比，未参考change log）：
[展示Change Points清单]
请确认：
上述变更点是否完整准确？是否有遗漏或误判？
是否有特定字段需要额外关注或排除？
确认无误后，我将基于这些变更点生成测试用例。"
选项：
   回复"确认" / "Proceed" → 进入第3章Analysis Layer生成测试用例
   回复"补充：[具体字段/变更]" → 更新Change Points清单，再次展示确认
   回复"排除：[具体字段]" → 从清单移除，说明原因后再次确认
   回复"停止" / "Stop" → 暂停生成，等待用户补充信息

** 阻断规则：
在收到用户明确确认（如"确认"、"Proceed"、"Go"）前，禁止进入第3章生成测试用例
若用户提出修正，更新Change Points后需再次展示完整清单供二次确认
所有确认记录需在Change History中备注用户反馈摘要
---

## 交付物清单

根据版本选择结果：

| 版本 | 交付物 |
|------|--------|
| 1 (EN) | `.xlsx` + `.md` (英文) |
| 2 (CN) | `_CN.xlsx` + `_CN.md` (中文) |
| 3 (双语) | `.xlsx` + `_CN.xlsx` + `.md` + `_CN.md` |

**命名格式**：
- Excel: `TKE IT Project SIT Test Case_{Project}_ITCM{number}.xlsx`
- Markdown: `TKE_IT_Project_SIT_Test_Case_{Project}_ITCM{number}.md`
