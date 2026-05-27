# CodeGraph 安装器 Skill

[English](README.md)

这个 Codex skill 用来让 CodeGraph 在本机的多个代码 agent 中自动可用。它会在需要时安装 CodeGraph CLI，初始化当前项目索引，注册 CodeGraph MCP Server，并挂载启动 Hook，让新会话可以自动检测和修复缺失配置。

当前实现面向 Colby McHenry 的 CodeGraph CLI 包 `@colbymchenry/codegraph`，MCP 注册流程使用 CodeGraph 自带的安装器。

## 提供能力

- 可直接克隆到 `~/.codex/skills/codegraph-bootstrap/` 的 Codex skill。
- 内置 PowerShell 脚本，集中放在 `scripts/` 目录。
- 为 Codex 和 Claude Code 写入启动 Hook。
- 为 OpenCode 写入本地启动插件文件。
- 为 Hermes 写入 `on_session_start` shell hook 配置。
- 提供 SQLite 查询兜底能力，查询食谱参考 `bennett-lee/codegraph-skill`，并适配当前 CodeGraph schema 和 C# 项目的 `method` 节点。

## 文件结构

```text
codegraph-bootstrap/
  SKILL.md
  README.md
  README.zh-CN.md
  agents/openai.yaml
  scripts/
    Ensure-CodeGraph.ps1
    Install-CodeGraphBootstrap.ps1
    Invoke-CodeGraphBootstrap.ps1
    Invoke-CodeGraphSql.ps1
  references/query-recipes.md
```

安装流程可能修改这些 agent 配置：

```text
~/.codex/config.toml
~/.codex/hooks.json
~/.codex/AGENTS.md
~/.claude.json
~/.claude/settings.json
~/.claude/CLAUDE.md
~/AppData/Roaming/opencode/opencode.jsonc
~/AppData/Roaming/opencode/AGENTS.md
~/AppData/Roaming/opencode/plugins/codegraph-bootstrap.mjs
~/.config/opencode/plugins/codegraph-bootstrap.mjs
~/.hermes/config.yaml
```

因为脚本默认执行 `codegraph install --target all`，所以也可能更新当前 CodeGraph 版本支持的其他 agent，例如 Cursor、Gemini CLI、Antigravity 或 Kiro。

## 快速开始

在任意项目根目录运行下面这一段。它会在 skill 不存在时自动克隆，然后安装或修复 CodeGraph，索引当前项目，注册 MCP Server，并挂载启动 Hook：

```powershell
$skill = "$HOME\.codex\skills\codegraph-bootstrap"
if (-not (Test-Path -LiteralPath $skill)) {
  git clone https://github.com/petrezhu/codegraph-installer-skill.git $skill
}
powershell -NoProfile -ExecutionPolicy Bypass -File "$skill\scripts\Invoke-CodeGraphBootstrap.ps1"
```

如果 skill 已经安装，可以直接运行短命令：

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File "$HOME\.codex\skills\codegraph-bootstrap\scripts\Invoke-CodeGraphBootstrap.ps1"
```

首次配置完成后，重启正在运行的 agent，让它们重新加载 MCP Server 和启动 Hook。

## 主要脚本

### Ensure-CodeGraph.ps1

检查并修复 CodeGraph 运行状态。

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File "$HOME\.codex\skills\codegraph-bootstrap\scripts\Ensure-CodeGraph.ps1" -Mode ensure
```

模式：

- `check`：检查 CodeGraph CLI 是否存在；如果缺失且 npm 可用，则自动安装。
- `ensure`：检查 CLI，初始化或同步当前项目索引，并注册 MCP。
- `hook`：用于启动 Hook。执行同样的修复流程，但只输出 Hook 可解析的 JSON，失败也不会阻塞 agent 启动。

常用参数：

- `-ProjectPath <path>`：指定要索引的项目目录。
- `-Targets all`：传给 `codegraph install` 的目标 agent。
- `-SkipIndex`：只修复 MCP/Hook 注册，不执行索引。
- `-Quiet`：只写日志，不打印普通输出。

### Install-CodeGraphBootstrap.ps1

运行 `Ensure-CodeGraph.ps1`，并写入各 agent 的启动 Hook。

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File "$HOME\.codex\skills\codegraph-bootstrap\scripts\Install-CodeGraphBootstrap.ps1" -Targets all
```

这是常规安装入口。

### Invoke-CodeGraphSql.ps1

直接查询 `.codegraph/codegraph.db`，不需要系统安装 `sqlite3.exe`。

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File "$HOME\.codex\skills\codegraph-bootstrap\scripts\Invoke-CodeGraphSql.ps1" -Recipe stats
```

