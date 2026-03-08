# SDK Feasibility RFC

## Goal

Create a developer-facing SDK that makes "native-feeling dictation" easy to add to third-party apps while reusing Tambourine's current split architecture:

- Client SDK embedded in the developer's app
- Hosted Tambourine server for STT + LLM formatting + pipeline orchestration

## Feasibility Summary

This is highly feasible with the current codebase. The core pieces already exist:

- WebRTC signaling + pipeline lifecycle on server (`server/main.py`)
- Typed client/server message protocol (`server/protocol/messages.py`, `server/protocol/providers.py`)
- Split runtime config surface:
  - HTTP for prompt sections, LLM formatting, and STT timeout (`server/api/config_api.py`)
  - RTVI messages for provider switching and recording control (`server/protocol/messages.py`, `server/processors/configuration.py`)
- Robust client connection/reconnect flow plus safe message send logic (`app/src/machines/connectionMachine.ts`, `app/src/lib/safeSendClientMessage.ts`)
- Existing client-side config sync logic for the HTTP config endpoints (`app/src-tauri/src/config_sync.rs`)

The main missing piece is productization for multi-tenant public use:

- Authentication and authorization (API keys, project isolation, quotas)
- Public SDK packaging and docs
- Stable versioned API surface

## Current Reusable Contract

The current protocol is already close to an SDK contract, but it is now explicitly split between REST endpoints and RTVI messages.

### HTTP endpoints

- Register + verify client UUID:
  - `POST /api/client/register`
  - `GET /api/client/verify/{client_uuid}`
- ICE config:
  - `GET /api/ice-servers` with `X-Client-UUID`
- WebRTC offer/patch:
  - `POST /api/offer`
  - `PATCH /api/offer`
- Runtime config:
  - `GET /api/prompt/sections/default`
  - `PUT /api/config/prompts`
  - `PUT /api/config/llm-formatting`
  - `PUT /api/config/stt-timeout`
  - `GET /api/providers`

Notes:

- `PUT /api/config/*` endpoints are per-client and require `X-Client-UUID`.
- `GET /api/providers` is global, but today it only returns provider/model info after at least one active connection has initialized provider services. A public SDK/server surface should remove that dependency.

### RTVI client messages

- Recording control:
  - `start-recording`
  - `stop-recording`
- Per-recording metadata:
  - `start-recording` can include optional `active_app_context`
- Runtime provider switching:
  - `set-stt-provider`
  - `set-llm-provider`

### RTVI server messages

- `raw-transcription`
- `recording-complete-with-zero-words`
- `config-updated`
- `config-error`

RTVI messages and provider selections are typed and forward-compatible (`Unknown*` / `Other*` patterns), which is exactly what you want for SDK stability.

## Proposed Product Architecture

### 1. Public SDK Surface

`@tambourine/sdk-web` (first target):

- `createTambourineClient(options)`
- `client.connect()`
- `client.disconnect()`
- `client.startRecording({ activeAppContext? })`
- `client.stopRecording()`
- `client.setProviders({ stt, llm })`
- `client.updatePromptSections(promptSections)`
- `client.setLlmFormattingEnabled(enabled)`
- `client.setSttTimeoutSeconds(seconds)`
- `client.getDefaultPromptSections()`
- `client.getAvailableProviders()`
- event callbacks:
  - `onTranscriptRaw`
  - `onEmptyTranscript`
  - `onConfigUpdated`
  - `onError`
  - `onConnectionStateChange`

Optional companion package:

- `@tambourine/sdk-react` with hooks (`useTambourine`, `useDictationState`)

### 2. Hosted Server Surface

Keep the current server model, add auth + tenancy:

- Project-scoped API keys
- Tenant isolation for usage and settings
- Rate limits + quota enforcement per key/project

### 3. Auth Model (Recommended)

Use a 2-key model:

- `secret_key` (server-to-server only)
- `publishable_key` (safe for client apps, restricted)

Recommended flow:

1. Developer backend uses `secret_key` to create a short-lived client session token.
2. Client SDK uses session token for `register/ice/offer/config` calls.
3. Session token encodes `project_id`, `expires_at`, `scopes`.

This avoids exposing a root secret in browser/mobile clients.

## Minimal Server Changes for SDK Readiness

1. Add auth middleware to all `/api/*` endpoints.
2. Replace in-memory registration with tenant-aware storage:
   - `(project_id, client_uuid, status, last_seen_at)`
3. Decouple `/api/providers` from live connection state so discovery works before the first session is established.
4. Add usage accounting and limits:
   - connection minutes, STT seconds, LLM tokens
5. Keep existing UUID behavior, but bind UUID to project/session.
6. Version the public API:
   - `/v1/client/register`, `/v1/offer`, etc.

## SDK Extraction Plan from Existing App Code

### Extract directly

- Connection lifecycle and retry logic:
  - `app/src/machines/connectionMachine.ts`
- Safe RTVI send logic:
  - `app/src/lib/safeSendClientMessage.ts`
- HTTP runtime config sync patterns:
  - `app/src-tauri/src/config_sync.rs`
- Typed provider selection/parsing:
  - `app/src/lib/tauri.ts` provider types + parsers
- Active app context payload shape:
  - `app/src/lib/activeAppContext.ts`

### Keep app-specific (do not put in SDK core)

- Tauri commands/events and window bridge code
- Global hotkeys and native text insertion
- Overlay UI

## Proposed Rollout

### Phase 1: Internal SDK package

- Move reusable TS client logic into `packages/sdk-web`
- Keep the current split contract:
  - REST for registration, ICE, offer/patch, prompt config, timeout, and LLM formatting
  - RTVI for recording control and provider switching
- Add an example web app in `examples/`

### Phase 2: Auth + tenant layer

- API keys, session tokens, project isolation
- Endpoint versioning (`/v1`)
- Backward-compatible bridge to current endpoints

### Phase 3: Public launch

- SDK docs and quick-start templates
- Dashboard for keys, quotas, usage
- Production observability and SLOs

### Phase 4: Platform SDKs

- React SDK wrappers
- React Native / iOS / Android wrappers reusing the same server contract

## Recommendation

Start with a web SDK MVP using the existing server contract, then layer auth/tenancy without changing core dictation semantics. The current architecture is already modular enough to support this path with low risk.
