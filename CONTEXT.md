# CONTEXT.md — ChemAI 领域词汇表

本文件定义 ChemAI 的核心领域术语。写 issue、提方案、起测试名、命名代码符号时，一律使用本表定义的术语，不要漂移到同义词。

**依据**：ChemAI 产品设计文档（编号 20–43），位于课程资料 `L3阶段/chemAI/产品设计/`。下文的「N 号文档」即指该目录下 `N-*.md`。凡本表标注「文档未定义」的术语，表示该词在设计文档中查无出处，使用前需先确认。

## 核心实体

依据：34 号《数据模型设计》§一 实体关系总览（自述「实体共 23 个，枚举类型 5 个」）。

| 中文 | English | 定义 |
| --- | --- | --- |
| 学校 | School | 组织链顶层。多租户隔离以 `school_id` 为单位。 |
| 年级 | Grade | 学校内的组织层级（School 包含多个 Grade）。**不是学段层级（初一/高一），也不是「成绩」**。 |
| 班级 | Class | 年级下的教学单位（Grade 包含多个 Class）。 |
| 学生 | Student | 学习主体，经 Account 登录，用 `status` 字段表示审批状态。 |
| 教师 | Teacher | 教学主体。**admin / 教务管理员 / 学科组长 / teacher 四种角色共用 Teacher 数据模型**，靠 `role` 字段区分（23 号文档）。 |
| 家长 | Parent | 经绑定关系（`student_id + bind_code + parent_id`）查看子女数据的角色，走独立认证通道（`phone + bind_code`），不携带 `school_id`。 |
| 账号 | Account | 统一账户。Teacher / Student / Parent 各与一个 Account 一对一关联。 |

组织链：`School → Grade → Class → Student`。

## 角色体系

依据：23 号《权限分级与访问控制系统设计》§一。完整角色清单（逐字）：

| 角色 | 权限范围 |
| --- | --- |
| `admin`（系统管理员） | 全局系统管理、跨校数据 |
| 教务管理员 | 本校教务管理、师生账户管理 |
| 学科组长 | 本校化学科只读权限、数据分析 |
| `teacher`（教师） | 本人班级教学、考试、诊断 |
| `student`（学生） | 本人学习数据、答题练习 |
| `parent`（家长） | 绑定子女的学习数据（独立认证通道） |

注意：23 号文档自身在「五角色体系」与「六角色体系」的表述上前后不一致（正文写「5 种用户角色 + 家长端独立认证」，表格列 6 行）。**不存在 `super_admin`**。`admin` / 教务管理员 / 学科组长不带英文标识，其 `role` 字面量就是中文。

## 学习与诊断

| 中文 | English | 定义 |
| --- | --- | --- |
| 障碍类型 | `BarrierType` | 学生化学学习障碍的三个维度，枚举取值 `concept` 概念理解型 / `reading` 审题障碍型 / `expression` 表述障碍型（27 号文档 §2.1）。 |
| 障碍分布 | （字段名 `barrier_distribution`） | 学生画像中的三维障碍权重，如 `concept:0.7 / reading:0.15 / expression:0.15`；班级级为 `class_barrier_distribution`，三个键各对应整数人数。 |
| 知识点 | Knowledge Point | 知识图谱节点（34 号实体 `KnowledgePoint`）。25 号 §9.1 按 `category` 分 8 类：电解质溶液 / 离子反应 / 氧化还原反应 / 电化学 / 化学计量 / 物质结构 / 化学反应速率与平衡 / 有机化学。 |
| 诊断 | Diagnosis | **根因诊断**——从「知道谁错了」进化到「知道为什么错」（27 号 §1.1）。 |
| 自适应练习引擎 | Adaptive Practice Engine | 消费诊断结果，为学生生成处于最近发展区（ZPD）的个性化练习题（28 号 §一）。 |
| 最近发展区 | Zone of Proximal Development, ZPD | 难度略高于学生当前水平但不致挫败的区间（28 号 §一）。 |
| 间隔重复复习 | Spaced Repetition | 答错后自动创建复习任务，按艾宾浩斯曲线定时提醒（29 号）。 |
| 错题本 | — | 错题强化训练的入口，学生主动选择错题集中训练（29 号 §7.1）。 |
| 学情预警 | — | 由 `EarlyWarningService` 承载的三层预警引擎（31 号 §六）。 |

