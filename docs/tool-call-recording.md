# Tool Call Recording Feature

This document describes how cursor-chat-browser extracts and displays tool call data — including MCP (Model Context Protocol) tool calls — from Cursor's internal storage.

## Background

Cursor stores all conversation data in a SQLite database. When an AI agent invokes tools during a conversation (file reads, terminal commands, MCP server calls, etc.), Cursor persists the tool name, arguments, and results alongside the conversation messages. This feature ensures that tool call data is surfaced in cursor-chat-browser's UI and exports.

---

## Cursor Database Structure

### Database Location

| Platform | Path |
|----------|------|
| **macOS** | `~/Library/Application Support/Cursor/User/globalStorage/state.vscdb` |
| **Windows** | `%APPDATA%/Cursor/User/globalStorage/state.vscdb` |
| **Linux** | `~/.config/Cursor/User/globalStorage/state.vscdb` |
| **WSL** | `/mnt/c/Users/<username>/AppData/Roaming/Cursor/User/globalStorage/state.vscdb` |
| **Remote Linux** | `~/.cursor-server/data/User/globalStorage/state.vscdb` |

The path can be overridden via the `WORKSPACE_PATH` environment variable (which points to the `workspaceStorage` directory; the global database is at `../globalStorage/state.vscdb` relative to it).

### Database Engine

SQLite, accessed via `better-sqlite3` in read-only mode.

### Tables

#### `cursorDiskKV` (Global Storage — current Cursor format)

A key-value store. Single table with two columns:

| Column | Type | Description |
|--------|------|-------------|
| `key` | TEXT | Namespaced key (e.g. `composerData:<id>`, `bubbleId:<chatId>:<bubbleId>`) |
| `value` | TEXT/BLOB | JSON-serialized data (may be stored as blob) |

#### `ItemTable` (Per-workspace Storage — legacy Cursor format)

| Column | Type | Description |
|--------|------|-------------|
| `key` | TEXT | Fixed key string |
| `value` | TEXT | JSON-serialized data |

Legacy keys used:
- `workbench.panel.aichat.view.aichat.chatdata` — chat tab data
- `composer.composerData` — composer conversation data

---

## Key Prefixes in `cursorDiskKV`

| Key Prefix | Description | Tool Call Data? |
|------------|-------------|-----------------|
| `composerData:<composerId>` | Conversation metadata, headers, status | No (headers only) |
| `bubbleId:<chatId>:<bubbleId>` | Individual message content | **Yes — `toolFormerData` field** |
| `codeBlockDiff:<chatId>:<diffId>` | Code diff / tool action results | Partial (file edits, commands) |
| `messageRequestContext:<composerId>:<contextId>` | Context sent with a message | No (project info, git status) |
| `agentKv:blob:<hash>` | Raw LLM API conversation turns (content-addressed) | **Yes — full `tool-call`/`tool-result` messages** |
| `composer.content.<hash>` | Cached markdown summaries | No |
| `checkpointId:<id>` | Agent checkpoint data | No |

---

## Where Tool Calls Are Stored

### 1. `bubbleId` entries — `toolFormerData` field (primary source)

Each message bubble stored in `bubbleId:<chatId>:<bubbleId>` is a JSON object with many fields. The tool-call-relevant fields are:

```typescript
interface BubbleData {
  // ... text, richText, codeBlocks, etc.

  toolFormerData: {
    tool: number;             // Numeric tool type ID (e.g. 6=list_dir, 15=run_terminal_cmd, 44=list_mcp_resources, 45=fetch_mcp_resource)
    toolCallId: string;       // Unique call identifier (e.g. "toolu_018H93EvFYqrD9bCwLtRh4zK")
    toolIndex: number;        // Index within a multi-tool call
    modelCallId: string;      // LLM call that generated this tool use
    status: string;           // "completed", "error", etc.
    name: string;             // Full tool name — see naming conventions below
    rawArgs: string;          // JSON string of arguments passed to the tool
    params: string;           // Parsed parameters (JSON string)
    result: string;           // JSON string of the tool's response
    additionalData: {
      status?: string;        // "error" if failed
      error?: string;         // Error message
    };
  };

  toolResults: any[];         // Array of tool result objects (usually empty in newer format)
  supportedTools: number[];   // Numeric IDs of tools available to the agent
  capabilityType?: string;    // Capability identifier
}
```

