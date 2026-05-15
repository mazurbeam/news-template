# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Wagtail CMS (Django 6.0 + Wagtail 7.4) bootstrapped from `wagtail-news-template`. Default admin login (after `load-data`): `admin` / `password` at `/admin`.

## Common commands

Backend (run inside venv):

```bash
make load-data    # createcachetable + migrate + load_initial_data + collectstatic
make start        # ./manage.py runserver
make reset-db     # delete db.sqlite3 and reload fixtures
make dump-data    # write fixtures/demo.json from current DB state
```

Frontend assets (webpack + Sass + Tailwind, output to `static_compiled/`):

```bash
npm run start     # webpack --watch (dev)
npm run build     # one-shot dev build
npm run build:prod
```

`webpack-dev-server` proxies to Django on `:8000` and serves on `:3000`. After changing static sources, run `python manage.py collectstatic --noinput` so Django picks them up.

Tests use Django's runner: `python manage.py test [app.module.TestClass.test_method]`. There is no separate lint/format config checked in.

## Architecture

- `fmrx/` — Django project. URL root is `fmrx.urls`; Wagtail's `wagtail_urls` is the catch-all at the bottom (page-served), with `/admin/`, `/django-admin/`, `/documents/`, `/search/` reserved above it.
- `fmrx/settings/` — split settings: `base.py`, `dev.py`, `production.py`. `DJANGO_SETTINGS_MODULE` selects which. Database via `dj_database_url` (`DATABASE_URL` env), defaults to local sqlite. Production storages can swing to S3 via `AWS_*` / `DEFAULT_*` env (`get_first_env` helper checks both Divio and AWS prefixes).
- Custom user model: `AUTH_USER_MODEL = "users.User"` (`fmrx.users`). Any auth-related migration must respect this.
- Wagtail apps (each is a Django app under `fmrx/`): `home`, `news`, `standardpages`, `forms`, `images`, `navigation`, `search`, `users`, `utils`. `fmrx.utils` holds shared `blocks.py` (StreamField blocks), `cache.py`, `query.py`, `struct_values.py`, `wagtail_hooks.py`, and a `templatetags/` library — extend page/block schemas there rather than per-app where shared.
- Templates live at repo-root `templates/` (project-level overrides + `components/`, `icons/`, `navigation/`, `pages/`); per-app templates live inside each app. Both are picked up because `APP_DIRS=True` and `templates/` is on `TEMPLATES.DIRS`.
- Static pipeline: source in `static_src/{javascript,sass,images,fonts}` → webpack → `static_compiled/` → `collectstatic` → `static/`. Production uses `ManifestStaticFilesStorage` (hashed filenames; rebuild + collectstatic after asset changes). WhiteNoise serves static in prod.
- Cache backend is `DatabaseCache` (`database_cache` table) — `createcachetable` is required before first migrate-and-run. `make load-data` does this for you.
- Fixtures: `fixtures/demo.json` loaded by the `load_initial_data` management command (custom; lives under one of the apps' `management/commands/`). `make dump-data` regenerates it while excluding noisy auto-tables (permissions, contenttypes, sessions, search index, rendition, reference index, page subscriptions).

## Deployment

- `Dockerfile` + `fly.toml` target Fly.io. The volume `wagtail` is mounted at `/data`; `DATABASE_URL=sqlite:////data/db.sqlite3` and `MEDIA_ROOT=/data/media` are set there. App auto-stops/auto-starts; static files served via WhiteNoise, media via Fly `[[statics]]` mapping `/data/media` → `/media`.
- Production gunicorn config: `gunicorn.conf.py`.
- After deploy, seed via `fly ssh console -u wagtail -C "./manage.py load_initial_data"`.

## Conventions

- This repo originated from `wagtail/news-template`. When backporting upstream template changes, replace `myproject` with `{{ project_name }}`, rename the project dir, and wrap template tags in `{% verbatim %}` so `wagtail start` doesn't render them. See README "Contributing".
- `requirements.txt` pins ranges, not exact versions, except `gunicorn==23.0.0`. Stay within `django>=6.0.5,<6.1` and `wagtail>=7.4,<7.5` unless intentionally upgrading.

## Known scaffold issues (upstream `wagtail-news-template`)

Dormant pre-existing import bugs — surface only on edge paths:
- `fmrx/utils/models.py:120-157` (`ArticleTopic.save`) — uses `router`, `transaction`, `IntegrityError`, `slugify` without imports; only fires when saving an `ArticleTopic` without a slug (fixtures pre-set slugs).
- `fmrx/news/models.py:106-108` — uses `PageNotAnInteger`/`EmptyPage` without imports; only fires on a malformed `?page=` query to the news index.
- `fmrx/search/tests/test_view.py` was missing `from django.urls import reverse` until the Django 6 upgrade.

## Fixture-load noise (Wagtail 7+)

`make load-data` / `manage.py load_initial_data` emits `modelsearch.tasks.insert_or_update_object_task` → `DoesNotExist` tracebacks mid-run. Non-fatal — the `django-tasks` async search indexer races fixture inserts. Loader prints "Awesome. Your data is loaded!" at the end on success.

## SEO_NOINDEX wiring

`fmrx/utils/context_processors.py` is **not** registered in `TEMPLATES.OPTIONS.context_processors` (`fmrx/settings/base.py:89`). Its `settings.SEO_NOINDEX` lookup is dead code. The value reaches `templates/base.html` only via the search view's per-request context (`fmrx/search/views.py:32`); on other pages the template variable is silently empty.
