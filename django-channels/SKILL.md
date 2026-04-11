---
name: django-channels
description: Use this skill when the user asks to implement WebSocket consumers, real-time messaging, or Django Channels features. Triggers on mentions of "consumer", "WebSocket", "channels", "pubsub", "real-time", or "ChatroomEvent"/"MessageConsumer".
---

# Django Channels — Implementation Guide

All backend code lives under `backend/`. All paths in this skill are relative to `backend/`.

This project uses Django Channels with a custom in-process PubSub system instead of a channel layer for real-time delivery.

## Architecture Overview

```
REST API (POST message)
    → MessageSerializer.acreate()
    → pubsub.async_publish('message/{user.id}', data)
    → MessageConsumer.on_message()
    → WebSocket.send() → Client
```

WebSocket endpoints are mounted at `/ws/messenger/` and defined in `messenger/routings.py`.

---

## ASGI Configuration (`server/asgi.py`)

```python
from channels.routing import ProtocolTypeRouter, URLRouter
from django.core.asgi import get_asgi_application
from messenger.routings import websocket_urlpatterns
from .auth_middleware import TokenAuthMiddlewareStack

application = ProtocolTypeRouter({
    "http": get_asgi_application(),
    "websocket": TokenAuthMiddlewareStack(
        URLRouter([
            path("ws/messenger/", URLRouter(websocket_urlpatterns))
        ])
    ),
})
```

---

## WebSocket Routing (`messenger/routings.py`)

```python
from django.urls import path
from . import consumers

websocket_urlpatterns = [
    path('messages/', consumers.MessageConsumer.as_asgi()),
    path('room/event/<str:room_id>/', consumers.ChatroomEvent.as_asgi()),
]
```

---

## Writing a Consumer (`messenger/consumers.py`)

Always inherit from `AsyncWebsocketConsumer`. The pattern is:

1. `accept()` first, then check auth — close with code `4401` if unauthorized
2. Subscribe to pubsub topic in `connect()`; store the unsubscribe callable on `self`
3. Call `self.unsubscribe()` in `disconnect()`

**User-scoped consumer** (e.g. personal message stream):

```python
import json
from channels.generic.websocket import AsyncWebsocketConsumer
from .utils import pubsub

class MessageConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        self.user = self.scope["user"]
        await self.accept()
        if not self.user.is_authenticated:
            await self.send(text_data=json.dumps({"type": "error", "message": "Unauthorized"}))
            await self.close(code=4401, reason='Unauthorized')
            return
        await self.send(text_data=json.dumps({"type": "ready"}))
        self.unsubscribe = pubsub.subscribe(f'message/{self.user.id}', self.on_message)

    async def disconnect(self, close_code):
        self.unsubscribe()

    async def on_message(self, message):
        await self.send(text_data=json.dumps({"type": "message", "data": message}))
```

**Room-scoped consumer** (e.g. chatroom events):

```python
class ChatroomEvent(AsyncWebsocketConsumer):
    async def connect(self):
        self.user = self.scope["user"]
        await self.accept()
        if not self.user.is_authenticated:
            await self.close(code=4401, reason='Unauthorized')
            return
        self.room_id = self.scope['url_route']['kwargs']['room_id']
        self.unsubscribe = pubsub.subscribe(f'chatroom/{self.room_id}', self.on_message)

    async def disconnect(self, close_code):
        self.unsubscribe()

    async def on_message(self, message):
        await self.send(text_data=json.dumps({"type": "event", "data": message}))
```

---

## PubSub System (`utils/pubsub.py`)

The project uses a custom in-process `PubSub` class — **not** Django Channels' channel layer. It supports both sync and async handlers.

**Singleton instantiation** in `messenger/utils.py`:

```python
from utils.pubsub import PubSub
pubsub = PubSub()
```

**API:**

```python
# Subscribe — returns an unsubscribe callable
unsubscribe = pubsub.subscribe("topic", handler)

# Unsubscribe
unsubscribe()

# Publish from sync context
pubsub.publish("topic", payload)

# Publish from async context (preferred in async views/serializers)
await pubsub.async_publish("topic", payload)
```

**Publishing from a serializer** (the standard pattern for delivering messages):

```python
async def acreate(self, validated_data):
    message = await super().acreate(validated_data)
    async for identity in message.room.identities.select_related('user').all():
        await pubsub.async_publish(f'message/{identity.user.id}', MessageSerializer(message).data)
    return message
```

**Key behaviour:**
- Thread-safe via `threading.Lock()`
- `async_publish` gathers all async handlers concurrently with `asyncio.gather()`
- `publish` uses `loop.create_task()` for async handlers when a loop is running
- **In-process only** — does not work across multiple server processes (fine for single-worker dev; switch to Redis channel layer for multi-process production)

---

## Token Auth Middleware (`server/auth_middleware.py`)

Handles DRF token auth for WebSocket handshakes. Reads the `Authorization` header and sets `scope['user']`.

```python
from channels.middleware import BaseMiddleware
from channels.auth import AuthMiddlewareStack
from rest_framework.authtoken.models import Token

class TokenAuthMiddleware(BaseMiddleware):
    header_name = b'authorization'

    async def __call__(self, scope, receive, send):
        headers = dict(scope['headers'])
        token = headers.get(self.header_name)
        if token is not None:
            token = token.decode('utf-8')
            try:
                t = await Token.objects.select_related('user').aget(key=token.split()[-1])
                scope['user'] = t.user
            except Token.DoesNotExist:
                pass
        return await super().__call__(scope, receive, send)

def TokenAuthMiddlewareStack(inner):
    return TokenAuthMiddleware(AuthMiddlewareStack(inner))
```

Client sends: `Authorization: Bearer <token>`

---

## Settings

```python
INSTALLED_APPS = [
    'daphne',   # must be first
    ...
    'channels',
]

CHANNEL_LAYERS = {
    "default": {
        "BACKEND": "channels.layers.InMemoryChannelLayer"
    }
}
```

Run with Daphne (ASGI server):

```bash
daphne -b 0.0.0.0 -p 8000 server.asgi:application
```

---

## Common Pitfalls

- **`AsyncWebsocketConsumer` not `AsyncWebSocketConsumer`**: The channels library uses a lowercase `s` in `Websocket`. Using `AsyncWebSocketConsumer` (capital `S`) will raise `ImportError: cannot import name 'AsyncWebSocketConsumer'`. Always use `from channels.generic.websocket import AsyncWebsocketConsumer`.

## Conventions

- Topic naming: `message/{user.id}` for user streams, `chatroom/{room_id}` for room events
- Always `accept()` before sending or closing — clients won't receive anything otherwise
- Use `4401` as the close code for unauthorized connections (custom code in the 4000–4999 range)
- Send `{"type": "ready"}` after successful connection so clients know the stream is live
- Store the unsubscribe callable on `self` in `connect()` and always call it in `disconnect()`
- Use `async_publish` from async contexts (serializers, views); use `publish` from sync signals
