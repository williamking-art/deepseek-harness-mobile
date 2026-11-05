# 连接器身份绑机器、不绑网络：一机一 id + 401 自解释（0.1.26 增补）

- 日期：2026-11-05
- 触发：本机面板「中转重连中」，nginx 上全是 `GET /ws/connector?id=local → 401`。
  排查：中转配置里 `local` 的摘要变成 `8a16d6d9…`，本机凭据还是 `975ccfbf…` ——
  **另一台电脑重签了同名连接器 id 的令牌**，把本机踢下线。
- 状态：implemented（0.1.26 的第二提交）

## 问题

1. 两台电脑都用默认 id `local`；中转按 id 存令牌摘要，**同 id 重签 = 覆盖**，
   另一台一更新/重签，本机立刻 401，面板只显示「重连中」。
2. 401 的旧文案只说"重签一张贴进来"，没告诉用户**根因是 id 撞了**，于是每次都在现场写令牌。
3. 电脑端身份本来只与 **connectorId + 令牌**绑定（与网络无关），但 id 撞车让"换环境/更新"看起来像网络问题。

## Decision

1. **一机一 id**：默认仍 `local`，但新增 `DSH_MOBILE_CONNECTOR_ID` 环境变量 ——
   首次读到就写进存储的 `global.connectorId`（之后以存储为准），
   让另一台电脑可以不改存储文件就切到自己的 id（如 `home-pc`）。
2. **401/403 文案自解释**：明确写出"常见原因是另一台电脑重签了同名 id 的令牌
   （同 id 重签会覆盖）"，并指出每台电脑要各用一个唯一 id。
3. **配对/身份与网络解耦**：身份只来自 `connectorId` + 凭据 seam 里的令牌，
   重连沿用同一份凭据；网络切换只会触发指数退避重拨，不会改变身份。
4. **一次性绑定工具**：`tools/dsh-fleet-sync/bind-connector.ps1`
   （在其他电脑上运行：写 `connectorId` + 令牌，重启 DSH 生效），另有配套 bundle。

## 验证

- `tsc` 0 / `build` ok / `pnpm test` 844 pass + 1 skipped（去掉 PATH 里的 Dart 后，
  跨语言用例按设计 skip；有 Dart 时它需要先 `flutter pub get`，本机尚未做）；
- 本机换网络（VPN 开关、IP 从 192.168.1.34 → 10.1.100.159）后重拨仍使用同一身份；
- 用新令牌 curl 升级 = 101；中转日志 `connector.open connectorId="local"`。

## 代码入口回指

- `connector/src/index.ts`（`DSH_MOBILE_CONNECTOR_ID` 读取与持久化）
- `connector/src/transport/relay-link.ts`（`describeUpgradeFailure` 的 401/403 文案）
