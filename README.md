# agent-mailbox

Harness（轻验证 / 量大便宜活）↔ 编程 Kun（要仓库上下文的编程活）之间的 **Git 信箱**。两边隔离、无隧道时，用本仓库交换任务与结果。

## 约定

| 路径 | 谁写 | 谁读 | 用途 |
|------|------|------|------|
| `tasks/task-<id>.md` | Harness 或用户 | 编程 Kun | 新任务 |
| `results/result-<id>.md` | 编程 Kun | Harness / 用户 | 结论与产物链接 |
| `status.json` | 双方都可更新自己那一侧字段 | 双方 | 看板：谁在忙、卡在哪 |

- 一个任务一个 id（建议短横线：`2026-11-03-login-fix`）。
- 任务文件存在且无对应 `results/result-<id>.md` = 待办或进行中；有结果文件 = 本轮结束。
- **不要**把密钥、token、私人聊天原文写进本仓库。
- 大代码改动不在本仓完成：编程 Kun 在目标业务仓库开 PR，这里只留 PR 链接与结论。

## 任务文件模板

见 `tasks/_TEMPLATE.md`。复制为 `tasks/task-<id>.md` 后填写。

## 结果文件模板

见 `results/_TEMPLATE.md`。复制为 `results/result-<id>.md`，`<id>` 与任务相同。

## status.json

```json
{
  "updated_at": "ISO-8601",
  "harness": { "state": "idle|working|blocked", "note": "" },
  "coding_kun": { "state": "idle|working|blocked", "note": "", "active_task": null }
}
```

## 谁干什么

- **Harness**：边界清晰的文档、样板、机械改写；把「需要动手改仓库」的活写成 `tasks/task-*.md`。
- **编程 Kun**：只接需要仓库上下文的活；用 CloudAgent 改代码、开/盯 PR；写回 `results/result-*.md` 并更新 `status.json`。
