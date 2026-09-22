# CLAUDE.md

本文件是本仓库的行为准则与工作约定。领域术语见 `CONTEXT.md`，架构决策见 `docs/adr/`。

## 项目背景

**ChemAI（智辅化学）** 是一个 AI 驱动的中学化学教学辅助平台。

**目标用户**：中国初中、高中化学教师、学生及家长。

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
- **内容安全**：所有 AI 生成内容须经四维安全审核引擎校验后方可输出，`blocked` 的内容打回重生成、不得输出

## 标准开发流程（工具链）

分层推进，每层有对应工具。不要跳过思考层直接写代码。

| 层 | 做什么 | 工具链 |
| --- | --- | --- |
| 思考层 | 想清楚要解决什么问题 | `/office-hours` → `/plan-ceo-review` → `/plan-eng-review`（Gstack） |
| 规格层 | 把想法固化成可审阅的规格 | `/opsx:propose` → `/opsx:apply` → `/opsx:archive`（OpenSpec） |
| 实现层 | 测试驱动落地 | 写测试（RED）→ Claude 生成（GREEN）→ 重构（TDD） |
| 质量层 | 验证 AI 行为质量 | L1 单元 → L2 集成 → L3 Golden → 基线对比（Evals） |
| 流程层 | 评审、发布、复盘 | `/review` `/cso` `/qa` `/ship` `/retro` `/investigate`（Gstack） |
| 骨架层 | 版本控制 | branch / commit / tag / merge / revert / push（Git） |

**思考层何时跳过**：产品设计文档（20–43 号）已把功能定义清楚，所以大部分阶段跳过 `/office-hours` `/plan-ceo-review` 直接进规格层；只有架构选型存在不确定性时才用 `/plan-eng-review`（本项目仅在 Agent 架构阶段保留）。已决策的事不反复讨论。

### 质量标准（Evals）

| 层级 | 内容 | 通过标准 | 运行时机 |
| --- | --- | --- | --- |
| L1 单元 | 纯函数逻辑（规则引擎、置信度、状态机） | ≥95% | pre-commit hook |
| L2 集成 | API 端点行为（结构、状态码、认证） | ≥90% | 每个工具组完成 |
| L3 质量 | AI 内容质量（科学性、诊断准确率、辅导安全性） | ≥70% | 每日收工、push 前 |

劣化处理：≤3% 记录 devlog 后继续；3–5% 触发 `/investigate` 定位 golden 样本并修复；>5% 回退到 bad commit 或直接修复，两种情况都要重跑 Evals 确认恢复。全量 Evals 只在三个时机跑：工具组完成、每日收工、push 前——单个工具开发时只跑该工具的 pytest。

### Git 约定

- commit message 用 Conventional Commits：`{type}({scope}): {description}`
  - type：`feat` / `fix` / `test` / `refactor` / `docs` / `chore` / `perf`
  - scope：`model` / `auth` / `review` / `exam` / `diagnosis` / `ocr` / `agent` / `eval` / `persona` / `guard` / `ui` / `golden-NNN`
- **每个模型或组件完成即提交**，不攒批
- 门禁：pre-commit 跑 L1 单元（<5 秒）；commit-msg 校验上述格式；pre-push 跑全量 Evals 并对比 `baseline.json`，劣化 >5% 阻断

## Agent skills

### Issue tracker

Issues and specs live as markdown files under `.scratch/<feature-slug>/` in this repo (local-markdown tracker — no remote, no `gh` CLI needed). See `docs/agents/issue-tracker.md`.

### Triage labels

Five canonical triage roles, using the default label strings. See `docs/agents/triage-labels.md`.

### Domain docs

Single-context: one `CONTEXT.md` plus `docs/adr/` at the repo root. See `docs/agents/domain.md`.
