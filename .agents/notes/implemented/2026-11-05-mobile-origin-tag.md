# 手机消息带来源标记 + 本地审计：会话里终于分得出谁发的

- 日期：2026-11-05
- 触发：用户问「现在这个插件没办法区分是移动端还是电脑端的 dsh 发出的信息吗」；
  实测确认不能（见下）。
- 状态：implemented（本机插件已用等价补丁验证过；本提交把同一逻辑落到 `connector/src`）

## Problem

DSH 的 prompt 接口**没有来源字段**：

```ts
// packages/api/session-controller/src/types.ts
export interface SessionPromptRequest {
  readonly requestId: SessionRequestId
  readonly sessionId: SessionId
  readonly mode: 'queue' | 'steer'
  readonly content: readonly PromptContentPart[]
  readonly clientTimeZone?: string
}
```

电脑 Web 与手机连接器走**同一个** `sessionController.prompt` 入口，落盘的 `user/message`
事件里也只有 `content` —— 会话层面天生区分不出"手机发的"与"电脑打的"。
（能区分的只有传输层的 `connectorId`/`deviceId`；那是路由元数据，不进会话。）

## Decision

在**通道这一侧**做两件事（用户 2026-11-05 选定的方案 1+4）：

1. **正文前缀（可见）**：`session.prompt` 投递前，给第一个 `text` 块前加 `【手机】`
   （`DSH_MOBILE_TAG` 可覆盖；空串 = 不打标只审计）。没有 text 块（纯图片）就补一个，
   保证每条手机消息都带标。电脑端输入不走这条通道，不受影响。
2. **本地审计（不记正文）**：追加一行到 `$DSH_HOME/logs/dsh-mobile-prompt-audit.jsonl`：
   `ts / iso / sessionId / requestId / mode / parts / chars`。写失败静默（旁路不该让
   消息投递失败）。

## Alternatives considered

1. **给 `SessionPromptRequest` 加 `source` 字段（内核方案）**
   - 最强理由：语义最干净，UI 能渲染成"来自手机"徽标，而不是正文里的一段前缀。
   - 为何没选：要改宿主 + 前端 + 事件兼容，且当前 schema 不接受额外字段；上游没有这个
     计划前，通道侧做不了。将来若宿主支持，应迁移过去并删掉前缀。
2. **只做审计、不加前缀**
   - 最强理由：零正文污染。
   - 为何没选：用户要的是"会话里一眼看出"；只在日志里查是事后行为，不能替代可见标记。
3. **只加前缀、不做审计**
   - 最强理由：少一个文件写入口。
   - 为何没选：前缀会进正文（复制/搜索都带），需要审计作为"内容之外的证据"；
     两者互补，成本都很低。
4. **把 `source.kind` 伪造成非 `user`**
   - 最强理由：完全不动正文。
   - 为何没选：DSH 的 UI 把 `source.kind !== 'user'` 的一律当上下文注入渲染
     （`ui-chat/src/client/conversation-nodes/message.ts:79` 一带），消息会从"用户气泡"
     变成"上下文注入"，比前缀更误导。

## Consequences

- 收益：会话里能直接看出哪条是手机发的；审计可按时间/会话核对，且不含正文。
- 代价：正文前多 4 个字符的标记（复制、搜索会带上）；`DSH_MOBILE_TAG` 可换更短的标记，
  但"空串"只用于调试。
- 代价：手机侧编辑/引用该消息时也会看到前缀（通道没有"只对发送端隐藏"的机制）。
- 不变：审批/提问回答（`source.kind='user-question-reply'`）不走 `session.prompt`，
  本条不覆盖它们 —— 需要时另做。

## 验证

- 本机等价补丁（打在已安装的 `lib/index.js` 上）2026-11-05 实测：
  - 手机发来一句话 → 会话 `user/message` 文本以 `【手机】` 开头（`seq 2876`，`rpcId`
    与审计里的 `requestId` 一致）；
  - `~/.dsh/logs/dsh-mobile-prompt-audit.jsonl` 新增一行，`mode=steer`、`chars=42`；
  - 电脑端输入无前缀。
- 本提交（源码版）验证：`pnpm exec tsc --noEmit`、`pnpm build`、`pnpm test` 全绿；
  `pnpm test` 的现有用例没有钉"正文不得被改写"的断言（若有，需同步调整）。

## 代码入口回指

- `connector/src/handlers.ts` 的 `tagMobilePrompt` / `auditMobilePrompt` 与
  `'session.prompt'` handler。
