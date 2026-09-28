# WhatsApp Automation Workflows

This directory contains reusable YAML workflows for WhatsApp Web automation, extracted from the WhatsApp Automation Service (WAS) project.

## Directory Structure

```
examples/whatsapp/
├── whatsapp.yaml          # Single entry point — dispatches to sub-workflows
├── README.md              # This file
├── locators.toml          # CSS selectors (update when WhatsApp UI changes)
├── _common/               # Shared helper sub-workflows
│   ├── ensure_ready.yaml  #   Navigate to WhatsApp Web + check auth
│   └── open_chat.yaml     #   Open chat by phone number (deep link)
├── auth/                  # Authentication sub-workflows
│   ├── qr_login.yaml     #   QR code authentication
│   ├── phone_login.yaml  #   Phone number authentication
│   ├── check_status.yaml #   Check authentication status
│   └── logout.yaml       #   Logout from WhatsApp Web
├── messaging/             # Message sending sub-workflows
│   ├── send.yaml         #   Universal send (text, media, or document)
│   ├── send_text.yaml    #   Send text message only
│   ├── send_media.yaml   #   Send image/video with optional caption
│   └── send_document.yaml#   Send document file
└── chat/                  # Chat management sub-workflows
    ├── get_chats.yaml    #   Get list of chats from sidebar
    ├── get_messages.yaml #   Get messages from a specific chat
    ├── watch_messages.yaml#  Watch for new incoming messages
    └── navigate.yaml     #   Navigate to a specific chat
```

## Architecture

All workflows are composed using the `call` action:

```
whatsapp.yaml (entry point)
    │
    ├── action=check_status  → call auth/check_status
    │                              └── call _common/ensure_ready
    ├── action=qr_login      → call auth/qr_login
    │                              └── call _common/ensure_ready
    ├── action=send_text     → call messaging/send_text
    │                              ├── call _common/ensure_ready
    │                              └── call _common/open_chat
    ├── action=get_chats     → call chat/get_chats
    │                              └── call _common/ensure_ready
    └── ...
```

**Shared helpers** (`_common/`) eliminate boilerplate:
- `_ensure_ready` — navigates to WhatsApp Web and returns `{ authorized, status }`
- `_open_chat` — opens a chat by phone number via deep link, returns `{ success, error }`

Each sub-workflow calls the helpers instead of duplicating the navigate + auth-check logic.

## Quick Start

### Single Entry Point (CLI)

Use `whatsapp.yaml` as the main entry point — pass `action=` to select the operation:

```bash
# Check auth status
automodus run examples/whatsapp/whatsapp.yaml action=check_status

# Login via QR code
automodus run examples/whatsapp/whatsapp.yaml action=qr_login

# Login via phone number
automodus run examples/whatsapp/whatsapp.yaml action=phone_login phone_number=+919876543210

# Send a text message
automodus run examples/whatsapp/whatsapp.yaml action=send_text phone=+919876543210 message="Hello!"

# Send media with caption
automodus run examples/whatsapp/whatsapp.yaml action=send_media phone=+919876543210 file_path=/path/to/image.jpg caption="Check this out"

# Send document
automodus run examples/whatsapp/whatsapp.yaml action=send_document phone=+919876543210 file_path=/path/to/doc.pdf

# Get chat list
automodus run examples/whatsapp/whatsapp.yaml action=get_chats limit=20

# Get messages from a chat
automodus run examples/whatsapp/whatsapp.yaml action=get_messages chat_id=919876543210 limit=50

# Watch for new messages
automodus run examples/whatsapp/whatsapp.yaml action=watch

# Navigate to a chat
automodus run examples/whatsapp/whatsapp.yaml action=navigate phone=+919876543210
```

### Running Sub-Workflows Directly

Each sub-workflow can also be run standalone:

```bash
automodus run examples/whatsapp/auth/check_status.yaml
automodus run examples/whatsapp/messaging/send_text.yaml phone=+919876543210 message="Hello!"
automodus run examples/whatsapp/chat/get_chats.yaml limit=20
```

### API Usage

The daemon HTTP server (default `http://127.0.0.1:8080`) exposes one generic
endpoint per workflow — there are no `/whatsapp/*` routes. Start the daemon
with the examples loaded so the workflows are registered:

```bash
export AUTOMODUS_WORKFLOWS=$PWD/examples
automodus daemon start

# Check auth
curl -X POST http://127.0.0.1:8080/api/workflows/whatsapp_check_status/run \
  -H "Content-Type: application/json" -d '{"params": {}}'

# Send text
curl -X POST http://127.0.0.1:8080/api/workflows/whatsapp_send_text/run \
  -H "Content-Type: application/json" \
  -d '{"params": {"phone": "+919876543210", "message": "Hello from Automodus!"}}'

# Get chats
curl -X POST http://127.0.0.1:8080/api/workflows/whatsapp_get_chats/run \
  -H "Content-Type: application/json" -d '{"params": {"limit": 20}}'

# Get messages
curl -X POST http://127.0.0.1:8080/api/workflows/whatsapp_get_messages/run \
  -H "Content-Type: application/json" \
  -d '{"params": {"chat_id": "919876543210", "limit": 50}}'

# Watch for new messages
curl -X POST http://127.0.0.1:8080/api/workflows/whatsapp_watch_messages/run \
  -H "Content-Type: application/json" -d '{"params": {}}'
```

The `on: api:` trigger blocks inside the workflow files are schema-only —
they validate and document intent, but no HTTP route is created for them.

## Workflow Parameters

### Authentication Workflows

