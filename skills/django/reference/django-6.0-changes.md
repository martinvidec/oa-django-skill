# Django 6.0 — detailed changes

Distilled from the official Django 6.0 release notes and topic docs
(`https://docs.djangoproject.com/en/6.0/releases/6.0/`). Consult the linked
release notes for the authoritative, complete list.

## Compatibility & minimum versions

- **Python 3.12, 3.13, 3.14** supported; minimum is now **Python 3.12** (Django 5.2.x
  was the last series to support Python 3.10/3.11).
- Minimum **`asgiref` 3.9.1**.
- **MariaDB 10.6+** required (MariaDB 10.5 dropped).
- **`cx_Oracle` support removed** (use the `oracledb` driver).

## Major new features

- **Content Security Policy (CSP)**: `django.middleware.csp.ContentSecurityPolicyMiddleware`,
  `django.template.context_processors.csp()`, settings `SECURE_CSP` and
  `SECURE_CSP_REPORT_ONLY`, and constants in `django.utils.csp` (`CSP.SELF`, `CSP.NONE`,
  `CSP.NONCE`, …). Example:
  `SECURE_CSP = {"default-src": [CSP.SELF], "frame-src": [CSP.NONE]}`.
- **Template partials**: `{% partialdef name %}…{% endpartialdef %}` and `{% partial name %}`,
  plus `template_name#partial_name` syntax in `get_template()`, `render()`, and `{% include %}`.
- **Background Tasks framework**: `django.tasks.task()` decorator, `.enqueue()`, the `TASKS`
  setting, and two built-in (dev/test) backends. Real execution needs an external worker.
- **Modern Python email API**: `EmailMessage.message()` returns `email.message.EmailMessage`
  (not `SafeMIMEText`/`SafeMIMEMultipart`); `attach()` accepts `email.message.MIMEPart`; new
  `policy` argument (defaults to `email.policy.default`).

## Minor features by area

- **admin**: Font Awesome Free 6.7.2 icons; `AdminSite.password_change_form`; distinct styling
  for `DEBUG`/`INFO` message levels.
- **auth**: PBKDF2 default iterations raised 1,000,000 → **1,200,000**.
- **gis**: `GEOSGeometry.hasm`; `Rotate()` and `GeometryType()` functions; `geom_type` lookup;
  `BaseGeometryWidget.base_layer`; MariaDB 12.0.1+ support for several GIS ops; widgets render
  without inline JS.
- **postgres**: `Lexeme` full-text-search expression; system checks for `django.contrib.postgres`;
  `hints` param on extension operations.
- **staticfiles**: `ManifestStaticFilesStorage` consistent path ordering; quieter `collectstatic`
  output for skipped files at verbosity 1.
- **i18n**: Haitian Creole added.
- **management commands**: `startproject`/`startapp` create the target dir if missing; `shell`
  auto-imports `connection`, `reset_queries`, `models`, `settings`, `timezone` (`--no-imports`
  to disable; `-i {ipython,bpython,python}` to pick the REPL).
- **migrations**: squash already-squashed migrations; serialize `zoneinfo.ZoneInfo`;
  deconstructible objects with non-identifier kwargs.
- **models**: `Constraint.check()` method; `Aggregate` gains `order_by` arg and `allow_order_by`
  attribute; **`StringAgg` now works on all backends**; new **`AnyValue`** aggregate; new
  **`Model.NotUpdated`** exception; `QuerySet.raw()` supports `CompositePrimaryKey`; composite-PK
  subqueries beyond `__in`; `JSONField` negative array indexing on SQLite; `GeneratedField` /
  expression fields refreshed after `save()` on RETURNING backends (SQLite/PostgreSQL/Oracle).
- **pagination**: `AsyncPaginator`, `AsyncPage`.
- **requests/responses**: multiple `Cookie` headers for HTTP/2 over ASGI.
- **templates**: `forloop.length`; `{% querystring %}` always prefixes `?` and accepts multiple
  mapping positional args.
