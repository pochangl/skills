---
name: django-test
description: Use this skill when writing Django test cases, test classes, or test files for this project. Trigger on requests like "write tests for", "add test coverage", "test this endpoint", "create test cases", or anytime a new test file or TestCase class needs to be authored.
---

# Django Test Writing Guide

All backend code lives under `backend/`. All paths in this skill are relative to `backend/`.

This project uses an async-first architecture with ADRF. Follow these conventions exactly.

## Imports

```python
from django.test import TestCase
from utils.test.auth import create_api_client, create_user
# app-level helpers:
from .utils import create_identity, create_chatroom  # adjust to app
```

For WebSocket / async tests, also import:
```python
from channels.testing import WebsocketCommunicator
from utils.test.auth import create_async_api_client
from server.asgi import application
```

For mocking:
```python
from unittest.mock import patch, ANY
from utils.test.matcher import AnyType  # checks isinstance, not exact value
```

## Test class skeleton

```python
class TestFooAPI(TestCase):
    def setUp(self):
        self.user = create_user()
        self.identity = create_identity(self.user)
        self.client = create_api_client(self.user)
        self.endpoint = '/api/messenger/foo/'

    def test_list(self):
        response = self.client.get(self.endpoint)
        self.assertEqual(response.status_code, 200, response.content)
        self.assertEqual(response.json(), [...])
```

- `setUp` initialises all fixtures fresh for every test method.
- Always pass `response.content` as the second arg to `assertEqual` on status codes — it makes failures self-explanatory.
- Use `201` for POST create, `200` for GET/PUT, `204` for DELETE.

## Creating test data

Use helpers from `utils/test/` and the app's own `tests/utils.py`:

```python
user     = create_user()                         # unique username via UUID
identity = create_identity(user)
room     = create_chatroom()
room.identities.add(identity)
client   = create_api_client(user)               # authenticated REST client
```

Create ORM objects directly with `.objects.create()` — no factories.

## Assertions

Check status code first, then response body:

```python
self.assertEqual(response.status_code, 200, response.content)
self.assertEqual(response.json(), {'id': self.identity.id, 'name': self.identity.name})
```

Use `AnyType(str)` when you only care about the type, not the exact value:
```python
from utils.test.matcher import AnyType
self.assertEqual(response.json(), {'token': AnyType(str), 'app_id': AnyType(str)})
```

Use `ANY` from `unittest.mock` to ignore a specific argument in mock assertions:
```python
mock_fn.assert_called_once_with(tokens=['tok'], topic='user-1', app=ANY)
```

## Isolation / multi-user tests

Verify that one user cannot see another's data:

```python
def test_isolation(self):
    user2 = create_user()
    client2 = create_api_client(user2)

    self.client.post(self.endpoint, {...})
    response1 = self.client.get(self.endpoint)
    response2 = client2.get(self.endpoint)

    self.assertNotEqual(response1.json(), response2.json())
```

## Async / WebSocket tests

```python
class TestMessageConsumer(TestCase):
    def setUp(self):
        self.user = create_user()
        self.identity = create_identity(self.user)

    async def test_receives_message(self):
        client = await create_async_api_client(self.user)
        communicator = WebsocketCommunicator(
            application=application,
            headers=client.ws_headers.items(),
            path='/ws/messenger/messages/'
        )
        connected, _ = await communicator.connect()
        self.assertTrue(connected)

        response = await client.post('/api/messenger/message/', {...})
        self.assertEqual(response.status_code, 201, response.content)

        event = await communicator.receive_json_from()
        self.assertEqual(event, {'type': 'message', 'data': response.json()})

        await communicator.disconnect()
```

For unauthorized WebSocket checks:
```python
async def test_unauthenticated(self):
    communicator = WebsocketCommunicator(application=application, path='/ws/...')
    await communicator.connect()
    event = await communicator.receive_output(timeout=1)
    self.assertEqual(event['type'], 'websocket.send')
    close_event = await communicator.receive_output(timeout=1)
    self.assertEqual(close_event['type'], 'websocket.close')
    self.assertEqual(close_event.get('code'), 4401)
```

## Mocking external services

Always use `patch` as a context manager and assert the call:

```python
def test_fcm_register(self):
    with patch('firebase_admin.messaging.subscribe_to_topic') as mock_sub:
        mock_sub.return_value = None
        response = self.client.post('/api/fcm/register/user/', {'token': 'tok'})
        self.assertEqual(response.status_code, 201, response.content)
        mock_sub.assert_called_once_with(tokens=['tok'], topic=f'user-{self.user.id}', app=ANY)
```

## Organising multiple test classes in one file

Group by feature, not by HTTP method:

```python
class TestMessageList(TestCase): ...
class TestMessageCreate(TestCase): ...
class TestMessageAuth(TestCase): ...   # error / unauthenticated cases
```

## What NOT to do

- Don't use sync DRF views/clients to test async endpoints — auth will fail.
- Don't share state between tests via class attributes — always use `setUp`.
- Don't assert on `response.text` when `.json()` is available.
- Don't leave `communicator.disconnect()` out of async WebSocket tests.
