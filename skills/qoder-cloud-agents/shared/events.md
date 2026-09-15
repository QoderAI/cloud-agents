# Events

HTTP mode only: read [setup and transport selection](guide.md) first. These Managed examples are separate from the MCP tool schemas; consult current operation docs for deployment-specific fields.

The event system is the core communication channel between client and Cloud Agent sessions. You send events (user messages, tool confirmations, interrupts) and receive events (agent responses, status changes) via SSE streaming or polling.

---

## Sending Events

Send events to a session via `POST /sessions/{id}/events`.

### user.message

```json
{
  "events": [
    {
      "type": "user.message",
      "content": [{"type": "text", "text": "Your message text here"}]
    }
  ]
}
```

> **Two hard rules**: (1) wrap in an `events` array — a bare event object returns 400; (2) `content` must be a **non-empty array of content blocks** — a plain string returns 400.

Accepted asynchronously; the response body is `{data: [...]}` containing the accepted event object(s).

### Other client event types

| Type | Required fields | Purpose |
|---|---|---|
| `user.interrupt` | — | Interrupt the current turn (optional `session_thread_id`) |
| `user.tool_confirmation` | `tool_use_id`, `result` (`allow`/`deny`) | Answer an `agent.tool_use` that paused under an `always_ask` permission policy. Optional `deny_message` (only with `deny`) |
| `user.custom_tool_result` | `custom_tool_use_id` | Return the result of a client-side `custom` tool after `agent.custom_tool_use`. Optional `content` (content-block array), `is_error` |
| `user.tool_result` | `tool_use_id` | Return a built-in tool result from a self-hosted worker |
| `user.define_outcome` | `description`, `rubric` | Define a success rubric the platform evaluates (`span.outcome_evaluation_*` events). `rubric` is `{"type":"text","content":"..."}` or `{"type":"file","file_id":"..."}` |
| `system.message` | `content` | Append system-prompt-level guidance. Max one per request, must be the final event in the batch |

---

## Receiving Events (SSE Stream)

