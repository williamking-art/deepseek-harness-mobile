# fork 自用连接器：默认中转域与更新清单回到我们自己的地址

- 日期：2026-11-05
- 触发：从 upstream main（f4cdf79）重建的 connector 0.1.22/0.1.23 **不拨号**；
  对比官方 0.1.21 产物发现两处默认值差异。
- 状态：implemented（0.1.24）

## Problem

官方 **0.1.21 的产物**里：

- `BUILTIN_RELAY_URL = wss://dsh.songzuo.online/ws/connector`；
- 检查更新问的是 `https://dsh.songzuo.online/version/connector.json`（自建清单）。

但 **upstream main 的源码**还停在旧值：

- `BUILTIN_RELAY_URL = wss://dsh.0t0.me/ws/connector`（旧中转域名，已停用）；
- `REGISTRY = https://registry.npmjs.org/dsh-mobile-connector/latest`。

也就是说，0.1.21 是从一个**尚未进 main 的分支/发布流程**里出来的；从 main 重建时，
如果 profile 里的 `config` 是空的（我们的行就是 `config: {}`），连接器会按默认值拨向
已停用的 `dsh.0t0.me` —— 表现是「进程在、8790 在、但中转侧完全看不到 `connector.open`」。
本地实测：0.1.22/0.1.23 装上后不拨号，回装 0.1.21 立即连上。

## Decision

在 fork 里把这两处默认值对齐官方产物：

1. `connector/src/config.ts` 的 `BUILTIN_RELAY_URL` → `wss://dsh.songzuo.online/ws/connector`；
2. `connector/src/admin/update.ts` 的 `REGISTRY` → 我们的 `/version/connector.json`
   （应答 `{version, url}`；有 `url` 就用它，否则回退 npm 包页）。

## Alternatives considered

1. **在 profile patch 里显式写 `relayUrl`**
   - 最强理由：不动源码，改配置即可；对所有版本生效。
   - 为何没选：只修了本机；别台机器/新装仍然是空 config + 旧默认值，fleet 里会再踩一次。
     源码默认值对齐才是"装完即用"。
2. **等上游把发布分支合进 main**
   - 最强理由：最干净，不产生 fork 偏离。
   - 为何没选：今晚就要可用；等合并不可控。之后若上游合并，本提交可回退。
3. **保持 main 原样，显式写死配置到每个 profile**
   - 最强理由：源码零偏离。
   - 为何没选：同上，配置是每机一份的第二处真值。

## Consequences

- 收益：从 main 重建的连接器默认就能连上我们的中转；检查更新也不再依赖 npm registry（国内更稳）。
- 代价：fork 与本仓 main 有两处默认值差异；上游合并后需回退本提交（或改成读发布配置）。
- 不变：`relayUrl` 仍是 `.volatile()` 可覆盖配置，用户改地址依旧走面板。

## 验证

- 本机：0.1.24 安装后（重启 DSH 生效）中转侧应出现 `connector.open`；
- `pnpm exec tsc --noEmit` / `pnpm build` / `pnpm test` 全绿；
- 线上清单 `https://dsh.songzuo.online/version/connector.json` 返回 0.1.24 与 tgz 地址。

## 代码入口回指

- `connector/src/config.ts` 的 `BUILTIN_RELAY_URL`。
- `connector/src/admin/update.ts` 的 `REGISTRY` 与 `check()` 的 `url` 取值。