三个障碍维度的关系，文档原文是「**这三个维度相互独立**」（27 号 §2.2），**不是「正交」**——「正交」二字在设计文档中从未出现。

## 内容实体

| 中文 | English | 定义 |
| --- | --- | --- |
| 题目 | Question | 最小内容单元。34 号实体。 |
| 试卷 | `ExamPaper` | 25 号中的**历史真题试卷对象**（含 `paper_id` / `source` / `year` / `region` / `total_score` / `questions`）。注意与「考试」是两个实体。 |
| 考试 | `ExamRecord` | 可发布、可作答的考试记录（34 号实体）。 |
| 题库文件夹 | `QuestionSet` | 顶层组织单元，教师可建多个文件夹分类管理题目（25 号 §5.2）。其下挂 `QuestionSetItem` 引用具体题目。 |
| 历年真题库 | `HistoricalExam` | 34 号实体，`HistoricalExam }o--|| Question : 关联`。 |
| 周报 | — | 面向家长推送的周期性学习总结，每周一 8:00 自动生成，同一周每生仅一份（33 号 §七）。 |

**`QuestionSet` 已裁决**：34 号 ER 图标注为「真题集」，25 号 §5.2 定义为「题库文件夹」。按 [ADR-0001](docs/adr/0001-设计文档冲突的裁决优先级.md)「题库概念以 25 号为主责文档」，取**「题库文件夹」**，34 号的「真题集」视为笔误。不要回写 34 号，也不要再开一次这个讨论。

## 题目与考试

| 中文 | English | 定义 |
| --- | --- | --- |
| 题目类型 | Question Type | 枚举取值 **5 种**：`choice` 选择 / `fill` 填空 / `calc` 计算 / `experiment` 实验 / `inference` 推断（25 号 §3.1.1）。 |
| 难度 | Difficulty | 枚举取值 `easy` / `medium` / `hard` / `competition`，默认 `medium`；34 号枚举表记为「简单/中等/困难/竞赛」。**`competition` 仅支持手动录入**（25 号 §一）。知识点的难度只有 `easy` / `medium` / `hard` 三档。 |
| 考试类型 | — | 枚举：月考 / 练习 / 作业（34 号枚举表）。 |
| 题目来源 | — | 枚举：AI 生成 / 手动录入 / 日常练习 / OCR 导入（34 号枚举表）。 |
| 题目审核状态 | — | 枚举：通过 / 警告可用 / 阻断不可用（34 号枚举表）。 |
| 四维安全审核引擎 | — | AI 生成化学内容的强制校验引擎（26 号文档全篇）。 |
| 考试状态 | Exam State | 状态机：`Draft` → `AddingQuestions` → `Published` → `InProgress` → `Completed`（25 号 §6.1）。 |

**前端展示的枚举粒度与数据模型不同，不要混用**（依据 `DESIGN.md`）：

- 考试状态数据模型 5 个（含 `AddingQuestions`），前端状态徽章只展示 4 个：草稿 / 已发布 / 进行中 / 已完成。
- 题目难度数据模型 4 档（含 `competition`），前端难度标签只展示 3 档：简单 / 中等 / 困难。
- 障碍类型前端标签配色为**平行无等级**：概念理解型（紫）/ 审题障碍型（蓝）/ 表述障碍型（青）——不要做成递进色阶。

### 四维安全审核的四个维度

依据：26 号文档各章标题（**不是**「科学性/难度匹配/知识点覆盖/区分度」）：

| 维度 | English | 可能状态 |
| --- | --- | --- |
| 系数配平审核 | Coefficient Balancing | `passed` / `blocked` |
| 反应条件审核 | Reaction Conditions | `passed` / `warning` / `failed` |
| 产物稳定性审核 | Product Stability | `passed` / `warning` / `failed` |
| 分子结构审核 | Structure Check | `passed` / `failed` |

