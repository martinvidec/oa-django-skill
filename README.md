# oa-django-skill

A Claude Code skill for building and extending **Django 6.0** projects, distilled
from the official Django 6.0 documentation (`https://docs.djangoproject.com/en/6.0/`).

It covers project/app layout, the model → migration → admin → view → URL → test
workflow, the ORM, forms, templates, authentication, permissions, security,
settings/deployment, async, the admin, testing — and what changed in Django 6.0.

## What's in here

```
.claude-plugin/plugin.json          # plugin manifest
skills/
  django/
    SKILL.md                        # the skill (auto-loaded when relevant)
    reference/
      django-6.0-changes.md         # detailed 6.0 new/changed/deprecated/removed list
```

## Install from GitHub

This repo is a Claude Code **plugin** containing one skill. Add it as a plugin
marketplace from GitHub, then install the plugin:

```
/plugin marketplace add martinvidec/oa-django-skill
/plugin install oa-django-skill
```

After installing, the `django` skill becomes available and Claude will load it
automatically when you work in a Django project or ask about Django.

### Alternative: install just the skill

If you only want the skill (not the plugin wrapper), copy the skill directory into
your skills folder:

- **Project scope** (this repo only): `.claude/skills/django/`
- **User scope** (all projects): `~/.claude/skills/django/`

```bash
# user scope
mkdir -p ~/.claude/skills
cp -r skills/django ~/.claude/skills/django
```

## Scope & versioning

The skill targets **Django 6.0** (Python 3.12–3.14). When a project uses a
different Django version, prefer that version's docs; the 6.0-specific notes in
`SKILL.md` and `reference/django-6.0-changes.md` won't all apply.

## License

MIT — see [LICENSE](LICENSE).
