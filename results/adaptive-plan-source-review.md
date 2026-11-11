# 自适应学习计划只读审阅

给后续改代码的代理用。本轮没有改产品代码，没有提交，没有开修复 PR，没有合并分支。

## 审阅范围

- 仓库：`Software-Fighters-Project`
- 分支：`feat/adaptive-plan-mvp`
- 精确提交：`af25e9ed0e4849bbe682cc15fe8099b0ceb5da56`
- 说明：`feat: add adaptive learning plans with reviewed adjustments and resumable chat`（作者 Codex，2026-11-10）
- 父提交：`741292e`（`feat: deliver evidence-based learning map trajectory suggestions and continuation`），`741292e` 是 `af25e9e` 的祖先。该分支不在 `for-work` 上。审阅时工作区干净，且没有这个分支的开放 PR。
- 计划功能就是这一次提交（相对 `741292e` 共 26 个文件，约 +805/−77）。下面按模块读的是这次提交加上它直接调用的既有代码，不是全仓库。

### 模块 1：自适应计划

- `backend/src/main/java/com/university/learning/controller/AdaptivePlanController.java`
- `backend/src/main/java/com/university/learning/service/AdaptivePlanService.java`
- `frontend/src/api/adaptivePlans.ts`
- `frontend/src/components/plans/AdaptivePlanWorkspace.vue`
- `frontend/src/views/LearningPlanPage.vue`（本提交把主界面换成上述组件，并保留快照动作条）
- 配套测试与验收记录只用来对照「声称验证了什么」，不当作实现本身：`AdaptivePlanServiceTest`、`AdaptivePlanIndependentReviewTest`、`AdaptivePlanBrowserBridge`、`AdaptivePlanWorkspace.spec.ts`、`docs/verification/adaptive-plan/`、`docs/acceptance/2026-11-10-adaptive-plan-mvp-delivery.md`、`docs/verification/adaptive-plan/INDEPENDENT-REVIEW.md`

计划调整没有接 LLM。`AdaptivePlanService.propose` 是子串/正则规则（`engine=rules-v1`）。聊天不会调用这些接口。

### 模块 2：和计划绑在一起的续学聊天，以及学习地图/快照只在「喂给计划或被计划页挡住」的那一段

- `frontend/src/views/QuestionNavChat.vue`：本提交加上的计划面板、续学草稿监听、发送成功后保留 `planId` 的 `router.replace`
- `frontend/src/utils/learningDraft.ts`：计划续学和知识地图续学共用的一次性内存草稿
- `frontend/src/views/KnowledgeGraphPage.vue` 的 `continueLearning`（约 38–47 行）：地图续学写同一草稿槽并进入同一聊天页
- `backend/src/main/java/com/university/learning/graph/LearningMapFeatures.java` 的 `continuation`：地图续学草稿的正文来源。地图节点、边、建议决策没有再往下读
- `ai-service/app/services/next_action.py` 里 `resume_plan` / `review_current_node` → `/learning-path` 的入口表（约 74–87 行）
- `LearningPlanPage.vue` 在带 `actionKind` 时改去打旧计划进度：`POST /api/plans/{id}/progress`（`LearningPlanController.updateProgress`）
- 聊天发送链路只核对了计划草稿会撞上的现成行为：`frontend/src/api/chat.ts` 的 `parseSSEStream`，`ChatController.saveAiResponse` 的空回复占位，`LoginInterceptor` + `application.yml`

### 故意没看

资源工作台、showcase、`archive/` 里的旧前端实验、知识图谱的布局和拖拽、练习/资源/画像接口、旧 `LearningPlan` 的完整 CRUD。`Home.vue` 在本提交里只改了窄屏侧栏宽度，与计划行为无关。

## 结论

服务层在「用户 id 可信」的前提下，所有权、乐观版本和「先预览再落库」是认真做的。默认配置让这个前提不成立。规则引擎会在用户没有要求、甚至写了否定句时改计划，而确认界面看不出断点被覆盖。学习快照上的「继续本周计划」打开的是同一条 `/learning-path`，但会藏起自适应计划，并用满分、0 分钟去完成另一套旧计划。

