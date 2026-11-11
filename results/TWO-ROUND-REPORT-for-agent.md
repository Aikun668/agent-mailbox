# 两轮工作报告（给后续改代码的 agent）

协调说明，不是产品代码。用户要的是：先按上一轮聊天/SSE/记忆审查修问题，隔约 10 分钟后再只审计划模块（最多两块），然后交出两份报告。本文件是给 agent 用的完整稿。用户精简版是另写的 Page。

不要擅自合并分支，不要在用户没要求时开新的修复 PR。

## 当前状态

| 事项 | 位置 | 状态 |
| --- | --- | --- |
| 聊天 / SSE / 记忆修复 | https://github.com/Aikun668/Software-Fighters-Project/pull/2 ，分支 `cursor/chat-sse-memory-fixes-9072`，基线 `for-work` @ `e620b2c`，头 `7891d3c` | 已开 PR，未合并。改了 35 个文件，+1355/−362 |
| 计划模块审查 | 分支 `feat/adaptive-plan-mvp` @ `af25e9ed0e4849bbe682cc15fe8099b0ceb5da56`（2026-11-10，父提交 `741292e` 是学习地图） | 只读，没有改代码，没有开修复 PR。该分支不在 `for-work` 上，也还没有自己的 PR |

仓库：https://github.com/Aikun668/Software-Fighters-Project

## 合并时必须知道的分叉

- PR #2 改的聊天页是 `archive/frontend/` 里的 `QuestionNavChat.vue`（以及 `ChatController`、`ConversationController`、`LoginInterceptor`、`PythonClient`、`application.yml` 等）。
- 计划模块用的是活的 `frontend/src/views/QuestionNavChat.vue`。把 PR #2 合进计划分支时，后端和 `application.yml` 可能冲突；它**不会自动修到**计划面板所在的那份聊天页。合并时以活的 `frontend/` 为准，不要只改 `archive/frontend`。
- 计划分支自 `e620b2c` 以来也改过 `ChatController`、`ConversationController`、`application.yml` 和活的 `frontend/src/api/chat.ts`。计划续学仍会撞上：SSE `error` 停读导致错过 `done`；空回复仍可能落成 `[AI已回复]`。这些问题本来就在这条分支上，计划提交没有把它们重新引进来，也不该在计划服务里修。

## 第一轮：已修（PR #2）

对照审查 `e620b2c`。云端 agent：https://cursor.com/agents/bc-e8471e56-25da-59e3-a641-83b90e3f9072

### P0

- **P0-1** Java 不再在 SSE `error` 上停读。`done` 仍会到，会话 id 写进 `meta` / `error` / `done`。失败落一条 `mode=notice` 的人话，不把已流出的坏 token 当助手原话。前端失败时钉住会话 id 并刷新侧栏。
- **P0-2** `learning.auth.required` 默认改为要求登录（`LEARNING_AUTH_REQUIRED=false` 才跳过）。鉴权关闭时不再覆盖已经解析出的用户。会话读/改/消息和画像按 `UserHolder` 校验，对不上按项目既有 `Result.fail(404)`。创建会话忽略 body 里的 `user_id`。不属于自己的 `conversation_id` 在发消息前返回 HTTP 404，不再悄悄开新会话。Python 在 `APP_ENV`/`ENV` 为 production、prod、staging 时拒绝缺省 `user_id`。JSON 接口继续用 `Result.fail(404)`，只有还没开始 SSE 的发消息用 HTTP 404。

### 其他已修

- 健全性失败或重试采纳时先发 `retract`，清掉已经打出的字。错误文案统一为人话，稳定 `code` 留给日志和帧。管道异常不再把 `str(exc)` 给用户。
- 没有正文时不再写入 `[AI已回复]`。历史占位符和 `notice` 不进下一轮模型上下文。
- 心跳注释转发给浏览器。超时、上游非 200、客户端断开都会给人话，并尽量把这轮记在当前会话里。生成中可以停止（AbortSignal）。生成中点另一条会话会停掉当前流再加载。
- `last_extracted_msg_index` 从画像快照带回下一轮（P1-5 局部）。
- 进度和中心文案不再直接显示图节点名。空态问候跟本地时间。快捷句作为第一条提问发出。非图片附件在发送前拒绝。推荐练习出现在气泡下。学情页和知识图谱去掉另一名学生的示例数字。
- 按 mode 过滤消息时保留用户行。请求日志不再打全文和图片 data URI。

### 验证（PR #2）

