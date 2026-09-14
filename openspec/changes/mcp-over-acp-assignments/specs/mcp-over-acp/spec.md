# mcp-over-acp Delta

## Purpose

Give users a first-class way to hand user-configured MCP servers to custom ACP agents (and any other ACP agent that reads no native MCP config): a special assignment target in the MCP settings whose servers codeg delivers to the agent over the ACP wire at session birth.

## ADDED Requirements

### Requirement: MCP-over-ACP assignment target in MCP settings

The local MCP settings SHALL offer a special assignment target `MCP-over-ACP` alongside the built-in apps, selectable in every app-assignment surface (server create/edit, marketplace install, scan readback). A server assigned to `MCP-over-ACP` SHALL NOT be written to any other agent's on-disk config, and removing it SHALL only affect the MCP-over-ACP store.

#### Scenario: Assign a local server to MCP-over-ACP

- **WHEN** the user enables `MCP-over-ACP` for a local MCP server in Settings - MCP and saves
- **THEN** the server's canonical spec is persisted in codeg's MCP-over-ACP store and no agent config file outside that store is modified for this assignment

#### Scenario: Reassignment follows the existing "these agents and no others" contract

- **WHEN** the user saves a server whose enabled apps exclude `MCP-over-ACP`
- **THEN** the server is removed from the MCP-over-ACP store, consistent with how reassignment removes it from every other unselected app

### Requirement: Wire delivery to custom agents

At ACP session creation for a custom agent, codeg SHALL load every server stored under `MCP-over-ACP` and forward them on the wire's `mcpServers` field, converted to the ACP server schema. Delivery SHALL remain gated by the custom agent's existing "MCP support" declaration and by the agent's advertised MCP transport capabilities (a transport the agent does not advertise is skipped with a warning, not fatal).

#### Scenario: MCP-support custom agent receives assigned servers

- **WHEN** a session starts for a custom agent whose MCP support is enabled and the store holds a stdio server
- **THEN** the session request carries that server in `mcpServers` together with the built-in codeg-mcp companion entry

#### Scenario: MCP-averse custom agent stays connectable

- **WHEN** a session starts for a custom agent whose MCP support is disabled
- **THEN** no MCP-over-ACP server is forwarded and session creation proceeds exactly as before this capability existed

#### Scenario: Agent rejecting wire MCP still surfaces the existing hint

- **WHEN** a custom agent fails session creation after MCP-over-ACP servers were attached
- **THEN** the existing MCP-suspect error path (pointing the user at the MCP support switch) applies unchanged

### Requirement: Store visibility and scan honesty

The scan SHALL report servers held by the MCP-over-ACP store as assigned to it, and an unreadable store SHALL surface as a scan warning under the same completeness contract as every other source (reads degrade, writes refuse to act on an incomplete picture).

#### Scenario: Scan round-trips the store

- **WHEN** the user opens Settings - MCP after assigning a server to MCP-over-ACP
- **THEN** the server appears with `MCP-over-ACP` among its enabled apps

### Requirement: pi unchanged

The pi agent SHALL NOT receive MCP-over-ACP servers; its sessions keep their current no-MCP behavior regardless of store contents.

#### Scenario: pi session ignores the store

- **WHEN** a session starts for pi while the MCP-over-ACP store is non-empty
- **THEN** no MCP server from the store is forwarded to pi
