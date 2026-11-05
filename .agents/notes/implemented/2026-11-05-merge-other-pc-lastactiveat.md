# 合并另一台电脑的未提交差异（0.1.26）

- 日期：2026-11-05
- 背景：0.1.25 是在**另一台电脑**上构建的：那边的改动没提交/没推，直接在 0.1.21 发布线上
  打了「0.1.25」。本机这条线已经从 upstream main（f4cdf79）重建过 0.1.24（含手机来源标记、
  默认中转/更新清单修复）。两边因此各带一部分差异。
- 状态：implemented（0.1.26 = 合并后的产物）

## 合并了什么

从另一台电脑的 0.1.25 产物里**反向提取**并落到源码：

1. **设备「最后活跃」持久化**（`lastActiveAt`）：
   - `store/index.ts` 的 `devices` 表加 `lastActiveAt?: number`（随表落盘，重启后仍能显示）；
   - `presence.ts` 的 `createDevicePresence` 加 `onSeen(devId, at)`，在**上线 / 声明前后台 /
     改订阅 / 答话**四个时机上报；
   - `index.ts` 装配处**节流 60 秒**后写设备表（高频信号 × JSON 存储，写一次有真实代价）；
   - 面板设备行（`admin/routes.ts` 的 `DeviceRow`、`handlers.ts` 的 `devices.list`）带出
     `lastActiveAt`；`client.js` 的「最后活跃」列改为 `lastActiveAt ?? lastInteractedAt`。
2. **更新检查端点命名**：那边用的是 `UPDATE_ENDPOINT`；本机 0.1.24 用 `REGISTRY` 且已支持
   `{version,url}` 应答 —— 行为等价，**保留本机实现**（少一次重命名噪音）。

## 本机这条线独有的、必须保留的

- `【手机】` 来源前缀 + 本地审计（`handlers.ts` 的 `tagMobilePrompt` / `auditMobilePrompt`）；
- 配对降级可见（`lanOnly` 载荷 + 三处提示，来自 upstream main）；
- 默认中转域与更新清单指向 `dsh.songzuo.online`（0.1.24 修的同款问题）；
- **去掉 `engines.dsh` 与 `@deepseek-ai/dsh-*` 精确 peer**（0.1.26 的兼容性目的，见
  `2026-11-05-connector-drops-dsh-engine.md`）。

## 验证

- `pnpm exec tsc --noEmit` / `pnpm build` / `pnpm test`（845 项，844 pass + 1 skipped，跨语言用例）；
- 装到本机后：中转 `connector.open` 恢复、面板「最后活跃」显示、手机消息仍带 `【手机】` 前缀、
  审计文件仍追加（需真机发一条消息确认）。

## 代码入口回指

- `connector/src/store/index.ts`（`lastActiveAt` schema）
- `connector/src/presence.ts`（`onSeen` 上报）
- `connector/src/index.ts`（60 秒节流写表 + 面板行）
- `connector/src/handlers.ts`（`devices.list` 带出）
- `connector/src/admin/routes.ts`（面板 `DeviceRow`）
- `connector/client.js`（「最后活跃」列回退链）
