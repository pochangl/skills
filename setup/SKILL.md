---
name: setup
description: Use this skill when the user asks to create, scaffold, or set up a new project. Triggers on mentions of "new project", "create project", "setup", "scaffold", "init", or "bootstrap".
---

# Project Setup — Default Stack & Rules

## General Rules

- Always use the official project creation method for each framework. Never generate boilerplate files manually.
  - Django: `django-admin startproject`
  - Next.js: `npx create-next-app`
  - Flutter: `flutter create`

## Backend (default)

- **Framework:** Django + Django REST Framework + ADRF (async DRF)
- **Database:** PostgreSQL
- **Async / real-time:** Django Channels + Redis
- **Python environment:** Use a venv. Load the existing venv if found; create one if not.

## Web Frontend (default)

- **Framework:** Next.js (always use the latest version)
- **Node version management:** Use nvm to select the Node.js version.

## Mobile (default)

- **Framework:** Flutter