## 发现

### P0 — 默认配置下，计划的所有权检查全部落在用户 1

预期：每个登录用户只能列出、读取、修改、应用、撤销自己的计划。验收文档把「跨用户拒绝」写成已通过。

实际：仓库里唯一的 `application.yml` 把认证关掉了。

```83:85:backend/src/main/resources/application.yml
learning:
  auth:
    required: false
```

`LoginInterceptor` 在这个开关为 false 时，不看 Redis 里已经还原出的登录用户，直接写成 id `1`、用户名 `dev_user`：

```18:25:backend/src/main/java/com/university/learning/utils/LoginInterceptor.java
        if (!authRequired) {
            UserDTO dev = new UserDTO();
            dev.setId(1L);
            dev.setUsername("dev_user");
            dev.setAvatarUrl(null);
            UserHolder.saveUser(dev);
            return true;
        }
```

拦截器顺序是 `RefreshTokenInterceptor`（order 0）先跑、`LoginInterceptor`（order 1）后跑，所以真实 token 用户也会被盖掉。`AdaptivePlanController.user()`（第 14 行）只检查 `UserHolder` 非空，随后所有 SQL 都按这个 id 过滤（`AdaptivePlanService` 第 44–45、51、54、87、90、100 行）。服务方法本身没有漏 `user_id` 条件，也没有从请求体收 userId。在这份默认配置下，那些条件比较的是同一个常量，任意浏览器会话都能读写同一批目标、断点和调整记录。

为什么现有测试绿：`AdaptivePlanServiceTest` / `AdaptivePlanIndependentReviewTest` 把 userId 当参数传入，不经过拦截器。`AdaptivePlanBrowserBridge` 用 `X-Test-User` 头在独立 MockMvc 里塞用户，同样绕开 `LoginInterceptor`。验收里的「越权全拒绝」没有覆盖进程实际会跑的身份来源。

和 PR #2 的关系：这就是那份聊天修复里说的 default userId=1。计划提交没有新开一个「客户端传 userId」的洞，但也没有关掉这个开关。不要在计划服务里再做一套身份；应让这几个接口走真实登录用户。在 `learning.auth.required` 于运行配置里为 true 之前，不能把所有权当成已验证。仓库里没有 `application-prod.yml`，也没有看到覆盖该键的配置文件。

### P1 — 否定句和无关句子仍会生成相反的改动，确认稿还把断点藏起来

预期：设计说明和 `INDEPENDENT-REVIEW.md` 都写了否定式请求拒绝猜测、不支持的句子不改计划。预览应让人看清将要落库的差异，取消则计划不变。

实际：否定检测只有这一种形式，而且在生成任何提案之前整句拒绝：

```69:72:backend/src/main/java/com/university/learning/service/AdaptivePlanService.java
        ObjectNode p=get(user,planId);version(p,body);String message=text(body,"message",2000,true);require(!Pattern.compile("(不要|不需要|不用|别|无需).{0,8}(延期|推迟|拆分|调整)").matcher(message).find(),"检测到否定指令，未生成更改。请明确要修改的部分。");ObjectNode candidate=p.deepCopy();
        ...
        boolean defer=message.contains("延期")||message.contains("推迟"); boolean blocked=message.contains("卡住")||message.contains("不懂")||message.contains("没懂");boolean changed=false;
```

因此：

