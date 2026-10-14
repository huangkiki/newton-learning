# Learning 迁移与 solver 导读核对

本次将当前仓库显示名改为 Newton Physics Learning，并将现行入口改到 `newton-learning`、总仓改到 `sim-learning`。正文、课程、开发导航与工程规则沿用新名称；原生 solver/API/模型身份没有因品牌迁移而改名。

新增 [solver 标签解读](solver-reading-guide.md)覆盖 DexLab 固定 taxonomy 中 Newton Physics 的 14 个 token（其中 `semiimplicit` / `semi_implicit` 是同一展示名的两个索引）。正文中文，配英文术语、摘要及 README 英文入口，复用 E3/E6 深入课程。先修教材通过总仓统一导航，不把教材方法追认为引擎默认配置。

## 保留的历史例外

- `docs/e1-validation.md` / `.json` 至 `docs/e6-validation.md` / `.json` 共 12 份交付快照保持逐字不变，其中旧仓库 Issue URL、路径和当时描述用于追溯原始证据。
- 既有官方固定提交、源文件 blob、模型和 API 标识保持不变。
- `docs/roadmap.md` 的 E1–E6 复审正文保留其当时的依赖判断，链接已迁移，顶端明确说明这些是历史记录。

历史链接是枚举过的快照例外，不是现行导航漏改。E7 的安装/总审校和原生运行继续暂停。

## 核对范围

基线为本仓提交 `7e31bbab6225a28605c69d2c782f2f1f14d6e3d4`。官方源码使用 Newton `713fecdc41caf0c9d726f5c016939f36e66e3dff`；DexLab 资料使用 `7e4a0774efb4f8b332e866e1495e032b4969c8aa`。本次重新下载的 14 份官方文件与既有 [sources.json](sources.json) 的 Git blob 一致；7 份 DexLab 斜面 profile 的 SHA-256 与其固定索引一致。此处没有重新审计整棵引擎源树，也没有核验历史 `1.7.0.dev0 @ 2dee3234` 的全部设置。

本地检查通过（2026-10-14）：`git diff --check`、`python3 scripts/check_docs.py`、`python3 scripts/check_examples.py`（8 个 Python 示例、5 个 Markdown Python 片段，只解析不执行）。新导读的 24 个固定来源链接均可下载，数字行号在对应源文件范围内；14 个 taxonomy token 已按显示名/别名覆盖，12 份历史验收快照与基线逐字相同。上述检查仅证明文本链接、语法与所读源文件身份，不证明候选构造片段可以运行或物理结果正确。远端交付以最终 PR 与合入提交为准。

English: current navigation uses Learning names. Twelve E1–E6 validation snapshots remain unchanged. The new guide is source/static teaching only; E7 and native execution remain deferred.
