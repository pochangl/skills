---
name: rest-framework-adrf
description: Use this skill when adding or modifying API endpoints in this repository. Triggers on mentions of "endpoint", "API", "view", "viewset", "serializer", "ADRF", or REST framework patterns.
---

# DRF + ADRF Skill (SealChat Style)

All backend code lives under `backend/`. All paths in this skill are relative to `backend/`.

Use this skill when adding or modifying API endpoints in this repository.

## Core Stack Pattern

- Prefer `adrf` classes/decorators for async endpoints:
  - `adrf.views.APIView`
  - `adrf.viewsets.ModelViewSet`, `ReadOnlyModelViewSet`
  - `adrf.generics.CreateAPIView`
  - `adrf.decorators.api_view`
- Return `rest_framework.response.Response`.
- Keep default auth/permission from `server/settings.py` unless endpoint must be public.

## ⭐ Best Practice: Async-First Architecture

**ALWAYS prioritize async implementation with ADRF when dealing with APIs in this project.**

This is a core architectural principle:

1. **Use async views**: Inherit from `adrf.views.APIView` (NOT `rest_framework.views.APIView`)
2. **Use async methods**: Define view methods as `async def post(self, request):`
3. **Use async ORM operations**:
   - `await Model.objects.aget()` instead of `Model.objects.get()`
   - `await Model.objects.acreate()` instead of `Model.objects.create()`
   - `await queryset.aexists()` instead of `queryset.exists()`
   - `await m2m_field.aadd()` instead of `m2m_field.add()`
4. **Use async serializer methods**:
   - Implement `async def avalidate()` for database validation
   - Implement `async def asave()` or `async def acreate()` for async operations
   - Call them with `await serializer.avalidate()` and `await serializer.asave()`

### Why Async?

- **Non-blocking I/O**: Database queries don't block the event loop
- **Better scalability**: Can handle more concurrent requests
- **WebSocket compatibility**: Shares the same event loop as Django Channels
- **Project standard**: The entire codebase (auth, messenger, fcm) uses async patterns

### Common Mistake to Avoid

❌ **Don't use sync DRF classes** - they conflict with async authentication:
```python
from rest_framework.views import APIView  # WRONG - will fail with async auth
```

✅ **Always use ADRF classes**:
```python
from adrf.views import APIView  # CORRECT - async compatible
```

## Global API Defaults (must align)

From `server/settings.py`:

- Authentication: `utils.adrf.authentications.TokenAuthentication`
- Permission: `utils.adrf.permissions.IsAuthenticated`
- Filter backend: `django_filters.rest_framework.DjangoFilterBackend`

For public endpoints, explicitly clear both in the view:

```python
class PublicView(APIView):
    permission_classes = []
    authentication_classes = []
```

## Authentication Style

- Token auth is the project default (`rest_framework.authtoken`).
- Use async wrappers in `utils/adrf/*` rather than raw DRF classes when dealing with async ADRF views.
- In tests, pass token as `Authorization: Token <token>`.

## When to Use ViewSets vs Plain Views

- **Use a ViewSet** when the endpoint performs CRUD on a single table. This covers most standard resource endpoints.
- **Use a plain APIView or @api_view** when the endpoint involves custom logic across multiple tables or doesn't map to standard CRUD.

## ViewSet Style

- Define `queryset` and `serializer_class` at class level.
- Restrict data by `request.user` in `get_queryset()`.
- For read-only relationships, use `ReadOnlyModelViewSet`.
- **Override `perform_acreate` / `perform_aupdate` / `perform_adestroy`** — NOT `perform_create` / `perform_update` / `perform_destroy`. ADRF's async mixins dispatch to the `a`-prefixed hooks; the unprefixed versions are never called.

Example pattern:

```python
from adrf.viewsets import ModelViewSet

class IdentityViewSet(ModelViewSet):
    queryset = Identity.objects.all()
    serializer_class = IdentitySerializer
    filterset_class = IdentityFilterSet

    def get_queryset(self):
        return self.queryset.filter(user=self.request.user)

    async def perform_acreate(self, serializer):
        await serializer.asave(user=self.request.user)
```

## Function Endpoint Style

- Use `@api_view([...])` from `adrf.decorators`.
- Validate user ownership before read/write.
- Return explicit status codes for domain errors (`400`, `403`, `404`).

Example pattern:

```python
from adrf.decorators import api_view
from rest_framework.response import Response

@api_view(['POST'])
def add_friend(request):
    try:
        # business validation
        ...
    except BadSignature:
        return Response({'error': 'Invalid friend token'}, status=400)
    return Response(payload, status=201)
```

## Serializer Style (Async-Capable)

- Prefer `adrf.serializers.ModelSerializer`.
- Use custom async lifecycle hooks where needed:
  - `acreate(self, validated_data)` for async side effects and writes.
- Keep timestamp conversion via `utils.adrf.fields.TimeStampField`.

Pattern used in repo:

```python
class MessageSerializer(ModelSerializer):
    created_at = TimeStampField(read_only=True)

    class Meta:
        model = Message
        fields = '__all__'

    async def acreate(self, validated_data):
        obj = await super().acreate(validated_data)
        # async side effect (pubsub / external integration)
        return obj
```

## Router + URL Style

- Use `adrf.routers.DefaultRouter()` in app URLs.
- Register resources with **singular** nouns (e.g. `'profile'`, not `'profiles'`) unless explicitly told to use plural.
- Register resources with explicit `basename`.
- Keep utility endpoints (`token`, `join`, etc.) as explicit `path()` entries beside router URLs.

## Error Handling Conventions

- Use plain error payload shape: `{'error': '<message>'}`.
- Common statuses in this repo:
  - `201` for successful creates and token issuance
  - `400` invalid input/token/credentials
  - `403` ownership/authorization mismatch
  - `404` missing domain resource

## Testing Conventions

- Sync API tests: `rest_framework.test.APIClient`
- Async flows and websocket-adjacent behavior: async test + `AsyncClient` / Channels communicator
- Authenticate test clients by token helper from `utils/test/auth.py`.

## New Endpoint Checklist

1. Choose ADRF base class/decorator.
2. Reuse project default auth/permission unless public endpoint.
3. Filter all object access by `request.user` ownership.
4. Implement serializer with `ModelSerializer` (and `acreate` if async side effects needed).
5. Register route in app `urls.py` router/path list.
6. Add/extend tests for success + failure statuses.

## Ready-to-Use Templates

### Async authenticated APIView

```python
from adrf.views import APIView
from rest_framework.response import Response

class ExampleView(APIView):
    async def get(self, request):
        return Response({'ok': True})
```

### Public token/login style endpoint

```python
from adrf.views import APIView
from rest_framework.response import Response

class LoginLikeView(APIView):
    permission_classes = []
    authentication_classes = []

    async def post(self, request):
        return Response({'token': '...'}, status=201)
```

### Async create serializer with side effects

```python
from adrf.serializers import ModelSerializer

class ExampleSerializer(ModelSerializer):
    class Meta:
        model = Example
        fields = '__all__'

    async def acreate(self, validated_data):
        obj = await super().acreate(validated_data)
        await do_async_side_effect(obj)
        return obj
```