- 「不要延期」会拒绝（测试只覆盖了这一句，见 `AdaptivePlanServiceTest` 第 29 行）。「不延期」「先不延期」「不拆分」不含那些前缀，`contains("延期")` / `contains("拆分")` 仍为真，会延期或进入拆分。
- 「没有卡住」「我不是不懂」含「卡住」「不懂」，第一个未完成任务会被标成 `blocked`，断点整段换成用户原文，`nextAction` 被换成固定句「先针对卡点做一个具体例子，再继续原任务」（第 81 行）。原先写好的断点和回来后的第一步被盖掉。
- 每日分钟的正则是 `(?:每天|每日|一天|daily)[^0-9]{0,8}(\\d{1,3})`（第 70 行）。「某一天我用了 20 分钟」「不是每天 20 分钟」都会把每日预算改成 20。预算路径没有否定词。
- 「不用调整，只要每天 20 分钟」会因为「不用…调整」整句失败，合法的预算改动也做不成。

拆分不是单独的用户意图，而是窗口内的副作用。第 78–82 行：前 3 个未完成且未延期的任务（或指定的那一个）只要分钟数大于预算，就拆，不管这句话是在报卡点、要求延期，还是只改截止日期。`defer` 和 `blocked` 互斥，一句里两件事先发生的那个赢，另一件被丢掉。延期只会打中窗口里的第一个任务（`!changed`），「都延期」不会延期其余任务，旁边超长的任务却可能被拆开。

确认界面放大了这个问题。

```26:30:frontend/src/components/plans/AdaptivePlanWorkspace.vue
const display = (value: unknown): string => {
  if (typeof value === 'string') return value
  if (Array.isArray(value)) return value.map(display).join('；')
  if (value && typeof value === 'object' && 'title' in value) { const t = value as AdaptiveTask; return `${t.title} · ${t.minutes} 分钟 · ${labels[t.status] || t.status}${t.nextAction ? ` · 下一步：${t.nextAction}` : ''}` }
  return JSON.stringify(value)
}
```

断点字段不在这行字里。卡点类改动的 `kind` 是 `"checkpoint"`，标签表只有 `keep/split/defer/budget`（模板约第 124 行），界面上会露出英文 `checkpoint`。`budget` 这个中文标签是死的，后端从不产生这种 `kind`。每个未改动的任务都会以「保留」占一行，真正的延期/拆分夹在里面。用户确认的是一份又吵又缺字段的稿，不是他写的那句话。

落库仍要再点一次确认，所以不是静默写库。但模块自己的安全声明（否定句不猜测）与实现不符，误确认会改写未完成任务的标题、状态、断点和下一步。已完成任务不会被拆，这一点是守住的。

建议方向：无法识别或检测到否定时，提案直接失败，不要用 `contains` 做正向动作。拆分只在用户明确要求拆、或明确改了每日分钟时发生，并且预览要列出全部将被拆的任务，而不是只拆眼前 3 个。卡点预览必须写出旧断点 → 新断点，并让用户选择是追加还是覆盖。

### P1 — 「继续本周计划」藏起自适应计划，并给另一套计划记满分

预期：学习计划页就是这份自适应计划。快照/地图上的「继续」要么进入这份计划的当前任务，要么明确说自己完成的是另一份旧计划。完成自适应任务只记自报，不写掌握度。

实际：本提交把 `LearningPlanPage` 的主体换成 `AdaptivePlanWorkspace`，但快照动作和它互斥。

```15:15:frontend/src/views/LearningPlanPage.vue
const isPlanAction = computed(() => actionKind.value === 'resume_plan' || actionKind.value === 'review_current_node')
```

```93:93:frontend/src/views/LearningPlanPage.vue
    <AdaptivePlanWorkspace v-if="!isPlanAction" :plan-id="typeof route.query.planId === 'string' ? route.query.planId : undefined" />
```

`ai-service/app/services/next_action.py` 第 80–87 行把 `resume_plan` 和 `review_current_node` 的 `entry` 定成 `/learning-path`。学情页按这个 entry 带上 `actionKind` 跳过来。父提交 `741292e` 的计划页是「动作条 + 原来的计划界面」同时在；这一提交改成动作条出现时整块自适应计划消失。

动作条上的「完成本次动作」调用的是旧数字 id 计划，不是 `adaptive_plan`：

