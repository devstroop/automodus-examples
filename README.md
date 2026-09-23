# Automodus Examples

Reusable YAML workflows for Automodus. Each subdirectory is self-contained — point `AUTOMODUS_WORKFLOWS` at the root or at a single pack.

```
examples/
├── browser/
│   ├── search_form.yaml      # DuckDuckGo search (goto, type, click, extract)
│   ├── screenshot.yaml       # Param screenshot service (viewport vs full_page)
│   ├── scrape_list.yaml      # Scrape titles/prices/ratings (extract many)
│   ├── multi_tab.yaml        # tab.new / switch / list / close (numeric indices)
│   └── selectors.yaml        # CSS, text:, text*:, role:, xpath: tour
├── http/
│   ├── http_api.yaml         # GET/POST/PUT/PATCH/DELETE vs JSONPlaceholder
│   └── headers_auth.yaml     # Bearer auth + custom headers verified via echo
├── control-flow/
│   ├── branching.yaml        # condition then/else, step if:, emit, lifecycle
│   └── retry_wait.yaml       # wait_for states, sleep ms, step retry, on_error
├── debug/
│   └── debug_demo.yaml       # workflow + step debug (delay, highlight, capture)
├── compose/
│   ├── pipeline.yaml         # call login + capture sub-workflows
│   ├── login.yaml            # Reusable login block
│   └── capture.yaml          # Reusable screenshot block
└── whatsapp/
    ├── whatsapp.yaml         # Entry point — dispatch via action= param
    ├── _common/              # ensure_ready, open_chat helpers
    ├── auth/                 # qr_login, phone_login, check_status, logout
    ├── messaging/            # send, send_text, send_media, send_document
    ├── chat/                 # get_chats, get_messages, watch_messages, navigate
    ├── locators.toml         # CSS selectors (update when WhatsApp UI changes)
    └── README.md             # WhatsApp-specific docs
```

## Run

```bash
# From workspace root, use examples/ as workflow dir
export AUTOMODUS_WORKFLOWS=$PWD/examples

automodus validate $AUTOMODUS_WORKFLOWS
automodus list

# Standalone demos (key=value overrides workflow param defaults)
automodus run examples/browser/search_form.yaml query="rust automation"
automodus run examples/browser/screenshot.yaml name=home full_page=false
automodus run examples/http/headers_auth.yaml token=secret-123
automodus run examples/control-flow/branching.yaml mode=ok
automodus run examples/control-flow/retry_wait.yaml
automodus run examples/debug/debug_demo.yaml
automodus run examples/browser/multi_tab.yaml
automodus run examples/browser/scrape_list.yaml
automodus run examples/browser/selectors.yaml

# Composition
automodus run examples/compose/pipeline.yaml

# WhatsApp entry point (action dispatch)
automodus run examples/whatsapp/whatsapp.yaml action=check_status
automodus run examples/whatsapp/whatsapp.yaml action=send_text phone=+919876543210 message="Hello!"

# Or run a sub-workflow directly (set AUTOMODUS_WORKFLOWS to the pack so `call` resolves)
export AUTOMODUS_WORKFLOWS=$PWD/examples/whatsapp
automodus run auth/check_status.yaml
```

## `call` resolution rules

`WorkflowLoader` resolves `workflow:` relative to `AUTOMODUS_WORKFLOWS` (walks up to 3 parent dirs, no recursive subdirectory search). When running a single file via `automodus run`, `AUTOMODUS_WORKFLOWS` wins; otherwise the file's parent directory is used.

**All `workflow:` paths in this repo are pack-root-relative to `examples/`** (e.g. `whatsapp/_common/ensure_ready`, `compose/login`). That form resolves from `AUTOMODUS_WORKFLOWS=examples/` and also via walk-up when the workflows dir is the file's parent:

| Workflows dir | Call | Resolves to |
|---|---|---|
| `examples/` | `compose/login` | `examples/compose/login.yaml` |
| `examples/` | `whatsapp/whatsapp` | `examples/whatsapp/whatsapp.yaml` |
| `examples/` | `whatsapp/auth/check_status` | `examples/whatsapp/auth/check_status.yaml` |
| `examples/` | `whatsapp/_common/ensure_ready` | `examples/whatsapp/_common/ensure_ready.yaml` |
| `examples/whatsapp/` | `whatsapp/_common/ensure_ready` | walks up → `examples/whatsapp/_common/ensure_ready.yaml` |

## Engine caveats (read before authoring)

These are real engine behaviors, not style preferences:

| Pitfall | What happens | Do this instead |
|---|---|---|
| `action: loop` | Parses, then **ActionNotFound** at runtime | Unroll steps, or `call` a sub-workflow |
| Step-level `on_success` / `on_failure` | Schema fields, **silently ignored** by the engine | Use `if:` / `action: condition`, workflow `on_error` |
| `sleep: "2s"` or `duration: "2s"` | `as_u64` fails → **1000ms** | `ms: 2000` (numeric only) |
| `wait_for timeout: "5s"` | String ignored → **30000ms** | `timeout: 5000` (numeric ms) |
| `tab.switch index: "{{steps.x.tab_index}}"` | Rendered string fails `as_u64` | Hardcode numeric `index:` (initial tab = 0, first `tab.new` = 1) |
| `full_page: "{{params.x}}"` | Templated string → `as_bool` always **false** | Two `if:`-gated screenshot steps (see `browser/screenshot.yaml`) |
| `debug: profile: verbose` in YAML | Valid schema; **ignored** on `automodus run` merge path | CLI `--profile=verbose`, or set explicit `enabled/delay/capture/...` |
| `on:` triggers (schedule/event/webhook) | Schema-only; never auto-fired by the engine | Run with `automodus run` or `POST /api/workflows/:name/run` |
| Condition string | Whole string **lowercased**, then `==` / `!=`, else truthiness | Unquoted RHS: `{{store.x.status}} == 200`, `{{params.action}} == send_text` — never `== 'send_text'` (quotes are compared literally) |
| Header echo checks | Arrays render as JSON (`["Bearer x"]`) | Compare against `["Bearer x"]` (see `http/headers_auth.yaml`) |
| `'{{params.x}}'` inside JS | Raw interpolation breaks on quotes / injects JS | Use `{{params.x \| json}}` (no surrounding quotes), or a native `type:` action input |
| HTTP body field with template | Always rendered as YAML string unless full-string `{{path \| json}}` | `userId: "{{params.user_id \| json}}"` keeps number/bool/null type |

Registered actions only: `goto/back/forward/reload`, `click/type/select/hover`, `wait_for/sleep`, `extract/eval`, `screenshot`, `tab.list/new/switch/close`, `emit/log`, `upload/wait_upload/file_chooser`, `http.get/post/put/patch/delete/request`, plus engine specials `call` and `condition`.

Template filters: `{{path}}` (raw string) and `{{path | json}}` (JSON-encoded literal — missing paths become `null`). Prefer `| json` whenever the value is embedded in a JS expression.

Stable public endpoints used here: `example.com`, `books.toscrape.com`, `httpbingo.org`, `jsonplaceholder.typicode.com`.