- Java：`SseEventAccumulatorTest`、`ChatTurnPersistenceTest`、`ConversationAccessTest`、`LoginInterceptorTest`（12 个，通过）。
- Python：`test_sse_contract.py`、`test_state_scope_contract.py`、`test_error_fallback_contract.py`，以及 dispatcher 的 retract 用例。云端环境缺 `DEEPSEEK_API_KEY`，原有 4 个会构造真实 LLM 的用例失败，与这次改动无关。
- 前端：`chat.spec.ts`、`QuestionNavChat.spec.ts`、`LearningAnalysisPage.spec.ts`、`ChatThread.spec.ts` 通过。
- 没有起浏览器打完整 Java/Python 链路。该 PR 的 GitHub Checks 当时为 0（仓库可能没有 Actions）。

### 第一轮推迟，不要在计划修复里顺手半接

- 结束会话 LLM 摘要：`POST /conversations/{id}/end` 前端没有入口；硬接进下一轮会改变提示词。答疑路径已经用 `SummaryMaintainer` 从历史确定性重放摘要。
- 可视化块落库：`messages` 没有存放 visualization JSON 的列。刷新后图仍会消失。
- 苏格拉底 / 规划选项只在 localStorage，会话表没有风格字段。
- 统一聊天路径没有 web_search，是能力缺口，不是这条 SSE 链路的缺陷。

## 第二轮：计划模块审查（只读）

云端 agent：https://cursor.com/agents/bc-3ae8ef60-1086-599b-927b-e00f4b9f9cff  
精确提交：`af25e9e`，相对父提交 `741292e` 约 26 个文件、+805/−77。没有跑服务，没有重放作者的 H2 / 浏览器验收。

### 看了什么（最多两块）

1. **自适应计划**：`AdaptivePlanController`、`AdaptivePlanService`、`frontend/src/api/adaptivePlans.ts`、`AdaptivePlanWorkspace.vue`、被改成承载它的 `LearningPlanPage.vue`。调整不是 LLM，而是 `rules-v1` 子串规则。聊天不会调用这些接口。
2. **交界**：`QuestionNavChat.vue` 的计划面板和一次性续学草稿、`learningDraft.ts`、知识地图续学如何抢同一个草稿槽，以及学习快照的 `resume_plan` / `review_current_node` 如何进入 `/learning-path`。

故意没看：资源工作台、showcase、`archive/`、地图的其余部分、旧 `LearningPlan` 的完整 CRUD。

### P0 — 默认配置下，计划的所有权检查全部落在用户 1

`application.yml` 里 `learning.auth.required: false`（约 83–85 行）。`LoginInterceptor` 在开关为 false 时，不看已经还原出的登录用户，直接写成 id `1`、`dev_user`（约 18–25 行）。拦截器顺序是 RefreshToken 先跑、Login 后跑，真实 token 用户也会被盖掉。

`AdaptivePlanController.user()` 只检查 `UserHolder` 非空，随后所有 SQL 都按这个 id 过滤。服务方法本身没有漏 `user_id`，也没有从请求体收 userId。在这份默认配置下，那些条件比较的是同一个常量，任意浏览器会话都能读写同一批目标、断点和调整记录。

现有测试直接传 userId，浏览器桥用 `X-Test-User`，都绕开了这个拦截器。验收里的「越权全拒绝」没有覆盖进程实际会跑的身份来源。

**不要在计划服务里再造一套身份。** 应让这些接口走真实登录用户。在运行配置把 `learning.auth.required` 打开之前，不能把所有权当成已验证。仓库里没有 `application-prod.yml`。这就是 PR #2 里那个 default userId=1；计划提交没有新开「客户端传 userId」的洞，但也没有关掉这个开关。PR #2 的修复还没合到这条分支。

### P1 — 否定句和无关句子仍会生成相反的改动，确认稿还把断点藏起来

`AdaptivePlanService.propose`（约 69–85 行）：否定检测只认「不要 / 不需要 / 不用 / 别 / 无需」后接延期/推迟/拆分/调整。「不延期」「不拆分」仍会 `contains("延期")` / `contains("拆分")` 并改计划。「没有卡住」「我不是不懂」会把第一个未完成任务标成 blocked，并覆盖原来的断点和 nextAction。每日分钟正则会把「某一天我用了 20 分钟」改成每日预算 20。

拆分是窗口副作用，不是单独的用户意图：报卡点、要求延期或只改截止日期时，窗口里超长的任务仍可能被拆开。延期只打中窗口里第一个任务。

确认界面 `display()`（`AdaptivePlanWorkspace.vue` 约 26–30 行）不展示断点正文。卡点 `kind` 是 `"checkpoint"`，标签表只有 keep/split/defer/budget，会露出英文。每个未改动的任务都以「保留」占一行。落库仍要再点确认，所以不是静默写库；但模块自己的安全声明（否定句不猜测）与实现不符。

已完成任务不会被拆，守住了。建议：识别不了或检测到否定就拒绝；拆分只在用户明确要求拆或明确改了每日分钟时发生；卡点预览写出旧断点 → 新断点。

