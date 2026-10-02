# WebMCP Task Console

A single-file proof of concept (`index.html`, vanilla HTML/CSS/JS, no build step) for **WebMCP**, the draft web standard from the W3C Web Machine Learning Community Group. WebMCP lets a page expose its own functions to an AI agent running in the browser as typed tools, so the agent calls those functions directly and does not have to scrape the DOM or simulate clicks.

The page is a small task dashboard. It exposes three tools:

| Tool | API | What it does |
|---|---|---|
| `create_task` | **Declarative**: `toolname` / `tooldescription` / `toolautosubmit` on a `<form>` | The browser builds the JSON Schema from the form controls. The submit handler sees `SubmitEvent.agentInvoked` and sends structured JSON back through `SubmitEvent.respondWith()`. |
| `query_tasks` | **Imperative**: `document.modelContext.registerTool()` | Filters, sorts and aggregates tasks (overdue count, counts by status). Marked `readOnlyHint`. |
| `update_task_status` | **Imperative** | Changes a task's status. Validates input and returns `isError` results the model can act on. |

## Design notes

- **Progressive enhancement.** The dashboard works in any browser. The WebMCP tools are added only when the browser supports them.
- **Feature detection.** The page uses `document.modelContext` first. It falls back to `navigator.modelContext`, which early previews used and Chrome 150 deprecated. If neither exists, or the page is not in a secure context, it shows a banner and keeps working.
- **One code path.** The human UI, the declarative form and the imperative tools all call the same functions (`createTask`, `queryTasks`, `updateTaskStatus`), so what an agent does matches what a person can do.
- **Error contract.** Each `execute()` is wrapped. A validation failure becomes `{ isError: true, content: [...] }` with a message the model can recover from. An unexpected exception is logged and not exposed to the model.
- **Lifecycle.** Tools are registered with an `AbortSignal` and removed on `pagehide`. The page also calls `unregisterTool()` for builds that don't support the signal.
- **Safety.** Text from the agent or the user is written with `textContent`, never `innerHTML`.

## Testing locally

### 1. Install a browser that has WebMCP
Install **Google Chrome Canary** (or Chrome Dev/Beta). WebMCP is an early preview behind a flag, so the stable channel may not have the current API shape.

### 2. Enable the flag
1. Open `chrome://flags/#enable-webmcp-testing`.
2. Set **WebMCP for testing** to **Enabled**.
3. Click **Relaunch**.

### 3. Serve the page from a secure context
WebMCP is only exposed in secure contexts. `http://localhost` counts as secure; `file://` URLs and LAN IP addresses do not.

```bash
git clone <this-repo> && cd <this-repo>
python3 -m http.server 8000      # or: npx serve .
```

Open <http://localhost:8000>. The badge at the top right should show a green dot and **document.modelContext · 3 tools**. In DevTools you can check `typeof document.modelContext.registerTool` (it should be `"function"`).

### 4. Install the inspector extension
Install the **Model Context Tool Inspector** extension from the Chrome Web Store. It lists the tools the current page registers, shows each tool's input schema, and lets you call tools by hand or through an AI agent.

### 5. Test the tools
1. Open the extension on the page. You should see `create_task` (from the form), `query_tasks` and `update_task_status`.
2. **Declarative:** call `create_task` with `{"title": "Email Prof. X", "priority": "high"}`. The form fills in, highlights while the agent is active (`:tool-form-active`), and submits on its own because of `toolautosubmit`. The inspector should get back `{"ok": true, "task": {...}}`.
3. **Imperative:** call `query_tasks` with `{"status": ["todo"], "sortBy": "due"}`, then `update_task_status` with an id from that result.
4. **Error path:** call `update_task_status` with `{"id": 999, "status": "done"}`. You should get an `isError` result, and the error appears in the *Agent activity* log.
5. **Agent run:** if the inspector has an agent or model option, give it a goal such as *"Mark my highest-priority open task as in progress and add a task to follow up on it next Friday."* Watch the activity log to see it chain the tools.

### 6. Test the fallback
Open the same URL in a browser without the flag (or in Firefox or Safari). The badge turns red, a banner explains why, and the dashboard plus the **Imperative tools** demo buttons still work.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| Badge says *WebMCP not available* | The flag is off, Chrome wasn't relaunched, or the Chrome build is too old. |
| Badge says *insecure context* | The page was opened via `file://` or a LAN IP. Use `http://localhost`. |
| Yellow badge, `navigator.modelContext` | An older build with the deprecated entry point. It works, but update Canary. |
| A tool fails to register (see the log) | The same tool name was registered twice, or the schema is invalid. The log shows the `DOMException` name. |

WebMCP is experimental and attribute and event names may change between Chrome releases. Before a demo, check the current explainer at <https://github.com/webmachinelearning/webmcp> and Chrome's docs at <https://developer.chrome.com/docs/ai/webmcp>.