**Key assumption:** Each bubble with tool call data has exactly one `toolFormerData` object. A conversation with N tool calls will have N separate bubble entries, each with its own `toolFormerData`.

### 2. `agentKv:blob:<hash>` entries (secondary source — not currently used)

These are content-addressed blobs containing raw LLM API messages:

```json
{
  "role": "assistant",
  "content": [
    {
      "type": "tool-call",
      "toolCallId": "toolu_01XLB536vCRQbXRUWfG6Sz4U",
      "toolName": "user-confluence-mcp-confluence_search",
      "args": { "query": "SLT Silicon Labs Tool", "limit": 20 }
    }
  ]
}
```

And tool results:

```json
{
  "role": "tool",
  "content": [
    {
      "type": "tool-result",
      "toolCallId": "toolu_01XLB536vCRQbXRUWfG6Sz4U",
      "toolName": "user-confluence-mcp-confluence_search",
      "result": "..."
    }
  ]
}
```

**Note:** `agentKv` entries are keyed by content hash and have **no direct link to a `composerId`**. They are currently not used by cursor-chat-browser because there is no reliable way to map them back to specific conversations. The `bubbleId` entries provide the same data through the `toolFormerData` field.

### 3. `codeBlockDiff` entries (pre-existing)

These contain code diffs and some tool action metadata (`toolName`, `parameters`, `result`, `command`, `filePath`) but in practice mostly store file edit diffs (`newModelDiffWrtV0`/`originalModelDiffWrtV0`). These were already extracted before this feature.

---

## MCP Tool Name Conventions

Cursor uses several naming conventions for MCP tools depending on the Cursor version and configuration:

| Pattern | Example | Description |
|---------|---------|-------------|
| `mcp-<server>-user-<server>-<tool>` | `mcp-confluence-mcp-user-confluence-mcp-confluence_search` | Older format with redundant prefixes |
| `mcp_<server>_<tool>` | `mcp_mcu_debug_tool_start_gdb_server` | Newer underscore-delimited format |
| `user-<server>-<tool>` | `user-slc-cli-slc_generate` | User-defined MCP server tools |
| `list_mcp_resources` | `list_mcp_resources` | Built-in MCP resource listing |
| `fetch_mcp_resource` | `fetch_mcp_resource` | Built-in MCP resource fetching |
| `CallMcpTool` | `CallMcpTool` | Generic MCP tool invocation |
| `FetchMcpResource` | `FetchMcpResource` | Generic MCP resource fetch |

Built-in Cursor tools (not MCP) use simple names: `read_file`, `run_terminal_cmd`, `codebase_search`, `grep_search`, `list_dir`, `edit_file`, `write`, etc.

---

## Implementation

### Overview

Tool call data flows through the system as follows:

```
state.vscdb
  └─ cursorDiskKV (bubbleId:* entries)
       └─ toolFormerData field on each bubble
            │
            ▼
  tabs/route.ts: extractTextFromBubble()
       │
       ├─ formatToolFormerData()  →  markdown-formatted tool call
       └─ formatToolResults()     →  markdown-formatted tool results
            │
            ▼
  Bubble text (markdown string including tool call details)
       │
       ├─ Rendered in workspace page via ReactMarkdown
       ├─ Exported via downloadMarkdown / downloadHTML / downloadPDF
       └─ Copied via CopyButton
```

### Files Modified

#### `src/app/api/workspaces/[id]/tabs/route.ts`

**`formatToolFormerData(toolFormerData)`** — New function that converts a `toolFormerData` object into readable markdown:

1. **MCP detection** — Determines if the tool is an MCP tool by checking if the name starts with `mcp`, `user-`, or matches known MCP tool names (`CallMcpTool`, `FetchMcpResource`, `list_mcp_resources`, `fetch_mcp_resource`). MCP tools get a 🔌 emoji label; regular tools get 🔧.

2. **Arguments formatting** — Tries `rawArgs` first (JSON string of what was passed to the tool), falls back to `params`. Parsed and pretty-printed as a JSON code block.

3. **Result formatting** — Parses the JSON result string and formats it. Results are truncated at 5000 characters to prevent UI overflow.