### P1 — 「继续本周计划」藏起自适应计划，并给另一套计划记满分

`LearningPlanPage.vue`：`actionKind` 为 `resume_plan` / `review_current_node` 时整块 `AdaptivePlanWorkspace` 被 `v-if="!isPlanAction"` 藏起（约 15、93 行）。「完成本次动作」调用旧数字 id 计划的 `updatePlanProgress`，固定 `score: 1`、`duration_minutes: 0`（约 49–56 行）。`actionPlanId` 来自快照 `active_plan.plan_id`，由旧 research 路径写入，自适应计划的创建/完成不会更新它。父提交里两块界面是同时在的。

地图续学（`KnowledgeGraphPage.continueLearning`、`LearningMapFeatures.continuation`）不读 `adaptive_plan`，也不带 `planId`，和计划续学抢同一个草稿槽。

计划面板写「普通问答或学习规划模式生成的文字不会修改此计划」。对 `adaptive_plan` 这句是真的，但 learning-planning / research 仍会另存一份旧 `LearningPlan`，之后快照可能把它当成「本周计划」。

建议：`/learning-path?actionKind=resume_plan` 不要藏起自适应计划；在两套计划打通之前，禁止用 `score: 1` 冒充这一页的完成。

### P2（摘要，细节见完整审阅稿）

- `list` `ORDER BY id`，id 是随机 UUID；未带 `planId` 时打开 `all[0]`，首页常打开错误的一份。表在首次请求 `CREATE TABLE IF NOT EXISTS`，没有 `user_id` 索引。
- 每日分钟只拆眼前 3 个未完成任务；没有截止日期时不产生 `capacityWarning`。`parts<=30` 会让超长任务整份提案失败。有截止日期且装不下时，强制选择延长或延期，这个保护不要拆掉。
- 「我已完成」没有确认，也不能改回。进度保存会清掉 `undo_json`（有新进度就不能用旧撤销，这是有意取舍）。「记录断点 / 暂停」默认把进行中改成暂停。
- 坏的 `planId` 把新建入口一起藏起来；全局 axios 对所有 HTTP 404 先弹「接口不存在」。
- `learningDraft.ts` 是单个内存槽。计划续学和地图续学后写的有效。刷新、新标签会丢草稿。连续点「用最新计划准备续学草稿」会往输入框追加而不是替换。不要退回「直接覆盖未发送输入」或「自动发送」。续学正文不进 URL，这点是对的。
- `planPanelOpen` 只在 setup 时读一次 `planId`。展开后面板最高约 55vh，对话区被顶住。
- 几乎不能改标题、目标、任务名；截止日期不能清空。没有删除计划/任务。

### 人性化（功能能跑，但不像给人用）

调整框文案像听得懂人话，实际是子串。两套计划共用导航名和 `/learning-path`。误点完成没有反悔。创建时不能写「回来先做什么」。聊天顶部展开计划后，对话区被半屏表单顶住。术语不统一（`undo`、`checkpoint`）。

### 不要退掉的部分

用户 id 可信时跨用户是 404；版本是条件更新，并发下只有一个写入成功；提案在确认前不改计划，过期或跨日不能再次应用；期限冲突必须明确选择延长或延期；已完成任务不会被拆，分钟数拆分后仍守恒；有后续进度时撤销被拒绝（缺的是「误点完成」自己的回退，不是把撤销再放开）；续学正文不进 URL，也不会自动发送；完成文案写的是「自报完成」，自适应计划写入路径不会去改知识图谱；规则引擎没有假装自己是模型。

### 建议的修改顺序（若用户要求修计划）

1. 先让 `/api/adaptive-plans` 使用真实登录用户，并用带拦截器的测试锁住「配置为 false 时不得把计划写成共享的 user 1」，或直接在非本机 profile 打开认证。不要另造 userId 参数。
2. 收紧 `propose`：识别不了就拒绝；拆分与否定分开测（`不延期`、`没有卡住`、`某一天…20分钟`、只报卡点却被拆分）。预览展示断点前后文，并去掉全量「保留」行。
3. `/learning-path` 上的快照动作不要覆盖自适应计划；打通之前禁止用 `score: 1` 冒充这一页的完成。
4. 列表按更新时间；误完成可恢复；任务标题和截止日期可改可清空。
5. 续学草稿与地图草稿分槽，或在覆盖前提示。聊天 SSE / `[AI已回复]` 留给 PR #2 那条线，合并时以活的 `frontend/` 为准。

## 未能验证

两轮都主要是源码 + 单元/契约测试。没有在浏览器里把 Java 和 Python 整条链路跑起来，也没有重跑计划作者的 H2 浏览器桥。如果部署时用环境变量把 `learning.auth.required` 设成 true，计划分支上的 P0 在那个环境里不成立；仓库内的默认文件不是这样。