```49:56:frontend/src/views/LearningPlanPage.vue
    const response = await updatePlanProgress(actionPlanId.value, {
      step_id: actionStepId.value,
      step_order: currentStep.value?.order ?? 0,
      status: 'completed',
      activity_type: actionActivityType.value,
      score: 1,
      duration_minutes: 0,
    })
```

`actionPlanId` 来自学习快照的 `active_plan.plan_id`。那张表由聊天 research 模式的 `ChatController.saveResearchPlan`（约 386–425 行）等旧路径写入，自适应计划的创建/完成不会更新它。`LearningPlanController.updateProgress` 自己的注释（约 310–318 行）已经写明：前端固定 `score: 1` 会让掌握度按虚假高分计算。自适应计划的文案反复说「完成不等于掌握」。同一个人从「继续本周计划」进来，一次点击得到的是相反的记账。

地图续学也不读自适应计划。`LearningMapFeatures.continuation` 只拼节点摘要和历史引文；`KnowledgeGraphPage.continueLearning` 把草稿送进 `/question-nav-chat`，query 里没有 `planId`。自适应计划的「带着计划去对话」另写一份草稿。两条「继续学习」互不知道对方。

计划面板第 123 行写「普通问答或学习规划模式生成的文字不会修改此计划」。对 `adaptive_plan` 这句是真的：没有任何工具调用这些 API。learning-planning / research 模式仍会另存一份旧 `LearningPlan`，之后快照就可能把它当成「本周计划」。这句话会让人以为聊天里的规划是安全的空谈。

建议方向：`/learning-path?actionKind=resume_plan` 不要藏起自适应计划。若暂时不能把快照指到 `adaptive_plan`，动作条必须写明它完成的是另一份计划的哪一步，并且不要再用 `score: 1`、`duration_minutes: 0`。地图续学和计划续学至少不要抢同一个草稿槽（见下方 P2）。

### P2 — 打开计划页时，选中的往往不是刚用的那一份

`AdaptivePlanService.list`（第 44 行）`ORDER BY id`。id 是随机 UUID，没有 `created_at` / `updated_at`。未带 `planId` 时，前端 `load` 取 `all[0]`（`AdaptivePlanWorkspace.vue` 第 51 行）。刷新后下拉顺序也会回到 UUID 序，而不是「最近编辑的在前」。多份计划时，学习计划首页稳定地打开错误的那一份。

表在首次请求时 `CREATE TABLE IF NOT EXISTS`（第 22–26 行），没有 `user_id` 索引，提案表没有清理。计划份数没有上限（任务上限 100、模块上限 20 只在单份计划内）。这是后一个规模问题，不是现在就会坏，但列表顺序是现在就会让人找错计划的问题。

### P2 — 每日 20 分钟只拆眼前 3 个任务

第 78 行的窗口是 `refined<3` 或指定 `taskId`。用户把每天 60 分钟改成 20 分钟时，只有前三个未完成任务会被拆；后面仍是 60 分钟一块。没有截止日期时，`decorate`（第 40–41 行）不产生 `capacityWarning`，因为警告只在有 deadline 时计算。确认稿里这些任务显示为「保留」，混在全部任务的保留行中，容易看成「已经按 20 分钟重排完了」。

有截止日期且装不下时，冲突分支会强制选择延长或延期，这个保护是有效的，不要在修窗口时拆掉它。

另外，`parts<=30`（第 82 行）会让「任务 480 分钟、每天 15 分钟」整份提案失败（32 段），而不是给出可确认的方案。用户如果只是在报卡点，超长任务也会触发这句「拆分过细」。

### P2 — 误点「我已完成」后，界面上改不回来

「我已完成」是一步按钮（`AdaptivePlanWorkspace.vue` 约第 121 行），没有确认。`updateTask`（第 65 行）接受 `pending/in_progress/paused/blocked/completed`，但不接受在调整流程之外把任务标成 `deferred`。完整路线里只有延期任务有「重新列入」。已完成任务没有「我点错了」。

