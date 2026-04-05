---
name: websocket-token-auth
description: Use this skill when adding or changing Channels WebSocket endpoints. Triggers on mentions of "WebSocket", "ws", "consumer", "token auth middleware", "4401", or WebSocket endpoint paths.
---

# WebSocket + Token Auth Skill (SealChat Style)

All backend code lives under `backend/`. All paths in this skill are relative to `backend/`.

Use this skill when adding or changing Channels websocket endpoints in this repository.

## Current Architecture

### ASGI composition

From `server/asgi.py`:

- `ProtocolTypeRouter` splits `http` and `websocket`.
- WebSocket stack is:
  - `TokenAuthMiddlewareStack(...)`
  - `URLRouter([ path("ws/messenger/", URLRouter(websocket_urlpatterns)) ])`

Meaning all websocket paths are nested under `/ws/messenger/`.

### Endpoint paths

From `messenger/routings.py`:

- `/ws/messenger/messages/` -> `MessageConsumer`
- `/ws/messenger/room/event/<room_id>/` -> `ChatroomEvent`

## Auth Handshake Convention

### Header format

From `server/auth_middleware.py` + test helpers:

- Middleware reads `authorization` header from ASGI scope.
- Client sends token as `Token <key>`.
- Middleware extracts last segment (`split()[-1]`) and loads `rest_framework.authtoken.Token`.

### Consumer connect behavior

From `messenger/consumers.py`:

1. `await self.accept()` first.
2. If unauthenticated:
   - send error payload: `{ "type": "error", "message": "Unauthorized" }`
   - close with code `4401`.
3. If authenticated:
   - send ready payload: `{ "type": "ready" }`
   - subscribe to pubsub channel.

This means unauthorized clients still complete HTTP->WebSocket upgrade, then are closed at app layer.

## Message/Event Envelope Convention

Use JSON messages with top-level `type` key.

Examples used in repo:

- Ready event:
  ```json
  {"type": "ready"}
  ```
- Unauthorized event:
  ```json
  {"type": "error", "message": "Unauthorized"}
  ```
- Message push:
  ```json
  {"type": "message", "data": {...}}
  ```

## PubSub Pattern

### Subscribe on connect

```python
self.unsubscribe = pubsub.subscribe(topic, self.on_message)
```

### Cleanup on disconnect

```python
if hasattr(self, "unsubscribe"):
    self.unsubscribe()
```

Prefer the `hasattr` guard to avoid disconnect errors in early-close paths.

## Recommended Middleware Pattern

For robust compatibility, keep these rules when touching token middleware:

- Read ASGI headers from `scope['headers']` (list of byte tuples).
- Use lowercase byte key `b'authorization'`.
- Decode bytes before parsing.
- Support both `Token <key>` and `Bearer <key>` formats if possible.

Template:

```python
from channels.middleware import BaseMiddleware
from channels.auth import AuthMiddlewareStack
from rest_framework.authtoken.models import Token


class TokenAuthMiddleware(BaseMiddleware):
    async def __call__(self, scope, receive, send):
        headers = dict(scope.get("headers", []))
        raw = headers.get(b"authorization")

        if raw:
            auth = raw.decode("utf-8", errors="ignore").strip()
            parts = auth.split()
            token_key = parts[-1] if parts else None
            if token_key:
                try:
                    token_obj = await Token.objects.select_related("user").aget(key=token_key)
                    scope["user"] = token_obj.user
                except Token.DoesNotExist:
                    pass

        return await super().__call__(scope, receive, send)


def TokenAuthMiddlewareStack(inner):
    return TokenAuthMiddleware(AuthMiddlewareStack(inner))
```

## Endpoint Implementation Checklist

1. Add websocket route in `messenger/routings.py`.
2. Confirm final URL under `/ws/messenger/...` due to ASGI nesting.
3. In consumer `connect()`:
   - `accept()`
   - auth check
   - send `ready` or send error + close `4401`
4. Subscribe to pubsub topics after auth success.
5. Guard cleanup in `disconnect()`.
6. Keep event payloads consistent with existing envelope format.
7. Add/extend async websocket tests for:
   - authenticated connect
   - no-header unauthorized close
   - event delivery shape

## Test Style in This Repo

From `messenger/tests/test_message.py`:

- Use `channels.testing.WebsocketCommunicator`.
- Authenticated ws tests pass headers from `create_async_api_client(...).ws_headers`.
- Unauthorized tests assert:
  - websocket send error event
  - close code `4401`

Minimal authenticated test shape:

```python
communicator = WebsocketCommunicator(
    application=application,
    headers=client.ws_headers.items(),
    path="/ws/messenger/messages/",
)

connected, _ = await communicator.connect()
assert connected
ready = await communicator.receive_json_from()
assert ready == {"type": "ready"}
```

## Deployment/Infra Quick Checks (for `wss` issues)

If websocket upgrade fails in production, verify outside Django too:

- Reverse proxy forwards upgrade headers (`Upgrade`, `Connection`).
- HTTP/1.1 is enabled between proxy and app.
- TLS terminates correctly and client uses `wss://`.
- Path sent by client exactly matches `/ws/messenger/...`.
- `Authorization` header is forwarded to upstream websocket app.
