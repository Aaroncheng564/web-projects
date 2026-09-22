# CLAUDE.md

本文件是本仓库的行为准则与工作约定。领域术语见 `CONTEXT.md`，架构决策见 `docs/adr/`。

## 项目背景

**ChemAI（智辅化学）** 是一个 AI 驱动的高中化学教学辅助平台。

**目标用户**：中国高中化学教师、学生及家长。设计文档只覆盖高中，不含初中——裁决见 [ADR-0004](docs/adr/0004-目标学段为高中.md)。

**核心功能**：

- AI Agent 对话系统
- 出题工作台
- 四维审核引擎
- 障碍诊断引擎
- 题库管理与考试生命周期
- 学生练习与错题本
- 家长端

**技术栈**：

| 层 | 选型 |
| --- | --- |
| 语言 | Python 3.11 |
| Web 框架 | FastAPI 0.109.2 + Uvicorn 0.27.1 |
| ORM / 迁移 | SQLAlchemy 2.0.25 + Alembic 1.13.1 |
| 数据库 | SQLite（WAL 模式，开发）/ MySQL（生产可选）。三个库：主库、Agent 检查点库、Agent 长期记忆库 |
| 向量库 | ChromaDB 0.4.22 + DashScope `text-embedding-v3`（1024 维） |
| AI 编排 | LangGraph `create_react_agent`（单 Agent ReAct） |
| 模型 | 三级 Fallback：MiMo-V2.5 → 通义千问 `qwen-turbo` → DeepSeek-V4-Flash |
| OCR | 三引擎降级链：百度 AI 教育 OCR → Qwen-VL-OCR → 腾讯 OCR |
| 前端 | Vanilla JS + Vite + Tailwind CSS CDN + KaTeX + Marked；Vue 3 CDN 仅限出题工作台 |
| 化学配平 | 自定义系数平衡算法 + RDKit 2024.3.1 |
| 桌面打包 | pywebview + PyInstaller 6.x |
| 调度 | APScheduler 3.10.4 |
| 测试 / CI | Pytest 8.0.0 / GitHub Actions |

代码根目录为 `chemai-backend/`。前端设计规格见课程资料中的 `DESIGN.md`（36 号设计系统 + 40 号页面规格浓缩而成），生成页面前必须先读。

## 行为准则

### 1. 先思考再编码

开始实现前明确陈述假设。假设错了，后面写的所有代码都是废的。

- 动手前说明：要做什么、影响哪些文件、依赖哪些前提
- 不确定就发问，不要靠猜补齐需求
- 有多种合理解法时，先给推荐方案和理由，而不是闷头选一种

### 2. 简单优先

只写解决问题所需的最少代码。

- 不做没被要求的功能、抽象和配置项
- 不为「以后可能要」预留扩展点
- 三行能解决的，不写一个类

### 3. 手术式修改

只触及必须修改的代码，匹配已有风格。

- 不顺手重构无关代码，不顺手改格式
- 新代码的命名、注释密度、错误处理方式向周围代码看齐
- 改不动的地方说明原因，不要绕过

### 4. 目标驱动执行

把任务转化为可验证的目标。

- 动手前明确「怎么算做完了」，最好是一条能跑的命令或一个能断言的测试
- 先写测试再实现（见下方「实现层」）
- 如实报告结果：测试没过就说没过并贴出输出，跳过的步骤要讲明

## 项目特定规范

- **语言**：所有代码注释和文档使用中文
- **代码风格**：Python 代码遵循 PEP 8
- **数据库**：数据模型使用 SQLAlchemy ORM 定义
- **API**：遵循 RESTful 设计
- **开发方式**：使用 TDD（测试驱动开发）
- **化学式渲染**：所有化学式、离子式、方程式必须经 KaTeX + mhchem 渲染——用 `$...$` 包裹并以 `\ce{}` 书写（如 `$\ce{Fe^3+ + e- -> Fe^2+}$`），禁止输出 `H2O`、`SO4^2-` 这类裸文本。后端返回的化学式视为已格式化，前端不再二次转换
- **前端样式**：一律用 Tailwind CDN 的工具类内联写在 HTML 中，**零构建步骤**，禁止引入需要编译的 CSS（见 [ADR-0005](docs/adr/0005-前端样式一律走-Tailwind-CDN.md)）。生成页面前必须先读 `DESIGN.md`
- **内容安全**：所有 AI 生成内容须经四维安全审核引擎校验后方可输出，`blocked` 的内容打回重生成、不得输出

