# 中转不再被客户端 RST 打挂：connection 级 error 收口

- 日期：2026-11-05
- 触发：线上 2026-11-05 00:23（UTC+8）中转进程 `exit 1`；崩溃前日志是
  `Unhandled 'error' event` + `Error: read ECONNRESET`（`TCP.onStreamRead`）。
  自动重启的 1–2 秒里连接器拿到反代的 502，而连接器 502 后不重拨 —— 面板一直显示
  「中转拒绝升级 (HTTP 502)」。
- 状态：implemented（本机部署已实测；真机/线上复验见「验证」）

## Problem

中转只做转发：连接器与手机都可能在任何一秒重置 TCP（App 切后台、Wi-Fi 切蜂窝、
反代空闲回收）。Node 对**没有 `error` 监听**的 socket 会把 `read ECONNRESET`
升级成 `uncaughtException` —— 默认行为是**整个进程退出**。

这不是"偶发网络问题"，而是**一条坏连接能杀掉整个中转**：所有用户的连接器与手机
同时掉线，且恢复要靠进程重启。

## Decision

在 `server.ts` 的 `http` 上挂 **connection 级** `error` 收口：

```ts
http.on('connection', (socket) => {
  socket.on('error', () => {
    // 连接级错误没有恢复动作：socket 已由 http/ws 自己销毁。
  })
})
```

判据是"错误的杀伤半径"：一条连接的错误只该杀这一条连接；进程级的
`uncaughtException` 兜底会把本来该暴露的编码错误也一起吞掉，等于把故障藏起来
（`main.ts` 头注本来就写着"不做接住所有异常继续跑"）。

## Alternatives considered

1. **进程级 `process.on('uncaughtException')` 兜底（只忽略 ECONNRESET/EPIPE）**
   - 最强理由：一行就能让线上不再 502，且与现有"优雅退出"逻辑不冲突。
   - 为何没选：它把**所有**连接的错误都吞在进程级，等于放弃"错误只影响一条连接"的
     边界；将来真正的编程错误（比如某处 socket 没挂 handler）会被同一条兜底掩盖。
     connection 级收口是同一件事的更小切口（本机部署试过进程级版本，能救，但收口位置
     不够准，换成了这一版）。
2. **把每个 `wss.handleUpgrade` 的 socket 单独挂 error**
   - 最强理由：更贴近"升级之后"的那批 socket。
   - 为何没选：`connection` 事件覆盖了升级前后的同一条底层 socket，且只挂一处；
     在三个 `handleUpgrade` 调用点上重复挂，只会多出三处要同步的地方。
3. **让连接器在 502 后自动重拨（客户端兜底）**
   - 最强理由：这次事故的"可见症状"是面板卡在 502，客户端兜底能更快恢复。
   - 为何没选：那是把"中转崩了"的正常后果藏起来；先修"崩"本身。客户端重拨是另一件事，
     需要单独评估（当前插件在非 101 升级失败后按设计不重试）。

## Consequences

- 收益：单条连接 RST 不再影响其他连接，也不再有"整台中转退出 → 全部 502"。
- 代价：这类错误不再出现在进程退出码里；排查时要看连接级日志（目前没有单独记，
  需要时再加 —— 不先加日志是为了避免被刷屏）。
- 不变：其他未处理异常仍按 Node 默认行为退出，运维仍应把 `restart: on-failure` 当作
  最后一道防线。

## 验证

- 本机部署（2026-11-05 00:26 起）：容器 `RestartCount=0`、`healthy`，连接器
  `connector.open` 正常，内存约 32 MiB。
- 原崩溃现场：容器 `die exit=1`，前一行是 `Error: read ECONNRESET`；补丁后未再复现。
- 变异校验（未做，说明原因）：无法在本机稳定复现"某条 socket 不带 error 监听"，
  只能靠真实 RST；线上观察窗口继续。

## 代码入口回指

- `relay/src/server.ts` 的 `http.on('connection', …)`（`createServer` 之后、`http.on('upgrade')` 之前）。
- 崩溃命令与恢复顺序见 `memory.md` 与 `docs/`（若后续补运维文档）。
