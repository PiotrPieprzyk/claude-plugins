# claude-plugins

Personal [Claude Code](https://claude.com/claude-code) plugin marketplace.

| Plugin | Skills | What for |
|---|---|---|
| `api` | `/api:story`, `/api:test` | API story specs and black-box API story tests, grouped by domain under `docs/stories/` |
| `ui` | `/ui:story`, `/ui:test`, `/ui:app` | UI story specs, UI story integration tests and story-driven component structure |

## Install

```
/plugin marketplace add PiotrPieprzyk/claude-plugins
/plugin install api@piotr-plugins
/plugin install ui@piotr-plugins
```

Pull new versions with `/plugin marketplace update piotr-plugins`.

To make a project prompt its contributors to install them, add to its `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "piotr-plugins": { "source": { "source": "github", "repo": "PiotrPieprzyk/claude-plugins" } }
  },
  "enabledPlugins": { "api@piotr-plugins": true, "ui@piotr-plugins": true }
}
```

## Developing

Load a plugin straight from disk while editing it: `claude --plugin-dir ./api`.
Validate before pushing: `claude plugin validate .`.
