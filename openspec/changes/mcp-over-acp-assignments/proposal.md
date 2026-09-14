# Proposal: MCP-over-ACP assignments for custom agents

## Why

自定义 ACP agent 目前拿不到任何用户配置的 MCP server：`read_servers_for_agent_type(AgentType::Custom(_))` 刻意返回空表（`src-tauri/src/commands/mcp.rs:3279-3282`），因此 `load_mcp_servers_for_agent`（`src-tauri/src/acp/connection.rs:4294`）经 `session/new.mcpServers` 转发的只有内建 `codeg-mcp` 伴生进程。同时 MCP 设置页的 `Enabled Apps`（`APP_OPTIONS`，15 个内建 app）不把自定义 agent 当作可分配目标。pi 的处境相同且更受限：pi-acp 接受 `mcpServers` 字段但直接丢弃（`connection.rs:4287-4289`），MCP 只能由 client 供给。issue #699 的诉求真实存在，且无任何现有设置可替代。

## What Changes

- 新增一个特殊的 MCP 分配目标 `McpAppType::McpOverAcp`（wire 名 `mcp_over_acp`），出现在 `Settings - MCP - Local MCP - Enabled Apps` 中，标签 `MCP-over-ACP`。
- codeg 为该目标维护自有 store（canonical spec 原样保存，stdio/http/sse 全部支持），完全复刻 DeepSeek `$DSH_HOME/mcp.json` 的"自有 store + 每次 session 出生走 ACP wire 转发"架构（`mcp.rs:2439-2462`）。
- `load_mcp_servers_for_agent` 对 `AgentType::Custom(_)` 改为读取该 store 并转换为 ACP wire 格式；转发仍受 `CustomAgentDef::supports_mcp`（UI 的 "MCP Support" 开关）与 agent 广告的 `mcpCapabilities` 双重门控，不新增选项（connection.rs:5383-5417 已有完整门控与失败提示 `McpRejectedByAgent`）。
- scan / upsert / remove / set-apps 全链路纳入新目标；`ALL_MCP_APPS` 增至 16 项，既有 `all_mcp_apps_is_exhaustive` 编译期测试保证不漏分支。
- pi 保持不转发（wire 会被丢弃），行为不变。

## Capabilities

### New Capabilities
- `mcp-over-acp`: codeg 自有的 MCP-over-ACP 服务器分配目标——本地 MCP 设置中的特殊 App、其 store 生命周期（scan/upsert/remove）、以及向启用 MCP Support 的自定义 agent 的 ACP wire 转发行为。

### Modified Capabilities

（`openspec/specs/` 目前为空，无既有 capability 可修改。）

## Impact

- 后端 `src-tauri/src/commands/mcp.rs`：`McpAppType`、`ALL_MCP_APPS`、store 读写（镜像 deepseek 三函数）、`read_servers_for_agent_type` 的 Custom 分支、`app_can_host_spec`、`scan_local_sources`。
- 后端 `src-tauri/src/acp/connection.rs`：`load_mcp_servers_for_agent` 的 Custom 分支（仅注释与读源变化，门控复用）。
- 前端 `src/lib/types.ts`（`McpAppType` 联合类型）、`src/components/settings/mcp-settings.tsx`（`APP_OPTIONS`）、`src/i18n/messages/*.json`（10 语言）。
- 无 API 破坏：新 app 为纯增量，旧配置不含 `mcp_over_acp` 时行为与现状一致。
