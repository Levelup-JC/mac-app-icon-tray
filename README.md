# mac-app-icon-tray

给 macOS 应用做 **App/Dock 图标** 与 **菜单栏 Tray（template image）** 的可复用 skill。

范围只含两套图标资产、加载代码、验收与定版；不含应用内 UI。

## 安装

把本仓库的 `SKILL.md` 拷进你的 agent skills / workflows 目录，或按你使用的 agent 平台的 skill 安装方式引用本仓库。

## 何时用

触发词：App logo、Dock、icns、菜单栏图标、Tray、template image、图标发糊、菜单栏看不见。

## 硬规则摘要

1. App 与 Tray 两套母版分离；禁止把带底板的 App 图标缩小当 Tray。
2. Tray：纯黑 `#000` + alpha，透明底；命名 `trayTemplate.png` / `trayTemplate@2x.png`。
3. **不要**对 Tray 调用 `resize()`（Retina 会糊）。
4. App：1024 母版 + Apple 边距 + `.icns` 打包。

全文见 [SKILL.md](./SKILL.md)。

## 来源

由 Jason / Levelup 实测（Electron Tray）后经 Codex Opus 审核定稿。

## License

MIT
