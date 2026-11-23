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
| `GET /status` | Required | Redacted transport, adapter, and catalog-fingerprint information |

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

`tools/list` returns the built-in tools in the order below, followed by any [application custom tools](#application-custom-tools). Each tool's `inputSchema` is self-contained JSON Schema with `additionalProperties: false`, and its `annotations` give `readOnlyHint`, `destructiveHint`, `idempotentHint`, and `openWorldHint`.

### Conventions

- **Vectors and quaternions** are JSON arrays: positions `[x, y, z]` in metres, quaternions `[x, y, z, w]`.
- **Coordinates** follow OpenXR: right-handed, +X right, +Y up, −Z forward.
- **`base_space`** is `local`, `stage`, `local_floor`, or `view`, default `local_floor`. The conversion tool's `source_space` and `target_space` take the same values.
- **`hand`** is `left` or `right`. **`pose_type`** is `grip` (default) or `aim`.
- **`session_handle`** is an optional positive integer returned by session tools. When omitted, the server picks the active session, or fails with `AMBIGUOUS_SESSION` or `NO_ACTIVE_SESSION`.
- **SteamVR limits:** head and grip poses, pose overrides, and every call for an OpenVR application go through SteamVR, which supports only `view`, `stage`, and `local_floor`; `local` returns `SPACE_UNSUPPORTED`. SteamVR treats `stage` and `local_floor` as the same space. `aim` poses can be read only from a focused OpenXR application; otherwise `aim` reads and overrides return `CAPABILITY_UNAVAILABLE`, and retrying will not help. `openxr_convert_pose_between_spaces` uses the application's own OpenXR spaces when it is a focused OpenXR application.
- **Timestamps:** names ending `_ns` are runtime monotonic nanoseconds; names ending `_utc` are `YYYY-MM-DDTHH:mm:ss.sssZ`.
- Numbers use the shortest round-trip decimal form, and object keys appear in a fixed order.

Observation results (the `openxr_get_*` tools and `openxr_convert_pose_between_spaces`) include a `_portalvr` object with `schema_version`, `backend` (`steamvr` for both OpenVR and OpenXR applications on SteamVR), `session_generation`, `state_sequence`, `sample_monotonic_ns`, `age_ms` (how old the observation is), and `synthetic`. `synthetic` is always true for instance, system, session, frame, and profile results from SteamVR, and true for a pose while an MCP override supplies it. The examples below omit `_portalvr`. The input and capture tools return their own `_portalvr` fields, shown below; the pose setters and `openxr_exit_session` return none.

### Summary

| Tool | Arguments | Effect |
| --- | --- | --- |
| `openxr_get_instance_info` | `include_api_layers`, `include_enabled_extensions`, `include_available_extensions` (booleans, default false) | Reads |
| `openxr_get_session_info` | `include_history` (default false) | Reads |
| `openxr_exit_session` | `session_handle` | Asks an OpenXR application to exit |
| `openxr_get_frame_info` | `include_history` (default false) | Reads |
| `openxr_get_system_info` | none | Reads |
| `openxr_get_head_pose` | `session_handle`, `base_space` | Reads |
| `openxr_get_controller_pose` | **`hand`**, `pose_type`, `base_space`, `session_handle` | Reads |
| `openxr_get_active_interaction_profile` | `hand` (`left`, `right`, or `both`; default `both`), `session_handle` | Reads |
| `openxr_list_spatial_entities` | `base_space`, `session_handle` | Not yet available |
| `openxr_capture_composited_image` | `eye` (`left` or `right`; default `left`), `session_handle` | Reads pixels |
| `math_calc` | **`expression`** | Pure |
| `math_dot`, `math_cross` | **`vector_a`**, **`vector_b`** | Pure |
| `math_add`, `math_sub`, `math_mul`, `math_div` | **`a`**, **`b`** | Pure |
| `math_build_quat` | **`front_direction`**, `up_direction` (default `[0,1,0]`), `return_basis_vectors` (default false) | Pure |
| `math_decompose_quat` | **`quaternion`** | Pure |
| `openxr_set_controller_input` | **`hand`**, **`component`**, **`value`**, `sub_component`, `auto_release`, `hold_duration`, `session_handle` | Injects input |
| `openxr_set_head_pose` | `position`, `orientation`, `base_space`, `duration_seconds`, `session_handle` | Overrides the head pose |
| `openxr_set_controller_pose` | **`hand`**, `pose_type`, `position`, `orientation`, `base_space`, `duration_seconds`, `session_handle` | Overrides a controller pose |
| `convert_unity_pose_to_openxr` | `position`, `orientation` | Pure |
| `convert_openxr_pose_to_unity` | `position`, `orientation` | Pure |
| `openxr_convert_pose_between_spaces` | `source_space` (default `local_floor`), `target_space` (default `stage`), `position` (default origin), `orientation` (default identity), `session_handle` | Reads the space relation |

Bold arguments are required.

### Application and session

`openxr_get_instance_info` returns the application, the reporting component as `engine` (`PortalVR SteamVR OpenXR API layer` for OpenXR applications, `PortalVR SteamVR server driver` for OpenVR applications), the API version, and optionally the requested name lists:

```json
{"application":{"name":"Sample","version":1},"engine":{"name":"PortalVR SteamVR OpenXR API layer","version":1},"api_version":{"major":1,"minor":1,"patch":0},"timestamp_utc":"2026-08-22T18:00:00.000Z","enabled_extensions":["XR_EXT_local_floor"]}
```

`openxr_get_system_info` returns `system_id`, `vendor_id`, `system_name`, `graphics_properties` (`max_swapchain_image_width`, `max_swapchain_image_height`, `max_layer_count`), and `tracking_properties` (`orientation_tracking`, `position_tracking`).

`openxr_get_session_info` returns `session_handle`, `state` (an uppercase OpenXR session state such as `FOCUSED`), and the booleans `focused`, `visible`, and `running`. With `include_history`, `history` lists retained transitions as `{"from","to","time_utc","reason"}`.

`openxr_get_frame_info` returns `frame_id`, `predicted_display_time_ns`, `predicted_display_period_ns`, `should_render`, `display_time_ns`, `environment_blend_mode` (`OPAQUE`, `ADDITIVE`, or `ALPHA_BLEND`), `layer_count`, `layers` (`index`, `type`, `flags`, `view_count`), and `submission_complete`. With `include_history`, `history` lists earlier frames in the same shape.

`openxr_exit_session` asks a focused OpenXR application to end its session through `xrRequestExitSession`. It does not promise that the process terminates. It returns `{"session_handle":42,"exit_requested":true,"observable_state_sequence":4824}`. SteamVR offers drivers no safe way to close an OpenVR application, so for OpenVR applications the tool returns `CAPABILITY_UNAVAILABLE`.

### Poses and profiles

`openxr_get_head_pose` takes a fresh sample:

```json
{"position":[0,1.65,0],"orientation":[0,0,0,1],"position_valid":true,"orientation_valid":true,"position_tracked":true,"orientation_tracked":true,"active":true,"simulated":false,"base_space":"local_floor","sample_time_ns":9876543210}
```

`linear_velocity` and `angular_velocity` arrays appear when the runtime reports them. `simulated` is true while an MCP pose override supplies the value. `openxr_get_controller_pose` returns the same shape plus `hand` and `pose_type`. An inactive controller returns `active: false` when the runtime reports it as inactive; otherwise a pose that is not available fails with `POSE_INVALID`.

`openxr_get_active_interaction_profile` returns `{"hand":"left","interaction_profile":"/interaction_profiles/valve/index_controller"}` for one hand, or `{"left":{"interaction_profile":…},"right":{"interaction_profile":…}}` for both. A `null` profile means none is currently selected.

### Scene and capture

`openxr_list_spatial_entities` is listed for compatibility, but the current server has no scene source: with an active session it always returns `CAPABILITY_UNAVAILABLE`. Do not retry it in a loop.

`openxr_capture_composited_image` returns the final composited image for one eye as a base64 PNG, without a `data:` prefix:

```json
{"content":[{"type":"image","data":"iVBORw0KGgo…","mimeType":"image/png"}],"structuredContent":{"eye":"left","width":1920,"height":1920,"format":"RGBA8_SRGB","frame_id":1234,"capture_time_utc":"2026-08-22T18:00:00.000Z","_portalvr":{"source_width":1920,"source_height":1920,"downscaled":false}},"isError":false}
```

Images larger than `capture_max_dimension`, or whose PNG would exceed the 12 MiB size limit, are downscaled, which `_portalvr.downscaled` reports. Setting `capture_max_png_bytes` above 12 MiB has no effect. Captures are limited by `limits.capture_per_second`. In a 2D PortalVR session, see [eye capture in 2D mode](#eye-capture-in-2d-mode).

### Input and pose control

`openxr_set_controller_input` sets one control on the [active controller](#controllers-for-input-and-pose-control):

| Hand | `component` values |
| --- | --- |
| `left` | `X`, `Y`, `Menu`, `Thumbstick`, `ThumbstickClick`, `Trigger`, `Grip` |
| `right` | `A`, `B`, `System`, `Thumbstick`, `ThumbstickClick`, `Trigger`, `Grip` |

Buttons take exactly `0` or `1`; `Trigger` and `Grip` take `[0, 1]`; `Thumbstick` takes `[-1, 1]` and requires `sub_component` `X` or `Y`, which is forbidden for other components. With `auto_release: true`, the control returns to zero after `hold_duration` seconds (default `0.2`, range `0.001` to `60`). Without it, the value holds until changed.

```json
{"session_handle":42,"hand":"right","component":"Trigger","sub_component":null,"value":1,"auto_release":true,"release_time_ns":9876743210,"applied":true,"observable_state_sequence":4822,"_portalvr":{"component_path":"/input/trigger/value","controller_type":"…","legacy_binding_installed":true,"visibility":"binding_dependent"}}
```

`applied` means the runtime accepted the value. Whether the application reacts depends on its input bindings; `_portalvr.visibility` is `binding_dependent` for ordinary controls and `runtime_only` for `System`, which SteamVR reserves.

`openxr_set_head_pose` and `openxr_set_controller_pose` override a pose. Supply `position`, `orientation`, or both; an omitted component keeps its current value. `duration_seconds` (range `0` to `60`, default `0`) interpolates to the target instead of jumping. For a focused OpenXR application, SteamVR derives both grip and aim from one controller origin, so a `pose_type: "aim"` controller override moves that origin, and the result still reports `aim`; see [SteamVR limits](#conventions) for other applications. Both return the following; a controller result also has `hand` and `pose_type` after `target`:

```json
{"session_handle":42,"target":"head","target_pose":{"position":[0,1.7,0],"orientation":[0,0,0,1]},"base_space":"local_floor","duration_seconds":0,"applied":true,"simulated":true,"observable_state_sequence":4823}
```

A new call replaces the previous override for the same target, starting from the currently effective pose. An override lasts until it is replaced, the XR session ends, MCP control is released (for example when the server stops), or **Emergency MCP Clear** is used. PortalVR also clears every override and held input at once when the SteamVR dashboard opens or closes, a PortalVR controller is deactivated, the headset becomes inactive, or the scene application changes. Physical tracking then resumes immediately. Injecting right-hand `System` normally opens the dashboard, so it clears all overrides too. Overrides and held inputs survive the disconnect of the client that set them, so a test harness should restore or clear them itself.

Setting the head pose moves the whole simulated rig: controllers and hands keep their live motion relative to the physical head but follow it to the new head pose. A controller override applies after the rig move and wins for that controller. A head target in `view` space is applied relative to the view at the moment the command takes effect.

### Coordinate and math tools

`math_*` and the two Unity conversion tools need no XR application; `openxr_convert_pose_between_spaces` needs an active session.

`convert_unity_pose_to_openxr` and `convert_openxr_pose_to_unity` convert between Unity's left-handed convention and OpenXR's right-handed one by mirroring Z. They are the same involutive transform: position `[x, y, z]` becomes `[x, y, -z]`, and a unit quaternion `[x, y, z, w]` becomes `[-x, -y, z, w]` (output quaternions are normalized, with `w ≥ 0`). Supply `position`, `orientation`, or both; only supplied components are returned.

`openxr_convert_pose_between_spaces` expresses the input pose, given in `source_space`, in `target_space`. For a focused OpenXR application it can only use spaces the application created, so even the default conversion can return `SPACE_UNSUPPORTED`. It returns `source_space`, `target_space`, `position`, `orientation`, `position_valid`, `orientation_valid`, and `sample_time_ns`.

| Tool | Example arguments | Result |
| --- | --- | --- |
| `math_calc` | `{"expression":"2 + 3 * 4"}` | `{"result":14}` |
| `math_dot` | `{"vector_a":[1,2,3],"vector_b":[4,5,6]}` | `{"result":32}` |
| `math_cross` | `{"vector_a":[1,0,0],"vector_b":[0,1,0]}` | `{"result":[0,0,1]}` |
| `math_add`, `math_sub`, `math_mul`, `math_div` | `{"a":[1,2,3],"b":2}` | `{"result":…}` |
| `math_build_quat` | `{"front_direction":[1,0,0],"return_basis_vectors":true}` | `{"quaternion":[0,-0.7071067811865475,0,0.7071067811865476],"front_direction":[1,0,0],"up_direction":[0,1,0],"right_direction":[0,0,1]}` |
| `math_decompose_quat` | `{"quaternion":[0,0,0,1]}` | `{"front_direction":[0,0,-1],"up_direction":[0,1,0],"right_direction":[1,0,0]}` |

`math_calc` evaluates one scalar expression with `+ - * /`, unary minus, `**` for exponentiation, parentheses, the constants `pi` and `e`, and the functions `abs`, `sqrt`, `exp`, `log`, `log10`, `pow`, `sin`, `cos`, `tan`, `asin`, `acos`, `atan`, `atan2`, `floor`, `ceil`, `round`, `min`, and `max`. `^` and `%` are invalid, and division by zero inside an expression is an `EXPRESSION_ERROR`. The arithmetic tools take a number or a 3-vector for each operand and apply per component.

`math_build_quat` returns the rotation whose local −Z faces `front_direction`, with `up_direction` as the approximate up; both are normalized first. Basis vectors appear only when `return_basis_vectors` is true. `math_decompose_quat` returns the rotated −Z, +Y, and +X axes.

## Application custom tools

An OpenXR application can add its own tools through the `XR_METAX1_agentic_external_tool` extension, version 1. PortalVR provides it through the explicit OpenXR API layer `XR_APILAYER_PORTALVR_steamvr_mcp_metax1`, which SteamVR lists as **Application controlled**. The application enables that layer and the extension before `xrCreateInstance`, then calls `xrAgenticRegisterExternalToolMETAX1` for each tool with a name, a description, parameter descriptors, and a callback. Ordinary MCP use does not need this layer.

- Tool names must match `^[A-Za-z0-9_.-]{1,63}$`. Names beginning `openxr_`, `math_`, `convert_`, or `portalvr.` are reserved, and duplicate names in one instance fail.
- Registered tools follow the built-ins in `tools/list`, in registration order, and are listed and callable only while the application's session is focused and selected. Their schemas are objects with `additionalProperties: false` built from the parameter descriptors.
- Tools are retired when the instance is destroyed or the application disconnects; version 1 has no unregister call.
- A call receives the arguments as JSON text, and the callback writes a JSON result. An object becomes the tool result as is; any other JSON value is wrapped as `{"result": …}`. A failing callback or invalid JSON is reported as `CUSTOM_TOOL_ERROR`, and calls have a 10-second deadline.

## Recovery

Use **Emergency MCP Clear** on the System page first. If PortalVR's UI is unavailable:

1. Create an empty file `%LOCALAPPDATA%\PortalVR\mcp\kill-switch`. The driver and OpenXR layer see it directly and drop pose overrides and pending input without waiting for the server.
2. End only the `portalvr-mcp-host.exe` process located in PortalVR's `steamvr-portalvr\bin\win64` folder.
3. Delete the stale `%LOCALAPPDATA%\PortalVR\mcp\discovery.json`.

MCP stays off while `kill-switch` exists. Deleting it re-enables MCP at the next PortalVR start unless **MCP Server** is off; turning **MCP Server** on also deletes it. Repair damaged binaries with PortalVR's normal installer.

## Compatibility notes

This reference was checked against the PortalVR MCP implementation on **2026-11-22**. The transport, JSON-RPC, error, and math examples were confirmed against a running server; pose, input, exit, and capture results follow the implementation's encoders and were not exercised against a live XR application for this document.

Tool names, arguments, and annotations follow the Meta XR Operator surface, but PortalVR does not reproduce that product's undocumented result formats byte for byte. PortalVR results always include `structuredContent`, use explicit `_ns` and `_utc` field names, and add `_portalvr` metadata to most XR results.
