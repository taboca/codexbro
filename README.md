# Codexbro

This fork is meant to make the Codex command-line agent easier to understand as a working system. The starting point is that Codex can look opaque from the outside because model selection, backend routing, tool execution, connectors, and turn handling are spread across several layers. The purpose of this README is to make those layers easier to inspect in plain English, so a builder can understand how a session works, how a turn works, and where the main decisions and integrations live in the code.

The previous repository README has been moved to [README_LEGACY.md](./README_LEGACY.md).

## 0. Introduction

This section gives a top-level mental model for the architecture used by this CLI.

### 0.1 User, Session, And Turn

This section introduces the three basic units that make the rest of the document easier to read. The starting point is that the system is not just a single prompt going to a model. It is a longer-lived interaction structure with a user, an ongoing session, and many turns inside that session. The practical value of naming these units early is that later sections can describe model requests, tool execution, reconnects, and backend routing without mixing together short-lived and long-lived parts of the system.

* User - the human interacting with Codex through the CLI or a host UI such as an editor integration

* Session - the longer-lived conversation or thread that carries the conversation identity over time

* Turn - one unit of work started by user input and completed only after the model and any follow-up tool activity for that input settle

A session can contain many turns. A turn is not just one outbound request. It can include model output, tool calls, tool results, retries, reconnects, and follow-up model requests before the turn is done. In the code, this idea shows up as turn-scoped state such as `TurnContext`, `TurnState`, `ActiveTurn`, and `ModelClientSession`.

### 0.2 Understanding Turns For A Given Session

This section names the main parts that show up when one turn runs inside a session. The starting point is that a turn is not a single isolated API call. It is a unit of work that passes through several parts of the system. The practical question is how to describe those parts clearly without pretending that the code uses exactly the same labels. In this README, the labels below are explanatory labels that help describe the turn structure in plain English, even when the code uses more specific internal names.

* User input - starts or steers the turn

* Session runtime - the local runtime that builds the request, selects the available tools, and manages the turn; in the code, this is closer to the `Session` plus turn-scoped state than to a separate formal layer named `CLI`

* Model request path - the model-facing network path, including requests such as `/responses`, `/models`, `/responses/compact`, and realtime transport

* Local tools path - the built-in local tools such as `list_dir`, `shell`, `apply_patch`, `grep_files`, and `read_file`

* Connectors path - the app and integration side of the system, where connector-backed tools and app metadata are surfaced through the ChatGPT connectors ecosystem

### 0.3 How A Turn Uses Tools

This is the core execution logic of a turn. The starting point is that the session already has a model choice, a provider, and working context. The complication is that the model may need to do more than produce text: it may need to inspect files, run commands, or use connector-backed capabilities. The practical question is who decides which tool to use and where that tool runs. In this codebase, the answer is that the CLI sends the model request together with the available tool schema, the model decides whether to call a tool, and the local runtime executes local tools and returns the result into the same turn until the work is complete.

* Model selection - starts from session configuration and collaboration-mode settings, then resolves against the available model catalog and provider configuration.

* Tool availability - for each turn, the CLI builds a prompt that includes the current tool menu.

* Model decision - the normal model request sends that tool schema to the model-facing backend, and the model can answer with plain content or with tool calls.

* Local execution - if the model calls a local tool, the CLI dispatches that tool locally and records the output back into the turn.

* Turn completion - the turn ends only after the model and any tool work for that user request have finished.

A `reconnecting` message in a UI fits this model. It usually means the streaming transport for the current turn is retrying or reconnecting, not that the whole session has started over.

## 1. Model Information

### 1.1 Catalog Sources

This repository uses a real model-list endpoint and also ships with an offline model catalog.

- Live model discovery comes from `GET /v1/models`.
- The local bundled fallback catalog lives at [`codex-rs/core/models.json`](./codex-rs/core/models.json).
- The model manager starts from the bundled catalog and can later refresh from the network.
- You can also force a custom offline catalog with `model_catalog_json` in config.
- The system is centered on the OpenAI `Responses` API rather than the older `chat` wire format.
- When the user is authenticated through ChatGPT instead of an API key, model traffic can be routed through the ChatGPT Codex backend at `https://chatgpt.com/backend-api/codex`.

