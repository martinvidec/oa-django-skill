---
name: django
description: >-
  Build and extend Django 6.0 projects following the official 6.0 documentation
  and best practices. Use when working in a Django codebase (manage.py,
  settings.py, apps/, models.py) or when the user asks to add or change Django
  models, the ORM/QuerySets, migrations, views, URLs, forms, templates, the
  admin, authentication, permissions, security, settings/deployment, async
  views, or tests. Covers project/app layout, the model → migration → admin →
  view → URL → test workflow, what changed in Django 6.0, and common pitfalls.
  Targets Django 6.0 (Python 3.12–3.14). For detailed 6.0 changes see
  reference/django-6.0-changes.md.
---

# Django 6.0

Guidance for working productively and safely in **Django 6.0** projects. Django 6.0
requires **Python 3.12, 3.13, or 3.14**. When the project targets a different Django
version, prefer that version's docs (`https://docs.djangoproject.com/en/<version>/`)
and ignore the 6.0-specific notes below.

## First: orient in the project

Before changing anything, learn the project's conventions instead of assuming defaults.

1. Find the project root: the directory containing `manage.py`.
2. Confirm the Django version: `python -m django --version`, or check
   `requirements*.txt` / `pyproject.toml`. Only apply the 6.0 notes if it really is 6.0.
3. Read settings to learn structure:
   - `INSTALLED_APPS`, `DATABASES`, `AUTH_USER_MODEL`, `MIDDLEWARE`, `STORAGES`.
   - Whether settings are split (`settings/base.py`, `settings/dev.py`, `settings/prod.py`).
4. Identify how the app is run and tested: `pytest`, `manage.py test`, Docker, `tox`, Makefile.
5. Match the existing style: app naming, fat-models vs. service-layer, DRF vs. plain views,
   import ordering, and where business logic lives.

Never invent a database or settings module — use what the project already defines.

## What's new in Django 6.0 (high level)

Full detail in **`reference/django-6.0-changes.md`**. The changes most likely to affect day-to-day work:

- **`DEFAULT_AUTO_FIELD` now defaults to `BigAutoField`** (was `AutoField`). New models get
  `BigAutoField` PKs automatically; you can drop the explicit setting from project templates.
- **Built-in Content Security Policy**: `django.middleware.csp.ContentSecurityPolicyMiddleware`
  plus `SECURE_CSP` / `SECURE_CSP_REPORT_ONLY` settings and `django.utils.csp.CSP` constants.
- **Forms default renderer is now `as_div`** — `{{ form }}` renders `<div>`-wrapped fields
  (was `as_p`). `as_p`, `as_table`, `as_ul` still available explicitly.
- **Template partials**: define with `{% partialdef name %}…{% endpartialdef %}`, render with
  `{% partial name %}`, and load fragments via `template_name#partial_name`.
- **Background Tasks framework**: `@task` decorator + `.enqueue()` + `TASKS` setting (real
  execution needs an external worker; built-in backends are for dev/test).
- **Async class-based views**: CBV handlers may be `async def` (all handlers in a class must be
  either all sync or all async). Plus `AsyncPaginator` / `AsyncPage`.
- **ORM**: new `AnyValue` aggregate; `StringAgg` now works on all backends; `Aggregate` gains
  `order_by`; `Model.save()` raises `Model.NotUpdated` when a forced update matches no rows.
- **`ADMINS` / `MANAGERS`** are now lists of plain email strings (not `(name, address)` tuples).
- **Minimum versions**: Python 3.12+, MariaDB 10.6+, `asgiref` 3.9.1+. `cx_Oracle` support removed.

## Core workflow: adding or changing a feature

The usual order when adding a model-backed feature:

1. **Model** — define/modify in `<app>/models.py`.
2. **Migration** — `python manage.py makemigrations <app>`, then **review the generated file**
   before applying. Apply with `python manage.py migrate`.
3. **Admin** — register in `<app>/admin.py` if it should be editable in the admin.
4. **View / serializer** — `views.py` (or DRF `serializers.py` + `viewsets.py`).
5. **URL** — wire into `<app>/urls.py` and `include()` it from the project `urls.py`.
6. **Test** — add tests covering the new behavior.
7. **Run** — start the dev server (`python manage.py runserver`) or tests to verify.

## Models & fields

- Subclass `django.db.models.Model`; define **`__str__`** on every model; add
  **`get_absolute_url()`** when objects map to a canonical URL.
- PKs: each model gets a `BigAutoField` PK from `DEFAULT_AUTO_FIELD` (new default in 6.0).
  `OneToOneField` does **not** auto-become the PK — pass `primary_key=True` if you want that.