综合状态 `overall_status` 只有 `passed` / `blocked`；任一维度被拦截即整体 `blocked`。语义：`block` 打回重生成、不可输出；`warning` 标记但不拦截、仍可输出（26 号 §6.3）。**没有 `pending`（待审）状态。**

## 评测体系

依据：32 号《评测体系设计》。**不要用 00 号 §3.2 的 L1/L2/L3 口径**——两者不兼容，裁决见 [ADR-0002](docs/adr/0002-评测分层以设计文档为准.md)。

| 中文 | English | 定义 |
| --- | --- | --- |
| 基线层 | Baseline | 三层金字塔底层。代码可精确断言、零 API 调用，每次 commit 跑 60 个场景。 |
| 边界层 | Boundary | 三层中层。异常输入与降级路径，可 mock 外部服务，每次 commit 跑 24 个场景。 |
| 回归层 | Regression | 三层顶层。历史缺陷与语义质量，需真实 API 调用，每次 PR / 发版前跑 25 个场景。 |
| Golden 数据集 | Golden Dataset | **109 个评测场景**的完整定义（输入、预期输出、评估标准），三层共用的参照基准，**不是某一层**。 |
| 确定性评测 | Deterministic Track | 按断言引擎切分的轨道之一，92 个场景走 pytest 精确断言。 |
| LLM 评分 | LLM-as-Judge | 另一条轨道，17 个场景由 LLM 打分（质量层阈值 ≥3.5/5 平均分）。 |
| 性能基线 | Performance Baseline | 回归层的第 14 个维度（8 场景），指标为延迟 < P95 目标。**与「基线层」是不同的东西**，不要混用。 |

「三层」与「双轨」是同一批 109 个场景的两种**不同切法**——三层按成本/触发频率划分，双轨按断言引擎划分。一个场景同时属于某一层和某一条轨，两者不冲突。

## Agent

| 中文 | English | 定义 |
| --- | --- | --- |
| 单 Agent | Single-Agent (ReAct) | 当前 v2 架构：**单一 ReAct Agent + Persona 过滤工具集**，30 个工具全量注入，LLM 自主选择（30 号 §二）。v1 的 Coordinator + Router + 6 Sub-Agent 多智能体模式仅作为回退保留。 |
| 意图类型 | — | Gateway 的分类结果，取值 **`chat` / `navigate`** 两种，无其他取值（30 号 §六）。 |
| 角色 | Persona | 取值 **`Teacher` / `Student` / `Tutor` / `Parent`** 四套（30 号 §4.x 标题）。`Tutor` 是「化学助教-通用」。 |
| 工具 | Tool | Agent 可调用的能力单元。**文档中「工具」与「技能 Skill」同义混用**（`skill_name` / `available_skills` / `chem_skills/`）。 |
| Guard 护栏 | Guard | 请求级安全基础设施，每位 Agent 调用创建新实例。**四层**：前置检查 `check_prerequisites` → 调用限制 `check_limit` → 去重检查 `check_dedup` → 审批门控 `requires_approval`（30 号 §5.1）。 |
| 护栏状态 | `GuardState` | Guard 实例内部持有的状态对象，跟踪去重、限制、审批。 |
| 网关 | Gateway | **意图分类器**——在请求进入 Agent 前预分类 `chat` / `navigate`，采用「LLM 优先 + 关键词兜底」双路径（30 号 §六）。 |

## OCR 与批改