撤销只在 `last_action='apply'` 时可用（第 99–100 行）。任何进度保存都会把 `undo_json` 写成 null（第 66 行）。这是设计文档里的明确取舍（有新进度就不能用旧撤销覆盖），不是意外。取舍的结果是：误点完成，既不能撤销这次点击，也不能撤销这次点击之前的那次调整。至少应能把自报完成改回未完成，并在点击前确认。

「记录断点 / 暂停」会把状态下拉默认成 `paused`，除非任务已经是完成（第 66 行）。人只想补一句断点、任务本来是「进行中」时，保存会把任务暂停。下拉里有「继续进行」，但是默认值偏了。

### P2 — 坏的 planId 把新建入口也藏起来

`load` 先 `list` 再 `get`，赋值在两次都成功之后（第 50–53 行）。`get` 404 时 `current` 仍是 null，`error` 有值。创建表单的条件是「没有当前计划且没有 error」（约第 109 行），于是错误页上不能新建。刷新调用 `load(current?.id)`，此时 id 为空，会去打开列表第一份，地址栏里的坏 `planId` 还在；下次整页刷新又 404。全局 axios 拦截器对所有 HTTP 404 弹「接口不存在」（`frontend/src/utils/request.ts` 第 76–78 行），计划不存在、模块不存在、提案不存在都会先看到这句，再在页面上看到后端的「计划不存在」。400/409 还会同时出现一条 ElMessage 和页面内错误。

### P2 — 续学草稿是单个内存槽，刷新、新标签和地图续学会互相冲掉

```1:14:frontend/src/utils/learningDraft.ts
// One-use handoff only. Source excerpts must never enter URLs or browser storage.
let pending: { handle: string; text: string } | undefined
export function setLearningDraft(text: string): string {
  ...
  pending = { handle, text }
  return handle
}
```

计划页「带着计划去对话」和地图「继续学习这个节点」写的是同一个 `pending`。后写的句柄有效，先写的 `consume` 得到空串。草稿不进 URL 是对的（测试也锁了这一点），代价是：

- 复制链接或新标签打开时，输入框是空的，只剩 `planId`（若走的是计划按钮）。
- 整页刷新发生在发送前，内存槽消失，未发送的续学上下文没了。计划面板还在的话可以重新生成；地图那份不行。
- 聊天页里连续点「用最新计划准备续学草稿」会往输入框追加，不会替换旧的一段。后退只改 URL 里的已消费 token，不会从输入框拿掉刚加上的文字。

同页追加、流式输出期间排队、不自动发送、从完整计划页进入时不夹带旧 `conversationId`，这些在 `QuestionNavChat.vue` 第 337–343 行和组件测试里是对的，不要在修草稿槽时退回「直接覆盖未发送输入」或「自动发送」。

`planPanelOpen` 只在 setup 时读一次 `route.query.planId`（`QuestionNavChat.vue` 第 18 行），之后 query 变了不会自动展开。从无 planId 的聊天回到带 planId 的历史记录时，面板保持收起，计划组件根本不挂载。

聊天列是 `height:100%; overflow:hidden` 的纵向 flex（约第 417–418 行）。计划面板 `flex-shrink:0`，展开后最高 55vh（组件样式约第 135 行），会话中部被压在剩余高度里，页面本身不能把面板滚走。窄屏是 50vh。能用，但是「边看对话边改计划」会挤成一条。

### P2 — 计划一旦写下，几乎不能改正写法

没有删除计划、删除任务、重命名、改目标、改任务标题。创建表单没有「第一步」字段，`nextAction` 只能事后补。截止日期可以通过调整提案改日期，但不能清空：前端只在 `proposedDeadline` 非空时才传 `deadline`。延期任务仍算在「x / y 任务自报完成」的分母里。

调整历史不记时间，也不记当时选的是延长还是延期。撤销记录没有 `message`，界面退回到原始的 `undo`（历史渲染约第 129 行）。`apply` 第 97 行只写入 `proposalId`、`message`、`action`。

这些都不破坏已保存的数据，但和「计划可以中断，进度不必重来」放在一起，人会卡在写错的标题和清不掉的截止日期上。