- `ForeignKey` requires **`on_delete`** — choose deliberately (`CASCADE`, `PROTECT`, `RESTRICT`,
  `SET_NULL` (needs `null=True`), `SET_DEFAULT`, `SET(callable)`, `DO_NOTHING`).
- Set explicit **`related_name`** on FK/M2M. Use string refs (`"app.Model"`) for circular/forward
  refs. Reference the user model via `settings.AUTH_USER_MODEL`, never a direct `User` import.
- Enumerations: use `models.TextChoices` / `models.IntegerChoices`; read display with
  `get_<field>_display()`.
- Money: `DecimalField(max_digits=…, decimal_places=…)` — never `FloatField`.
- `null` is DB-level; `blank` is validation. Avoid `null=True` on string fields (use `""`),
  except `CharField(unique=True, blank=True, null=True)` to avoid empty-string unique clashes.
- Never use mutable defaults — pass a callable: `default=list`, `default=dict`.
- Indexes: prefer `Meta.indexes = [models.Index(fields=[…])]`; add `db_index=True` for single hot
  fields. Use `Meta.constraints` (`CheckConstraint(condition=Q(...), name=…)`, `UniqueConstraint`).
- `db_default=` sets a DB-computed default (e.g. `db_default=Now()`); `GeneratedField` is a
  DB-computed column (6.0 auto-refreshes it after `save()` on SQLite/PostgreSQL/Oracle).
- Overridden `save()`/`delete()` are **not** called for bulk QuerySet ops or cascade deletes —
  use signals (`pre_save`/`post_save`/…). In 6.0, `Field.pre_save()` may be called more than
  once, so keep it idempotent and side-effect free.

## Managers

- Default manager is `objects = models.Manager()` (class-level, not on instances).
- Customize via `get_queryset()`; build a manager from a custom `QuerySet` with
  `SomeQuerySet.as_manager()` or `Manager.from_queryset(SomeQuerySet)()`.
- The **base manager must not filter rows** — filtering there breaks related-object access. Use a
  separate manager for filtered views. Set `use_in_migrations = True` for managers used in migrations.

## Migrations

- Workflow: edit models → `makemigrations` → **review the generated migration** → `migrate` →
  commit model changes and migration files together.
- Guard CI with `python manage.py makemigrations --check --dry-run` to catch missing migrations.
- Adding a non-nullable field to existing data needs a default (`default=`/`db_default=`) or a
  two-step migration.
- Data migrations: use `migrations.RunPython(forward, reverse)`; **always** load models via
  `apps.get_model("app", "Model")` (historical models lack custom methods/`save()`). Provide a
  reverse function or reversing raises `IrreversibleError`.
- Don't edit an already-applied, shared migration — add a new one. Squash with `squashmigrations`
  (6.0 can re-squash already-squashed migrations).
- DDL is wrapped in a transaction on SQLite/PostgreSQL, not on MySQL/Oracle; set `atomic = False`
  on the `Migration` for non-transactional operations.

## ORM & performance

- **Avoid N+1**: `select_related(*fields)` for FK/one-to-one (JOIN); `prefetch_related(*lookups)`
  for M2M / reverse FK (separate query); `Prefetch()` to filter the prefetched set.
- Do work in the DB: `filter()`/`exclude()`, `F()` expressions for column math, `annotate()` /
  `aggregate()`. Use `RawSQL`/`raw()` only when the ORM can't express it (always pass `params=`).
- Retrieve only what you need: `values()` / `values_list(flat=True)`, `only()` / `defer()`; use
  `entry.blog_id` instead of `entry.blog.id` to skip a query.
- Cheap checks: `exists()` not `if qs:`; `count()` not `len(qs)`; `contains(obj)` not `obj in qs`
  — but if you also need the data, evaluate the QuerySet once and reuse its cache.
- Bulk writes: `bulk_create(...)`, `bulk_update(objs, [fields])`, `QuerySet.update(**kw)` /
  `.delete()`. Note `update()` does **not** call `save()` and fires no signals; it supports
  `F()` (e.g. `.update(n=F("n") + 1)`). Stream large reads with `iterator()`.
- Aggregation correctness: `Count("rel", distinct=True)` to avoid JOIN inflation; per-aggregate
  `filter=Q(...)` for conditional aggregates; `annotate().filter()` is not commutative with
  `filter().annotate()`; pass `default=0` to `Sum`/`Avg` for empty sets.
- Debug with `qs.explain()`, `str(qs.query)`, and `django.db.connection.queries`.

## Transactions

- Use `@transaction.atomic` / `with transaction.atomic():`. Nested blocks become savepoints.
- **Catch DB exceptions outside the `atomic()` block** — catching inside can leave the
  transaction broken (`TransactionManagementError`).
