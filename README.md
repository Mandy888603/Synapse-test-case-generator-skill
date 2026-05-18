# Synapse SIT Test Case Generator Skill

<div align="center">

**TKE Synapse Serverless SIT/UAT 自动测试用例生成器**

[![Version](https://img.shields.io/badge/version-2.6-blue.svg)](./SKILL.md)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)
[![Platform](https://img.shields.io/badge/platform-Azure%20Synapse-orange.svg)](https://azure.microsoft.com/en-us/products/synapse-analytics/)

[English](#english) | [中文](#chinese)

</div>

---

<a name="chinese"></a>

## 📋 项目概述

这是一个用于 TKE Synapse Serverless 数据平台的 **AI 驱动测试用例自动生成工具**。它能够根据功能规格文档（FS/TS）和 Notebook 代码自动生成符合 TKE IT 模板格式的专业 SIT/UAT 测试用例。

### 核心价值

- ✅ **自动化生成**：从需求到测试用例全流程自动化
- 🎯 **高质量输出**：符合 TKE IT 标准模板格式
- 🔍 **智能分析**：自动识别变更点、业务规则和数据流
- 📊 **多格式支持**：生成 Excel 和 Markdown 格式
- 🌐 **双语支持**：英文/中文/双语测试用例
- 🚀 **项目类型智能识别**：New Development / Enhancement 自动判定

---

## 🎯 主要功能

### 1. 智能文档解析
- 自动提取 FS/TS 文档中的业务规则和数据字典
- 大文档分页处理策略（>50段落或>500KB）
- 主键字段自动识别

### 2. 代码分析
- **Enhancement 项目**：新旧 Notebook 代码对比，精准定位变更点
- **New Development**：全量代码提取和逻辑分析
- 多数据层支持（DM → DWT → Staging → Ingress）

### 3. 测试用例生成
- **四段式 SQL 验证**：Coverage → Cardinality → Value Diff → Conclusion
- **Excel 输出**：符合 TKE IT 模板格式，自动格式化
- **Markdown 输出**：条件触发，便于审阅
- **质量门禁**：9 项强制检查清单

### 4. 分层测试策略
```
DM 层 (gold.dbo.dm_*)     → 优先级最高
   ↓
DWT 层 (gold.dbo.dwt_*)   → 次优先级
   ↓
Staging 层                → 中间层
   ↓
Ingress 层                → 最底层
```

---

## 📁 文件结构

```
openspec-generate-testcase-document - Synapse/
│
├── README.md                          # 本文档
├── SKILL.md                           # Skill 定义和快速入门
├── index.md                           # 文档地图和完整决策树
│
├── rules/                             # 核心规则
│   ├── rule-a.md                     # 核心原则与硬性约束
│   ├── rule-b.md                     # 输入提取规则
│   └── rule-c.md                     # 分析决策规则
│
├── fs/                               # 功能规格处理
│   ├── fs-core.md                    # FS/TS 文档提取核心逻辑
│   └── fs-edge.md                    # 大文档处理策略
│
├── notebooks/                        # Notebook 处理
│   ├── nb-extraction.md              # Notebook 提取规则
│   └── nb-diff.md                    # 代码对比规则（Enhancement）
│
├── output/                           # 输出生成
│   ├── output-excel.md               # Excel 生成规范
│   └── output-sql.md                 # SQL 编写规范
│
└── quality/                          # 质量控制
    └── quality-check.md              # 质量控制清单（9 项）
```

---

## 🚀 快速开始

### 前置要求

确保您的工作目录包含以下文件：

#### 必需文件
1. **功能规格文档**（FS/TS）
   - 文件名格式：`TKE_IT_Template_Functional_Requirement_Specification_*_ITCM*.docx`
   - 包含：业务规则、数据字典、字段映射

2. **测试用例模板**
   - 文件名格式：`TKE IT Project SIT Test Case - template_ITCMXXXXX.xlsx`
   - 用途：提取项目信息和基础格式

3. **新版本 Notebook**
   - 文件名格式：`[notebook_name]_new.ipynb`
   - 包含：最新的业务逻辑代码

#### 可选文件（Enhancement 项目）
4. **旧版本 Notebook**
   - 文件名格式：`[notebook_name]_old.ipynb`
   - 用途：代码对比，识别变更点

### 执行流程（九步导航）

```
┌─────────────────────────────────────────────────────────────┐
│ Step 1: 扫描目录 → 检测文件存在性                             │
│         验证 FS/TS, Template, Notebook 文件                  │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 2: 加载模板 → 强制加载 template.xlsx                     │
│         提取项目信息（ITCM号、项目名称、PM、环境）              │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 3: 提取 FS/TS → 业务规则提取                            │
│         大文档触发分页策略（>50段落或>500KB）                  │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 3b: 主键识别 → 从代码/注释推断 Primary Key              │
│         提取 GROUP BY, DISTINCT, JOIN 字段                    │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 4: 版本确认 → 🛑 阻断点：用户选择                        │
│         1) English Only  2) Chinese Only  3) Bilingual       │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 5: Change Points → 变更点识别                           │
│         Enhancement: 代码对比 | New Dev: 全量提取            │
│         🛑 阻断点：确认测试范围                                │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 6: Field Inventory → 整合多源数据                        │
│         合并 FS数据字典 + Notebook字段 + PK字段                │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 7: 分析决策 → 分层策略 + 场景设计                        │
│         应用 DM→DWT→Staging 优先级规则                        │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 8: 输出生成 → Excel + Markdown（条件触发）              │
│         应用 TKE IT 模板格式规范                             │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 9: 质量门禁 → 强制检查清单                              │
│         9 项检查：格式、SQL、逻辑、完整性                      │
└─────────────────────────────────────────────────────────────┘
```

### 项目类型自动判定

```python
if old_notebook.exists():
    project_type = "Enhancement"      # 代码对比流程
    workflow = "新旧代码对比 → 变更点识别 → 增量测试"
else:
    project_type = "New Development"  # 全量提取流程
    workflow = "完整代码提取 → 全量逻辑分析 → 完整测试"
```

---

## 📖 文档导航

| 文件 | 用途 | 加载时机 |
|------|------|----------|
| [SKILL.md](./SKILL.md) | Skill 定义和快速入门 | 始终加载 |
| [index.md](./index.md) | 文档地图/完整决策树 | 始终加载 |
| [rules/rule-a.md](./rules/rule-a.md) | 核心原则与硬性约束 | Step 1 |
| [rules/rule-b.md](./rules/rule-b.md) | 输入提取规则 | Step 3-6 |
| [rules/rule-c.md](./rules/rule-c.md) | 分析决策规则 | Step 7 |
| [fs/fs-core.md](./fs/fs-core.md) | FS/TS 文档提取核心 | Step 3 |
| [fs/fs-edge.md](./fs/fs-edge.md) | 大文档处理策略 | Step 3（大文档） |
| [notebooks/nb-extraction.md](./notebooks/nb-extraction.md) | Notebook 提取规则 | Step 5-6 |
| [notebooks/nb-diff.md](./notebooks/nb-diff.md) | 代码对比规则 | Step 5（Enhancement） |
| [output/output-sql.md](./output/output-sql.md) | SQL 编写规范 | Step 8 |
| [output/output-excel.md](./output/output-excel.md) | Excel 生成规范 | Step 8 |
| [quality/quality-check.md](./quality/quality-check.md) | 质量控制清单 | Step 9 |

---

## 🎓 使用示例

### 示例 1: New Development 项目

**输入文件：**
```
openspec/changes/ITCM123456/
├── TKE_IT_Template_Functional_Requirement_Specification_CallPerformanceReport_ITCM123456.docx
├── TKE IT Project SIT Test Case - template_ITCM123456.xlsx
└── call_performance_report_new.ipynb
```

**生成输出：**
```
openspec/changes/ITCM123456/output/
├── TKE_IT_SIT_Test_Cases_ITCM123456.xlsx       # Excel 测试用例
└── Test_Cases_Preview_ITCM123456.md             # Markdown 预览（可选）
```

### 示例 2: Enhancement 项目

**输入文件：**
```
openspec/changes/ITCM789012/
├── TKE_IT_Template_Functional_Requirement_Specification_DataQuality_ITCM789012.docx
├── TKE IT Project SIT Test Case - template_ITCM789012.xlsx
├── data_quality_report_old.ipynb  # 旧版本
└── data_quality_report_new.ipynb  # 新版本
```

**处理流程：**
1. 自动对比新旧代码
2. 识别变更的表、字段、逻辑
3. 生成针对变更点的测试用例
4. 输出 Excel 和 Markdown

---

## 🔧 核心规范

### 测试用例 ID 格式
```
格式：TC.{L1}.{L2}
示例：TC.1.1, TC.2.3, TC.10.5

规则：
✅ 使用自然数（1, 2, 3...）
❌ 禁止前导零（TC.1.01）
❌ 禁止三级编号（TC.1.1.1）
```

### 表名规范
```
格式：<database>.dbo.<table>
示例：gold.dbo.dm_call_performance

有效数据库：
- gold (DM, DWT 层)
- silver
- prepared
- staging
- ingress
- curated
```

### SQL 验证结构（四段式）
```sql
-- 1. [Coverage] 覆盖范围验证
SELECT COUNT(*) as total_records
FROM gold.dbo.dm_table
WHERE date >= '2024-01-01';

-- 2. [Cardinality] 基数验证
SELECT field1, COUNT(*) as record_count
FROM gold.dbo.dm_table
GROUP BY field1
HAVING COUNT(*) > 1;

-- 3. [Value Diff] 值差异验证
SELECT field1, field2
FROM gold.dbo.dm_table
WHERE field2 IS NULL OR field2 < 0;

-- 4. [Conclusion] 结论
-- Expected: All records should have valid values
```

---

## 📊 质量控制

### 9 项强制检查清单

1. ✅ **ID 格式检查**：所有 TC ID 符合 `TC.{L1}.{L2}` 格式
2. ✅ **表名格式检查**：所有表名包含 `database.dbo.table`
3. ✅ **SQL 结构检查**：包含四段式标记
4. ✅ **主键一致性**：FS 与代码 PK 字段一致
5. ✅ **测试层级检查**：遵循 DM→DWT→Staging 优先级
6. ✅ **双语一致性**：英中文描述对应
7. ✅ **Excel 格式检查**：正确的 Sheet、表头位置、格式
8. ✅ **步骤精简性**：Procedure ≤ 3 步
9. ✅ **完整性检查**：所有必填字段已填写

---

## 🌟 高级特性

### 1. 大文档处理策略
当 FS/TS 文档 >50 段落或 >500KB 时：
- 自动触发分页提取
- 先提取目录和结构
- 用户确认后分批加载详细内容

### 2. 智能主键识别
自动从以下位置识别主键：
- Notebook 代码中的 `GROUP BY` 子句
- `DISTINCT` 字段列表
- `JOIN ON` 条件
- 代码注释中的 PK 标记

### 3. 条件化 Markdown 输出
自动判断是否生成 Markdown 预览：
- 用例数量 > 10：生成预览便于审阅
- 复杂 SQL 查询：生成以便调试
- 用户明确请求

---

## ❓ 常见问题

<details>
<summary><b>Q1: 如何判断项目是 Enhancement 还是 New Development？</b></summary>

**A:** 系统自动判定：
- 如果存在 `*_old.ipynb` 文件 → Enhancement
- 如果只有 `*_new.ipynb` 文件 → New Development
</details>

<details>
<summary><b>Q2: 为什么需要两个阻断点？</b></summary>

**A:** 
- **阻断点 1（Step 4）**：确认测试用例语言版本，影响后续所有生成内容
- **阻断点 2（Step 5）**：确认测试范围，避免生成不必要的测试用例
</details>

<details>
<summary><b>Q3: SQL 为什么必须是四段式？</b></summary>

**A:** 四段式结构确保测试的完整性：
1. **Coverage**：验证数据覆盖范围
2. **Cardinality**：验证数据基数（唯一性/重复性）
3. **Value Diff**：验证数据值的正确性
4. **Conclusion**：明确预期结果
</details>

<details>
<summary><b>Q4: 如何处理多层数据流（Ingress → Staging → Gold）？</b></summary>

**A:** 系统按优先级自动选择测试层级：
1. 优先测试 **DM 层**（如果存在且与 DWT 逻辑不一致）
2. 其次测试 **DWT 层**
3. 必要时测试 **Staging** 和 **Ingress** 层
</details>

<details>
<summary><b>Q5: Excel 输出时为什么不能写入默认 Sheet？</b></summary>

**A:** TKE IT 模板要求：
- 项目信息必须写入 **SIT Case** Sheet
- 表头位置固定在行 3-6，列 E
- 使用 `wb.active` 可能指向错误的 Sheet
</details>

---

## 📌 技术规格

| 项目 | 规格 |
|------|------|
| **版本** | v2.6 |
| **平台** | Azure Synapse Serverless SQL Pool |
| **输入格式** | DOCX (FS/TS), XLSX (Template), IPYNB (Notebook) |
| **输出格式** | XLSX (Test Cases), MD (Preview) |
| **支持语言** | English, Chinese, Bilingual |
| **数据层** | Ingress, Staging, Silver, Gold (DWT/DM) |
| **项目类型** | New Development, Enhancement |

---

## 🤝 贡献

本 Skill 由 **TKE DA IT Team** 开发和维护。

如有问题或建议，请联系：
- **作者**: DA IT Team
- **版本**: 1.0
- **生成工具**: openspec CLI 1.2.0

---

## 📜 许可证

MIT License

---

<a name="english"></a>

## 📋 Project Overview (English)

This is an **AI-powered automated test case generator** for TKE Synapse Serverless data platform. It automatically generates professional SIT/UAT test cases compliant with TKE IT template format based on Functional Specifications (FS/TS) and Notebook code.

### Key Features

- ✅ **End-to-end Automation**: From requirements to test cases
- 🎯 **High Quality Output**: Compliant with TKE IT standard template
- 🔍 **Intelligent Analysis**: Auto-identify change points, business rules, and data flows
- 📊 **Multi-format Support**: Excel and Markdown outputs
- 🌐 **Bilingual Support**: English/Chinese/Bilingual test cases
- 🚀 **Smart Project Type Detection**: New Development / Enhancement auto-detection

### Core Workflow

```
Step 1: Scan Directory → Validate file existence
Step 2: Load Template → Extract project metadata
Step 3: Extract FS/TS → Parse business rules (large doc pagination)
Step 3b: Identify Primary Keys → Infer from code/comments
Step 4: Version Selection → 🛑 User selects language (EN/CN/Bilingual)
Step 5: Change Points → Code diff (Enhancement) or full extraction (New Dev)
Step 6: Field Inventory → Consolidate multi-source data
Step 7: Analysis & Decision → Layer strategy + scenario design
Step 8: Output Generation → Excel + Markdown (conditional)
Step 9: Quality Gate → 9-item mandatory checklist
```

### Project Type Detection

```python
if old_notebook.exists():
    project_type = "Enhancement"      # Code diff workflow
else:
    project_type = "New Development"  # Full extraction workflow
```

### Data Layer Priority

```
DM Layer (gold.dbo.dm_*)     → Highest priority
   ↓
DWT Layer (gold.dbo.dwt_*)   → Secondary priority
   ↓
Staging Layer                → Middle layer
   ↓
Ingress Layer                → Base layer
```

### Quality Checklist (9 Items)

1. ✅ **ID Format**: All TC IDs follow `TC.{L1}.{L2}` pattern
2. ✅ **Table Naming**: All tables include `database.dbo.table`
3. ✅ **SQL Structure**: Contains 4-section markers
4. ✅ **PK Consistency**: FS and code PK fields aligned
5. ✅ **Layer Testing**: Follows DM→DWT→Staging priority
6. ✅ **Bilingual Consistency**: EN/CN descriptions match
7. ✅ **Excel Format**: Correct sheet, header position, formatting
8. ✅ **Step Conciseness**: Procedure ≤ 3 steps
9. ✅ **Completeness**: All required fields populated

---

## 🔗 Quick Links

- 📚 [SKILL Definition](./SKILL.md)
- 🗺️ [Document Map](./index.md)
- 📋 [Core Rules](./rules/rule-a.md)
- 🔍 [Quality Checklist](./quality/quality-check.md)

---

<div align="center">

**Made with ❤️ by TKE DA IT Team**

</div>
