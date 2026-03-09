---
name: django-rest-framework
description: This skill should be used when the user asks to implement a REST API with Django, uses "Django REST Framework", "DRF", mentions "serializers", "viewsets", "APIView", "routers", or asks about building API endpoints with Django.
---

# Django REST Framework — Implementation Guide

## Project Setup

### Installation

```bash
pip install djangorestframework djangorestframework-simplejwt django-filter
```

Add to `INSTALLED_APPS` in `settings.py`:

```python
INSTALLED_APPS = [
    ...
    'rest_framework',
    'django_filters',
]
```

### Base DRF Settings

```python
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',

    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 20,
    'DEFAULT_FILTER_BACKENDS': [
        'django_filters.rest_framework.DjangoFilterBackend',
        'rest_framework.filters.SearchFilter',
        'rest_framework.filters.OrderingFilter',
    ],
}
```

---

## Serializers

Serializers validate input and control output shape. Always prefer `ModelSerializer` for model-backed resources.

```python
from rest_framework import serializers
from .models import Article

class ArticleSerializer(serializers.ModelSerializer):
    author_name = serializers.CharField(source='author.get_full_name', read_only=True)

    class Meta:
        model = Article
        fields = ['id', 'title', 'body', 'author_name', 'created_at']
        read_only_fields = ['id', 'created_at']

    def validate_title(self, value):
        if len(value) < 5:
            raise serializers.ValidationError("Title must be at least 5 characters.")
        return value
```

**Nested serializers** — use `depth` for simple nesting, explicit nested serializers for write support:

```python
class CommentSerializer(serializers.ModelSerializer):
    class Meta:
        model = Comment
        fields = ['id', 'body', 'created_at']

class ArticleDetailSerializer(ArticleSerializer):
    comments = CommentSerializer(many=True, read_only=True)

    class Meta(ArticleSerializer.Meta):
        fields = ArticleSerializer.Meta.fields + ['comments']
```

---

## Views

### Prefer ViewSets over plain APIView for CRUD resources

**ModelViewSet** — full CRUD with minimal code:

```python
from rest_framework import viewsets, permissions
from .models import Article
from .serializers import ArticleSerializer

class ArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.select_related('author').order_by('-created_at')
    serializer_class = ArticleSerializer
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]
    filterset_fields = ['author', 'status']
    search_fields = ['title', 'body']

    def perform_create(self, serializer):
        serializer.save(author=self.request.user)
```

**ReadOnlyModelViewSet** — list + retrieve only:

```python
class CategoryViewSet(viewsets.ReadOnlyModelViewSet):
    queryset = Category.objects.all()
    serializer_class = CategorySerializer
    permission_classes = [permissions.AllowAny]
```

**APIView** — for non-resource endpoints (e.g. stats, actions):

```python
from rest_framework.views import APIView
from rest_framework.response import Response

class DashboardStatsView(APIView):
    def get(self, request):
        data = {
            'total_articles': Article.objects.count(),
            'your_articles': Article.objects.filter(author=request.user).count(),
        }
        return Response(data)
```

**Custom actions on ViewSets** — use `@action`:

```python
from rest_framework.decorators import action

class ArticleViewSet(viewsets.ModelViewSet):
    ...

    @action(detail=True, methods=['post'], permission_classes=[permissions.IsAuthenticated])
    def publish(self, request, pk=None):
        article = self.get_object()
        article.status = 'published'
        article.save()
        return Response({'status': 'published'})
```

---

## URL Routing

Use `DefaultRouter` for ViewSets — it auto-generates all standard routes:

```python
# urls.py
from rest_framework.routers import DefaultRouter
from django.urls import path, include
from .views import ArticleViewSet, DashboardStatsView

router = DefaultRouter()
router.register(r'articles', ArticleViewSet, basename='article')

urlpatterns = [
    path('api/v1/', include(router.urls)),
    path('api/v1/dashboard/stats/', DashboardStatsView.as_view()),
]
```

Generated routes from the router:
- `GET /api/v1/articles/` — list
- `POST /api/v1/articles/` — create
- `GET /api/v1/articles/{pk}/` — retrieve
- `PUT/PATCH /api/v1/articles/{pk}/` — update
- `DELETE /api/v1/articles/{pk}/` — destroy
- `POST /api/v1/articles/{pk}/publish/` — custom action

---

### Session Auth (for browsable API / same-origin SPAs)

```python
urlpatterns += [
    path('api-auth/', include('rest_framework.urls')),
]
```

---

## Permissions

Use built-in permissions or subclass `BasePermission`:

```python
from rest_framework.permissions import BasePermission

class IsOwnerOrReadOnly(BasePermission):
    def has_object_permission(self, request, view, obj):
        if request.method in ('GET', 'HEAD', 'OPTIONS'):
            return True
        return obj.author == request.user
```

Apply per-viewset:

```python
class ArticleViewSet(viewsets.ModelViewSet):
    permission_classes = [permissions.IsAuthenticatedOrReadOnly, IsOwnerOrReadOnly]
```

---

## Pagination

Custom pagination class:

```python
from rest_framework.pagination import PageNumberPagination

class StandardPagination(PageNumberPagination):
    page_size = 20
    page_size_query_param = 'page_size'
    max_page_size = 100
```

Set globally in `REST_FRAMEWORK` settings or per-viewset:

```python
class ArticleViewSet(viewsets.ModelViewSet):
    pagination_class = StandardPagination
```

---

## Key Conventions

- Always version your API: `/api/v1/`
- Use `select_related` / `prefetch_related` in queryset to avoid N+1 queries
- Use separate serializers for list vs. detail views when field sets differ significantly
- Keep `perform_create` / `perform_update` for side effects (set owner, send signals)
- Use `get_queryset()` for dynamic filtering (e.g., scope to current user):

```python
def get_queryset(self):
    return Article.objects.filter(author=self.request.user)
```

- Use `get_serializer_class()` to return different serializers per action:

```python
def get_serializer_class(self):
    if self.action == 'list':
        return ArticleListSerializer
    return ArticleDetailSerializer
```