列出可用查询食谱：

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File "$HOME\.codex\skills\codegraph-bootstrap\scripts\Invoke-CodeGraphSql.ps1" -ListRecipes
```

## 启动 Hook 行为

Hook 不会常驻启动一个独立的 CodeGraph Server。MCP 客户端会在自身启动时根据 MCP 配置运行：

```powershell
codegraph serve --mcp
```

启动 Hook 只负责确认：

- CodeGraph CLI 存在；
- 当前项目存在 `.codegraph/codegraph.db`，除非使用 `-SkipIndex`；
- 支持的 agent 已注册 CodeGraph MCP；
- 失败会写入日志，但不会阻塞 agent 会话启动。

日志路径：

```text
~/.codegraph-agent-bootstrap.log
```

## Agent 说明

### Codex

MCP 注册配置类似：

```toml
[mcp_servers.codegraph]
command = "codegraph"
args = ["serve", "--mcp"]
```

安装流程还会启用 `hooks` 功能，并写入 `~/.codex/hooks.json`。

验证：

```powershell
codex mcp get codegraph
```

### Claude Code

CodeGraph 会把 MCP 配置写入 `~/.claude.json`，并把允许调用的 CodeGraph MCP 工具写入 `~/.claude/settings.json`。本 skill 还会在同一个 settings 文件中追加 `SessionStart` command hook。

配置完成后需要重启 Claude Code。

### OpenCode

CodeGraph 会在 Windows 上写入：

```text
~/AppData/Roaming/opencode/opencode.jsonc
```

本 skill 还会把启动插件写到两个常见插件目录：

```text
~/AppData/Roaming/opencode/plugins/codegraph-bootstrap.mjs
~/.config/opencode/plugins/codegraph-bootstrap.mjs
```

OpenCode 启动时会加载插件目录中的 JavaScript 或 TypeScript 文件。

### Hermes

CodeGraph MCP 配置写在 `~/.hermes/config.yaml` 的 `mcp_servers.codegraph` 下。本 skill 会追加类似下面的 shell hook：

```yaml
hooks:
  on_session_start:
    - command: 'powershell -NoProfile -ExecutionPolicy Bypass -File ".../Ensure-CodeGraph.ps1" -Mode hook -Targets "all" -Quiet'
      timeout: 120
hooks_auto_accept: true
```

Hermes 修改 MCP 后可能需要新建会话，或运行 `/reload-mcp`。

## SQL 兜底查询

当当前 agent 会话还没有加载 CodeGraph MCP 工具，或者你需要 MCP 未暴露的自定义图查询时，可以用 SQL 兜底。

示例：

```powershell
# 搜索 symbol 名称、qualified name、docstring 和 signature
powershell -NoProfile -ExecutionPolicy Bypass -File "$HOME\.codex\skills\codegraph-bootstrap\scripts\Invoke-CodeGraphSql.ps1" -Recipe search -Symbol FloorManagementService -Json

# 查直接调用方
powershell -NoProfile -ExecutionPolicy Bypass -File "$HOME\.codex\skills\codegraph-bootstrap\scripts\Invoke-CodeGraphSql.ps1" -Recipe callers -Symbol SaveFloors

# 查递归影响范围
powershell -NoProfile -ExecutionPolicy Bypass -File "$HOME\.codex\skills\codegraph-bootstrap\scripts\Invoke-CodeGraphSql.ps1" -Recipe impact -Symbol SaveFloors -Depth 4 -Limit 100

# 查单个文件中的 symbols
powershell -NoProfile -ExecutionPolicy Bypass -File "$HOME\.codex\skills\codegraph-bootstrap\scripts\Invoke-CodeGraphSql.ps1" -Recipe file-symbols -FilePath "Services/Floor/FloorManagementService.cs"
```

更多查询食谱：

```text
~/.codex/skills/codegraph-bootstrap/references/query-recipes.md
```

## 验证

检查 CLI：

```powershell
codegraph --version
```

检查项目索引：

```powershell
codegraph status --json
```

检查 Codex MCP：

```powershell
codex mcp list
```

检查 Hook 输出是否为单个 JSON 对象：

```powershell
'{"cwd":"E:\\C#\\ZDBim.Revit.BimMaster\\ZDZS.DecorateDesign\\DockPanel","hook_event_name":"SessionStart"}' |
  powershell -NoProfile -ExecutionPolicy Bypass -File "$HOME\.codex\skills\codegraph-bootstrap\scripts\Ensure-CodeGraph.ps1" -Mode hook -Quiet -SkipIndex
```

## 排查

### MCP 工具没有出现

重启对应 agent。MCP Server 配置通常只会在进程启动时加载。

### Hook 看起来没有执行

查看日志：

```powershell
Get-Content -Tail 80 "$HOME\.codegraph-agent-bootstrap.log"
```

### 系统没有安装 `sqlite3`

使用 `Invoke-CodeGraphSql.ps1`。它走 Python 内置的 `sqlite3` 模块，不依赖 `sqlite3.exe`。

### 索引存在但过期

运行：

```powershell
codegraph sync .
```

或者从项目根目录再次运行 bootstrap 脚本。

### 项目没有索引

运行：

```powershell
codegraph init .
codegraph index .
```

或者从项目根目录运行 bootstrap 脚本。

## 回滚

移除 CodeGraph MCP 注册：

```powershell
codegraph uninstall --target all --location global --yes
```

移除 bootstrap Hook 和 skill：

```powershell
Remove-Item -LiteralPath "$HOME\.codex\hooks.json" -Force
Remove-Item -LiteralPath "$HOME\.codex\skills\codegraph-bootstrap" -Recurse -Force
```

这些位置可能还需要手动清理 Hook 片段：

```text
~/.claude/settings.json
~/.hermes/config.yaml
~/AppData/Roaming/opencode/plugins/codegraph-bootstrap.mjs
~/.config/opencode/plugins/codegraph-bootstrap.mjs
```

回滚后重启受影响的 agent。

## 参考

- Colby McHenry CodeGraph: https://colbymchenry.github.io/codegraph/
- CodeGraph MCP server reference: https://colbymchenry.github.io/codegraph/reference/mcp-server/
- Bennett Lee CodeGraph skill: https://github.com/bennett-lee/codegraph-skill
- OpenCode plugin docs: https://dev.opencode.ai/docs/plugins/
- Hermes shell hook docs: https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/hooks.md
- Hermes MCP config reference: https://hermes-agent.nousresearch.com/docs/reference/mcp-config-reference/