Connect to `GET /sessions/{id}/events/stream` for real-time events. Since 2026-08-24, no `Last-Event-ID` means tail-only delivery, not historical replay. Establish SSE before triggering execution, or resume with an accepted/processed event ID; use List Events for history. See the [official change notice](https://docs.qoder.com/cloud-agents/sse-initial-connection-behavior-change).

```
GET /sessions/{id}/events/stream
Authorization: Bearer <PAT>
Accept: text/event-stream
Last-Event-ID: evt_xxx          # optional — resume after this event
```

### SSE Wire Format

Each event is delivered with standard SSE fields — note the top-level `id:` line, which is what you feed back as `Last-Event-ID`:

```
id: evt_xxx
event: <event_type>
data: {"id":"evt_xxx","type":"<event_type>",...}

```

The server sends `: heartbeat` comment lines periodically to keep the connection alive. Filter comment lines when parsing.

By default the stream delivers **buffered events**: each agent response arrives as one complete event (e.g. `agent.message`) after generation finishes, and is recorded in session event history.

---

## Incremental Streaming (event_deltas)

To render agent output token-by-token, opt in per connection with the repeatable `event_deltas[]` query param (allowed values: `agent.message`, `agent.thinking`):

```
GET /sessions/{id}/events/stream?event_deltas[]=agent.message&event_deltas[]=agent.thinking
```

Incremental `agent.message` output emits `event_start` followed by one or more `event_delta` frames, then the buffered final event:

```
id: evt_00jjujk9fbnr4wkj2gh8
event: event_start
data: {"event":{"id":"evt_00jjujk9fbnr4wkj2gh8","type":"agent.message"},"type":"event_start"}

id: evt_00jjujk9fbnr4wkj2gh8
event: event_delta
data: {"delta":{"content":{"text":"Hello","type":"text"},"index":0,"type":"content_delta"},"event_id":"evt_00jjujk9fbnr4wkj2gh8","type":"event_delta"}
```

Incremental `agent.thinking` is **start-only** — an `event_start` frame, no deltas (reasoning content is never exposed).

Rules to build on:
- The SSE `id:`, `event_start.event.id`, every `event_delta.event_id`, and the buffered event `id` are **identical** for one message. Accumulate deltas per `event_id`, keyed by `delta.index` per content block.
- Delta frames are stream-only: no top-level JSON `id`/`processed_at`, and they never appear in event list/history responses.
- The **buffered event is the authoritative result** — prefer replacing your accumulated text with the buffered `content` when it arrives.
- The selection applies only to the current connection; other connections to the same session are unaffected. Thread event streams do not support this parameter.

---

## Event Types

### Typical Turn Lifecycle

```
session.status_running
session.thread_status_running
user.message                    ← echo of your message
span.model_request_start
agent.thinking (0+)             ← marker only, no content
agent.tool_use / agent.tool_result (0+)
agent.message (1+)
span.model_request_end
session.thread_status_idle
session.status_idle             ← USE THIS as stop signal
```

Not every turn contains every event. Note that `session.status_running` arrives **before** the `user.message` echo.

### Full public event type list

`user.*`: `message`, `interrupt`, `tool_confirmation`, `tool_result`, `custom_tool_result`, `define_outcome`; `system.message`;
`agent.*`: `message`, `thinking`, `tool_use`, `tool_result`, `custom_tool_use`, `mcp_tool_use`, `mcp_tool_result`, `artifact_delivered`, `thread_message_sent`, `thread_message_received`, `thread_context_compacted`;
`session.*`: `status_running`, `status_idle`, `status_rescheduled`, `status_terminated`, `error`, `updated`, `deleted`, `thread_created`, `thread_status_running`, `thread_status_idle`, `thread_status_rescheduled`, `thread_status_terminated`;
`span.*`: `model_request_start`, `model_request_end`, `outcome_evaluation_start`, `outcome_evaluation_ongoing`, `outcome_evaluation_end`.

### Payload notes

- **`agent.thinking` carries no content** — only `id`, `processed_at`, `type`. It is a marker that the agent paused to reason; reasoning text is intentionally not exposed.
- `agent.message` `content` is always a content-block **array**: `[{"type":"text","text":"..."}]`. Extract via `content[0].text` (iterate for multi-block).
- `agent.tool_use` carries the tool `name` and `input`; when a permission policy is `always_ask`, the turn pauses until you send `user.tool_confirmation` with its `tool_use_id`.
- `agent.artifact_delivered` follows a `DeliverArtifacts` tool call and carries `file_id`, `original_filename`, `content_type`, `size` — the file is downloadable via `GET /files/{id}/content`.
- `span.model_request_end` carries `is_error`, `model_request_start_id` (matches the start event's `id`), and `model_usage` when credits are available for that call: `{"credits": 5.94}`. Credits are **floored** to at most 2 decimals and may be `0`; the field is omitted when unavailable. Spans expose no token counts or internal timing.
- `processed_at` is optional on many agent-generated events — treat it as such when parsing.
- `session.status_idle` carries `stop_reason`. Types seen in practice: `end_turn` (normal completion), `cancelled` (after cancel/interrupt), and `requires_action` with `event_ids` pointing at the pending `agent.tool_use` — the turn is **paused waiting for your `user.tool_confirmation`**, not finished.
- For cost tracking, prefer the session object's cumulative `usage.total_credits` (`GET /sessions/{id}`) over summing per-call `model_usage.credits`; the session value is an authoritative snapshot.
- `agent.tool_use` under an `always_ask` policy carries `evaluated_permission: "ask"`; answer with `user.tool_confirmation` referencing the event's `id` as `tool_use_id`.

### Terminal vs transient states

- `session.status_idle` — inspect stop_reason: end_turn is done, cancelled is interrupted, requires_action needs confirmation.
- `session.status_terminated`, `session.deleted` — **terminal**: stop reconnecting, no further events will arrive.
- `session.status_rescheduled` — transient: the stream may briefly drop, reconnect once the runtime is ready.
- `session.error` — processing error; payload has `error: {message, type}`.

---

## Reconnection

Reconnect with the **`Last-Event-ID` header** (standard SSE — `EventSource` does this automatically) set to the last received SSE `id:`:

```
GET /sessions/{id}/events/stream
Last-Event-ID: evt_<last_received>
```

For buffered events, the stream resumes after that ID. Deduplicate by `event.id` in case of boundary replay.

When an incremental message is mid-generation, three cases apply:
1. Cursor is an event **before** the current `event_start` → the stream replays that message's `event_start` + retained deltas, then continues live.
2. Cursor **equals** the in-progress event's ID → historical deltas are skipped; you get only future deltas plus the buffered final event.
3. Generation already **completed** → no delta replay; you get the buffered `agent.message` (or resume after it).

So: to rebuild an in-progress message after a disconnect, reconnect from the buffered event ID **before** its `event_start` and discard your partial local state (case 1). If you only need the final text, reconnecting with the in-progress ID and waiting for the buffered event is simpler (case 2).

---

## Polling (Alternative to SSE)

If SSE is impractical, poll `GET /sessions/{id}/events?after_id=<last_id>` (or use `page`/`next_page`) periodically. `order=asc|desc` is supported.

Recommended interval: 1-2 seconds while session status is `running`; inspect stop_reason on `session.status_idle`; stop on completion/cancellation or pause for required user input, with a deadline. Event deltas never appear in list responses — polling only sees buffered events.