| Workflow | Parameters | Description |
|----------|------------|-------------|
| `qr_login` | None | Returns QR code as base64 |
| `phone_login` | `phone_number` (required) | Returns verification code |
| `check_status` | None | Returns auth status |
| `logout` | None | Logs out of WhatsApp |

### Messaging Workflows

| Workflow | Parameters | Description |
|----------|------------|-------------|
| `send` | `phone` (required), `message`, `file` | Universal send |
| `send_text` | `phone` (required), `message` (required) | Text only |
| `send_media` | `phone` (required), `file_path` (required), `caption` | Image/Video |
| `send_document` | `phone` (required), `file_path` (required), `caption` | Document |

### Chat Workflows

| Workflow | Parameters | Description |
|----------|------------|-------------|
| `get_chats` | `limit` (default: 50) | List chats |
| `get_messages` | `chat_id` (required), `limit`, `load_more` | Chat messages |
| `watch_messages` | None | New incoming messages |
| `navigate` | `phone` or `name` | Open a chat |

## Browser Configuration

WhatsApp workflows set:

```yaml
browser:
  headless: false          # WhatsApp requires a visible window for login
  data_dir: "data/whatsapp_profile"
  width: 1280
  height: 800
```

What the engine actually honors today:

- `headless: false` — ✅ used on `automodus run` (WhatsApp Web needs a
  visible window while scanning the QR code).
- `data_dir`, `width`, `height` — accepted by the schema but **not
  consumed** by the launch paths; standalone runs use an isolated temp
  profile that is deleted on exit, so login does **not** persist between
  separate `automodus run` invocations.

To keep a session alive, keep the browser alive: use the daemon
(`automodus daemon start` + shell/API) and leave its session open, or stay
in one shell session for the whole login + messaging flow.

**Chromium-only:** `send`/`send_media`/`send_document` use the `upload`
action (`DOM.setFileInputFiles`), which is supported on Chromium only —
Firefox and Lightpanda report it as unsupported.

## CSS Selectors

These workflows use well-tested CSS selectors from the WAS project. Key selectors:

| Element | Selector |
|---------|----------|
| Auth pane | `#pane-side` |
| QR code canvas | `canvas[aria-label='Scan this QR code to link a device!']` |
| Message input | `#app #main footer div[aria-placeholder='Type a message']` |
| Send button | `button[aria-label='Send']` |
| Attach button | `button[title='Attach']` |
| Menu button | `button[title='Menu']` |

## Error Handling

All workflows include error handling:

- **on_error.screenshot**: Takes screenshot on failure
- **on_error.emit**: Emits error event for monitoring

Example error response:
```json
{
  "status": "send_failed",
  "success": false,
  "error": "Message input not found"
}
```

## Tips

1. **Rate Limiting**: WhatsApp may temporarily block accounts that send messages too quickly. Add delays between messages.

2. **Session Management**: `data_dir` is not wired up yet (see Browser
   Configuration), so each standalone run starts fresh — log in and act
   within one shell session, or run against a daemon session that stays
   open between operations.

3. **Headless Mode**: WhatsApp Web requires a visible browser window during
   QR code scanning. Headless is not used by these workflows
   (`headless: false`), and the daemon always launches with the config
   headless setting.

4. **Phone Format**: Always include country code (e.g., `+919876543210` or `919876543210`).

5. **File Paths**: Use absolute paths for file attachments, or paths relative to the workflow execution directory.

## Composition Example

Build higher-level workflows by calling `whatsapp.yaml` or the sub-workflows directly:

```yaml
# bulk_message.yaml - Send to multiple recipients
name: bulk_message
params:
  recipients:
    type: array
    required: true
  message:
    type: string
    required: true

steps:
  # Ensure we're authenticated first
  - action: call
    workflow: whatsapp/whatsapp
    params:
      action: check_status
    store_as: auth

  - action: condition
    if: "{{store.auth.authorized}} == false"
    then:
      - action: log
        level: error
        message: "Not authenticated"
      - action: emit
        event: bulk_message_aborted

  # NOTE: `action: loop` is a pseudo-action (`items:`, `as:`, `index_as:`,
  # `steps:`). Sequence-valued `items:` must keep their type:
  #   items: "{{vars.recipients | json}}"   # ✅ one item per recipient
  #   items: "{{vars.recipients}}"          # ❌ renders as a single string
  # Loop vars are read as `{{vars.<as>}}` and, when `index_as:` is set,
  # `{{vars.<index_as>}}`.
  # Unrolling recipients explicitly also works, or call a sub-workflow per step:
  #
  #   - action: call
  #     workflow: whatsapp/whatsapp
  #     params:
  #       action: send_text
  #       phone: "+919876543210"
  #       message: "{{params.message}}"
  #
  #   - action: sleep
  #     ms: 2000  # Rate limiting
```

Or call sub-workflows directly for finer control:

```yaml
steps:
  - action: call
    workflow: whatsapp/_common/ensure_ready
    store_as: ready

  - action: condition
    if: "{{store.ready.authorized}} == true"
    then:
      - action: call
        workflow: whatsapp/messaging/send_text
        params:
          phone: "+919876543210"
          message: "Hello!"
```

## Extracted From

These workflows were extracted from the WhatsApp Automation Service (WAS) project, specifically:

- `was/src/browser/core.rs` - WhatsAppEngine methods
- `was/src/browser/locators.rs` - CSS selectors
- `was/src/services/whatsapp/chat.rs` - Chat service implementation
- `was/src/services/auth/auth.rs` - Authentication service

The automation patterns have been converted from Rust code to declarative YAML workflows.
