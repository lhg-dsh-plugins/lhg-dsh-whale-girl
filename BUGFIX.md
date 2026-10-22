# 本版修复（BUGFIX）

## 标题

无 ref 的 git 安装查上游时落进 GitHub 提交**列表**端点，永远判「无法确认上游版本」（上游 issue #30）。

## 证据链

- 上游 open issue [#30](https://github.com/vlln/whale-girl/issues/30)（2026-10-17）描述该缺陷与成因；
  在上游固定 SHA `4b4350478c82d22d506a93567e52ab0ff90c2585`（v0.2.3）上核对：`lib/src/update-host.mjs`
  的 `gitUpstream` 在 `parsed.ref === undefined` 时后缀为空，请求 `…/commits`，缺陷仍在。
- 修复前最小复现（假网络忠实模拟 GitHub：裸 `/commits` 返回数组、无顶层 `sha`；`/commits/HEAD`
  返回带 `sha` 的单提交对象；安装来源 `github:vlln/whale-girl`、锁文件在旧提交）：
  请求实际打到 `https://api.github.com/repos/vlln/whale-girl/commits`，结果
  `state: unknown | latest: {}`，脚本 exit 1。
- 基线测试：修复前 `npm test` 247 通过 / 0 失败——缺陷不在既有覆盖内
  （`tests/update-host.test.mjs` 原有 git 夹具都带 `#main`，无 ref 分支从未被走到）。

## 根因

`gitUpstream` 把「没有 ref」拼成「没有路径段」：GitHub 的 `GET /repos/{o}/{r}/commits` 是
提交列表端点，返回数组而非单提交对象，`payload?.sha` 恒为 `undefined`，于是 `decideUpdate`
对这类安装永远只能判 `unknown`。`parseInstallSource` 认无 ref spec（语义为跟默认分支）、
`updateSpecs` 也为它生成两步更新计划，唯独查上游这一步从未成立。

## 改动

- `lib/src/update-host.mjs`（唯一实现改动）：`gitUpstream` 改为恒带路径段，
  无 ref 时用 `parsed.ref ?? 'HEAD'` 查 `commits/HEAD`（GitHub 将 HEAD 解析为仓库默认分支尖端），
  与显式 ref 同一请求路径；注释写明裸 `commits` 是列表端点、路径段不能省。`latest` 不带 `ref`。
- `tests/update-host.test.mjs`：新增回归用例——无 ref 来源只认 `/commits/HEAD` 的夹具下，
  上游新提交判 `update-available`（且 `latest.ref === undefined`）、同提交判 `up-to-date`。
- `decisions/implemented/bug-fix/2026-10-17-update-ref-less-git-default-branch.md`：按上游
  AGENTS.md 的决策记录契约新增记录，并与 `feature/2026-10-05-plugin-update-row.md` 双向互链
  （部分取代其「git 安装问 `commits/<ref>`」的无 ref 分支表述）。
- `package.json`：version 0.2.3 → 0.2.4；repository.url 改指本 fork 落点仓。其余字段不动。

## 复验结果

- 复现脚本修复后 exit 0：请求变为 `…/commits/HEAD`，`state: update-available`，
  `latest: {"sha":"b2b2…","commit":"b2b2b2b"}`。
- 反向核验：只回退 `lib/src/update-host.mjs` 时新增用例单独变红（22 通过 / 1 失败），恢复后转绿。
- `node --test tests/update-host.test.mjs`：23/23 通过。
- 全量 `npm test`：248 通过 / 0 失败（基线 247 + 新增 1），无回归。
- `node scripts/gates/run.mjs`：12 个门禁全部通过（含 verify-decisions 链接契约、verify-prose）。

## 边界

- 只修 git 安装的查上游一步；registry、本地路径安装路径未触碰。
- 依赖 GitHub `commits/HEAD` 的解析语义（HEAD → 默认分支尖端）；若上游改此语义，
  无 ref 安装会退回 `unknown`，不会判错线。
- 未在真实 DSH 宿主里端到端验证（沙箱无宿主）；验证面为上游自带单测 + 门禁 + 假网络复现，
  与仓库 AGENTS.md 对 `lib/src/` 改动要求的证据口径一致。
