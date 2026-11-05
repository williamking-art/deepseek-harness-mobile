# 中转 5xx/非 101 升级失败后必须重试（0.1.26 增补）

- 日期：2026-11-05
- 触发：另一台电脑在 09:19:35 重启中转容器（新增它自己的 `ma-li-desktop` 令牌），
  本机连接器升级拿到 **一次 502** 后，面板停在「重连中 · 中转拒绝升级 (HTTP 502)」，
  nginx 日志显示此后 **7 分钟没有任何重拨**；人工重载插件才恢复。
- 状态：implemented（0.1.26）

## 问题

`transport/relay-link.ts` 原先只在 ws 的 `close` 事件里排退避重连；
非 101 升级走 `unexpected-response` → `options.onFailure(...)` → `request.destroy()`，
**指望随后会来一个 `close`**。实测（2026-11-05）在 502 上 `close` 没来：
`control` 一直是旧 socket、`retry` 从未排上，于是永久停在「重连中」。

中转重启窗口里的 502 是**可自愈**的（几秒后 upstream 就绪），连接器必须退避重试，
不能要求人工重载。

## Decision

1. 抽出 `dropControl(socket)`：控制连接掉线的**唯一收口**，负责清 `control`/看门狗、
   关闭数据面、并 `scheduleRetry()`（幂等，已有定时器不重复排）；
2. `close` 与 `unexpected-response` 都走 `dropControl`；
3. 5xx 额外记一条 warn（"多为中转重启窗口，进入退避重连"），便于事后判断；
4. 401/403/404 仍走 `describeUpgradeFailure` 的自解释文案，但**同样重试**——
   这样令牌在别处被修好后，本机会自己恢复，不用人工重载。

## Alternatives considered

- **只对 5xx 重试、4xx 终止**：4xx 是身份/地址错误，终止能让用户看到明确错误；
  但现场修复令牌后还得人工重载。重试成本只有 ≤30 秒一次请求，收益是"自己恢复"，选重试。
- **进程级看门狗**：在插件外再起一个看门狗轮询面板状态，过重且要额外进程。

## 验证

- `tsc` 0 / `build` ok / `pnpm test`（844 pass + 1 skipped，Dart 探针按设计 skip）；
- 真机复验需要一次中转重启（属**需明确授权**的操作）；下次授权窗口里观察：
  502 之后 ≤30 秒应出现新的 `GET /ws/connector`，恢复后 `connector.open`。

## 代码入口回指

- `connector/src/transport/relay-link.ts`（`dropControl` / `scheduleRetry` / `unexpected-response`）
