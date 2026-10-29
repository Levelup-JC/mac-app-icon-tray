---
name: mac-app-icon-tray
description: >-
  为 macOS 应用（Electron 或原生 AppKit/SwiftUI）设计并落地两套图标资产时使用：App/Dock 图标（icon.png /
  .icns）与菜单栏 Tray 图标（template image / NSStatusItem）。触发词：App logo、Dock
  图标、icns、菜单栏图标、menu bar icon、status bar item、Tray、template
  image、图标发糊、菜单栏看不见。不含应用内 UI、窗口外观、交互流程。
---
# macOS App 图标 + 菜单栏 Tray 图标

**范围**：只做两套图标资产（App/Dock 图标、菜单栏 Tray 图标）及其加载代码、验收、定版。
**不在范围**：应用内界面、窗口外观与布局、颜色系统、交互流程、快捷键等产品 UI；Windows/Linux 的托盘资产（规则不同，另行处理）。

## 硬规则

1. **两套资产独立，各有母版。**
   - App/Dock 图标：彩色、可有底板与材质，母版 1024×1024，**不设** template。
   - 菜单栏 Tray：透明底、仅 glyph、单色 template，母版为矢量（SVG），**必须**是 template image。
   - 禁止把带底板的 App 图标缩小后直接当 Tray 用。
2. **Tray 像素规则**：只有纯黑 `#000000` + alpha 通道；背景 alpha=0；层次只能靠 alpha，不要用灰度或彩色（系统渲染 template 时只读 alpha，颜色会被丢弃）。
3. **Tray 尺寸与命名固定**：
   - `trayTemplate.png`：高 16px（宽随 glyph，常见 16–22px）
   - `trayTemplate@2x.png`：1x 的两倍（高 32px）
   - glyph 本体高度约 12–16pt，外框不超过菜单栏高度（22pt）。
4. **不要在代码里对 Tray 图调用 `resize()`**。按命名加载，让系统挑 @2x；resize 会丢掉 2x 表示，Retina 屏上发糊。
5. **Tray 笔画按菜单栏标准单独定**，不从大图标等比缩放：@2x 下约 2–3px（≈1–1.5pt），直线和端点对齐像素网格；最终以与系统图标并排时的视觉重量为准。
6. **App 图标按 Apple macOS 图标模板留边距**：1024 画布上圆角矩形主体约 824×824，居中，四周透明（阴影落在边距内）。不要满版。具体数值以 Apple Design Resources 当期模板为准。
7. **打包图标以 `.icns` 为准**：`app.dock.setIcon()` 只影响当次运行，不改 bundle 图标，不能代替打包配置。
8. **用户选定前不装机**；选定后的每次调整只改一个变量（见步骤 5）。

## 步骤

### 1. 定母题与入口
- 和用户确认：隐喻/字形（例如首字母、工具本身的物件、一个抽象符号）、App 图标是否允许底板与材质、需要的状态数（例如正常/活动/错误）。
- 确定换标说明文件位置，默认 `brand/ICON.md`（模板见文末）。

### 2. 出候选
- **App 图标**：出 3–5 版，1:1，按硬规则 6 留边距。可以用生图工具出图。
- **Tray**：每版 App 图标配一个从中提炼的 glyph，用 SVG 手工绘制或简化，不直接用生图结果（位图生成的线稿有灰度、噪点、不对齐像素）。
- 呈现方式：把每版做成「1024 App 图标 + 16/32px Tray 在浅色、深色条上的预览」编号拼成一张图（或单页 HTML），让用户回复编号。

### 3. 导出

| 用途 | 文件 | 规格 |
|------|------|------|
| App 母版 | `icon-1024.png` | 1024×1024，RGBA，带边距 |
| App 打包 | `icon.icns` | 由 iconset 生成（见下） |
| Tray 1x | `trayTemplate.png` | 高 16px，纯黑 + alpha |
| Tray 2x | `trayTemplate@2x.png` | 高 32px，纯黑 + alpha |
| Tray 母版 | `tray.svg` | 矢量，按 16pt 画板绘制 |

生成 `.icns`（macOS 自带工具）：

```bash
mkdir icon.iconset
for s in 16 32 128 256 512; do
  sips -z $s $s icon-1024.png --out icon.iconset/icon_${s}x${s}.png
  sips -z $((s*2)) $((s*2)) icon-1024.png --out icon.iconset/icon_${s}x${s}@2x.png
done
iconutil -c icns icon.iconset -o icon.icns
```

Tray 从 SVG 光栅化到 1x、2x 两个文件（rsvg-convert、Inkscape CLI 等都可以），**分别从矢量导出**，不要从 2x 缩成 1x。小尺寸 1x 如果发糊，允许针对 1x 单独微调路径。

### 4. 接入代码

**Electron**

```js
const { app, Tray, nativeImage } = require('electron')
const path = require('path')

let tray // 模块级引用，防止被 GC 后图标消失

app.whenReady().then(() => {
  // 文件名以 Template 结尾 + 同目录 @2x，Electron 自动识别 template 与 2x
  const img = nativeImage.createFromPath(path.join(__dirname, 'assets', 'trayTemplate.png'))
  img.setTemplateImage(true) // 双保险
  tray = new Tray(img)
})
```

