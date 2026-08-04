# Pi Coding Agent 主题分析报告

> 生成时间: 2026-07-23
> 来源: Pi 官方文档 + npm 搜索

---

## 一、Pi 主题系统概述

Pi 支持通过 JSON 文件自定义 TUI 配色，主题文件包含 **51 个必填颜色 token**。

### 内置主题
| 主题 | 说明 |
|------|------|
| `dark` | 深色主题（默认） |
| `light` | 浅色主题 |

### 主题文件位置（加载顺序）
| 位置 | 作用域 |
|------|--------|
| 内置 | `dark`, `light` |
| `~/.pi/agent/themes/*.json` | 全局 |
| `.pi/themes/*.json` | 项目（需信任） |
| npm/git packages | `themes/` 目录或 `package.json` 中声明 |
| `--theme <path>` CLI 参数 | 单次运行 |

### 主题格式
```json
{
  "name": "my-theme",
  "vars": {
    "primary": "#00aaff",
    "gray": 242
  },
  "colors": {
    "accent": "primary",
    "border": "primary",
    "text": "",
    ...
  }
}
```

支持 4 种颜色值格式: Hex (`#ff0000`)、256 色索引 (`39`)、变量引用 (`"primary"`)、终端默认 (`""`)

**亮点**: 编辑主题文件后自动热重载，即时生效。

---

## 二、可安装的主题包（npm）

### 纯配色主题

| #   | 主题名称                  | npm 包                                 | 风格描述                           | 版本    | 作者        |
| --- | --------------------- | ------------------------------------- | ------------------------------ | ----- | --------- |
| 1   | **Tokyo Night**       | `@wishx127/pi-tokyo-night`            | Tokyo Night 配色 + Powerline 状态栏 | 1.0.4 | ny9u      |
| 2   | **Tokyo Night Storm** | `pi-theme-tokyo-night-storm-improved` | Tokyo Night Storm 改良版          | 1.0.0 | h1v35     |
| 3   | **Synthwave '84**     | `pi-theme-synthwave-84`               | 赛博朋克霓虹风                        | 1.0.1 | robzolkos |
| 4   | **Cyberdyne**         | `ameno-cyberdyne`                     | 高对比度赛博朋克（电青/热粉/酸绿/琥珀金）         | 1.0.1 | ameno     |
| 5   | **Jellybeans**        | `@aliou/pi-theme-jellybeans`          | Jellybeans Mono（深色+浅色双版本）      | 0.1.6 | aliou     |
| 6   | **Nerisma**           | `@nerisma/pi-theme-nerisma`           | 深紫色暗色主题                        | 1.0.0 | nerisma   |
| 7   | **Pi Coder Theme UI** | `pi-coder-theme`                      | 完整 UI 套件（主题+编辑器+紧凑工具渲染）        | 0.1.0 | vurihuang |

### 实用工具类

| #   | 工具名称           | npm 包                                 | 功能                            |
| --- | -------------- | ------------------------------------- | ----------------------------- |
| 1   | **系统主题同步**     | `pi-system-theme`                     | 跟随 macOS 深色/浅色外观自动切换          |
| 2   | **终端主题同步**     | `pi-theme-sync`                       | 跟随终端配色（本地 + SSH，通过 OSC 11 查询） |
| 3   | **Ghostty 同步** | `@ogulcancelik/pi-ghostty-theme-sync` | 跟随 Ghostty 终端配色               |
| 4   | **自动主题同步**     | `pi-auto-theme`                       | 自动跟随 OS 外观                    |
| 5   | **主题切换器**      | `@codewithkenzo/pi-theme-switcher`    | 会话中实时切换/预览主题                  |
| 6   | **主题选择器**      | `pi-theme-picker`                     | 添加 `/theme` 命令，交互式切换主题        |
| 7   | **主题同步(备)**    | `@sherif-fanous/pi-theme-sync`        | 自动跟随终端或 OS 外观切换               |

---

## 三、安装与使用

### 安装主题包
```bash
pi install npm:<package-name>
```

示例:
```bash
pi install npm:@wishx127/pi-tokyo-night
pi install npm:pi-theme-synthwave-84
pi install npm:ameno-cyberdyne
```

### 启用主题

编辑 `~/.pi/agent/settings.json`:

```json
{
  "theme": "tokyo-night"
}
```

或通过 Pi 内的 `/settings` 命令交互设置。

### 创建自定义主题
```bash
mkdir -p ~/.pi/agent/themes
```

创建 `~/.pi/agent/themes/my-theme.json`，参考 [官方 schema](https://raw.githubusercontent.com/earendil-works/pi/main/packages/coding-agent/src/modes/interactive/theme/theme-schema.json) 定义 51 个颜色 token。

---

## 四、颜色 Token 分类

### 核心 UI（11 色）
`accent`, `border`, `borderAccent`, `borderMuted`, `success`, `error`, `warning`, `muted`, `dim`, `text`, `thinkingText`

### 背景与消息（11 色）
`selectedBg`, `userMessageBg`, `userMessageText`, `customMessageBg`, `customMessageText`, `customMessageLabel`, `toolPendingBg`, `toolSuccessBg`, `toolErrorBg`, `toolTitle`, `toolOutput`

### Markdown（10 色）
`mdHeading`, `mdLink`, `mdLinkUrl`, `mdCode`, `mdCodeBlock`, `mdCodeBlockBorder`, `mdQuote`, `mdQuoteBorder`, `mdHr`, `mdListBullet`

### Diff（3 色）
`toolDiffAdded`, `toolDiffRemoved`, `toolDiffContext`

### 语法高亮（9 色）
`syntaxComment`, `syntaxKeyword`, `syntaxFunction`, `syntaxVariable`, `syntaxString`, `syntaxNumber`, `syntaxType`, `syntaxOperator`, `syntaxPunctuation`

### 思考级别边框（7 色）
`thinkingOff`, `thinkingMinimal`, `thinkingLow`, `thinkingMedium`, `thinkingHigh`, `thinkingXhigh`, `thinkingMax`

### Bash 模式（1 色）
`bashMode`

### HTML 导出（可选）
```json
{
  "export": {
    "pageBg": "#18181e",
    "cardBg": "#1e1e24",
    "infoBg": "#3c3728"
  }
}
```

---

## 五、注意事项

1. **无集中主题画廊** — Pi 主题社区还在早期，目前通过 npm 分发，可用 `npm search pi-theme` 发现最新主题
2. **颜色格式** — 推荐使用 Hex 24 位色（大部分现代终端支持），256 色作为 fallback
3. **深色/浅色** — 深色终端用饱和高明度，浅色终端用暗沉低对比
4. **VS Code 用户** — 建议设置 `terminal.integrated.minimumContrastRatio` 为 `1` 以获得准确颜色
5. **检查 truecolor 支持**: `echo $COLORTERM` 应输出 `truecolor` 或 `24bit`

---

## 六、发现更多主题

```bash
# 搜索主题包
npm search pi-theme
npm search pi-coding-agent

# 搜索 GitHub
# https://github.com/topics/pi-coding-agent-theme
# https://github.com/topics/pi-theme
```
