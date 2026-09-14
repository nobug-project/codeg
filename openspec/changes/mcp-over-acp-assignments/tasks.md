# Tasks: MCP-over-ACP assignments

## 1. 后端 store

- [x] 1.1 `src-tauri/src/commands/mcp.rs`：镜像 DeepSeek 三函数新增 `read/upsert/remove_mcp_over_acp_server(_at)` 与 `mcp_over_acp_store_path()`（`<data_dir>/mcp-over-acp.json`，owner-only 写入），并用 `_at(path)` 变体补单元测试（读写删 + 空文件容忍）；验证 `cargo test --features test-utils mcp_over_acp`
- [x] 1.2 `McpAppType` 新增 `#[serde(rename = "mcp_over_acp")] McpOverAcp` 变体，`ALL_MCP_APPS` 增至 16 项，补齐 `upsert_server_for_app` / `remove_server_for_app` 分支；验证 `cargo check` 被 `all_mcp_apps_is_exhaustive` 逼出的全部 match 分支已处理
- [x] 1.3 `app_can_host_spec(McpOverAcp, _) => true`，并新增测试：stdio/http/sse canonical spec 均可写入该目标；验证对应单测通过
- [x] 1.4 `scan_local_servers` 源列表紧随 DeepSeek 插入 MCP-over-ACP store 源，未读告警走 `LocalMcpSourceWarning` 同一契约；验证 scan 单测：store 内 server 以 `mcp_over_acp` app 出现

## 2. 转发路径

- [x] 2.1 `read_servers_for_agent_type` 的 `AgentType::Custom(_)` 分支改为 `read_mcp_over_acp_servers()`，pi 分支保持空表并保留注释；验证单测：Custom agent 的 `load_mcp_servers_for_agent` 返回 store 内容（stdio → `McpServer::Stdio`）
- [x] 2.2 确认门控不变：`supports_mcp=false` 的 custom agent 不转发（复用 connection.rs 既有过滤与 `McpRejectedByAgent` 提示路径）；验证既有 connection 层相关测试全绿，必要时补一个 supports_mcp 门控用例

## 3. 前端

- [x] 3.1 `src/lib/types.ts` `McpAppType` 联合类型加 `"mcp_over_acp"`；验证 `pnpm build` 类型检查通过
- [x] 3.2 `src/components/settings/mcp-settings.tsx` `APP_OPTIONS` 加 `MCP-over-ACP` 项（可分配目标，不进 `SCAN_ONLY_APP_LABELS`）；验证 mcp-settings 相关 vitest 通过
- [x] 3.3 `src/i18n/messages/*.json` 10 语言补 label 相关键（zh 系用 `MCP-over-ACP` 原文）；验证 `pnpm test` i18n 快照/键完整性检查通过

## 4. 备份与收尾

- [x] 4.1 `src-tauri/src/commands/backup/sections.rs` 增加 mcp-over-acp store 的 section；验证备份 round-trip 单测覆盖该文件
- [x] 4.2 全量验证（实际执行）：改动文件 `pnpm exec eslint` + `npx tsc --noEmit` + `pnpm build` 全过；`pnpm exec vitest run src/components/settings src/lib` 163 文件 2812 用例全绿；`cargo clippy --no-default-features --bin codeg-server --lib -- -D warnings` 干净；`cargo test --no-default-features --bin codeg-server --lib` 3494 全过；桌面模式 `cargo check` 通过。未跑：`--features test-utils` 桌面全量测试与 `--all-targets` clippy（本次改动均为 feature 无关代码，桌面测试与 server 测试同源）
- [x] 4.3 冒烟（server 模式端到端）：起 `codeg-server`，`POST /api/mcp_upsert_local_server`（apps=["mcp_over_acp"]）→ store 文件 `mcp-over-acp.json` 正确写入；`/api/mcp_scan_local` 返回 ctx7 且 apps 含 `mcp_over_acp`；`/api/mcp_remove_server` 后 scan 不再出现。桌面 UI 侧手动冒烟待用户在有 agent CLI 的环境执行
