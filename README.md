# Brew Bot: an agent-powered café (WebMCP demo)

**Live demo:** https://prachi050.github.io/agentic-product-management-dashboard/

Order coffee by asking in plain English. Type *"a large oat latte with an extra shot"* and an agent adds it to your order. Type *"place my order"* and you can watch it go from Received to Brewing to Ready.

This is a proof of concept for **WebMCP**, the draft web standard from the W3C Web Machine Learning Community Group. WebMCP lets a website hand AI agents in the browser a list of actions it supports, so an agent calls those actions directly instead of scraping the page or simulating clicks. Everything is plain HTML, CSS and JavaScript in one file (`index.html`), with no build step, server or database.

| Tool | API | What it does |
|---|---|---|
| `add_to_order` | **Declarative**: `toolname` / `tooldescription` / `toolautosubmit` on the "Customize a drink" `<form>` | The browser builds the input schema from the form fields. The submit handler sends a structured result back with `SubmitEvent.respondWith()`. |
| `get_menu`, `view_order` | **Imperative**: `document.modelContext.registerTool()` | Read the menu, prices and current order. Marked `readOnlyHint`. |
| `remove_from_order`, `clear_order` | **Imperative** | Edit the order. `clear_order` is marked `destructiveHint`. |
| `place_order` | **Imperative** | Places the order, but only after the customer confirms. |

- **The chat box.** Simple requests are handled instantly by a rule-based parser that works in every browser. Free-form requests (*"something sweet and cold"*) go to Chrome's built-in on-device model (Gemini Nano, through the Prompt API) when it's available. The model returns a JSON plan constrained by a schema, and the plan can only name the tools above.
- **One set of tools.** The chat box, the menu buttons and outside AI agents all call the same tool handlers.
- **Simple by default.** Technical details (WebMCP status, the tool list and a live activity log) sit behind the **How it works** button.

## Second demo: Agentic Product Management Dashboard (`dashboard.html`)

**Live:** https://prachi050.github.io/agentic-product-management-dashboard/dashboard.html

A more technical, developer-facing version of the same ideas: a product backlog that agents can read and update.

| Tool | API | What it does |
|---|---|---|
| `create_task` | **Declarative** form tool | The submit handler sees `SubmitEvent.agentInvoked` and returns JSON through `SubmitEvent.respondWith()`. |
| `query_tasks` | **Imperative** | Filters, sorts and aggregates tasks (overdue count, counts by status). Marked `readOnlyHint`. |
| `update_task_status` | **Imperative** | Changes a task's status. Validates input and returns `isError` results the model can act on. |
| `delete_task` | **Imperative** | Removes an item. Marked `destructiveHint`, and an AI-planned delete asks the user to confirm first. |

It has its own command box (`add Fix login bug high priority due friday #bug`, `what's overdue?`, `help`) with an engine menu (Auto / Rules only / AI only) to show each path on its own.

## Design notes

- **Progressive enhancement.** Both pages work in any browser. The WebMCP tools are added only when the browser supports them.
- **Feature detection.** The page uses `document.modelContext` first. It falls back to `navigator.modelContext`, which early previews used and Chrome 150 deprecated. If neither exists, or the page is not in a secure context, it shows a banner and keeps working.
- **One code path.** The human UI, the declarative form and the imperative tools all call the same functions, so what an agent does matches what a person can do.
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
git clone https://github.com/prachi050/agentic-product-management-dashboard.git
cd agentic-product-management-dashboard
python3 -m http.server 8000      # or: npx serve .
```

Open <http://localhost:8000> for Brew Bot (click **How it works**: it should say *Agent mode is on*) or <http://localhost:8000/dashboard.html> for the dashboard (the badge should show a green dot). In DevTools you can check `typeof document.modelContext.registerTool` (it should be `"function"`).

### 4. Install the inspector extension
Install the **Model Context Tool Inspector** extension from the Chrome Web Store. It lists the tools the current page registers, shows each tool's input schema, and lets you call tools by hand or through an AI agent.

### 5. Test the tools
1. Open the extension on Brew Bot. You should see `add_to_order` (from the form) and the five imperative tools. Try `add_to_order`, then `place_order`, and confirm the prompt. The steps below use the dashboard (`dashboard.html`); open the extension there and you should see `create_task`, `query_tasks`, `update_task_status` and `delete_task`.
2. **Declarative:** call `create_task` with `{"title": "Add SSO to enterprise plan", "priority": "high"}`. The form fills in, highlights while the agent is active (`:tool-form-active`), and submits on its own because of `toolautosubmit`. The inspector should get back `{"ok": true, "task": {...}}`.
3. **Imperative:** call `query_tasks` with `{"status": ["todo"], "sortBy": "due"}`, then `update_task_status` with an id from that result.
4. **Error path:** call `update_task_status` with `{"id": 999, "status": "done"}`. You should get an `isError` result, and the error appears in the *Agent activity* log.
5. **Agent run:** if the inspector has an agent or model option, give it a goal such as *"Move the highest-priority open item to in progress and add a follow-up item due next Friday."* Watch the activity log to see it chain the tools.

### 6. Test the fallback
Open the same URL in a browser without the flag (or in Firefox or Safari). The badge turns red, a banner explains why, and both pages still work.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| Badge says *WebMCP not available* | The flag is off, Chrome wasn't relaunched, or the Chrome build is too old. |
| Badge says *insecure context* | The page was opened via `file://` or a LAN IP. Use `http://localhost`. |
| Yellow badge, `navigator.modelContext` | An older build with the deprecated entry point. It works, but update Canary. |
| A tool fails to register (see the log) | The same tool name was registered twice, or the schema is invalid. The log shows the `DOMException` name. |

WebMCP is experimental and attribute and event names may change between Chrome releases. Before a demo, check the current explainer at <https://github.com/webmachinelearning/webmcp> and Chrome's docs at <https://developer.chrome.com/docs/ai/webmcp>.
