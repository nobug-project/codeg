# Design: MCP-over-ACP assignments

## Context

自定义 agent 的 MCP 转发管道已存在且已门控：`run_connection` 在 init 后按 `supports_mcp`（custom）与广告的 `mcpCapabilities` 过滤 `load_mcp_servers_for_agent` 的输出，再放入 `session/new`（`connection.rs:5383-5417`）；`tag_mcp_suspect` 在失败时指向 MCP Support 开关（`connection.rs:1198`）。缺的只是数据源——`read_servers_for_agent_type(AgentType::Custom(_))` 返回空（`mcp.rs:3279`）。DeepSeek（`$DSH_HOME/mcp.json`，`mcp.rs:2439-2602`）证明"codeg 自有 store + 每次会话出生 wire 转发"是本仓库已验证的模式，本设计直接复制它。

## Goals / Non-Goals

**Goals:**

- 新增 `McpAppType::McpOverAcp`（serde 名 `mcp_over_acp`），Enabled Apps 三处网格均可选。
- codeg 自有 store 承载 canonical spec；custom agent 会话出生时全量转发。
- scan / upsert / remove / set-apps / marketplace install 全链路纳入。

**Non-Goals:**

- 不做按 custom agent 粒度的分配（全局伪 App，见 Decisions D2）。
- 不给 pi、内建 agent 转发该 store。
- 不引入新开关；复用 `supports_mcp`（issue 中"或者一个专门的新选项"不需要）。

## Decisions

### D1. Store 形态：完全镜像 DeepSeek 三函数

`read/upsert/remove_mcp_over_acp_server(_at)` + `mcp_over_acp_store_path()`，canonical spec 原样保存（无 foreign schema 转换）。`read_json_file`/`write_json_file` 复用；upsert 写入走 DeepSeek 式 owner-only 权限路径（`write_deepseek_store` 已示范，stdio env 常含 token）。

- 备选：放 DB（SeaORM）。否——MCP 设置页整页建立在"读盘 + `require_complete_scan` 完整性契约"上，改 DB 会牵动 scan/write 的全部路径；文件 store 与 DeepSeek 同构，diff 最小。

### D2. Store 位置：codeg 数据目录 + backup section

`<data_dir>/mcp-over-acp.json`，经 `CODEG_HOME` → `CODEG_DATA_DIR` → `~/.codeg` 的既有解析约定（`paths.rs` 模式）。在 `backup/sections.rs` 增加一条 section，使其随备份走——DeepSeek 把 store 放 harness home 正是为备份一致性，这里没有 harness home，数据目录 + section 是等价物。

### D3. 转发接入点：只改 `read_servers_for_agent_type` 的 Custom 分支

`AgentType::Custom(_) => read_mcp_over_acp_servers()`。门控、转换（`canonical_spec_to_mcp_server`）、按 `mcpCapabilities` 过滤、失败提示全部复用，`connection.rs` 除注释外零改动。pi 分支保持 `Ok(BTreeMap::new())`。

- 备选：在 `load_mcp_servers_for_agent` 内单独为 Custom 读 store。否——`read_servers_for_agent_type` 的现有注释本来就把 Custom 描述为"纯 wire 转发、无自有 store"，如今有了 store，改这里语义自洽；且未来任何新的 wire 转发调用点自动获得正确行为。

### D4. `app_can_host_spec(McpOverAcp, _) => true`

store 保存 canonical spec 原文，三种 transport 都能放；http/sse 到不了的 agent 由会话出生时的能力过滤兜底（跳过 + warn，与现状一致）。

### D5. Scan 源插入位置：紧随 DeepSeek 之后

`scan_local_servers` 按"首个读到该 server 的 agent 拥有 canonical spec"解析（`merge_kimi_extension_fields` 的 owner 概念依赖顺序）。MCP-over-ACP store 是 codeg 自有 canonical 的忠实副本，插在 DeepSeek 同类位置即可；不排最前，避免抢占其他 agent 亲自声明的扩展字段。

### D6. 前端：`APP_OPTIONS` 增一项 + 10 语言 i18n

`src/lib/types.ts` 联合类型加 `"mcp_over_acp"`；`mcp-settings.tsx` `APP_OPTIONS` 加 `{ value: "mcp_over_acp", label: "MCP-over-ACP" }`（不进 `SCAN_ONLY_APP_LABELS`——它是可分配目标）；i18n 仅新增 label 相关键（现有 `local.enabledApps` 文案复用）。`appsToDraft` / `hiddenLegacyApps` / `normalizeApps` 均按 `APP_OPTIONS` 驱动，无需额外改动。

## Risks / Trade-offs

- [自定义 agent 挂上 server 后拒绝 `session/new` 整体失败] → 既有 `McpRejectedByAgent` + `supports_mcp` 提示路径覆盖；spec 场景锁定该行为不变。
- [全局伪 App 粒度粗：所有开 MCP Support 的 custom agent 都拿到全部 server] → 与 issue 的方案一致（一个特殊 App）；单 agent 粒度留作后续（需要 per-agent store 或矩阵 UI，本 change 不做）。
- [用户误以为 pi 也能收到] → UI label 与设置页说明文案明确"经 ACP wire 供自定义 agent"；pi 分支显式排除并在代码注释保留原因引用。
- [`ALL_MCP_APPS` 增员波及所有 match 分支] → `all_mcp_apps_is_exhaustive` 编译期测试强制处理每一处；预计需补的只有 `upsert_server_for_app` / `remove_server_for_app` / `app_can_host_spec` / scan 源列表。

## Migration Plan

纯增量。旧数据无 `mcp_over_acp` 字段，行为不变；回滚即删代码，store 文件残留无害（scan 不再读、转发不再发生）。

## Open Questions

- 无阻塞项。若实现时发现 `scan_local_servers` 的源列表有比"紧随 DeepSeek"更强的排序约束（如 `require_complete_scan` 的警告归因），按实测调整插入位置即可，不影响 spec。