## 标准开发流程（工具链）

分层推进，每层有对应工具。不要跳过思考层直接写代码。

| 层 | 做什么 | 工具链 |
| --- | --- | --- |
| 思考层 | 想清楚要解决什么问题 | `/office-hours` → `/plan-ceo-review` → `/plan-eng-review`（Gstack） |
| 规格层 | 把想法固化成可审阅的规格 | `/opsx:propose` → `/opsx:apply` → `/opsx:archive`（OpenSpec） |
| 实现层 | 测试驱动落地 | 写测试（RED）→ Claude 生成（GREEN）→ 重构（TDD） |
| 质量层 | 验证 AI 行为质量 | L1 单元 → L2 集成 → L3 质量（Evals，见下） |
| 流程层 | 评审、发布、复盘 | `/review` `/cso` `/qa` `/ship` `/retro` `/investigate`（Gstack） |
| 骨架层 | 版本控制 | branch / commit / tag / merge / revert / push（Git） |

**思考层何时跳过**：产品设计文档（20–43 号）已把功能定义清楚，所以大部分阶段跳过 `/office-hours` `/plan-ceo-review` 直接进规格层；只有架构选型存在不确定性时才用 `/plan-eng-review`（本项目仅在 Agent 架构阶段保留）。已决策的事不反复讨论。

### 质量标准（Evals）

**实现依据是 32 号《评测体系设计》，不是 00 号的口径**——两者不兼容，裁决见 [ADR-0002](docs/adr/0002-评测分层以设计文档为准.md)。

| 层级 | 判定方式 | 触发频率 | 场景数 |
| --- | --- | --- | --- |
| 基线层 | 代码可精确断言（对/错二值） | 每次 commit | 60 |
| 边界层 | 多数可精确断言，少数需语义判断 | 每次 commit | 24 |
| 回归层 | 精确断言 + LLM 评分 | 每次 PR / 发版前 | 25 |

通过标准按门禁优先级而非按层：安全层 **100%，不可降级**；路由层 ≥85%；质量层 ≥3.5/5 平均分；性能层 P95 达标。基线通过率目标 ≥95%。

**Golden 数据集 = 109 个评测场景**的完整定义（输入、预期输出、评估标准），按三层组织。它是数据资产，**不是第三层**——不要写成「L3 Golden」。同一批场景另有两条切法（**不要用「正交」**，见 `CONTEXT.md` 已废弃术语表）：确定性 92 场景（pytest）+ LLM-as-Judge 17 场景。

劣化处理：≤3% 记录 devlog 后继续；3–5% 触发 `/investigate` 定位 golden 样本并修复；>5% 回退到 bad commit 或直接修复，两种情况都要重跑 Evals 确认恢复。

> 00 号 §3.2 另有「L1 单元 / L2 集成 / L3 质量（≥95% / ≥90% / ≥70%）」一套口径，那是**教程叙述**。它的 L2「API 端点层」在 32 号中无对应层，L3 的「辅导安全性」在 32 号中属基线层且阈值 100%。**不要拿它的阈值做门禁。**

### Git 约定

- commit message 用 Conventional Commits：`{type}({scope}): {description}`
  - type：`feat` / `fix` / `test` / `refactor` / `docs` / `chore` / `perf`
  - scope：`model` / `auth` / `review` / `exam` / `diagnosis` / `ocr` / `agent` / `eval` / `persona` / `guard` / `ui` / `golden-NNN`
- **每个模型或组件完成即提交**，不攒批
- 门禁：pre-commit 跑基线层 + 边界层（纯逻辑、<5 秒）；commit-msg 校验上述格式；pre-push 跑全量 Evals 并对比基线快照，劣化 >5% 阻断（快照文件名待阶段五确定，00 号暂用 `baseline.json`）

## Agent skills

### Issue tracker

Issues and specs live as markdown files under `.scratch/<feature-slug>/` in this repo (local-markdown tracker — no remote, no `gh` CLI needed). See `docs/agents/issue-tracker.md`.

### Triage labels

Five canonical triage roles, using the default label strings. See `docs/agents/triage-labels.md`.

### Domain docs

Single-context: one `CONTEXT.md` plus `docs/adr/` at the repo root. See `docs/agents/domain.md`.
