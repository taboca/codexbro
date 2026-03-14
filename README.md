# Codexbro

This fork exists to help the builder user understand, inspect, and explore the capabilities of the Codex command-line agent and its user interaction flows.

The previous repository README has been moved to [README_LEGACY.md](./README_LEGACY.md).

## 1. Terminology

- Model traffic: the network requests that the CLI sends to the model-serving backend to get model behavior. In practice, this means requests such as `POST /v1/responses`, `GET /v1/models`, websocket connections for `responses`, realtime websocket sessions, and similar calls that directly support model selection, inference, streaming output, or model metadata. When I said model traffic can be routed through the ChatGPT Codex backend, I meant those model-related requests may go to `https://chatgpt.com/backend-api/codex` instead of directly to `https://api.openai.com/v1`, depending on how the user is authenticated.

- Model-facing requests: another way to describe model traffic. These are the requests that are sent toward the service layer that fronts model execution and model metadata. They are "facing" the model layer because they ask for model output, model streaming, model selection, model metadata, or related context-management behavior.

- Model request stream for the conversation: the sequence of model-facing requests that together represent an ongoing CLI conversation. Instead of thinking of the chat as one giant opaque blob, it is more accurate to think of it as a stream of requests and responses over time: user turn submission, streaming response output, tool-related follow-up, model lookup, compaction, and sometimes realtime session traffic.

- ChatGPT Codex backend: the backend service that receives the CLI's conversation-related model-facing requests before those requests are fulfilled by actual model execution. In this repo, that backend is represented by `https://chatgpt.com/backend-api/codex`.

- ChatGPT backend ecosystem used by Codex: a broader term than `ChatGPT Codex backend`. This wider bucket includes the ChatGPT Codex backend itself plus adjacent ChatGPT-side backend families such as `wham`, connectors directory routes, plugin routes, and transcription routes.

- Connectors: ChatGPT-side integration surfaces that let the system discover and work with external tools or apps through directory-style endpoints such as `/backend-api/connectors/directory/list` and `/backend-api/connectors/directory/list_workspace`. In this repo, connectors are part of the broader ChatGPT backend ecosystem used by Codex rather than part of the narrow `codex` model-serving family.

### 1.1 High-Level Session Surfaces

This is a good conceptual model for how a user session fans out through the CLI:

- User
  Briefly: the human interacting with the session
- Codex CLI
  Briefly: the local entry point that orchestrates requests, tools, and backend communication
- Model API layer
  Briefly: the model-facing network layer, including paths such as `/responses`, `/models`, `/responses/compact`, and realtime transport
- Local computer tool surface
  Briefly: the built-in local tools such as `list_dir`, `shell`, `apply_patch`, `grep_files`, and `read_file`
- Connectors surface
  Briefly: the app/integration side of the system, where connector-backed tools and app metadata are surfaced through the ChatGPT connectors ecosystem

Practical reading of that split:

- `Model API layer` is about talking to the model-serving backend.
- `Local computer tool surface` is about operating on the local machine or local workspace.
- `Connectors surface` is about external app/integration capabilities rather than local built-in tools.

Connector examples such as Slack, GitHub, Notion, or Google Drive are reasonable illustrative examples for this mental model, but I have not verified those exact names as a fixed connector inventory from this repo's runtime code.

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

## 3. Choreography Components

This section is meant to make the overall request choreography easier to reason about.

### 3.1 Distinction

- Model: the actual GPT/Codex model, such as `gpt-5.4` or `gpt-5-codex`.
- Backend: the HTTP service that the CLI talks to when it sends model-facing requests.
- Direct API mode: the backend is `https://api.openai.com/v1`.
- ChatGPT Codex backend mode: the backend is `https://chatgpt.com/backend-api/codex`.

### 3.2 Scope And Taxonomy

This subsection narrows the terms so they are easier to use in a mutually exclusive and collectively exhaustive way within this README.

- `OpenAI public API`: the public API surface rooted at `https://api.openai.com/v1`
- `ChatGPT Codex backend`: the narrow Codex model-serving family rooted at `https://chatgpt.com/backend-api/codex`
- `ChatGPT backend ecosystem used by Codex`: the broader ChatGPT-side backend surface touched by this codebase

Working partition used in this README:

- Public API bucket: `https://api.openai.com/v1/...`
- Narrow ChatGPT Codex bucket: `https://chatgpt.com/backend-api/codex/...`
- Broader ChatGPT backend ecosystem bucket: ChatGPT-side routes outside the narrow `codex` family, including:
- `https://chatgpt.com/backend-api/wham/...`
- `https://chatgpt.com/backend-api/connectors/directory/...`
- `https://chatgpt.com/backend-api/plugins/...`
- `https://chatgpt.com/backend-api/transcribe`

Working rule for this README:

- When we say `ChatGPT Codex backend`, we mean only the narrow `.../backend-api/codex` family.
- When we say `ChatGPT backend ecosystem used by Codex`, we mean the full ChatGPT-side surface used by this codebase, including the narrow `codex` family plus the adjacent families listed above.
- Section 4 is therefore intentionally broader than the term `ChatGPT Codex backend`.

### 3.3 How To Think About The Conversation

- The CLI does not only "hold a chat locally" and then somehow reveal it all at once.
- The conversation is expressed over time through a model request stream for the conversation.
- That stream can include user-turn submission, model streaming output, tool-related events, model catalog lookups, compaction requests, and realtime session traffic.
- In ChatGPT-authenticated mode, those conversation-related model-facing requests can be routed through the ChatGPT Codex backend at `https://chatgpt.com/backend-api/codex`.
- That backend should not be thought of as the model itself. It is better understood as the service layer or gateway that receives those requests and handles account, workspace, orchestration, and routing concerns before or around model execution.

### 3.4 Documentation Status

- Public OpenAI API documentation exists for the public API surfaces such as the Responses API and related endpoints.
- Useful public starting points are:
- Codex docs: `https://developers.openai.com/codex`
- Responses API docs: `https://platform.openai.com/docs/api-reference/responses`
- Audio transcription docs are part of the OpenAI platform API documentation.
- I did not find a public formal API reference in this repo for the ChatGPT-specific backend routes such as `https://chatgpt.com/backend-api/codex/...` or `/wham/...`.
- For this README, the safest working assumption is that the public OpenAI docs describe the public API at `api.openai.com`, while the ChatGPT backend routes seen here behave like product/backend integration endpoints rather than a separately documented public API.

## 4. Local Computer Tool Surface

This section clusters the built-in tools that operate on the local machine or local workspace during a session. These are separate from connectors. The exact set available in a session depends on configuration, feature flags, and tool mode.

### 4.1 Local File And Directory Inspection

- `list_dir`
  Briefly: lists entries in a local directory with numbered output and simple type labels
- `read_file`
  Briefly: reads a local file with line numbers and can expand around indentation-aware code blocks
- `grep_files`
  Briefly: searches local file contents for a regex pattern and returns matching file paths
- `view_image`
  Briefly: loads a local image file by filesystem path for model inspection

### 4.2 Local Command Execution

- `shell`
  Briefly: runs a local shell command on the current machine
- `shell_command`
  Briefly: runs a shell script string in the user's default shell
- `local_shell`
  Briefly: compatibility alias for local shell execution
- `exec_command`
  Briefly: unified exec-style local command runner with richer session control
- `write_stdin`
  Briefly: sends stdin to a running unified exec session

### 4.3 Local Editing And Runtime Helpers

- `apply_patch`
  Briefly: edits local files by applying structured patches
- `js_repl`
  Briefly: runs local JavaScript in the bundled runtime
- `artifacts`
  Briefly: runs local JavaScript against the preinstalled artifact runtime for generated files such as presentations or spreadsheets

### 4.4 How This Differs From Connectors

- Local computer tools act directly on the current machine or local workspace.
- Connectors are app/integration identities associated with ChatGPT-side app surfaces and connector-backed tools.
- A request such as "list a local directory" maps to the local tool surface, not to the connectors directory.
- `list_dir` is the clearest example of a built-in local tool rather than a connector-backed capability.

### 4.5 Tooling Dispatch Model

This repo does not appear to use a separate "tool catalog dispatch" API call. Instead, the available tool menu is attached to the normal model request for the turn.

- Step 1: the CLI builds the available tool set locally.
  Briefly: the tool router assembles built-in tools, MCP tools, connector-backed app tools, and dynamic tools, then keeps the subset that is model-visible for the turn
- Step 2: the turn prompt includes that tool set.
  Briefly: the prompt carries the model-visible tools as part of the turn request state
- Step 3: the tool specs are serialized into Responses API JSON.
  Briefly: the local tool definitions are converted into the `tools` array sent to the model-facing backend
- Step 4: the normal model request sends the tools to the server.
  Briefly: the standard `/responses` request, or websocket equivalent, carries `tools`, `tool_choice`, and `parallel_tool_calls`