## 人性化问题

上面若干条已经是行为缺陷。这里只记「功能能跑，但不像给一个人用」的部分，避免和 P0/P1 重复论证。

1. 调整框的占位是「告诉计划发生了什么」，读起来像听得懂人话。实现是几个中文子串。人会写「不延期」「没有卡住」「某一天学了 20 分钟」，然后在一长串「保留」里点确认。应把能力说成固定句式（例如必须选任务、必须填分钟数），或者在识别失败时停下来问，而不是猜。
2. 两套「学习计划」共用导航名和 `/learning-path`。侧栏叫学习计划，聊天空态有「梳理 Transformer 学习路径」也进这里，快照叫「继续本周计划」，自适应计划又说聊天不会改这份计划。人无法知道自己完成的是哪一份、掌握度会不会被改。
3. 眼前永远只有 3 步，这是好的；但「我已完成」没有反悔，断点表单默认把人暂停，延期任务要先「重新列入」再调整。这些规则只写在控件附近的小字里，误点的成本是一份改不回去的自报进度。
4. 创建时不能写「回来先做什么」，只能写标题和分钟。真正续学要靠的断点，要等计划存完再找到那一项补。空的远期模块可以加，这点是好的；写错的小任务不能删。
5. 从计划页去对话，草稿是一次性的，刷新就没了。面板上的说明要求人再点一次「用最新计划准备续学草稿」才更新上下文，这个克制是对的（避免静默改已经写好的问题），但追加而不是替换会让输入框里同时躺着两版计划。人很容易把旧版本一起发出去。
6. 聊天里展开计划后，对话区被一块最高半屏的表单顶住。人是来接着学的，不是来在聊天顶部再操作一遍完整计划编辑器的。窄屏更挤。
7. 术语不统一：状态是中文，历史动作和卡点类型会露出 `undo`、`checkpoint`。每日预算的变化不在差异列表里，而在另一段「A → B」里，和任务差异是两套语言。

## 做得扎实的地方

后续修改不要为了修上面的问题把这些退掉。

- 写接口的 SQL 都带 `user_id`，跨用户是 404，不泄露计划是否存在。提案的应用同时核对提案 id、计划 id、用户 id。在用户 id 来自真实登录时，这一层是对的。
- 版本号放在文档和列上，更新是 `WHERE version=?` 的单行条件写。并发测试期望恰好一个 200、一个 409。请求体里的版本和库里的版本不一致会 409，而不是后写覆盖先写。
- 提案写入另一张表，应用前计划文档不变。版本变了、或 `evaluatedDate` 不是服务器当天，旧提案不能应用。取消只清前端状态。同一提案在版本前进后不能再打进去。
- 冲突时必须带 `resolution=extend|defer`。延长日期和将要延期的任务列表在提案里先算出来；应用时按同一天重算，而不是相信客户端传来的任务正文。客户端申请体只有版本和 resolution。
- 拆分按段相加等于原分钟数；已完成任务不进入拆分/延期窗口，断点还在。测试 `completedAndCheckpointPreserved`、`proposalDoesNotMutateAndSplitConservesEffort` 锁的是这条，不是否定句。
- 延期任务离开「接下来 3 步」，重新列入走 `pending`，不会被叫完成。空模块不会让整份计划变成已完成。
- 有后续进度保存时，撤销被拒绝，避免用调整前的快照盖掉新断点。这个方向应保留；缺的是「误点完成」自己的回退，不是把撤销再放开。
- 续学正文不进 URL。已有聊天里更新草稿会留下 `conversationId`。流式过程中到来的草稿会等到发送结束再填进输入框，且不自动发出。从计划页进入聊天会丢掉无关 query，避免把别人的会话 id 带进来。保存进行中如果地址栏的 `planId` 变了，结束后会按新地址重读。
- 完成文案写的是「自报完成」，自适应计划的写入路径不会去改知识图谱。空库不会造一份假计划。
- 规则引擎没有假装自己是模型。问题是识别面比文案窄，而且失败方向常常是「做了相反的修改」，不是「承认不懂」。

