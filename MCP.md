# PortalVR MCP server

PortalVR includes a local [Model Context Protocol](https://modelcontextprotocol.io) (MCP) server for applications running on SteamVR. An MCP client, such as an AI agent or an automated test harness, can use it to inspect the running XR application, read head and controller poses, inject poses and controller input, capture the composited eye image, and call tools the application registers itself.

This document covers enabling the server, connecting a client, the transports, the security model, and every built-in tool. It describes PortalVR's Windows SteamVR driver; the standalone PortalVR OpenXR runtime and other vendors' OpenXR runtimes do not expose MCP.

## Contents

- [Scope and compatibility](#scope-and-compatibility)
- [Enable the server](#enable-the-server)
- [Connect a client](#connect-a-client)
- [Local files and configuration](#local-files-and-configuration)
- [Security model](#security-model)
- [JSON-RPC rules](#json-rpc-rules)
- [Legacy HTTP+SSE transport](#legacy-httpsse-transport)
- [Streamable HTTP transport](#streamable-http-transport)
- [Health and status endpoints](#health-and-status-endpoints)
- [Tool results and errors](#tool-results-and-errors)
- [Built-in tools](#built-in-tools)
- [Application custom tools](#application-custom-tools)
- [Recovery](#recovery)
- [Compatibility notes](#compatibility-notes)

## Scope and compatibility

The server observes and drives applications owned by SteamVR:

- modern OpenVR applications that use SteamVR Input;
- legacy OpenVR applications that read controller state directly; and
- OpenXR applications whose active OpenXR runtime is SteamVR.

An OpenXR application is exposed with its OpenXR details (instance information, `openxr_exit_session`, and its custom tools) only while its session is **focused**. At other times, such as before its first focus or while the SteamVR dashboard is open, calls without a `session_handle` reach it as a plain SteamVR session or return `NO_ACTIVE_SESSION`.

Session handles are not stable. Each time the application gains focus, the dashboard opens or closes, or the application changes, the server can issue a new handle, and the old one then returns `STALE_SESSION`. Omit `session_handle`, or read it again with `openxr_get_session_info` after any of those changes.

The 25 built-in tools use the tool names, arguments, and annotations of the Meta XR Operator MCP surface, so a client written for that surface can use PortalVR without renaming tools. Tool names keep the `openxr_` prefix for compatibility even when the application uses OpenVR. Results are PortalVR's own; see [built-in tools](#built-in-tools).

| Item | Value |
| --- | --- |
| Server name / version | `PortalVR XR Operator` / `1.0.0` |
| Legacy MCP revision | `2024-11-05` over HTTP+SSE |
| Streamable HTTP revision | `2026-07-28` (stateless) |
| Default listener | `127.0.0.1:8720` and `[::1]:8720`, loopback only |
| Stdio proxy | `steamvr-portalvr\bin\win64\portalvr-mcp-proxy.exe` in the PortalVR installation |

## Enable the server

The **MCP Server** switch on PortalVR's System settings page is the only control. It defaults on when the MCP configuration is first created; an explicit off choice is preserved across updates. When it is on, every local client can call every tool: there is no pairing, approval prompt, or per-tool permission.

Turning the switch on or off also registers or removes PortalVR's OpenXR API layers for the current user, then asks you to restart SteamVR and the VR application so the OpenXR loader applies the change. Registration needs no administrator prompt. A failed update offers a Retry action in System settings.

The System page also shows the server status and loopback endpoint. While a client is active, PortalVR shows an on-screen **MCP control active** badge (any accepted request in the last 30 seconds), or **MCP capture active** during a capture. **Emergency MCP Clear** (the **Disconnect and Clear** button on the System page) stops the server, clears pending input, pose overrides, and captures, and turns **MCP Server** off; the XR application keeps running. Turn the switch back on to let clients reconnect.

### Controllers for input and pose control

Input injection and controller pose overrides act on PortalVR's active controller for that hand; with none, they return `CAPABILITY_UNAVAILABLE`. When an MCP client first becomes active and no controller is connected, PortalVR selects its mouse-and-keyboard controller so those tools have a target.

### Eye capture in 2D mode

A 2D PortalVR session normally asks SteamVR for a single shared eye image, which cannot provide distinct left and right captures. When an MCP client first becomes active during such a session, PortalVR offers a restart into a two-eye source layout. The visible output stays 2D and the desktop window keeps showing the left eye, while `openxr_capture_composited_image` can then select either eye. Only detected MCP activity starts this flow; there is no separate setting.

## Connect a client

### Stdio clients (recommended)

Most desktop MCP clients launch a server as a child process and talk newline-delimited JSON-RPC on its stdin/stdout. Point them at the proxy executable in your PortalVR installation:

```json
{
  "mcpServers": {
    "portalvr": {
      "command": "C:\\path\\to\\PortalVR\\steamvr-portalvr\\bin\\win64\\portalvr-mcp-proxy.exe"
    }
  }
}
```

The proxy reads PortalVR's private [discovery record and token](#local-files-and-configuration) itself, connects only to loopback listeners, and never accepts a remote address. Clients speak MCP `2024-11-05` to it: `initialize`, then `notifications/initialized`, then `tools/list` and `tools/call`. It forwards calls to the server concurrently, so responses can arrive in a different order from requests; match them by `id`.

When legacy HTTP+SSE is enabled, the proxy passes the server's `notifications/tools/list_changed` through to the client. When PortalVR is not running, `tools/list` still returns the 25 built-in tools from the proxy's embedded catalog, and `tools/call` returns the tool error `SERVER_OFFLINE`. The proxy reconnects automatically when the server returns. It writes only MCP messages to stdout and exits when stdin closes or its parent process exits.

Options, for diagnostics only: `--config PATH`, `--token-file PATH`, and `--discovery PATH` override the default [file locations](#local-files-and-configuration); `--start-host` lets an interactively launched proxy start the server when MCP is enabled and no server is running. `--registry-info` prints the embedded catalog's tool count and SHA-256, and `--registry-json` prints the catalog.

### HTTP clients

An HTTP client connects directly to the URLs in the discovery record and sends the bearer token on every request. Use [Streamable HTTP](#streamable-http-transport) at `/mcp` for new clients. [Legacy HTTP+SSE](#legacy-httpsse-transport) at `/sse` exists for clients that only implement MCP `2024-11-05`.

Read the URLs from `discovery.json` rather than assuming the default port. For example, in PowerShell:

```powershell
$mcp = "$env:LOCALAPPDATA\PortalVR\mcp"
$discovery = Get-Content "$mcp\discovery.json" | ConvertFrom-Json
$token = Get-Content "$mcp\token" -Raw
$meta = @{
  "io.modelcontextprotocol/protocolVersion" = "2026-07-28"
  "io.modelcontextprotocol/clientCapabilities" = @{}
}
$body = @{
  jsonrpc = "2.0"; id = 1; method = "tools/call"
  params = @{ name = "math_calc"; arguments = @{ expression = "2 + 3 * 4" }; _meta = $meta }
} | ConvertTo-Json -Depth 5 -Compress
Invoke-RestMethod -Method Post -Uri $discovery.mcp_url -Body $body -ContentType "application/json" -Headers @{
  Authorization = "Bearer $token"
  Accept = "application/json, text/event-stream"
  "MCP-Protocol-Version" = "2026-07-28"
  "Mcp-Method" = "tools/call"
  "Mcp-Name" = "math_calc"
}
```

## Local files and configuration

PortalVR keeps MCP state in `%LOCALAPPDATA%\PortalVR\mcp`:

| File | Contents |
| --- | --- |
| `config.json` | Server configuration, below |
| `token` | 64 hexadecimal characters, no newline: the bearer token for this server |
| `discovery.json` | Where the running server listens; removed when it stops |
| `kill-switch` | Marker that keeps MCP off while it exists; see [recovery](#recovery) |
| `adapter-credentials` | Private credentials between the server and PortalVR's runtime adapters |
| `logs\host.jsonl`, `logs\proxy.jsonl` | Structured server and proxy logs |

The directory and its files are readable only by the current user and SYSTEM, and the server refuses files with any other access. Never copy `token`, `discovery.json`, `adapter-credentials`, or logs into a package, bug report, or support bundle.

`discovery.json` is written after the listener is ready:

```json
{
  "pid": 43216,
  "process_start_time": 134398427693272199,
  "legacy_url": "http://127.0.0.1:8720/sse",
  "mcp_url": "http://127.0.0.1:8720/mcp",
  "protocols": ["2024-11-05", "2026-07-28"],
  "schema_fingerprint": "sha256:4b437b31…"
}
```

`pid` and `process_start_time` (the Windows process creation time) identify the owning process; a record whose process no longer matches is stale. A URL is empty when its transport is disabled. `schema_fingerprint` identifies the built-in tool catalog.

`config.json` accepts exactly these fields; unknown fields are rejected. Edit the file in place: an editor that replaces it with a new file can change its access permissions, and the server then refuses to start.

| Field | Default | Meaning |
| --- | --- | --- |
| `enabled` | `true` | Required. Mirrors the **MCP Server** switch; saving `false` stops a running server at once |
| `bind` | `["127.0.0.1", "::1"]` | Listener addresses; only these two loopback values are accepted |
| `port` | `8720` | TCP port for every bound address |
| `legacy_sse` | `true` | Serve `/sse` and `/message` |
| `streamable_http` | `true` | Serve `/mcp` |
| `require_token` | `true` | Require the bearer token. `false` is accepted only with `streamable_http` set to `false`; otherwise the server does not start |
| `allowed_origins` | `[]` | Exact `Origin` values permitted on requests |
| `history_frames` | `120` | Accepted for compatibility; frame history is fixed at 120 frames |
| `history_session_transitions` | `64` | Accepted for compatibility; session history is fixed at 64 transitions |
| `capture_max_dimension` | `2048` | Longest capture edge in pixels; larger images are downscaled |
| `capture_max_png_bytes` | `12582912` | Maximum encoded PNG size |
| `log_level` | `"info"` | `error`, `warn`, `info`, or `debug` |
| `limits.calls_per_second` | `20` | Requests per second per client, with bursts up to twice the rate; `math_*` tools, the two Unity conversion tools, and cancellations are exempt. Fixed per-category caps also apply |
| `limits.outstanding_per_client` | `64` | Concurrent calls per client |
| `limits.capture_per_second` | `2` | Captures per second |

Allowed origins, log level, rate limits, and capture limits apply while the server runs; removing an origin closes its sessions. `bind`, `port`, `require_token`, the transport switches, and the `history_*` fields are restart-only: a saved file that changes any of them is not applied until the next server start, including its other changes. Keep at least one transport enabled.

## Security model

The server is a local automation interface, not a remote-control service. Its safety boundaries are:

- It listens on numeric loopback addresses only, and validates the `Host` header against the bound address and port.
- A request carrying an `Origin` header is rejected with `403` unless that exact origin is in `allowed_origins`. `Origin: null` is always rejected. Because the list is empty by default, web pages in a browser cannot call the server.
- Every MCP route requires `Authorization: Bearer <token>` when `require_token` is on (the default). Only processes running as the same Windows user can read the token.
- Request bodies, connections, queues, and concurrent calls are bounded; overload produces `429` or `503` rather than blocking an XR timing thread.

Anyone who can run code as your Windows user while **MCP Server** is on can drive the running VR application: inject input, move the head and controllers, capture what you see, and request that the session exit. Turn the switch off when you do not need it.

Runtime logs contain event IDs, hashed client and application identities, counters, and durations. They do not contain tokens, tool arguments, images, scene data, or custom-tool results.

## JSON-RPC rules

Both transports carry JSON-RPC 2.0 in UTF-8:

- `jsonrpc` must be `"2.0"`. IDs may be strings or integers and are echoed exactly; a request may not use a null ID.
- A message without `id` is a notification and never receives a response, even when invalid.
- Batches, duplicate object keys, invalid UTF-8, nesting deeper than 64, strings or keys over 4096 bytes, objects with more than 1024 members, and messages with more than 16384 JSON values are rejected.
- A request body may be at most 1 MiB. A response, including a base64 capture, may be up to 18 MiB.
- Protocol errors use `-32700` (parse error), `-32600` (invalid request), `-32601` (unknown method), and `-32602` (invalid params).
- An unknown tool name, or invalid arguments for a known tool, is a [tool error](#tool-results-and-errors) with `isError: true`, not a JSON-RPC error.

On legacy HTTP+SSE and through the stdio proxy, `notifications/cancelled` with `params.requestId` cancels an outstanding call. The server stops waiting, drops commands that have not yet been applied, and suppresses the late result. A pose or input change that has already been applied stays applied. A direct [Streamable HTTP](#streamable-http-transport) client cancels a call by closing that request's connection. A separate `notifications/cancelled` request matches the call only when both requests carry the same [client nonce](#streamable-http-transport); without one it is accepted but has no effect.

Each call has a deadline, after which it fails with `TIMEOUT`:

| Tools | Deadline |
| --- | ---: |
| `math_*`, `convert_unity_pose_to_openxr`, `convert_openxr_pose_to_unity` | 1 s |
| Other built-in `openxr_*` tools | 2 s |
| `openxr_exit_session`, `openxr_capture_composited_image`, `openxr_list_spatial_entities` | 5 s |
| Application custom tools | 10 s |

## Legacy HTTP+SSE transport

This is the MCP `2024-11-05` transport.

1. Open `GET /sse` with `Accept: text/event-stream` and the bearer token. The response is `200` with `Content-Type: text/event-stream` and `Cache-Control: no-cache, no-transform`. Its first event names the session's message URL:

   ```text
   event: endpoint
   data: /message?session_id=87bebe83e97319e6dc7b4cd4d7de9a9c
   ```

2. Send each JSON-RPC message as `POST /message?session_id=…` with `Content-Type: application/json` and the same bearer token. The server answers `202 Accepted` with no body.
3. Read each response from the SSE stream as a `message` event:

   ```text
   event: message
   data: {"jsonrpc":"2.0","id":3,"result":{"content":[{"type":"text","text":"{\"result\":32}"}],"structuredContent":{"result":32},"isError":false}}
   ```

Begin with `initialize`:

```json
{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"my-client","version":"1.0"}}}
```

`clientInfo.name` and `clientInfo.version` must be nonempty strings. The server always answers with revision `2024-11-05`:

```json
{"jsonrpc":"2.0","id":1,"result":{"protocolVersion":"2024-11-05","serverInfo":{"name":"PortalVR XR Operator","version":"1.0.0"},"capabilities":{"tools":{"listChanged":true}}}}
```

Then send `notifications/initialized`. Until then, only `tools/list` is accepted; other requests fail with `-32600` "Server is not initialized". The supported methods are `initialize`, `notifications/initialized`, `notifications/cancelled`, `ping`, `tools/list`, and `tools/call`.

When the set of [application custom tools](#application-custom-tools) changes, the stream carries `{"jsonrpc":"2.0","method":"notifications/tools/list_changed"}`; list the tools again.

The stream sends a `: ping` comment after 15 seconds of inactivity. A session ID is bound to its stream and token, and expires 15 seconds after the stream closes. An unknown, expired, or differently authorized session ID gets `404`. Each session may have at most 32 queued responses; a client that stops reading its stream is disconnected. The server accepts at most 16 SSE streams.

## Streamable HTTP transport

`POST /mcp` implements stateless MCP revision `2026-07-28`. There is no `initialize` handshake and no session: each request is self-contained and receives one `application/json` response. `GET /mcp` and `DELETE /mcp` return `405`. A client that only implements the earlier session-based Streamable HTTP (with `initialize`) should use the [stdio proxy](#stdio-clients-recommended) or [legacy HTTP+SSE](#legacy-httpsse-transport) instead.

Every request needs these headers:

| Header | Value |
| --- | --- |
| `Authorization` | `Bearer <token>` |
| `Content-Type` | `application/json` |
| `Accept` | One header listing both `application/json` and `text/event-stream` |
| `MCP-Protocol-Version` | `2026-07-28` |
| `Mcp-Method` | The JSON-RPC `method`, exactly |
| `Mcp-Name` | For `tools/call` only: the tool name, exactly. A name that is not printable ASCII is sent as `=?base64?<base64 of UTF-8>?=` |

Every request's `params` carries MCP metadata. `io.modelcontextprotocol/protocolVersion` (`2026-07-28`) and `io.modelcontextprotocol/clientCapabilities` are required; `io.modelcontextprotocol/clientInfo` is optional and, if present, needs nonempty `name` and `version`. To make cancellation work across requests, also send the PortalVR extension `io.portalvr/clientNonce`: the same random 32-hex-character value, with the same `clientInfo`, on every request from one client (a nonce without `clientInfo` is rejected with `-32602`):

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "math_calc",
    "arguments": {"expression": "2 + 3 * 4"},
    "_meta": {
      "io.modelcontextprotocol/protocolVersion": "2026-07-28",
      "io.modelcontextprotocol/clientCapabilities": {}
    }
  }
}
```

Results add `resultType` and server metadata:

```json
{"jsonrpc":"2.0","id":2,"result":{"resultType":"complete","_meta":{"io.modelcontextprotocol/serverInfo":{"name":"PortalVR XR Operator","version":"1.0.0"}},"content":[{"type":"text","text":"{\"result\":14}"}],"structuredContent":{"result":14},"isError":false}}
```

The supported methods are `server/discover`, `tools/list`, `tools/call`, and `notifications/cancelled`. `server/discover` returns `supportedVersions: ["2026-07-28"]` and `capabilities: {"tools": {}}`. `server/discover` and `tools/list` results include `ttlMs: 300000` and `cacheScope: "private"`; this transport has no list-changed notification, so list again after the TTL expires or when a call names an unknown custom tool.

| Condition | Result |
| --- | --- |
| Missing or wrong token | `401` |
| Disallowed `Origin` | `403` |
| Rate limit reached | `429` with `Retry-After`, no JSON-RPC body |
| `Content-Type` is not JSON | `415` |
| `Accept` lacks either media type | `406` |
| Missing or invalid `_meta` | `400`, JSON-RPC `-32602` |
| `MCP-Protocol-Version`, `Mcp-Method`, or `Mcp-Name` disagrees with the body | `400`, JSON-RPC `-32020` "Header mismatch" |
| Unsupported protocol revision | `400`, JSON-RPC `-32022` |
| Unknown method | `404`, JSON-RPC `-32601` |
| Notification accepted, or call cancelled | `202`, empty body |
| Server overloaded or stopping | `503` with `Retry-After` |

## Health and status endpoints

These are plain HTTP endpoints, not MCP methods:

| Endpoint | Token | Result |
| --- | --- | --- |
| `GET /health/live` | Not required | `{"status":"live"}` while the process runs |
| `GET /health/ready` | Not required | `{"status":"ready"}` once the listener and tool catalog are ready, even with no XR application; otherwise `503` |
| `GET /status` | Required | Redacted transport, adapter, and catalog-fingerprint information; `404` when `require_token` is off |

Health responses include the headers `X-PortalVR-Capture-Active` and `X-PortalVR-Client-Connected` (`0` or `1`), and `X-PortalVR-Tool-Registry-Revision`, which increases when the custom-tool set changes.

## Tool results and errors

A successful call returns the result object twice: as compact JSON text in `content`, and as `structuredContent`. The capture tool instead returns an image content item. A failed call sets `isError: true` and puts `CODE: message` in the text:

```json
{"content":[{"type":"text","text":"NO_ACTIVE_SESSION: no active XR session is available"}],"structuredContent":{"error":{"code":"NO_ACTIVE_SESSION","message":"no active XR session is available","retryable":true}},"isError":true}
```

`retryable` says whether the same call may succeed later without changing its arguments.

| Code | Retryable | Meaning |
| --- | :---: | --- |
| `INVALID_ARGUMENT` | no | Unknown tool, or arguments that fail the schema or semantic rules |
| `NO_ACTIVE_RUNTIME` | yes | No XR runtime adapter is connected |
| `AMBIGUOUS_RUNTIME` | no | `openxr_get_instance_info` or `openxr_get_system_info` found several runtimes and none has an active session to choose by |
| `NO_ACTIVE_SESSION` | yes | No XR session is running |
| `AMBIGUOUS_SESSION` | no | Several sessions qualify; pass `session_handle` |
| `STALE_SESSION` | no | `session_handle` names a session that has ended or been given a new handle; read the current handle again |
| `BACKEND_OFFLINE` | yes | The runtime adapter disconnected, or its latest observation is stale |
| `CAPABILITY_UNAVAILABLE` | yes | The selected runtime cannot provide this information now |
| `SPACE_UNSUPPORTED` | no | The requested reference space is unavailable |
| `POSE_INVALID` | yes | The requested pose is not currently valid |
| `INPUT_UNSUPPORTED` | no | The control does not exist on the current controller profile |
| `BUSY` | yes | Server queue full; through the stdio proxy, also an HTTP rate limit |
| `TIMEOUT` | yes | The call missed its [deadline](#json-rpc-rules) |
| `CANCELLED` | yes | The call was cancelled |
| `EXPRESSION_ERROR` | no | `math_calc` expression is invalid |
| `DIVIDE_BY_ZERO` | no | Division by zero |
| `DEGENERATE_VECTOR`, `INVALID_QUATERNION` | no | Reserved; built-in tools report zero-length directions and invalid quaternions as `INVALID_ARGUMENT` |
| `PERMISSION_DENIED` | no | Reserved; the current server does not return it |
| `CAPTURE_FAILED` | yes | The composited image could not be captured |
| `CUSTOM_TOOL_ERROR` | yes | An application custom tool reported failure or an invalid result |
| `INTERNAL_ERROR` | yes | Unexpected server failure |
| `SERVER_OFFLINE` | yes | Returned by the stdio proxy when no server is running |

Error messages never include file paths, pipe names, secrets, or stack traces.

## Built-in tools

This section is the reference for all 25 built-in tools. `tools/list` returns them in the order of the [index](#tool-index), followed by any [application custom tools](#application-custom-tools). Each tool's advertised `inputSchema` is self-contained JSON Schema with `additionalProperties: false`, so an unknown argument is an `INVALID_ARGUMENT` error.

### Conventions

- **Vectors and quaternions** are JSON arrays: positions `[x, y, z]` in metres, quaternions `[x, y, z, w]`. Quaternion arguments are normalized before use; a zero or non-finite quaternion is rejected.
- **Coordinates** follow OpenXR: right-handed, +X right, +Y up, −Z forward.
- **`base_space`** is `local`, `stage`, `local_floor`, or `view`, default `local_floor`. The conversion tool's `source_space` and `target_space` take the same values.
- **`hand`** is `left` or `right`. **`pose_type`** is `grip` (default) or `aim`.
- **`session_handle`** is an optional positive integer; read it from [`openxr_get_session_info`](#openxr_get_session_info). When omitted, the server picks the active session. Handles change, so prefer omitting it; see [scope and compatibility](#scope-and-compatibility).
- **SteamVR limits:** head and grip poses, pose overrides, and every call for an OpenVR application go through SteamVR, which supports only `view`, `stage`, and `local_floor`; `local` returns `SPACE_UNSUPPORTED`. SteamVR treats `stage` and `local_floor` as the same space. `aim` poses can be read only from a focused OpenXR application; otherwise `aim` reads and overrides return `CAPABILITY_UNAVAILABLE`, and retrying will not help.
- **Timestamps:** names ending `_ns` are runtime monotonic nanoseconds; names ending `_utc` are `YYYY-MM-DDTHH:mm:ss.sssZ`.
- **Encoding:** numbers use the shortest round-trip decimal form, and object keys appear in the order shown.
- **Validation first:** arguments are checked against the schema and the rules below before a session is selected, so a malformed call returns `INVALID_ARGUMENT` even when no XR application is running.

**`_portalvr` metadata.** Results of the observation tools (the `openxr_get_*` tools and `openxr_convert_pose_between_spaces`) end with a `_portalvr` object:

| Field | Meaning |
| --- | --- |
| `schema_version` | `1` |
| `backend` | `steamvr` for both OpenVR and OpenXR applications on SteamVR |
| `session_generation` | Changes when the session's adapter restarts |
| `state_sequence` | Increases with each observed state change |
| `sample_monotonic_ns` | When the reported value was sampled |
| `age_ms` | How old the observation was when the result was built |
| `synthetic` | Always `true` for instance, system, session, frame, and profile results from SteamVR; for a pose, `true` while an MCP override supplies it |

The examples below omit `_portalvr` unless a tool returns different fields. The input and capture tools return their own `_portalvr` fields; the pose setters, `openxr_exit_session`, and the math and Unity conversion tools return none.

**Errors every XR tool can return.** Besides the errors listed per tool, any tool that reads or drives an XR application can return `NO_ACTIVE_RUNTIME`, `NO_ACTIVE_SESSION`, `AMBIGUOUS_SESSION`, `STALE_SESSION`, `BACKEND_OFFLINE`, `BUSY`, `TIMEOUT`, `CANCELLED`, and `INTERNAL_ERROR`. See [tool results and errors](#tool-results-and-errors).

### Tool index

Annotations: **R** read-only, **D** destructive, **I** idempotent, **W** open-world (depends on or affects the live XR application). Bold arguments are required.

| Tool | Arguments | Annotations | Needs |
| --- | --- | --- | --- |
| [`openxr_get_instance_info`](#openxr_get_instance_info) | `include_api_layers`, `include_enabled_extensions`, `include_available_extensions` | R I | Runtime |
| [`openxr_get_session_info`](#openxr_get_session_info) | `include_history` | R | Session |
| [`openxr_exit_session`](#openxr_exit_session) | `session_handle` | D I W | Focused OpenXR application |
| [`openxr_get_frame_info`](#openxr_get_frame_info) | `include_history` | R | Session with frames |
| [`openxr_get_system_info`](#openxr_get_system_info) | none | R I | Runtime |
| [`openxr_get_head_pose`](#openxr_get_head_pose) | `base_space`, `session_handle` | R W | Active headset |
| [`openxr_get_controller_pose`](#openxr_get_controller_pose) | **`hand`**, `pose_type`, `base_space`, `session_handle` | R W | Known controller |
| [`openxr_get_active_interaction_profile`](#openxr_get_active_interaction_profile) | `hand`, `session_handle` | R W | Known controller |
| [`openxr_list_spatial_entities`](#openxr_list_spatial_entities) | `base_space`, `session_handle` | R W | Not available |
| [`openxr_capture_composited_image`](#openxr_capture_composited_image) | `eye`, `session_handle` | R W | Session with frames |
| [`math_calc`](#math_calc) | **`expression`** | R I | Nothing |
| [`math_dot`](#math_dot) | **`vector_a`**, **`vector_b`** | R I | Nothing |
| [`math_cross`](#math_cross) | **`vector_a`**, **`vector_b`** | R I | Nothing |
| [`math_add`](#math_add) | **`a`**, **`b`** | R I | Nothing |
| [`math_sub`](#math_sub) | **`a`**, **`b`** | R I | Nothing |
| [`math_mul`](#math_mul) | **`a`**, **`b`** | R I | Nothing |
| [`math_div`](#math_div) | **`a`**, **`b`** | R I | Nothing |
| [`math_build_quat`](#math_build_quat) | **`front_direction`**, `up_direction`, `return_basis_vectors` | R I | Nothing |
| [`math_decompose_quat`](#math_decompose_quat) | **`quaternion`** | R I | Nothing |
| [`openxr_set_controller_input`](#openxr_set_controller_input) | **`hand`**, **`component`**, **`value`**, `sub_component`, `auto_release`, `hold_duration`, `session_handle` | W | Active controller |
| [`openxr_set_head_pose`](#openxr_set_head_pose) | `position`, `orientation`, `base_space`, `duration_seconds`, `session_handle` | I W | Active headset |
| [`openxr_set_controller_pose`](#openxr_set_controller_pose) | **`hand`**, `pose_type`, `position`, `orientation`, `base_space`, `duration_seconds`, `session_handle` | I W | Active controller |
| [`convert_unity_pose_to_openxr`](#convert_unity_pose_to_openxr) | `position`, `orientation` | R I | Nothing |
| [`convert_openxr_pose_to_unity`](#convert_openxr_pose_to_unity) | `position`, `orientation` | R I | Nothing |
| [`openxr_convert_pose_between_spaces`](#openxr_convert_pose_between_spaces) | `source_space`, `target_space`, `position`, `orientation`, `session_handle` | R W | Session with a head pose |

"Runtime" means a connected SteamVR runtime, with or without a running application. "Known controller" means PortalVR reports a controller profile for that hand; "active controller" means that controller is currently connected. See [controllers for input and pose control](#controllers-for-input-and-pose-control).

### `openxr_get_instance_info`

Gets the current XR application's name, the reporting component, the API version, and optionally its API layers and extensions.

| Argument | Type | Default | Meaning |
| --- | --- | --- | --- |
| `include_api_layers` | boolean | `false` | Include enabled API layer names |
| `include_enabled_extensions` | boolean | `false` | Include enabled extension names |
| `include_available_extensions` | boolean | `false` | Include extensions available to the instance |

This tool selects a runtime rather than a session and takes no `session_handle`.

```json
{"application":{"name":"Sample","version":1},"engine":{"name":"PortalVR SteamVR OpenXR API layer","version":1},"api_version":{"major":1,"minor":1,"patch":0},"timestamp_utc":"2026-08-22T18:00:00.000Z","enabled_extensions":["XR_EXT_local_floor"]}
```

- `application` is the OpenXR application name and version. For an OpenVR application it is the SteamVR application name, or `SteamVR` when none is known.
- `engine` names the PortalVR component reporting the data: `PortalVR SteamVR OpenXR API layer` for a focused OpenXR application, `PortalVR SteamVR server driver` otherwise. It does not identify the application's game engine.
- `api_layers`, `enabled_extensions`, and `available_extensions` appear only when requested, as arrays (possibly empty). Enabled names keep instance-creation order; available names are sorted. OpenVR applications report empty lists.

**Errors:** `NO_ACTIVE_RUNTIME`; `AMBIGUOUS_RUNTIME` when several runtimes are connected and none has an active session; `CAPABILITY_UNAVAILABLE`.

### `openxr_get_session_info`

Gets the selected session's handle and lifecycle state, optionally with its transition history. Use it to discover the current `session_handle`.

| Argument | Type | Default | Meaning |
| --- | --- | --- | --- |
| `include_history` | boolean | `false` | Include the retained lifecycle transitions |

```json
{"session_handle":42,"state":"FOCUSED","focused":true,"visible":true,"running":true,"history":[{"from":"VISIBLE","to":"FOCUSED","time_utc":"2026-08-22T18:00:00.000Z","reason":"runtime"}]}
```

- `state` is one of `IDLE`, `READY`, `SYNCHRONIZED`, `VISIBLE`, `FOCUSED`, `STOPPING`, `EXITING`, `LOSS_PENDING`, or `UNKNOWN`.
- `focused` is true only in `FOCUSED`; `visible` is true in `VISIBLE` or `FOCUSED`; `running` is true while the session runs.
- `history`, present only when requested, lists up to the last 64 transitions, oldest first. `reason` is always `runtime`.

**Errors:** only the common XR errors.

### `openxr_exit_session`

Asks the selected application to end its XR session. This may cause the application to exit.

| Argument | Type | Default | Meaning |
| --- | --- | --- | --- |
| `session_handle` | number | active session | Session to end |

It works only for a focused OpenXR application, through `xrRequestExitSession`, and does not promise that the process terminates.

```json
{"session_handle":42,"exit_requested":true,"observable_state_sequence":4824}
```

**Errors:** `CAPABILITY_UNAVAILABLE` for OpenVR applications and for OpenXR applications that are not focused, because SteamVR gives drivers no safe way to close an application; `INTERNAL_ERROR` if the application's runtime rejects the request.

### `openxr_get_frame_info`

Gets the latest frame's predicted display timing, render flag, submitted display time, blend mode, and composition layers, optionally with recent frames.

| Argument | Type | Default | Meaning |
| --- | --- | --- | --- |
| `include_history` | boolean | `false` | Include retained frames |

```json
{"frame_id":1234,"predicted_display_time_ns":9876543210,"predicted_display_period_ns":11111111,"should_render":true,"display_time_ns":9876543210,"environment_blend_mode":"OPAQUE","layer_count":1,"layers":[{"index":0,"type":"projection","flags":0,"view_count":2}],"submission_complete":true}
```

- `environment_blend_mode` is `OPAQUE`, `ADDITIVE`, `ALPHA_BLEND`, or `UNKNOWN`.
- Each layer's `type` is `projection`, `projection_depth`, `quad`, `cube`, `cylinder`, `equirect`, `passthrough`, or `unknown`.
- `history`, present only when requested, lists up to the last 120 frames in the same shape, oldest first.

**Errors:** `CAPABILITY_UNAVAILABLE` until the session has submitted a frame; `BACKEND_OFFLINE` when the latest frame is older than two display periods plus 50 ms, for example while the application is paused.

### `openxr_get_system_info`

Gets the XR system's identity, graphics limits, and tracking capabilities. Takes no arguments.

```json
{"system_id":1,"vendor_id":0,"system_name":"PortalVR","graphics_properties":{"max_swapchain_image_width":2016,"max_swapchain_image_height":2240,"max_layer_count":1},"tracking_properties":{"orientation_tracking":true,"position_tracking":true}}
```

For an OpenVR application, the SteamVR driver reports an opaque per-session `system_id`, the headset's recommended render size as the maximum swapchain size, and a `max_layer_count` of `1`. A focused OpenXR application reports the values its runtime returned. Like instance information, this tool selects a runtime and takes no `session_handle`.

**Errors:** `NO_ACTIVE_RUNTIME`; `AMBIGUOUS_RUNTIME`; `CAPABILITY_UNAVAILABLE`.

### `openxr_get_head_pose`

Gets a fresh headset pose. Always call it again when the current pose matters; the user may have moved.

| Argument | Type | Default | Meaning |
| --- | --- | --- | --- |
| `base_space` | string | `local_floor` | Space to express the pose in |
| `session_handle` | number | active session | Session to sample |

```json
{"position":[0,1.65,0],"orientation":[0,0,0,1],"position_valid":true,"orientation_valid":true,"position_tracked":true,"orientation_tracked":true,"active":true,"simulated":false,"base_space":"local_floor","sample_time_ns":9876543210}
```

- `*_valid` says the component is usable; `*_tracked` says it is currently tracked rather than held or estimated.
- `simulated` is true while an [`openxr_set_head_pose`](#openxr_set_head_pose) override supplies the pose.
- `linear_velocity` and `angular_velocity` arrays follow `simulated` when the runtime reports them.

**Errors:** `CAPABILITY_UNAVAILABLE` while the headset is inactive or has not reported a pose; `SPACE_UNSUPPORTED` for `local`; `POSE_INVALID` when the head pose is inactive or not valid.

### `openxr_get_controller_pose`

Gets a fresh grip or aim pose for one controller. A simulated pose matches what the application observes.

| Argument | Type | Default | Meaning |
| --- | --- | --- | --- |
| `hand` | string | required | `left` or `right` |
| `pose_type` | string | `grip` | `grip` or `aim` |
| `base_space` | string | `local_floor` | Space to express the pose in |
| `session_handle` | number | active session | Session to sample |

The result has the [head pose](#openxr_get_head_pose) fields followed by `hand` and `pose_type`, then `base_space` and `sample_time_ns`. A controller that PortalVR knows but that is currently disconnected returns successfully with `active: false`.

**Errors:** `CAPABILITY_UNAVAILABLE` when PortalVR reports no controller profile, and for `aim` except on a focused OpenXR application; `SPACE_UNSUPPORTED`; `POSE_INVALID` when an active controller's pose is not valid.

### `openxr_get_active_interaction_profile`

Gets the interaction profile bound for one or both hands. Use it to see which controls can drive the application.

| Argument | Type | Default | Meaning |
| --- | --- | --- | --- |
| `hand` | string | `both` | `left`, `right`, or `both` |
| `session_handle` | number | active session | Session to read |

```json
{"hand":"left","interaction_profile":"/interaction_profiles/oculus/touch_controller"}
```

With `hand: "both"` the result is `{"left":{"interaction_profile":…},"right":{"interaction_profile":…}}`. `null` means no profile is currently bound. PortalVR's controllers report `/interaction_profiles/oculus/touch_controller`, `/interaction_profiles/valve/index_controller` (Index emulation), or `/interaction_profiles/htc/vive_controller`; a focused OpenXR application reports the profile its runtime bound.

**Errors:** `CAPABILITY_UNAVAILABLE` when no controller profile is known.

### `openxr_list_spatial_entities`

Would list physical-environment entities such as floors, walls, and tables. It is listed for compatibility, but the current server has no scene source.

| Argument | Type | Default | Meaning |
| --- | --- | --- | --- |
| `base_space` | string | `local_floor` | Space for entity poses |
| `session_handle` | number | active session | Session to query |

**Errors:** with an active session it always returns `CAPABILITY_UNAVAILABLE`; do not retry it in a loop. Without one it returns `NO_ACTIVE_SESSION`.

### `openxr_capture_composited_image`

Captures the final composited image for one eye, after PortalVR's compositor, as a PNG.

| Argument | Type | Default | Meaning |
| --- | --- | --- | --- |
| `eye` | string | `left` | `left` or `right` |
| `session_handle` | number | active session | Session to capture |

```json
{"content":[{"type":"image","data":"iVBORw0KGgo…","mimeType":"image/png"}],"structuredContent":{"eye":"left","width":1920,"height":1920,"format":"RGBA8_SRGB","frame_id":1234,"capture_time_utc":"2026-08-22T18:00:00.000Z","_portalvr":{"source_width":1920,"source_height":1920,"downscaled":false}},"isError":false}
```

- `data` is base64 PNG without a `data:` prefix. Exactly one eye is returned.
- Images larger than `capture_max_dimension` are downscaled, then shrunk further in steps until the PNG fits `capture_max_png_bytes` (effectively at most 12 MiB; larger settings have no effect). `_portalvr.source_width`, `source_height`, and `downscaled` report it.
- Captures are limited to `limits.capture_per_second` and one at a time per client.
- In a 2D PortalVR session both eyes show the same image until the [two-eye restart](#eye-capture-in-2d-mode).

**Errors:** `CAPABILITY_UNAVAILABLE` until the session has frames and a capture source; `CAPTURE_FAILED`.

### `math_calc`

Evaluates one scalar expression safely. Vectors are not supported.

| Argument | Type | Default | Meaning |
| --- | --- | --- | --- |
| `expression` | string | required | The expression |

`{"expression":"sqrt(2) * sin(pi / 4) + max(1, 2, 3)"}` returns `{"result":4}`.

The grammar has numbers, `+ - * /`, unary minus, `**` for exponentiation, parentheses, the constants `pi` and `e`, and the functions `abs`, `sqrt`, `exp`, `log` (natural, or `log(x, base)`), `log10`, `pow`, `sin`, `cos`, `tan`, `asin`, `acos`, `atan`, `atan2`, `floor`, `ceil`, `round` (ties to even; optionally `round(x, digits)` with an integer `digits` from −15 to 15), `min`, and `max`. Angles are radians.

**Errors:** `EXPRESSION_ERROR` for `^`, `%`, variables, unknown functions, syntax errors, division by zero, and non-finite results.

### `math_dot`

Computes the dot product of two 3-vectors.

| Argument | Type | Default | Meaning |
| --- | --- | --- | --- |
| `vector_a` | 3 numbers | required | First vector |
| `vector_b` | 3 numbers | required | Second vector |

`{"vector_a":[1,2,3],"vector_b":[4,5,6]}` returns `{"result":32}`.

**Errors:** `INVALID_ARGUMENT` when a vector does not have exactly three finite numbers.

### `math_cross`

Computes the right-handed cross product `vector_a × vector_b`.

| Argument | Type | Default | Meaning |
| --- | --- | --- | --- |
| `vector_a` | 3 numbers | required | First vector |
| `vector_b` | 3 numbers | required | Second vector |

`{"vector_a":[1,0,0],"vector_b":[0,1,0]}` returns `{"result":[0,0,1]}`.

**Errors:** `INVALID_ARGUMENT` as for `math_dot`.

### `math_add`

Computes `a + b`, componentwise.

| Argument | Type | Default | Meaning |
| --- | --- | --- | --- |
| `a` | number or 3 numbers | required | First addend |
| `b` | number or 3 numbers | required | Second addend |

Two numbers give a number; two vectors combine per component; a number and a vector apply the number to each component, in operand order. `{"a":[1,2,3],"b":[4,5,6]}` returns `{"result":[5,7,9]}`.

**Errors:** `INVALID_ARGUMENT` for a vector that does not have exactly three numbers.

### `math_sub`

Computes `a − b`, componentwise.

| Argument | Type | Default | Meaning |
| --- | --- | --- | --- |
| `a` | number or 3 numbers | required | Minuend |
| `b` | number or 3 numbers | required | Subtrahend |

Two numbers give a number; two vectors combine per component; a number and a vector apply the number to each component, in operand order. `{"a":10,"b":[1,2,3]}` returns `{"result":[9,8,7]}`.

**Errors:** `INVALID_ARGUMENT` for a vector that does not have exactly three numbers.

### `math_mul`

Computes `a × b`, componentwise (not a dot or cross product).

| Argument | Type | Default | Meaning |
| --- | --- | --- | --- |
| `a` | number or 3 numbers | required | First factor |
| `b` | number or 3 numbers | required | Second factor |

Two numbers give a number; two vectors combine per component; a number and a vector apply the number to each component, in operand order. `{"a":[1,2,3],"b":2}` returns `{"result":[2,4,6]}`.

**Errors:** `INVALID_ARGUMENT` for a vector that does not have exactly three numbers.

### `math_div`

Computes `a ÷ b`, componentwise.

| Argument | Type | Default | Meaning |
| --- | --- | --- | --- |
| `a` | number or 3 numbers | required | Dividend |
| `b` | number or 3 numbers | required | Divisor |

Two numbers give a number; two vectors combine per component; a number and a vector apply the number to each component, in operand order. `{"a":[2,4,6],"b":2}` returns `{"result":[1,2,3]}`.

**Errors:** `INVALID_ARGUMENT` for a vector that does not have exactly three numbers. `DIVIDE_BY_ZERO` when the divisor, or any divisor component, is zero.

### `math_build_quat`

Builds the quaternion whose local −Z faces `front_direction` and whose local +Y follows `up_direction` as closely as possible.

| Argument | Type | Default | Meaning |
| --- | --- | --- | --- |
| `front_direction` | 3 numbers | required | Direction for local −Z; normalized first |
| `up_direction` | 3 numbers | `[0,1,0]` | Approximate local +Y; normalized first |
| `return_basis_vectors` | boolean | `false` | Also return the orthonormal basis |

`{"front_direction":[1,0,0],"return_basis_vectors":true}` returns:

```json
{"quaternion":[0,-0.7071067811865475,0,0.7071067811865476],"front_direction":[1,0,0],"up_direction":[0,1,0],"right_direction":[0,0,1]}
```

Without `return_basis_vectors`, only `quaternion` is returned.

**Errors:** `INVALID_ARGUMENT` when either direction is zero or the two are parallel.

### `math_decompose_quat`

Returns the directions of a quaternion's local axes: front (−Z), up (+Y), and right (+X).

| Argument | Type | Default | Meaning |
| --- | --- | --- | --- |
| `quaternion` | 4 numbers | required | `[x, y, z, w]`; normalized first |

`{"quaternion":[0,0,0,1]}` returns `{"front_direction":[0,0,-1],"up_direction":[0,1,0],"right_direction":[1,0,0]}`.

**Errors:** `INVALID_ARGUMENT` for a zero or non-finite quaternion.

### `openxr_set_controller_input`

Sets one button, trigger, grip, or thumbstick axis on the [active controller](#controllers-for-input-and-pose-control), optionally returning it to zero after a hold.

| Argument | Type | Default | Meaning |
| --- | --- | --- | --- |
| `hand` | string | required | `left` or `right` |
| `component` | string | required | Control; see the table below |
| `value` | number | required | Buttons exactly `0` or `1`; `Trigger` and `Grip` in `[0, 1]`; `Thumbstick` in `[-1, 1]` |
| `sub_component` | string | none | `X` or `Y`; required for `Thumbstick`, forbidden otherwise |
| `auto_release` | boolean | `false` | Return the control to zero after `hold_duration` |
| `hold_duration` | number | `0.2` | Seconds to hold when `auto_release` is true, `0.001` to `60` |
| `session_handle` | number | active session | Session to drive |

| Hand | `component` values |
| --- | --- |
| `left` | `X`, `Y`, `Menu`, `Thumbstick`, `ThumbstickClick`, `Trigger`, `Grip` |
| `right` | `A`, `B`, `System`, `Thumbstick`, `ThumbstickClick`, `Trigger`, `Grip` |

Without `auto_release`, the value holds until changed or [cleared](#override-lifetime).

```json
{"session_handle":42,"hand":"right","component":"Trigger","sub_component":null,"value":1,"auto_release":true,"release_time_ns":9876743210,"applied":true,"observable_state_sequence":4822,"_portalvr":{"component_path":"/input/trigger/value","controller_type":"oculus_touch","legacy_binding_installed":true,"visibility":"binding_dependent"}}
```

- `release_time_ns` is `null` without `auto_release`.
- `applied` means SteamVR accepted the value. Whether the application reacts depends on its input bindings.
- `_portalvr.component_path` is the SteamVR input path that was set; it depends on the controller, for example `/input/joystick/x` on Touch and `/input/thumbstick/x` on Index.
- `_portalvr.controller_type` is `oculus_touch`, `knuckles` (Index), `vive_controller`, `portalvr`, or `unknown`. `legacy_binding_installed` says a controller profile is known for that hand.
- `_portalvr.visibility` is `binding_dependent` for ordinary controls and `runtime_only` for `System`, which SteamVR reserves. Injecting `System` normally opens the SteamVR dashboard, which [clears all overrides](#override-lifetime).

**Errors:** `INVALID_ARGUMENT` for a component that does not exist on that hand, a value out of range, or a missing or forbidden `sub_component`; `CAPABILITY_UNAVAILABLE` when no controller is active; `INPUT_UNSUPPORTED` when the active controller lacks the control.

### `openxr_set_head_pose`

Overrides all or part of the headset pose, instantly or with smooth interpolation. The application observes the simulated pose.

| Argument | Type | Default | Meaning |
| --- | --- | --- | --- |
| `position` | 3 numbers | current | Target position in metres, in `base_space` |
| `orientation` | 4 numbers | current | Target orientation quaternion |
| `base_space` | string | `local_floor` | Space of the target |
| `duration_seconds` | number | `0` | Interpolation time, `0` to `60`; `0` jumps |
| `session_handle` | number | active session | Session to drive |

Supply `position`, `orientation`, or both. Moving the head moves the whole simulated rig: controllers and hands keep their live motion relative to the physical head but follow it to the new head pose. A head target in `view` space is applied relative to the view at the moment the command takes effect. Interpolation is linear for position and normalized shortest-arc for orientation.

```json
{"session_handle":42,"target":"head","target_pose":{"position":[0,1.7,0],"orientation":[0,0,0,1]},"base_space":"local_floor","duration_seconds":0,"applied":true,"simulated":true,"observable_state_sequence":4823}
```

`target_pose` is the full resolved target, including any component you omitted.

**Errors:** `INVALID_ARGUMENT` when neither `position` nor `orientation` is supplied or a value is out of range; `CAPABILITY_UNAVAILABLE` while the headset is inactive; `SPACE_UNSUPPORTED` for `local`; `POSE_INVALID` when the current head pose is not valid.

### `openxr_set_controller_pose`

Overrides all or part of one controller's grip or aim pose, instantly or with smooth interpolation.

| Argument | Type | Default | Meaning |
| --- | --- | --- | --- |
| `hand` | string | required | `left` or `right` |
| `pose_type` | string | `grip` | `grip` or `aim` |
| `position`, `orientation`, `base_space`, `duration_seconds`, `session_handle` | | | As for [`openxr_set_head_pose`](#openxr_set_head_pose) |

The result matches the head-pose result with `target: "controller"` and `hand` and `pose_type` after `target`. A controller override applies after any head-rig move and wins for that controller; a target in `view` follows the simulated head.

For a focused OpenXR application, SteamVR derives both grip and aim from one controller origin, so an `aim` override moves that origin and the result still reports `aim`. For other applications `aim` overrides are unavailable.

**Errors:** as for `openxr_set_head_pose`, plus `CAPABILITY_UNAVAILABLE` when no controller is active on that hand or for `aim` outside a focused OpenXR application, and `POSE_INVALID` when the controller's current pose is not valid, for example while it is connected but untracked.

#### Override lifetime

A new override replaces the previous one for the same target, starting from the currently effective pose. An override or held input lasts until it is replaced, the XR session ends, MCP control is released (for example when the server stops), or **Emergency MCP Clear** is used. PortalVR also clears every override and held input at once when the SteamVR dashboard opens or closes, a PortalVR controller is deactivated, the headset becomes inactive, or the scene application changes. Physical tracking then resumes immediately.

Overrides and held inputs survive the disconnect of the client that set them, so a test harness should restore or clear them itself.

### `convert_unity_pose_to_openxr`

Converts a position and/or quaternion from Unity's left-handed convention to OpenXR's right-handed one. Needs no XR application.

| Argument | Type | Default | Meaning |
| --- | --- | --- | --- |
| `position` | 3 numbers | none | Unity position |
| `orientation` | 4 numbers | none | Unity quaternion |

The transform mirrors Z: position `[x, y, z]` becomes `[x, y, -z]`, and quaternion `[x, y, z, w]` becomes `[-x, -y, z, w]` after normalization, with `w ≥ 0`. Only supplied components are returned. `{"position":[1,2,3],"orientation":[0,0.7071068,0,0.7071068]}` returns:

```json
{"position":[1,2,-3],"orientation":[0,-0.7071067811865475,0,0.7071067811865475]}
```

**Errors:** `INVALID_ARGUMENT` when neither component is supplied or the quaternion is zero.

### `convert_openxr_pose_to_unity`

Converts a position and/or quaternion from OpenXR's right-handed convention to Unity's left-handed one. Needs no XR application.

| Argument | Type | Default | Meaning |
| --- | --- | --- | --- |
| `position` | 3 numbers | none | OpenXR position |
| `orientation` | 4 numbers | none | OpenXR quaternion |

The transform is the same Z mirror as [`convert_unity_pose_to_openxr`](#convert_unity_pose_to_openxr), which is its own inverse. `{"position":[1,2,-3]}` returns `{"position":[1,2,3]}`.

**Errors:** `INVALID_ARGUMENT` when neither component is supplied or the quaternion is zero.

### `openxr_convert_pose_between_spaces`

Expresses a pose given in one reference space in another, using fresh tracking data. With no pose, it converts the source space's origin.

| Argument | Type | Default | Meaning |
| --- | --- | --- | --- |
| `source_space` | string | `local_floor` | Space the input pose is in |
| `target_space` | string | `stage` | Space to express it in |
| `position` | 3 numbers | `[0,0,0]` | Input position |
| `orientation` | 4 numbers | `[0,0,0,1]` | Input orientation |
| `session_handle` | number | active session | Session whose spaces to use |

```json
{"source_space":"view","target_space":"local_floor","position":[0,1.65,0],"orientation":[0,0,0,1],"position_valid":true,"orientation_valid":true,"sample_time_ns":9876543210}
```

Through SteamVR, `stage` and `local_floor` are the same space, so converting between them returns the input unchanged, and `local` is unsupported. A focused OpenXR application converts with its own OpenXR spaces and can use only the spaces it created, so even the default conversion can return `SPACE_UNSUPPORTED`.

**Errors:** `CAPABILITY_UNAVAILABLE` until the headset reports a pose; `SPACE_UNSUPPORTED`; `POSE_INVALID` when the converted pose is not valid.

## Application custom tools

An OpenXR application can add its own tools through the `XR_METAX1_agentic_external_tool` extension, version 1. PortalVR provides it through the explicit OpenXR API layer `XR_APILAYER_PORTALVR_steamvr_mcp_metax1`, which SteamVR lists as **Application controlled**. The application enables that layer and the extension before `xrCreateInstance`, then calls `xrAgenticRegisterExternalToolMETAX1` for each tool with a name, a description, parameter descriptors, and a callback. Ordinary MCP use does not need this layer, and the layer is registered only while **MCP Server** is on, so enumerate API layers before requesting it.

The C declarations ship with PortalVR in `steamvr-portalvr\mcp\sdk\metax1_agentic_external_tool.h`: the extension name, the structure types `XrAgenticExternalToolParameterMETAX1` (type `1000679000`) and `XrAgenticExternalToolRegisterInfoMETAX1` (type `1000679001`), the parameter types (string `1`, number `2`, boolean `3`, array `4`, object `5`), the callback flags, and the function pointer types. The callback has this signature:

```c
XrResult callback(char *resultBuffer, uint32_t resultBufferCapacityInput,
                  uint32_t *resultBufferCountOutput, const char *parameters, void *userData);
```

- Tool names must match `^[A-Za-z0-9_.-]{1,63}$`. Names beginning `openxr_`, `math_`, `convert_`, or `portalvr.` are reserved, and duplicate names in one instance fail.
- Registered tools follow the built-ins in `tools/list`, in registration order, and are listed and callable only while the application's session is focused and selected. Their schemas are objects with `additionalProperties: false` built from the parameter descriptors.
- Tools are retired when the instance is destroyed or the application disconnects; version 1 has no unregister call.
- `parameters` is the call's arguments as JSON text. The layer calls the callback twice, in the usual OpenXR two-call pattern: first with a null buffer and zero capacity to read the required size, then with a buffer of exactly that size. The count includes the terminating NUL, and the result may be up to 1 MiB.
- The result must be JSON. An object becomes the tool result as is; any other JSON value is wrapped as `{"result": …}`. A failing callback, a size mismatch, or invalid JSON is reported as `CUSTOM_TOOL_ERROR`, and calls have a 10-second deadline.
- `toolCallbackInfoFlags` chooses the thread. With no flag, the callback runs on a layer worker thread. With `XR_AGENTIC_EXTERNAL_TOOL_CALLBACK_INFO_APPLICATION_MAIN_THREAD_BIT_METAX1` it runs inside the application's `xrPollEvent` calls; with `XR_AGENTIC_EXTERNAL_TOOL_CALLBACK_INFO_RENDERING_MAIN_THREAD_BIT_METAX1` it runs inside `xrWaitFrame`. Setting both fails registration with `XR_ERROR_VALIDATION_FAILURE`.

## Recovery

Use **Emergency MCP Clear** on the System page first. If PortalVR's UI is unavailable:

1. Create an empty file `%LOCALAPPDATA%\PortalVR\mcp\kill-switch`. The driver and OpenXR layer see it directly and drop pose overrides and pending input without waiting for the server.
2. End only the `portalvr-mcp-host.exe` process located in PortalVR's `steamvr-portalvr\bin\win64` folder.
3. Delete the stale `%LOCALAPPDATA%\PortalVR\mcp\discovery.json`.

MCP stays off while `kill-switch` exists. Deleting it re-enables MCP at the next PortalVR start unless **MCP Server** is off; turning **MCP Server** on also deletes it. Repair damaged binaries with PortalVR's normal installer.

## Compatibility notes

This reference was checked against the PortalVR MCP implementation on **2026-11-22**. The transport, JSON-RPC, error, and math examples were confirmed against a running server; pose, input, exit, and capture results follow the implementation's encoders and were not exercised against a live XR application for this document.

Tool names, arguments, and annotations follow the Meta XR Operator surface, but PortalVR does not reproduce that product's undocumented result formats byte for byte. PortalVR results always include `structuredContent`, use explicit `_ns` and `_utc` field names, and add `_portalvr` metadata to most XR results.