### 1.2 Bundled Model Slugs

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

### 1.3 Model Layer Notes

Additional notes about the model layer:

- Main inference path: `POST /v1/responses`
- Model catalog path: `GET /v1/models`
- Conversation compaction path: `POST /v1/responses/compact`
- Responses WebSocket transport is supported
- Realtime voice/session transport is supported through `/v1/realtime`
- Audio transcription is handled separately through `/v1/audio/transcriptions`
- A local model catalog can be supplied through `model_catalog_json`
- The code explicitly rejects `wire_api = "chat"` and expects `wire_api = "responses"`

## 2. Choreography Components

This section is meant to make the overall request choreography easier to reason about.

### 2.1 Distinction

- Model: the actual GPT/Codex model, such as `gpt-5.4` or `gpt-5-codex`.
- Backend: the HTTP service that the CLI talks to when it sends model-facing requests.
- Direct API mode: the backend is `https://api.openai.com/v1`.
- ChatGPT Codex backend mode: the backend is `https://chatgpt.com/backend-api/codex`.

### 2.2 Scope And Taxonomy

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

### 2.3 How To Think About The Conversation

- The CLI does not only "hold a chat locally" and then somehow reveal it all at once.
- The conversation is expressed over time through a model request stream for the conversation.
- That stream can include user-turn submission, model streaming output, tool-related events, model catalog lookups, compaction requests, and realtime session traffic.
- In ChatGPT-authenticated mode, those conversation-related model-facing requests can be routed through the ChatGPT Codex backend at `https://chatgpt.com/backend-api/codex`.
- That backend should not be thought of as the model itself. It is better understood as the service layer or gateway that receives those requests and handles account, workspace, orchestration, and routing concerns before or around model execution.

### 2.4 Documentation Status

- Public OpenAI API documentation exists for the public API surfaces such as the Responses API and related endpoints.
- Useful public starting points are:
- Codex docs: `https://developers.openai.com/codex`
- Responses API docs: `https://platform.openai.com/docs/api-reference/responses`
- Audio transcription docs are part of the OpenAI platform API documentation.
- I did not find a public formal API reference in this repo for the ChatGPT-specific backend routes such as `https://chatgpt.com/backend-api/codex/...` or `/wham/...`.
- For this README, the safest working assumption is that the public OpenAI docs describe the public API at `api.openai.com`, while the ChatGPT backend routes seen here behave like product/backend integration endpoints rather than a separately documented public API.

## 3. Local Computer Tool Surface

This section clusters the built-in tools that operate on the local machine or local workspace during a session. These are separate from connectors. The exact set available in a session depends on configuration, feature flags, and tool mode.

### 3.1 Local File And Directory Inspection

* `list_dir` - lists entries in a local directory with numbered output and simple type labels

* `read_file` - reads a local file with line numbers and can expand around indentation-aware code blocks

* `grep_files` - searches local file contents for a regex pattern and returns matching file paths

* `view_image` - loads a local image file by filesystem path for model inspection

### 3.2 Local Command Execution

* `shell` - runs a local shell command on the current machine

* `shell_command` - runs a shell script string in the user's default shell

* `local_shell` - compatibility alias for local shell execution

* `exec_command` - unified exec-style local command runner with richer session control

* `write_stdin` - sends stdin to a running unified exec session

### 3.3 Local Editing And Runtime Helpers

* `apply_patch` - edits local files by applying structured patches

* `js_repl` - runs local JavaScript in the bundled runtime

* `artifacts` - runs local JavaScript against the preinstalled artifact runtime for generated files such as presentations or spreadsheets

### 3.4 How This Differs From Connectors