| 中文 | English | 定义 |
| --- | --- | --- |
| 上传会话 | `UploadSession` | 一次文件上传的完整生命周期。状态机：`UPLOADED` → `PREVIEWING` → `READY` → {`IMPORTING` → `IMPORTED` \| `GRADING` → `GRADED`} → `DONE`；非终态可转 `DISCARDED`，出错为 `ERROR`（24 号 §三）。 |
| 预览 | （状态 `PREVIEWING`） | OCR 预览进行中；`READY` 表示预览完成、等待用户操作。 |
| 题库导入 | （状态 `IMPORTING` / `IMPORTED`） | 把校对后的识别结果写入题库，产出结构化题目列表（24 号 §三）。 |
| 批改 | Grading | 对作答评分。主词是「**批改**」（`Answer Sheet OCR Grading System`），「判卷」只出现在按钮复合词「批改判卷」中。 |
| 批改任务 | （`OCRTask`） | 状态机 `pending` → `processing` → `done` / `failed`，`failed` 可重试回 `pending`（24 号）。 |
| 降级链 | Degradation Chain | 外部能力不可用时的多层兜底路径（24 号 §…）。另有「兜底」「退化」两个相关词：退化路径的结果标记 `degraded=True`，降级使用时标记 `fallback_used=true`。 |
| 轮询 | — | **双轮询器**：后端 APScheduler 每 5 秒扫表 + 前端每 5 秒查批次状态；百度 `correct_edu` 异步接口另有 3 秒 / 最长 120 秒轮询（24 号）。 |

## 已废弃的术语（勿再使用）

以下术语曾出现在本项目早期草稿中，**经全量核对，设计文档（20–43 号）中不存在**。不要在任何产出中使用：

| 废弃术语 | 核对结论 | 正确说法 |
| --- | --- | --- |
| 迷思概念类别 / Misconception Category（6 类） | 文档无此分类。全目录「迷思」仅出现 1 处，是 28 号 §3.2.1 选择题干扰项的内联注释（"迷思：溶液≠电解质"） | 知识点按 `category` 分 8 类，见上表 |
| 障碍类型 = 概念理解/审题/表述 | 取值对，但中文全名带「型」 | 概念理解型 / 审题障碍型 / 表述障碍型 |
| 障碍类型与迷思概念「正交」 | 「正交」二字文档从未出现 | 「相互独立」（27 号 §2.2），且指的是三个障碍维度之间 |
| 题型 `multi_choice` / `true_false` / `short_answer` / `essay` | 文档无此 4 种题型 | 题型只有 5 种，见上表 |
| 难度 1–5 | 文档是枚举，非数字 | `easy` / `medium` / `hard` / `competition` |
| 四维审核 = 科学性/难度匹配/知识点覆盖/区分度 | 四维内容与文档不符 | 系数配平 / 反应条件 / 产物稳定性 / 分子结构 |
| Persona 含 `admin` | 文档无 admin Persona | `Teacher` / `Student` / `Tutor` / `Parent` |
| 考试状态含 `grading` / `archived` / `cancelled` | 考试状态机无这些状态 | `Draft` → `AddingQuestions` → `Published` → `InProgress` → `Completed`。`GRADING` 属于 OCR 的 `UploadSession`，不是考试状态 |
| 试卷导入 / Exam Import | 文档是「题库导入」 | 题库导入（写入题库，非写入试卷） |
| Gateway 负责统一收口外部模型调用、路由、限流、降级 | 文档 Gateway 只是意图分类器 | 模型多路回退在 38 号 §十四，OCR 降级链在 24 号 |
| 账号一个绑定一种角色 | 文档只显示 Account 携带单一 `role` 字段，未如此表述 | 见「账号」词条 |

## 用语约定

- **枚举值大小写不统一，按来源文档区分**：障碍类型、题目类型、难度、Persona、Agent 意图为 snake_case 或单词（`concept` / `choice` / `easy` / `Teacher` / `chat`）；考试状态与 OCR 状态机为 CamelCase / UPPER_SNAKE（`Draft` / `InProgress` / `UPLOADED` / `PREVIEWING`）。命名时以所属文档为准，不要自行统一。
- 「年级」指组织层级；表示分数时用「成绩」「正确率」「平均分」「及格率」。
- 「障碍」指 `BarrierType`，不与任何知识领域分类混用。
- 「批改」是 OCR 判分的主词，不叫「判卷」；「题库导入」不叫「试卷导入」。
