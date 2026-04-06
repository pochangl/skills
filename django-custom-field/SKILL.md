---
name: django-custom-field
description: Use this skill when creating or modifying custom Django model fields (subclassing ImageField, CharField, etc.). Triggers on mentions of "custom field", "extend field", "subclass field", "ProfileImage", or when writing a class that inherits from a Django model field.
---

# Custom Django Model Field Guide

When creating custom model fields that extend Django built-in fields, always override `deconstruct()` to remove all references to non-Django source code from migration files.

## Why

Migration files should only contain references to Django's own library. This keeps migrations portable and avoids import errors if custom field code is moved, renamed, or refactored.

## Pattern

```python
class ProfileImage(models.ImageField):
    def __init__(self, *args, **kwargs):
        kwargs.setdefault('upload_to', profile_image_upload_to)
        super().__init__(*args, **kwargs)

    def deconstruct(self):
        name, path, args, kwargs = super().deconstruct()
        path = 'django.db.models.ImageField'  # replace with base Django field path
        kwargs.pop('upload_to', None)          # remove non-Django callable references
        return name, path, args, kwargs
```

## Rules

1. **Replace the field path** — set `path` to the base Django field (e.g., `django.db.models.ImageField`, `django.db.models.CharField`).
2. **Remove non-Django kwargs** — pop any kwargs whose values are custom callables or non-Django objects (e.g., custom `upload_to` functions, custom validators).
3. **Keep Django-native kwargs** — leave standard kwargs like `blank`, `null`, `default`, `max_length` intact.
