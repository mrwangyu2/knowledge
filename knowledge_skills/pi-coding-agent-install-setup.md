# Pi Coding Agent 安装配置记录

> 生成时间: 2026-08-04  
> 参考源: [earendil-works/pi](https://github.com/earendil-works/pi) · [mattpocock/skills](https://github.com/mattpocock/skills) · npm `pi-coder-theme` by vurihuang

---

## 一、前置要求

| item | version/说明 |
|------|----------|
| Node.js | ≥20 (Pi fork 推荐 LTS) |
| npm     | 随 Node 自带即可（也可配置 `yarn/pnpm`） |
| OS      | macOS / Linux / Windows(WSL或PowerShell/Cygwin/msys2支持，[Windows文档](packages/coding-agent/docs/windows.md)) |
| 终端    | Truecolor(True Color) 终端以正常显示主题 |

---

## 二、安装 Pi Coding Agent（基于 earendil-works/pi）

### 1. npm 全局安装（推荐）

```bash
npm install -g --ignore-scripts @earendil-works/pi-coding-agent
```

`--ignore-scripts`: 不运行第三方依赖的 lifecycle scripts，Pi 自身不需要。

> 也可以通过以下 installer 脚本安装（底层也是调用 node/npm）:
> ```bash
> curl -fsSL https://pi.dev/install.sh | sh
> ```

### 2. API Key / Provider 配置

Pi 支持两种认证方式：环境变量或交互式登录。

**方式 A — 环境变量 (推荐，简单):**

```bash
# macOS/Linux
export ANTHROPIC_API_KEY="sk-ant-your-key-here"

# Windows PowerShell
$env:ANTHROPIC_API_KEY="sk-ant-your-key-here"

# 或写入 shell profile（~/.zshrc / ~/.bashrc）持久化：
echo 'export ANTHROPIC_API_KEY="your-key"' >> ~/.zshrc && source ~/.zshrc
```

**方式 B — 交互式订阅登录( Claude Pro/Max, GitHub Copilot等):**

首次启动时:

```bash
pi
/login   # Pi TUI 中运行，按提示选择 provider 并登录
```

支持 Provider：Anthropic（API key）, OpenAI, Google Gemini, DeepSeek, Mistral, Groq, xAI, Vercel AI Gateway, Cloudflare Workers AI等。部分也支持订阅（ChatGPT Plus/Pro Codex、Claude Pro/Max）。

### 3. 配置模型 & Thinking Level(可选)

推荐在 `~/.pi/agent/settings.json`中设置默认：

```json
{
  "defaultProvider": "anthropic",
  "defaultModel": "claude-sonnet-4-20250514",
  "defaultThinkingLevel": "medium"
}
```

也可以每次启动时覆盖:

```bash
pi --provider anthropic --model claude-sonnet-4-20250514 --thinking medium
```

### 4. 验证安装并启动

```bash
pi --version   # 确认成功安装，显示版本号
pi             # 启动交互模式: Ctrl+C twice 退出; /quit 也退出
```

---

## 三、安装主题：pi-coder-theme by vurihuang

[package.json](#)摘录信息（从 npm registry）

- **包名：** `pi-coder-theme`  
- **作者：** `vurihuang <vengeancehuang@gmail.com>`  
- **描述：** Pi Coder Theme UI suite for Pi: theme, editor chrome, and compact tool display.  
- **标签：** `pi-package`, `pi-theme`, `pi-extension`, `pi-coding-agent`

含义：这不仅是一个配色主题包，而是一个完整的 UI 扩展套件（自定义编辑器边框、紧凑工具渲染等），通过 Pi Package 标准分发。

### 1. 使用 pi install 安装(推荐)

进入你的工作目录然后在Terminal中运行:

```bash
pi install npm:pi-coder-theme
```

这会将包下载并装到 `~/.pi/agent/npm/pi-coder-theme`，Pi会自动加载其中的：

- 扩展（extensions）: 主题编辑器 UI、thinking steps展示工具等
- 主题文件（themes）
- Skills(如有)

### 2. 可选：项目级安装

如果你只希望某个项目生效，加 `-l`:

```bash
cd your-project-root
pi install npm:pi-coder-theme -l   # 装到 .pi/npm/pi-coder-theme
```

### 3. 启用主题/扩展状态检查

已安装的包默认启用，如需管理：

```bash
pi config    # TUI查看可启用的extensions/skills/themes，勾选或取消
```

---

## 四、安装 pi-theme-picker（通过 /theme 命令切换主题）

从 npm info 获取的信息：

- **包名：** `pi-theme-picker`  
- **描述：** Pi extension that adds a /theme command for switching themes.  
- **标签：** `pi-package`, `pi-extension`, `pi`

### 1. 安装

```bash
pi install npm:pi-theme-picker
# 或只作用于当前项目：
pi install npm:pi-theme-picker -l
```

重启 Pi，或在内部运行 `/reload`，即可使用新命令：

- **`/theme`** — 交互式主题选择器，列出所有可用主题（内置 + 已安装的主题包），选中即切换并持久化到设置。

### 2. 与现有主题的联动

配合之前安装的 `pi-coder-theme`（提供多个自定义 UI 样式）以及其他通过 npm/git 获得的 Pi Package 主题可以一站式切换：

```bash
/theme   # 例如选择 "Coder Theme"、"Tokyo Night"、“dark”等等
```

---

## 五、安装 mattpocock skills（可选，与 Claude Code/Codex/Pi等 Agent-Skills标准兼容）

### 1. 通过 npx / skills.sh 统一安装器安装

适合所有支持 `.agents/skills`标准的工作流。这将会把skills作为普通文件复制进仓库的 `.agents/skills/`目录供当前 agent直接发现（包括 Pi）

```bash
npx skills@latest add mattpocock/skills
```

会交互式提示选择：

- 要安装的 skills清单（包含 `setup-matt-pocock-skills`, `grill-with-docs`, `tdd`, `diagnosing-bugs`, `code-review`等）
- 目标 agent类型（Codex、其他兼容框架等），Pi用户也可以直接选择通用/Editable方式让 skills写进 `.agents/skills/`.

### 2. Pi 中发现 mattpocock skills

Pi按照 [Agent Skills standard](https://agentskills.io)自动扫描以下位置：

- `~/.pi/agent/skills/`（全局）
- `.pi/skills/` & `.agents/skills/`（从当前目录向上走到根目录，每个包含 SKILL.md 的目录即是一个 skill）
- packages中声明的 skills路径：通过 Pi Package安装

所以，当 `npx skills add mattpocock/skills`将 skills写入 `.agents/skills/`后，重新加载Pi即可发现:

```bash
# 如果已在 pi内部:
/reload    # 立即刷新加载新技能

# 或退出再进入
pi -c      # continue上次session
```

### 3. 运行初始化 skill（一次性）

建议首次安装时在项目里运行：

```bash
/skill:setup-matt-pocock-skills
# 或简称 /setup-matt-pocock-skills，Pi在启用skill命令后会自动注册
```

它会引导你:

- 选择 issue tracker(GitHub/Linear/local文件)
- 配置 triage 标签（配合 `/triage`使用）
- 设置 domain-related docs的存储位置(Context.md / ADRs路径)

之后即可在项目中使用如 `/grill-with-docs`, `/tdd`, `/diagnosing-bugs`, `/code-review`, `/implement`等。

---

## 五、目录结构小结（安装完成后）

```text
~/.pi/agent/config
├── settings.json              # Pi全局配置(model,theme...)
├── keybindings.json           # 快捷键(如有自定义)
├── trust.json                 # 项目信任记录
├── sessions/                  # 会话JSONL文件（按工作目录组织）
│   └── <cwd_hash>/session-*.jsonl
├── npm/                       # Pi Package：npm安装位置
│   └── pi-coder-theme         # ←你的主题包
│       ├── node_modules/
│       ├── extensions/
│       ├── themes/
│       └── package.json
├── git/                       # Pi Package：git源安装位置(如有用 git:install)

your-project-root/
├── .pi/settings.json          # (可选)项目局部覆盖配置
├── .agents/skills/            # mattpocock skills在此，Pi自动发现
│   ├── engineering/grill-with-docs/SKILL.md
│   ├── engineering/tdd/SKILL.md
│   └── ...其余skills按目录结构分布
├── CLAUDE.md / AGENTS.md      # 项目指导语（也会被 Pi加载）
└── CONTEXT.md                 # domain共享语言（通过skill生成）
```

---

## 六、快速启动示例

### (1) 全新启动

```bash
cd ~/my-project
export ANTHROPIC_API_KEY="secret_api_key"
pi --model claude-sonnet-4-20250514 --thinking medium -n "My project dev session"
/reload          # 确保技能与扩展生效
/skill:grill-with-docs   # or /grill-me
```

### (2) 恢复上次会话(在 Pi中通过快捷命令或 CLI):

```bash
pi -c     # continue上次最近的一个session
/r        # resume:浏览可选历史会话列表并选择继续的那个
```

---

## 七、提示

- **代理配置**: 如果你的网络需要HTTP代理，可以在 `~/.pi/agent/settings.json`中设置：
  
    ```json
    { "httpProxy": "http://127.0.0.1:7890" }
    ```

- Telemetry和版本检查默认开启，不想的话可在settings里关掉或设环境变量 `PI_OFFLINE=1`禁止启动期所有联网行为。

---

## 八个、相关文档链接

| Doc | Link / Command |
|-----|------------------|
| Pi 官方文档 | https://pi.dev/docs/latest |
| earendil-works/pi GitHub | https://github.com/earendil-works/pi |
| mattpocock/skills README 完整内容(本地存档) | [[mattpocock-skills.md]] |
| Skill标准说明 | https://agentskills.io |