4. **Error handling** — If `additionalData.status === 'error'` or `additionalData.error` exists, the error is displayed.

**`formatToolResults(toolResults)`** — New function that formats the `toolResults` array found on some bubbles. Each entry is rendered with the tool name and result data.

**`extractTextFromBubble(bubble)`** — Enhanced to call both `formatToolFormerData()` and `formatToolResults()` after extracting the regular text content. This means:
- Bubbles that previously had only tool data and no text are no longer silently dropped
- Tool call information is appended to the end of the bubble's text content

#### `src/types/workspace.ts`

Added `ToolFormerData` interface documenting the schema of Cursor's `toolFormerData` object for reference and type safety.

#### `src/lib/download.ts`

Updated the placeholder for empty AI messages from `[TERMINAL OUTPUT NOT INCLUDED]` to `[No text content — may be a tool-only interaction]` to be more accurate.

---

## Assumptions and Limitations

### Assumptions

1. **Cursor stores tool call data in `toolFormerData`** — Each bubble in `bubbleId:*` entries has a `toolFormerData` object when that bubble represents a tool invocation. This is the primary data source.

2. **One tool call per bubble** — Each bubble has at most one `toolFormerData` object. Multi-tool-call turns are split into separate bubbles by Cursor.

3. **`rawArgs` and `result` are JSON strings** — They can be parsed with `JSON.parse()`. If parsing fails, they are displayed as raw text.

4. **MCP tool names follow known prefixes** — The MCP detection heuristic relies on name prefixes (`mcp`, `user-`) and known special names. If Cursor changes its naming convention, the MCP vs. regular tool distinction may break (the tool call data itself would still be displayed, just without the 🔌 MCP label).

5. **The `agentKv:blob:*` entries are not linkable to conversations** — These content-addressed blobs contain full LLM API turns with tool calls/results, but their keys are content hashes with no conversation ID. There is currently no reliable way to associate them with a specific `composerData` entry.

6. **Tool results can be very large** — MCP tools like `confluence_get_page` can return entire page contents. Results are truncated at 5000 characters in `formatToolFormerData()` and 3000 characters in `formatToolResults()` to keep the UI responsive.

### Limitations

- **No Cursor configuration controls storage** — There is no CLI flag, settings.json option, or environment variable in Cursor that controls what conversation data is persisted. Cursor stores everything automatically. This is a Cursor design decision, not something this project can control.

- **`agentKv` data is not used** — The `agentKv:blob:*` entries contain the richest tool call data (full LLM API messages including `tool-call` and `tool-result` typed content), but cannot be mapped back to conversations. If Cursor adds a conversation-to-agentKv mapping in the future, this data source could be integrated.

- **Tool numeric IDs are opaque** — The `tool` field in `toolFormerData` is a numeric ID (e.g. `6`, `15`, `44`, `45`). The mapping of IDs to tool types is not documented by Cursor and may change between versions. The `name` field is used instead.

- **`codeBlockDiff` tool data is limited** — The previously existing `formatToolAction()` handles `codeBlockDiff` entries which can contain `toolName`, `parameters`, and `result` fields, but in practice these entries mostly contain code edit diffs, not MCP tool call data.

---

## Example Output

A bubble with MCP tool call data renders as:

```markdown
**🔌 MCP Tool Call: `mcp-confluence-mcp-user-confluence-mcp-confluence_search`** (completed)
<sub>Call ID: toolu_01XLB536vCRQbXRUWfG6Sz4U</sub>

**Arguments:**
​```json
{
  "query": "SLT Silicon Labs Tool",
  "limit": 20
}
​```

**Result:**
​```json
{
  "results": [
    { "id": "755997561", "title": "SLT MCP Server Design", "excerpt": "..." },
    { "id": "762997017", "title": "SLT Register Command", "excerpt": "..." }
  ]
}
​```
```

A regular (non-MCP) tool call:

```markdown
**🔧 Tool Call: `read_file`** (completed)
<sub>Call ID: tool_f460600d-f741-4d61-ba32-b91a1e460a3</sub>

**Arguments:**
​```json
{
  "target_file": "app.c",
  "should_read_entire_file": true
}
​```

**Result:**
​```json
{
  "contents": "#include \"app.h\"\n\nvoid app_init(void) {\n  // ...\n}\n"
}
​```
```