- Local computer tools act directly on the current machine or local workspace.
- Connectors are app/integration identities associated with ChatGPT-side app surfaces and connector-backed tools.
- A request such as "list a local directory" maps to the local tool surface, not to the connectors directory.
- `list_dir` is the clearest example of a built-in local tool rather than a connector-backed capability.

### 3.5 Tooling Dispatch Model

This repo does not appear to use a separate "tool catalog dispatch" API call. Instead, the available tool menu is attached to the normal model request for the turn.

* Step 1 - the CLI builds the available tool set locally. The tool router assembles built-in tools, MCP tools, connector-backed app tools, and dynamic tools, then keeps the subset that is model-visible for the turn.

* Step 2 - the turn prompt includes that tool set. The prompt carries the model-visible tools as part of the turn request state.

* Step 3 - the tool specs are serialized into Responses API JSON. The local tool definitions are converted into the `tools` array sent to the model-facing backend.

* Step 4 - the normal model request sends the tools to the server. The standard `/responses` request, or websocket equivalent, carries `tools`, `tool_choice`, and `parallel_tool_calls`.

* Step 5 - the model decides whether to call a tool. This repo uses `tool_choice = "auto"` in the normal Responses request path.

* Step 6 - the CLI receives the tool call and dispatches it locally. The streamed response item is parsed into a local tool invocation, matched to a registered handler, and executed on the client side.

* Step 7 - the CLI sends the tool result back into the conversation. The result is returned as a `function_call_output`-style item and fed into the next step of the turn.

In plain terms:

- The server sees the tool schema.
- The model chooses whether to call a tool.
- The local CLI owns the actual execution of local tools.
- Tool output is fed back to the model as structured conversation state, not as an out-of-band side channel.

## 4. ChatGPT Backend Ecosystem Used By Codex

This section is a repo-local documentation cluster for the broader ChatGPT-side backend surfaces used by this codebase. It is descriptive, not official.

### 4.1 Bases And Scope

- Broad ChatGPT-side base in this repo: `https://chatgpt.com/backend-api/`
- Narrow Codex model-serving base used for model-facing requests: `https://chatgpt.com/backend-api/codex`
- Scope of this section: the broader `ChatGPT backend ecosystem used by Codex`, not only the narrow `ChatGPT Codex backend`
- Common auth shape: `Authorization: Bearer <token>`
- Common ChatGPT account header: `ChatGPT-Account-ID`

### 4.2 `codex` Family

These are the ChatGPT-side Codex model-serving routes. They are the closest analogue to the direct `api.openai.com/v1` model API.

* `/backend-api/codex/responses` - the ChatGPT-side counterpart of `POST /v1/responses`

* `/backend-api/codex/models` - the ChatGPT-side counterpart of `GET /v1/models`

* `/backend-api/codex/responses/compact` - the ChatGPT-side counterpart of `POST /v1/responses/compact`

* `/backend-api/codex/...` websocket-derived `responses` transport - websocket transport for responses when the provider is the ChatGPT Codex backend

### 4.3 `wham` Family

These routes appear to cover product/backend features around tasks, usage, requirements, and environments.

* `/backend-api/wham/usage` - usage and rate-limit style information

* `/backend-api/wham/tasks/list` - list tasks

* `/backend-api/wham/tasks/{id}` - fetch a specific task

* `/backend-api/wham/tasks/{id}/turns/{turn_id}/sibling_turns` - fetch sibling attempts or related turn variants

* `/backend-api/wham/tasks` - create tasks

* `/backend-api/wham/config/requirements` - fetch managed requirements or config requirements

* `/backend-api/wham/environments` - list environments

* `/backend-api/wham/environments/by-repo/{provider}/{owner}/{repo}` - map a repository to candidate environments

* `/backend-api/wham/apps` - legacy apps MCP gateway path

### 4.4 Connectors Directory Family

These routes are used to discover connector/app metadata.

* `/backend-api/connectors/directory/list` - list discoverable connector directory entries

* `/backend-api/connectors/directory/list_workspace` - list workspace-specific connector entries