- Use `transaction.on_commit(func)` for side effects (emails, cache invalidation, enqueueing
  tasks) that must run only after a successful commit.
- `durable=True` asserts the block is the outermost atomic block. `ATOMIC_REQUESTS=True` wraps
  each view in a transaction (opt out with `@transaction.non_atomic_requests`).

## Views & URLs

- URLs: define `urlpatterns` with `path(route, view, name=…)`; use `re_path()` only for regex.
  Converters: `str`, `int`, `slug`, `uuid`, `path`. **Always name patterns** and reverse them —
  `reverse()`/`reverse_lazy()` in Python, `{% url 'name' %}` in templates; never hardcode paths.
- Namespace included URLconfs with `app_name` + `include()`, reverse as `"namespace:name"`.
  Error handlers (`handler400/403/404/500`) go in the **root** URLconf only.
- Views: function or class-based. Keep business logic out of views (put it on the model, a
  manager, or a service module). Use `get_object_or_404` / `get_list_or_404` instead of manual
  `try/except DoesNotExist`. Use `HttpResponse(status=…)` for explicit status codes.
- CBVs: subclass `View` or generics (`ListView`, `DetailView`, `FormView`, `CreateView`,
  `UpdateView`, `DeleteView`); register with `as_view()`; customize via `get_context_data()` /
  `get_queryset()`. Use CBVs to reuse generic behavior; FBVs for simple, one-off flows.

## Forms & templates

- Prefer **`ModelForm`** when a form maps to a model; plain `forms.Form` otherwise.
- View flow: bind on POST (`MyForm(request.POST, request.FILES)`), check `form.is_valid()`, read
  `form.cleaned_data[...]`, then **redirect** (POST/redirect/GET). Keep validation in the form
  (`clean_<field>()`, `clean()`), not the view.
- Always put `{% csrf_token %}` inside POST `<form>`. `{{ form }}` now renders `as_div` (6.0);
  use `as_p`/`as_table`/`as_ul` explicitly if needed. Include `{{ form.media }}` for widget assets.
- Templates: autoescaping is **on by default** — rely on it. Only mark output safe deliberately
  (`|safe`, `{% autoescape off %}`, `mark_safe()`) on trusted content. Never let users supply
  templates (XSS). Namespace templates per app (`app/templates/app/...`); pass `request` so
  `csrf_token`/context processors work.

## Authentication & permissions

- **Custom user model from day one**: set `AUTH_USER_MODEL = "app.User"` before the first
  migration. Subclass `AbstractUser` (keep default fields) or `AbstractBaseUser` +
  `PermissionsMixin` (full control, define `USERNAME_FIELD`, a `BaseUserManager`, etc.).
- Never import `User` directly: use `settings.AUTH_USER_MODEL` in models and
  `get_user_model()` at runtime. Set passwords with `set_password()` / `create_user()`.
- Check `request.user.is_authenticated` (property). After a custom password change, call
  `update_session_auth_hash(request, user)` to avoid logging the user out.
- Access control: `@login_required`, `@permission_required("app.codename", raise_exception=True)`,
  `@user_passes_test(fn)` for FBVs; `LoginRequiredMixin`, `PermissionRequiredMixin`,
  `UserPassesTestMixin` (place first in MRO) for CBVs. Default per-model perms: `add/change/
  delete/view_*`; check with `user.has_perm("app.codename")`.
- Async auth helpers exist: `aauthenticate`, `alogin`, `alogout`, `request.auser()`.

## Security

- **CSRF**: keep `CsrfViewMiddleware` on; `{% csrf_token %}` in every POST form; avoid
  `@csrf_exempt`. Set `CSRF_COOKIE_SECURE=True` and `CSRF_TRUSTED_ORIGINS` in production.
- **SQL injection**: trust the ORM's parameterization; avoid `extra()`/`RawSQL`/`raw()`, and if
  unavoidable always pass `params=` rather than f-strings.
- **XSS**: template autoescaping is on; use `mark_safe`/`|safe` sparingly on trusted content only.
- **Secrets**: load `SECRET_KEY` from the environment; rotate via `SECRET_KEY_FALLBACKS`. Keep
  DB credentials out of source control.
- **Production**: `DEBUG=False`, correct `ALLOWED_HOSTS`, HTTPS settings (`SECURE_SSL_REDIRECT`,
  `SECURE_HSTS_SECONDS`, `SESSION_COOKIE_SECURE`, `CSRF_COOKIE_SECURE`). Consider `SECURE_CSP`.
- Run `python manage.py check --deploy` against production settings (use `--fail-level` in CI).

