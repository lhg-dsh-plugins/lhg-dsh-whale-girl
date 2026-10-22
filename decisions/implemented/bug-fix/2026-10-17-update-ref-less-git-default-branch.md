# Decision: 无 ref 的 git 安装查 commits/HEAD 判上游

Status: implemented

## Problem

profile 允许不带 `#ref` 的 git spec（`github:vlln/whale-girl`，语义是跟仓库默认分支）：`parseInstallSource` 认这种写法，`updateSpecs` 也为它生成两步计划。但 `gitUpstream` 在 `ref` 为空时拼的是 `…/commits`——提交**列表**端点，返回数组、没有顶层 `sha`，于是 `payload.sha` 恒为 `undefined`、`decideUpdate` 判 `unknown`，界面永远显示「无法确认上游版本」。

这类安装既不会报「已是最新」，也永远发现不了新版；`tests/update-host.test.mjs` 原有的 git 夹具都带 `#main`，这条分支没被走到。

## Decision

`gitUpstream` 无 ref 时查 `commits/HEAD`（GitHub 把 `HEAD` 解析为仓库默认分支的尖端），与显式 ref 走同一条请求路径；`latest` 不因此带上 `ref`（它跟的是默认分支、不是某条具名线）。裸 `commits` 不能省成列表端点这一约束写进 `gitUpstream` 的注释里——端点形状（列表 vs 单提交）是外部 API 契约，不看注释就会再踩。

回归防线：`tests/update-host.test.mjs` 新增无 ref 用例，夹具只认 `/commits/HEAD`（落进裸 `/commits` 即抛「未预期的请求」），断言上游新提交判 `update-available`、同提交判 `up-to-date`、`latest` 不带 `ref`。

## 取代检查

部分取代 [feature/2026-10-05-plugin-update-row.md](../feature/2026-10-05-plugin-update-row.md)：该记录「检查走 Node half」一条里「git 安装问 `commits/<ref>`」的表述由本记录补上无 ref 分支；更新行其余语义（来源只认 profile spec、两步重装、本地安装跟默认分支等）仍归它。两条保持活跃并互链。除此之外无重叠。

## Alternatives considered

**无 ref 时先查仓库信息拿 `default_branch` 再查 `commits/<branch>`。** 与本地路径安装的 `localUpstream` 同形，但那条路是因为没有别的线索；git 安装多一次请求、多一个失败点，而 `commits/HEAD` 一次请求即可，且不依赖默认分支名。

**把列表端点的第一项当最新提交。** 列表按提交时间排序、第一项通常就是默认分支尖端，但「通常」不是契约：分页与排序语义都不是为单提交查询设计的，判错了是把别的线当成上游。

**在 `parseInstallSource` 里给无 ref 的 spec 补一个默认 ref（如 `main`）。** 把「跟默认分支」偷换成「跟一条猜出来的具名分支」：默认分支不叫 main 的仓库会跟错线，且 spec 归一层不该发明 profile 没记的事实。

## Consequences

- 不带 ref 的 git 安装从此能判三态：有新版可一键更新（两步计划本就为它生成过，只是从没被触发），同提交报已是最新。
- 每次检查对这类安装仍是一次 GitHub 请求，速率限制口径不变。
- 依赖 GitHub `commits/HEAD` 的解析语义（HEAD → 默认分支尖端）；若上游改此语义，无 ref 安装会退回 `unknown`，不会判错线。