### 4.5 Plugins Family

These routes are used for remote plugin status synchronization.

* `/backend-api/plugins/list` - fetch remote plugin installation or enabled-state summaries

### 4.6 Transcription Family

These routes are used when voice transcription goes through ChatGPT-authenticated mode instead of the public OpenAI audio API.

* `/backend-api/transcribe` - audio transcription endpoint used in ChatGPT-authenticated mode

### 4.7 Documentation Caveat

- I do not see a formal public API reference for these ChatGPT backend routes in this repo.
- The public OpenAI documentation appears to document the public API surface, not these product/backend routes.
- For that reason, this section should be treated as a reverse-engineered route map from the codebase.

## 5. OpenAI APIs And Services Used

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

## 6. How To Inspect Models Without Executing Requests

Use the bundled catalog:

```bash
sed -n '1,120p' codex-rs/core/models.json
```

List only model slugs:

```bash
rg -o '"slug":\s*"[^"]+"' codex-rs/core/models.json | sed 's/.*"slug":\s*"//; s/"$//'
```

If you want the application to use a local catalog without contacting the server, point config at a JSON file through `model_catalog_json`.

## 7. Terminology

* Model traffic - the network requests that the CLI sends to the model-serving backend to get model behavior. In practice, this means requests such as `POST /v1/responses`, `GET /v1/models`, websocket connections for `responses`, realtime websocket sessions, and similar calls that directly support model selection, inference, streaming output, or model metadata. When I said model traffic can be routed through the ChatGPT Codex backend, I meant those model-related requests may go to `https://chatgpt.com/backend-api/codex` instead of directly to `https://api.openai.com/v1`, depending on how the user is authenticated.

* Model-facing requests - another way to describe model traffic. These are the requests that are sent toward the service layer that fronts model execution and model metadata. They are "facing" the model layer because they ask for model output, model streaming, model selection, model metadata, or related context-management behavior.

* Model request stream for the conversation - the sequence of model-facing requests that together represent an ongoing CLI conversation. Instead of thinking of the chat as one giant opaque blob, it is more accurate to think of it as a stream of requests and responses over time: user turn submission, streaming response output, tool-related follow-up, model lookup, compaction, and sometimes realtime session traffic.

* Session - the longer-lived conversation or thread identity that persists across multiple user interactions. In the code, this is closely related to the session-scoped `ModelClient` and the `conversation_id` or thread identity carried through the system.

* Session-scoped - state or objects that are intended to live across multiple turns within one conversation or thread. In this repo, `ModelClient` is explicitly documented as session-scoped, while the thread identity also appears as `conversation_id` or `thread_id` depending on the code path.

* Turn - one unit of user work within a session. A turn may include more than one model request because tool calls, tool outputs, reconnects, retries, or follow-up requests can all occur before that turn finishes. In the code, this idea appears in `TurnContext`, `TurnState`, and `ActiveTurn`.

* Turn-scoped session - a per-turn streaming or model client context. In this repo, `ModelClientSession` is turn-scoped: it can reuse websocket and sticky-routing state within one turn, but a session as a whole can contain many turns.

* ChatGPT Codex backend - the backend service that receives the CLI's conversation-related model-facing requests before those requests are fulfilled by actual model execution. In this repo, that backend is represented by `https://chatgpt.com/backend-api/codex`.

* ChatGPT backend ecosystem used by Codex - a broader term than `ChatGPT Codex backend`. This wider bucket includes the ChatGPT Codex backend itself plus adjacent ChatGPT-side backend families such as `wham`, connectors directory routes, plugin routes, and transcription routes.

* Connectors - ChatGPT-side integration surfaces that let the system discover and work with external tools or apps through directory-style endpoints such as `/backend-api/connectors/directory/list` and `/backend-api/connectors/directory/list_workspace`. In this repo, connectors are part of the broader ChatGPT backend ecosystem used by Codex rather than part of the narrow `codex` model-serving family.
