# Brew Bot: an agent-powered café (WebMCP demo)

**Live demo:** https://prachi050.github.io/agentic-product-management-dashboard/

Order coffee by asking in plain English. Type *"a large oat latte with an extra shot"* and an agent adds it to your order. Type *"place my order"* and you can watch it go from Received to Brewing to Ready.

This is a proof of concept for **WebMCP**, the draft web standard from the W3C Web Machine Learning Community Group. WebMCP lets a website hand AI agents in the browser a list of actions it supports, so an agent calls those actions directly instead of scraping the page or simulating clicks. Everything is plain HTML, CSS and JavaScript in one file (`index.html`), with no build step, server or database.

## Tools the page gives AI agents

| Tool | API | What it does |
|---|---|---|
| `add_to_order` | **Declarative**: `toolname` / `tooldescription` / `toolautosubmit` on the "Customize a drink" `<form>` | The browser builds the input schema from the form fields. When an agent submits, the handler returns a structured result with `SubmitEvent.respondWith()`. |
| `get_menu` | **Imperative**: `document.modelContext.registerTool()` | Drinks, sizes, milk options and prices. Marked `readOnlyHint`. |
| `view_order` | **Imperative** | What's in the order and the total. Marked `readOnlyHint`. |
| `remove_from_order` | **Imperative** | Removes one line from the order. |
| `clear_order` | **Imperative** | Empties the order. Marked `destructiveHint`. |
| `place_order` | **Imperative** | Places the order, but only after the customer confirms. |

## The chat box

The "What can I get you?" box is the page's own small agent, and it uses the same tools.

- **Rules first.** Common requests (*"two small black coffees and an iced mocha"*, *"remove the mocha"*, *"place my order"*) are understood instantly by a rule-based parser that works in every browser.
- **On-device AI second.** Free-form requests (*"something sweet and cold"*) go to Chrome's built-in model, Gemini Nano, through the Prompt API, when it's available. The model sees the menu and the current order. It returns a JSON plan constrained by a schema (`responseConstraint`), and the plan can only name the tools above.
- **Same tools everywhere.** The chat box, the menu's **Add** buttons and outside AI agents all call the same tool handlers, so every action shows up in the activity log.

## Design notes

- **Simple by default.** Visitors see the chat, the menu and their order. WebMCP status, the tool list, the chat-engine menu and a live activity log sit behind the **How it works** button.
- **Progressive enhancement.** The page works in any browser. WebMCP tools are registered only when the browser supports them.
- **Feature detection.** The page uses `document.modelContext` first. It falls back to `navigator.modelContext`, which early previews used and Chrome 150 deprecated. If neither exists, or the page isn't in a secure context, the chat and menu keep working without agent mode.
- **Confirmation before risky actions.** `place_order` asks the customer before an outside agent can submit. It uses `client.requestUserInteraction()` when the browser provides it, and `confirm()` otherwise. Orders planned by the on-device model are confirmed the same way.
- **Error contract.** Every tool is wrapped. Bad input becomes `{ isError: true, content: [...] }` with a message the agent can recover from, such as *"'unicorn' isn't on the menu"*. Unexpected exceptions are logged and not exposed to the model.
- **Lifecycle.** Tools are registered with an `AbortSignal` and removed on `pagehide`, with `unregisterTool()` as a fallback.
- **Safety.** Text typed by users or produced by agents is written with `textContent`, never `innerHTML`.

## Trying it

### In any browser
Open the live demo and type into the chat box, or tap **Add** on a menu item. Try:
- `A large oat latte with an extra shot`
- `Two small black coffees and an iced mocha`
- `What's on the menu?` · `What's in my order?`
- `Remove the mocha` · `Cancel my order`
- `Place my order`

### With AI agents in Chrome
1. **Turn on WebMCP.** Open `chrome://flags/#enable-webmcp-testing`, set **WebMCP for testing** to **Enabled**, and click **Relaunch**. If the flag isn't listed, use Chrome Canary.
2. **Check it's on.** Open the demo and click **How it works**. It should say *"Agent mode is on. 6 actions are available to AI agents in this browser."*
3. **Install the inspector.** Add the **Model Context Tool Inspector** extension from the Chrome Web Store and open it on the demo tab. It should list the six tools above.
4. **Run the tools:**
   - `add_to_order` with `{"drink": "latte", "size": "large", "milk": "oat", "extraShot": true}`: the drink appears in **Your order**.
   - `view_order`: returns the order and the total.
   - `add_to_order` with `{"drink": "unicorn"}`: returns an error the agent can act on.
   - `place_order`: asks you to confirm, then the order tracker starts.
5. **Agent run.** If the inspector offers an agent or model option, give it a goal such as *"Order me a large iced mocha and a small americano, then check out."* Watch the activity log under **How it works**.

### Running it locally
WebMCP needs a secure context. `http://localhost` counts; `file://` URLs and LAN IP addresses don't.

```bash
git clone https://github.com/prachi050/agentic-product-management-dashboard.git
cd agentic-product-management-dashboard
python3 -m http.server 8000
```

Then open <http://localhost:8000>.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| **How it works** says agent mode is off | The flag is off, Chrome wasn't relaunched, the Chrome build is too old, or the page was opened from `file://`. |
| Free-form requests say the on-device AI isn't available | This Chrome or this computer doesn't support the built-in model. Simple requests still work. |
| The first free-form request is slow | Chrome is downloading the on-device model. This only happens once. |
| A tool fails to register (see the activity log) | The same tool name was registered twice, or a schema is invalid. The log shows the error name. |

WebMCP is experimental, and attribute and event names may change between Chrome releases. Before a demo, check the current explainer at <https://github.com/webmachinelearning/webmcp> and Chrome's docs at <https://developer.chrome.com/docs/ai/webmcp>.
