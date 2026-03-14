# Codexbro

This fork exists to help the builder user understand, inspect, and explore the capabilities of the Codex command-line agent and its user interaction flows.

The previous repository README has been moved to [README_LEGACY.md](./README_LEGACY.md).

## 1. Terminology

- Model traffic: the network requests that the CLI sends to the model-serving backend to get model behavior. In practice, this means requests such as `POST /v1/responses`, `GET /v1/models`, websocket connections for `responses`, realtime websocket sessions, and similar calls that directly support model selection, inference, streaming output, or model metadata. When I said model traffic can be routed through the ChatGPT Codex backend, I meant those model-related requests may go to `https://chatgpt.com/backend-api/codex` instead of directly to `https://api.openai.com/v1`, depending on how the user is authenticated.

## 2. Model Information

### 2.1 Catalog Sources

This repository uses a real model-list endpoint and also ships with an offline model catalog.

- Live model discovery comes from `GET /v1/models`.
- The local bundled fallback catalog lives at [`codex-rs/core/models.json`](./codex-rs/core/models.json).
- The model manager starts from the bundled catalog and can later refresh from the network.
- You can also force a custom offline catalog with `model_catalog_json` in config.
- The system is centered on the OpenAI `Responses` API rather than the older `chat` wire format.
- When the user is authenticated through ChatGPT instead of an API key, model traffic can be routed through the ChatGPT Codex backend at `https://chatgpt.com/backend-api/codex`.

### 2.2 Bundled Model Slugs

Current model slugs present in the bundled local catalog:

- `gpt-5.3-codex`
- `gpt-5.4`
- `gpt-5.2-codex`
- `gpt-5.1-codex-max`
- `gpt-5.1-codex`
- `gpt-5.2`
- `gpt-5.1`
- `gpt-5-codex`
- `gpt-5`
- `gpt-oss-120b`
- `gpt-oss-20b`
- `gpt-5.1-codex-mini`
- `gpt-5-codex-mini`

### 2.3 Model Layer Notes

Additional notes about the model layer:

- Main inference path: `POST /v1/responses`
- Model catalog path: `GET /v1/models`
- Conversation compaction path: `POST /v1/responses/compact`
- Responses WebSocket transport is supported
- Realtime voice/session transport is supported through `/v1/realtime`
- Audio transcription is handled separately through `/v1/audio/transcriptions`
- A local model catalog can be supplied through `model_catalog_json`
- The code explicitly rejects `wire_api = "chat"` and expects `wire_api = "responses"`

## 3. OpenAI APIs And Services Used

Primary OpenAI API paths used by this codebase:

- `POST /v1/responses`
- `GET /v1/models`
- `POST /v1/responses/compact`
- WebSocket transport for `responses`
- Realtime WebSocket at `/v1/realtime`
- `POST /v1/audio/transcriptions`
- Connectors MCP gateway at `/v1/connectors/gateways/flat/mcp`

OpenAI-owned backend services used by the system:

- `https://api.openai.com/v1`
- `https://auth.openai.com`
- `https://chatgpt.com/backend-api/codex`
- `https://chatgpt.com/backend-api/wham/...`
- `https://chatgpt.com/backend-api/connectors/directory/...`

ChatGPT-backed routes used in the repo include:

- `/wham/usage`
- `/wham/tasks/list`
- `/wham/tasks/{id}`
- `/wham/tasks/{id}/turns/{turn_id}/sibling_turns`
- `/wham/config/requirements`
- `/wham/tasks`
- `/connectors/directory/list`
- `/connectors/directory/list_workspace`
- `/wham/apps`

Auth and request behavior:

- Standard auth header: `Authorization: Bearer <token>`
- ChatGPT-backed requests also use `ChatGPT-Account-ID`
- The built-in OpenAI provider can send `OpenAI-Organization` from `OPENAI_ORGANIZATION`
- The built-in OpenAI provider can send `OpenAI-Project` from `OPENAI_PROJECT`
- Default clients attach an `originator` header
- Realtime requests include `x-session-id`
- Subagent traffic can include `x-openai-subagent`

OAuth and login services:

- OAuth issuer defaults to `https://auth.openai.com`
- Browser login uses `/oauth/authorize`
- Token exchange uses `/oauth/token`
- Refresh-token handling also uses `https://auth.openai.com/oauth/token`

Notably absent from the runtime code paths inspected here:

- No runtime use of `chat/completions`
- No runtime use of the Assistants API
- No runtime use of Embeddings
- No runtime use of Moderations
- No runtime use of image generation endpoints
- No runtime use of fine-tuning endpoints

## 4. How To Inspect Models Without Executing Requests

Use the bundled catalog:

```bash
sed -n '1,120p' codex-rs/core/models.json
```

List only model slugs:

```bash
rg -o '"slug":\s*"[^"]+"' codex-rs/core/models.json | sed 's/.*"slug":\s*"//; s/"$//'
```

If you want the application to use a local catalog without contacting the server, point config at a JSON file through `model_catalog_json`.
