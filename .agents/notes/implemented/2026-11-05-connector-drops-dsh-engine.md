# connector 去掉 DSH 版本限制（0.1.26）：先能装，再谈适配

- 日期：2026-11-05
- 触发：DSH 0.2.2-rc.1 已推送（nightly 通道，staging 30%）。0.2.2 起第三方插件兼容性
  **优先按 `engines.dsh` 判断**（声明了就只按它判、peer 被忽略）；connector 0.1.24 的
  `engines.dsh = ">=0.2.1-rc.5 <0.2.2"` 在 semver 预发布规则下**不包含 0.2.2-rc.1**，
  会被判不兼容、装不上，也就无法做兼容冒烟。
- 状态：implemented（0.1.26）

## Decision

一个提交里做两件事：

1. `package.json` 删掉 `engines.dsh`（保留 `engines.node`）；
2. `peerDependencies` 去掉 `@deepseek-ai/dsh-typert-protocol` 的精确钉死（`0.2.1-rc.5`），
   `@deepseek-ai/cordis` / `@deepseek-ai/schemastery` 放开为 `>=` 范围。

因为 0.2.2 的兼容回退规则是"没有 `engines.dsh` 时，每个 `@deepseek-ai/dsh` 或
`@deepseek-ai/dsh-*` peer 都必须匹配"——只删 `engines.dsh`、留着精确 peer，仍然会被挡。

## 为什么敢去掉

- 去掉的是**安装期门槛**，不是运行期保障：装上去之后该崩还是崩，冒烟会立刻暴露；
- 我们的发布链路是 fork 自管（fleet 清单 + 版本端点），不面向公众分发，风险面可控；
- 先解绑才能在 0.2.2-rc.1 的**临时环境**里测；测完再决定是恢复范围还是继续放开。

## 风险与回退

- 风险：0.2.2 若改了 typert 协议/cordis 语义，运行期可能出错（正是冒烟要查的）；
- 回退：把 `engines.dsh` 与精确 peer 加回，重发一版即可（清单/版本端点都按版本号走）。

## 验证

- `pnpm exec tsc --noEmit` / `pnpm build` / `pnpm test` 全绿；
- 装到本机（0.2.1-rc.5）后连接器仍正常连中转（本次实测 `connector.open` 恢复）；
- 0.2.2-rc.1 上的实测另起一轮（见 `proposed/2026-11-05-absorb-official-mobile-022.md` §5）。

## 代码入口回指

- `connector/package.json` 的 `engines` / `peerDependencies`。