- Step 5: the model decides whether to call a tool.
  Briefly: this repo uses `tool_choice = "auto"` in the normal Responses request path
- Step 6: the CLI receives the tool call and dispatches it locally.
  Briefly: the streamed response item is parsed into a local tool invocation, matched to a registered handler, and executed on the client side
- Step 7: the CLI sends the tool result back into the conversation.
  Briefly: the result is returned as a `function_call_output`-style item and fed into the next step of the turn

Practical reading of the model:

- The server sees the tool schema.
- The model chooses whether to call a tool.
- The local CLI owns the actual execution of local tools.
- Tool output is fed back to the model as structured conversation state, not as an out-of-band side channel.

## 5. ChatGPT Backend Ecosystem Used By Codex

This section is a repo-local documentation cluster for the broader ChatGPT-side backend surfaces used by this codebase. It is descriptive, not official.

### 5.1 Bases And Scope

- Broad ChatGPT-side base in this repo: `https://chatgpt.com/backend-api/`
- Narrow Codex model-serving base used for model-facing requests: `https://chatgpt.com/backend-api/codex`
- Scope of this section: the broader `ChatGPT backend ecosystem used by Codex`, not only the narrow `ChatGPT Codex backend`
- Common auth shape: `Authorization: Bearer <token>`
- Common ChatGPT account header: `ChatGPT-Account-ID`

### 5.2 `codex` Family

These are the ChatGPT-side Codex model-serving routes. They are the closest analogue to the direct `api.openai.com/v1` model API.

- `/backend-api/codex/responses`
  Briefly: the ChatGPT-side counterpart of `POST /v1/responses`
- `/backend-api/codex/models`
  Briefly: the ChatGPT-side counterpart of `GET /v1/models`
- `/backend-api/codex/responses/compact`
  Briefly: the ChatGPT-side counterpart of `POST /v1/responses/compact`
- `/backend-api/codex/...` websocket-derived `responses` transport
  Briefly: websocket transport for responses when the provider is the ChatGPT Codex backend

### 5.3 `wham` Family

These routes appear to cover product/backend features around tasks, usage, requirements, and environments.

- `/backend-api/wham/usage`
  Briefly: usage and rate-limit style information
- `/backend-api/wham/tasks/list`
  Briefly: list tasks
- `/backend-api/wham/tasks/{id}`
  Briefly: fetch a specific task
- `/backend-api/wham/tasks/{id}/turns/{turn_id}/sibling_turns`
  Briefly: fetch sibling attempts or related turn variants
- `/backend-api/wham/tasks`
  Briefly: create tasks
- `/backend-api/wham/config/requirements`
  Briefly: fetch managed requirements/config requirements
- `/backend-api/wham/environments`
  Briefly: list environments
- `/backend-api/wham/environments/by-repo/{provider}/{owner}/{repo}`
  Briefly: map a repository to candidate environments
- `/backend-api/wham/apps`
  Briefly: legacy apps MCP gateway path

### 5.4 Connectors Directory Family

These routes are used to discover connector/app metadata.

- `/backend-api/connectors/directory/list`
  Briefly: list discoverable connector directory entries
- `/backend-api/connectors/directory/list_workspace`
  Briefly: list workspace-specific connector entries

### 5.5 Plugins Family

These routes are used for remote plugin status synchronization.

- `/backend-api/plugins/list`
  Briefly: fetch remote plugin installation/enabled-state summaries

### 5.6 Transcription Family

These routes are used when voice transcription goes through ChatGPT-authenticated mode instead of the public OpenAI audio API.

- `/backend-api/transcribe`
  Briefly: audio transcription endpoint used in ChatGPT-authenticated mode

### 5.7 Documentation Caveat

- I do not see a formal public API reference for these ChatGPT backend routes in this repo.
- The public OpenAI documentation appears to document the public API surface, not these product/backend routes.
- For that reason, this section should be treated as a reverse-engineered route map from the codebase.

## 6. OpenAI APIs And Services Used

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

## 7. How To Inspect Models Without Executing Requests

Use the bundled catalog:

```bash
sed -n '1,120p' codex-rs/core/models.json
```

List only model slugs:

```bash
rg -o '"slug":\s*"[^"]+"' codex-rs/core/models.json | sed 's/.*"slug":\s*"//; s/"$//'
```

If you want the application to use a local catalog without contacting the server, point config at a JSON file through `model_catalog_json`.