## 和未合并的聊天修复 PR 的关系

PR：https://github.com/Aikun668/Software-Fighters-Project/pull/2  
分支 `cursor/chat-sse-memory-fixes-9072`，基线是 `for-work` 的 `e620b2c`，标题「fix: 失败的聊天也留在当前会话里」。未合并。本轮没有移植也没有合并。

计划提交没有把那些 bug 重新引进来：这条分支在 `e620b2c` 之后还有前端复位、记忆/所有权和学习地图，那些问题本来就在。计划功能新碰过的聊天文件是活的 `frontend/src/views/QuestionNavChat.vue`。PR #2 改的是 `archive/frontend/` 下的同名页面，加上 `ChatController`、`ConversationController`、`LoginInterceptor`、`PythonClient`、`application.yml` 等。自 `e620b2c` 以来，本分支也改过 `ChatController`、`ConversationController`、`application.yml` 和活的 `frontend/src/api/chat.ts`。把 PR #2 合并进本分支时，后端和 yml 可能冲突；它不会自动改到计划面板所在的那份 `QuestionNavChat.vue`。

计划续学实际还走着这些现成路径：

- `frontend/src/api/chat.ts` 第 295–301 行：SSE `error` 事件直接 `return`，后面的 `done` 不会进 `onDone`。发送一开始就把输入框清空（`QuestionNavChat.vue` 的 `send`）。失败后再刷新，未写入会话的那次续学正文只活在当时的内存里。打开聊天时如果 URL 里已经有 `planId`，面板可以再生成一份草稿。成功路径上新的 `router.replace`（约第 278–280 行）会留下 `planId` 和已经消费过的 `learningContext` token，token 不是正文。
- `ChatController.saveAiResponse` 第 227–228 行：抽不出回复文本时仍落库 `[AI已回复]`。本分支的 `PythonClient` 大约第 777–790 行已经尝试把非 token 的 `content` 帧积进 `_response_text`，所以不是 `for-work` 上完全没修过的那一版，但占位符分支还在。计划续学若走「学习规划」这类不发 token 的模式，刷新后仍可能只剩占位句。这条不在计划服务里修。

## 未能验证

只读源码。没有启动 Java、前端、MySQL、Redis、Python/LLM，没有重跑作者的 H2 浏览器桥和 vitest。因此没有亲自证实并发 409、跨日提案和窄屏布局，只核对了测试在断言什么、以及测试没经过 `LoginInterceptor`。

Spring Boot 父 POM 是 3.4.6（Jackson 2.18）。`NullNode.asText("")` 在这个 Jackson 里会返回默认值 `""`，所以「不填截止日期」不应在 `decorate` 里把 `"null"` 解析成日期。这一点没有跑过。

如果部署时用环境变量把 `learning.auth.required` 设成 true，P0 在那个环境里不成立。仓库内的默认文件不是这样。

## 建议的修改顺序

1. 先让 `/api/adaptive-plans` 使用真实登录用户，并用带拦截器的测试锁住「配置为 false 时不得把计划写成共享的 user 1」或直接在非本机 profile 打开认证。不要另造 userId 参数。
2. 收紧 `propose`：识别不了就拒绝；拆分与否定分开测（`不延期`、`没有卡住`、`某一天…20分钟`、只报卡点却被拆分）。预览展示断点前后文，并去掉全量「保留」行。
3. `/learning-path` 上的快照动作不要覆盖自适应计划；在两套计划打通之前，禁止用 `score: 1` 冒充这一页的完成。
4. 列表按更新时间；误完成可恢复；任务标题和截止日期可改可清空。
5. 续学草稿与地图草稿分槽，或在覆盖前提示。聊天 SSE / `[AI已回复]` 留给 PR #2 那条线，合并时以活的 `frontend/` 为准，不要只改 `archive/frontend`。