## Settings & deployment

- Read sensitive/environment-specific values from the environment; respect an existing
  split-settings layout. Add new apps to `INSTALLED_APPS` and middleware in the correct order.
- Static files: configure storage via the **`STORAGES`** setting (dict with a `"staticfiles"`
  alias). Namespace files per app (`app/static/app/...`); in production set `STATIC_ROOT` and run
  `collectstatic`; serve via the web server (not `runserver`). Treat `MEDIA_ROOT` as untrusted.
- `ADMINS`/`MANAGERS` are now plain email-string lists in 6.0. Set real `DEFAULT_FROM_EMAIL` /
  `SERVER_EMAIL`. Serve with a real WSGI/ASGI server — never `runserver` in production.

## Async

- Declare async views with `async def` (function views, or CBV HTTP-method handlers — all
  handlers in one class must be all sync or all async). Run under an ASGI server for real benefit.
- Prefer native async ORM methods (`aget`, `acreate`, `afirst`, `asave`, `async for` over a
  QuerySet) over wrapping. Wrap sync/async-unsafe calls in
  `asgiref.sync.sync_to_async(fn, thread_sensitive=True)`; use `async_to_sync` for the reverse.
- **Transactions are not yet supported in async** — wrap transactional logic in `sync_to_async()`.
- Disable persistent DB connections (`CONN_MAX_AGE`) in async; any sync middleware forces
  thread-per-request and negates async gains. Never set `DJANGO_ALLOW_ASYNC_UNSAFE` in production.

## Admin

- Register with `@admin.register(Model)` on a `ModelAdmin` subclass (or `admin.site.register`).
- Common options: `list_display`, `list_filter`, `search_fields` (prefixes: `^` startswith,
  `=` exact, `@` full-text), `ordering`, `date_hierarchy`, `list_select_related` (perf),
  `readonly_fields`, `fieldsets`, `autocomplete_fields` (needs `search_fields` on the related
  admin), `prepopulated_fields`, `inlines` (`TabularInline`/`StackedInline`).
- Use `@admin.display(description=…, ordering=…, boolean=…)` for custom `list_display` methods and
  `format_html()` for HTML. Scope querysets with `formfield_for_foreignkey` / `get_queryset`.
  Enforce access with `has_view/add/change/delete_permission`.

## Testing

- Match the project's runner (`pytest` + `pytest-django`, or `manage.py test`).
- Base classes: `TestCase` (default; transaction + rollback per test), `SimpleTestCase` (no DB),
  `TransactionTestCase` (real commit/rollback; slower), `LiveServerTestCase` (browser/Selenium).
- Create read-only fixtures once with `setUpTestData(cls)`; use `setUp` only for per-test mutation.
  Override config with `@override_settings(...)` — never mutate `settings` directly.
- Use `self.client` (`Client`); assertions like `assertContains`, `assertRedirects`,
  `assertTemplateUsed`, `assertFormError`, `assertNumQueries`. Use
  `captureOnCommitCallbacks(execute=True)` to test `on_commit` hooks.
- Async tests: write `async def test_…`, use `self.async_client` and `await` every request
  (`alogin`/`aforce_login`/`alogout`).
- Speed: `--parallel`, `--keepdb`; check order-independence with `--shuffle` / `--reverse`.

## Handy commands

```bash
python manage.py startapp <app>                     # scaffold an app
python manage.py makemigrations [app]               # create migrations
python manage.py migrate                            # apply migrations
python manage.py makemigrations --check --dry-run   # CI guard for missing migrations
python manage.py sqlmigrate <app> <name>            # preview a migration's SQL
python manage.py showmigrations                     # migration state
python manage.py runserver                          # dev server (dev only)
python manage.py shell                              # shell (6.0 auto-imports models/connection/…)
python manage.py createsuperuser                    # admin user
python manage.py check --deploy                     # production system checks
python manage.py collectstatic                      # gather static files
python manage.py test                               # run tests (or: pytest)
```

## Common pitfalls

- Forgetting to create/apply a migration after a model change (catch with `makemigrations --check`).
- Importing `User` directly instead of `settings.AUTH_USER_MODEL` / `get_user_model()`.
- Mutable default arguments (`default=[]`) — use `default=list` / `default=dict`.
- N+1 queries from accessing related objects in loops/templates.
- Catching DB exceptions *inside* `transaction.atomic()` instead of outside.
- Editing an already-applied, shared migration instead of adding a new one.
- Assuming `{{ form }}` renders `as_p` — in 6.0 it renders `as_div`.
- Using transactions inside async code (not yet supported) — wrap in `sync_to_async`.
- A base manager that filters rows, silently breaking related-object lookups.