- **tests**: `DiscoverRunner` parallel execution on the `forkserver` start method.
- **views/forms**: forms default renderer is now **`as_div`**; async class-based views.

## Backwards-incompatible changes

- **`DEFAULT_AUTO_FIELD` now defaults to `BigAutoField`** (was `AutoField`). Set `AutoField`
  explicitly if you need the old behavior.
- **`ADMINS` and `MANAGERS`** are now lists of plain email-address strings, not `(name, address)`
  tuples.
- DB backend API renames: `return_insert_columns()` → `returning_columns()`;
  `fetch_returned_insert_rows()` → `fetch_returned_rows()` (now takes `cursor` +
  `returning_params`); `fetch_returned_insert_columns()` removed. No more `CASCADE` when dropping
  columns.
- Custom ORM expression `as_sql()` must return params as a **tuple**, not a list.
- Email: undocumented `mixed_subtype`/`alternative_subtype`/`encoding` (legacy `Charset`)
  properties dropped.
- JSON serializer now writes a trailing newline even without `indent`.
- `Field.pre_save()` may be called multiple times — must be idempotent / side-effect free.

## Deprecated in 6.0

- **Positional args in `django.core.mail` APIs**: `get_connection()`, `mail_admins()`,
  `mail_managers()`, `send_mail()`, `send_mass_mail()`, and `EmailMessage` /
  `EmailMultiAlternatives` (only the first four positional args allowed) — the rest must be keywords.
- `BaseDatabaseCreation.create_test_db(serialize)` → use `serialize_db_to_string()`.
- PostgreSQL-specific `StringAgg` and `OrderableAggMixin` → use the general `StringAgg` /
  `Aggregate.order_by`; string `delimiter` → use `Value`.
- `URLIZE_ASSUME_HTTPS` setting (`urlize`/`urlizetrunc` will default to HTTPS in 7.0).
- `ADMINS`/`MANAGERS` as `(name, address)` tuples → use email-string lists.
- `Paginator`/`AsyncPaginator` with `orphans >= per_page`.
- Legacy email helpers: `MIMEBase` to `attach()`, `BadHeaderError`, `SafeMIMEText`,
  `SafeMIMEMultipart`, `forbid_multi_line_headers()`, `sanitize_address()`.

## Removed in 6.0 (from earlier deprecation cycles)

- From 5.0: positional args to `BaseConstraint`; `DjangoDivFormRenderer` /
  `Jinja2DivFormRenderer`; `field_cast_sql()`; `request` now required in
  `ModelAdmin.lookup_allowed()`; `format_html()` without args; `forms.URLField` default scheme
  now `https`; `FORMS_URLFIELD_ASSUME_HTTPS` removed; `cx_Oracle` support removed; `ChoicesMeta`
  alias; `Prefetch.get_current_queryset()`, `get_prefetch_queryset()`.
- From 5.1: `register_converter()` can't override existing converters;
  `ModelAdmin.log_deletion()`; `LogEntryManager.log_action()`; `is_iterable()`;
  `GeoIP2.coords()`/`.open()`; positional args to `Model.save()`/`asave()`; `CheckConstraint`
  `check` kwarg (use `condition`); `FieldCacheMixin.get_cache_name()`;
  `FileSystemStorage.OS_OPEN_FLAGS`.

## Version-attribution caveats (do not advertise these as new in 6.0)

- `CompositePrimaryKey` — new in **5.2**.
- `UniqueConstraint.violation_error_code` / `violation_error_message` honored without a
  `condition` — changed in **5.2**.
- `values()`/`values_list()` respecting field order — changed in **5.2**.
- The async ORM `a`-prefixed methods, `AsyncClient`, and async auth helpers are mature features
  current in 6.0 but were introduced earlier — treat them as "current best practice," not 6.0 additions.
- `db_default`, `GeneratedField` (the field itself), `JSONField`, constraint `include` /
  `opclasses` / `deferrable` / `nulls_distinct` all predate 6.0.
