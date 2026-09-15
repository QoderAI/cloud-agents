# HTTP client patterns

Read [events](events.md) for wire format and [HTTP setup](guide.md) for transport/credential selection.

## Completion and confirmation

`span.model_request_end` closes one model call, not the task. For the target turn, inspect `session.status_idle.stop_reason`: `end_turn` finishes the turn, `cancelled` records interruption, and `requires_action` is paused for tool confirmation. Resolve the exact pending event only with the required user authorization. Do not auto-send `allow`. Read final `agent.message` blocks and business errors before reporting success.

## SSE startup, history and reconnect

Without `Last-Event-ID`, a connection starts at the current tail. For every live event, establish SSE before sending the message. Alternatively, after sending, resume from the accepted message event ID and recover any needed earlier events through paginated List Events. Use the last processed SSE ID on reconnect and deduplicate buffered events. If the cursor is unknown, read history rather than assume a new stream replays it. Event deltas need the additional replay handling in [events](events.md).

Use a task deadline, bounded reconnect attempts and backoff. Stop on user cancellation, terminal status, unrecoverable error or deadline. A disconnected stream is not evidence that remote execution stopped. Cold starts may be slow; heartbeats indicate an open stream, not task completion. Fall back to HTTP event polling when SSE is impractical, without resubmitting the task.

## Multi-turn and cancellation

Wait for the current turn to settle before sending a new message. To interrupt it, cancel and read back status/events until settled or deadline. A cancel acknowledgement is not completion. Keep the same Session for an intended conversation; do not create a replacement merely because observation failed.

## Errors and retry

Inspect status, structured errors and request IDs. Honor Retry-After on 429 (seconds or HTTP date), otherwise use capped exponential backoff with jitter and bounded concurrency. For ambiguous writes, inspect state before retrying; a network timeout may occur after acceptance. Do not retry paid execution or create resources blindly after session.error. Authentication/permission errors require correct credentials/access, not a different endpoint or identity.

## Cleanup

Only clean up exact resources covered by the user's request. Pause/archive dependent schedules first, cancel running sessions and wait for settlement, then delete/archive intended resources. Used Environments may retain references after Session deletion; archive them instead of forcing deletion. Never clean up shared resources automatically.