- 打包图标：electron-builder 用 `mac.icon: "build/icon.icns"`；Electron Forge 用 `packagerConfig.icon`（指向 icns，可省略扩展名）。
- 确认 `assets/` 被打进包（检查 `files` / `extraResources` 配置）；放在 asar 外的资源用 `process.resourcesPath` 取路径。
- 状态切换：换另一张 template 图（如 `trayActiveTemplate.png`）或用 `tray.setTitle()`；不要为状态改成彩色图，除非接受非 template 并自备浅色、深色两套，监听 `nativeTheme` 切换。

**原生（AppKit）**

```swift
// 强引用保存 statusItem，否则会被释放
statusItem = NSStatusBar.system.statusItem(withLength: NSStatusItem.variableLength)
let image = NSImage(named: "TrayIcon")! // Asset Catalog 中 Render As = Template Image
image.isTemplate = true
statusItem.button?.image = image
```

- App 图标放进 Asset Catalog 的 `AppIcon`（导入 1024 母版和各尺寸）。
- SwiftUI 的 `MenuBarExtra` 同样要用 template 资源。

### 5. 验收

**文件级（必须全过）**

```bash
sips -g pixelWidth -g pixelHeight -g hasAlpha trayTemplate.png trayTemplate@2x.png icon-1024.png
iconutil -c iconset icon.icns -o /tmp/check.iconset && ls /tmp/check.iconset
```

- 尺寸符合硬规则 3；三张图 `hasAlpha: yes`。
- Tray 四角 alpha=0、不存在非黑的不透明像素（可用任意图像库脚本抽查）。
- `.icns` 解包后含 16–512 的 1x 与 @2x 共 10 张。
- Electron 中 `img.isTemplateImage() === true`，`img.getScaleFactors()` 含 `2`。

**真机（截图留证）**
1. 浅色、深色菜单栏各截一张，与系统图标（Wi-Fi、电池、控制中心）并排：大小、粗细、垂直居中一致。
2. 点击 Tray：高亮态下图标正确反色，菜单或点击行为正常。
3. Retina 屏清晰；有外接非 Retina 屏时也看一次。
4. 打包后的 App 在 Dock、Finder、Cmd+Tab 中与系统 App 并排：大小一致，不显示默认图标或系统套的容器。
5. 刘海屏机型上菜单栏挤满时，图标宽度不过大。

**调整规则**：用户说偏大/偏小/偏粗时，只改 glyph 在画布内的大小、留白或笔画，画布尺寸不动；只有用户明确提到「点击区域」或「占位宽度」时才改画布宽度。每次只改一个变量，重新截图对比。

### 6. 定版
- 入库：`tray.svg`、`trayTemplate*.png`、`icon-1024.png`、`icon.icns`（或 Asset Catalog）。
- 更新 `brand/ICON.md`；需要时打 Git tag（如 `brand-v1`）。

## 常见坑

| 现象 | 原因 / 处理 |
|------|------|
| 菜单栏图标发糊 | 对 Tray 调了 `resize()`，或缺 @2x，或从 2x 缩成 1x → 按硬规则 3、4 重做 |
| 菜单栏是实心方块或灰块 | 背景不透明或不是 template → 背景 alpha=0 + template |
| 浅色/深色下看不见或发灰 | 用了灰色、半透明笔画 → 纯黑 + 足够的 alpha 与笔画宽度 |
| 比系统图标大 | 画布高超过 16pt 或 glyph 太满 → 固定画布，缩 glyph |
| 线太细或太粗 | 从大图标等比缩放 → 按硬规则 5 单独定笔画 |
| 运行一会儿图标消失 | Tray / statusItem 被回收 → 模块级或强引用 |
| 打包后没有图标 | 资源路径或打包配置漏了 → 查 `files` / `extraResources`，用正确路径 |
| Dock 显示默认图标 | 只调了 `app.dock.setIcon` → 配打包 `.icns`；必要时清图标缓存后重启 Dock |
| Dock 图标比别的大或被套容器 | 满版、没留边距 → 按 Apple 模板边距重导 |
| 刘海屏上图标被遮住 | 菜单栏空间不够 → 缩小宽度，避免带长 title |

## 检查清单

- [ ] App 图标与 Tray 两套母版分离（`icon-1024.png` / `tray.svg`）
- [ ] Tray：`trayTemplate.png` 16px 高 + `@2x` 32px 高，纯黑 + alpha，透明背景
- [ ] Tray 已是 template，代码里没有对它 `resize()`；App 图标不是 template
- [ ] App 图标按模板留边距，`.icns` 含 10 个尺寸并接入打包配置
- [ ] 浅色/深色、高亮态、Dock 并排截图已获用户确认
- [ ] `brand/ICON.md` 已更新，资产已入库

## `brand/ICON.md` 最小模板

```markdown
# 图标说明
- 母题：<一句话>
- App 母版：brand/icon-1024.png → build/icon.icns（生成命令见下）
- Tray 母版：brand/tray.svg → assets/trayTemplate.png / @2x（1x 高 16px）
- 规则：Tray 纯黑 + alpha、template、不 resize；App 按 Apple 模板留边距
- 加载代码位置：<文件:行>
- 打包配置：<electron-builder mac.icon / Forge packagerConfig.icon / AppIcon>
- 导出命令：<粘贴步骤 3 的命令>
- 版本：<tag 或日期>
```
