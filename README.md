# claude-plugins

Personal [Claude Code](https://claude.com/claude-code) plugin marketplace.

| Plugin | Skills | What for |
|---|---|---|
| `back` | `/back:story`, `/back:test` | API story specs and black-box API story tests, grouped by domain under `docs/stories/` |
| `front` | `/front:story`, `/front:test`, `/front:app` | UI story specs, UI story integration tests and story-driven component structure |

## Install

```
/plugin marketplace add PiotrPieprzyk/claude-plugins
/plugin install back@piotr-plugins
/plugin install front@piotr-plugins
```

Pull new versions with `/plugin marketplace update piotr-plugins`.

To make a project prompt its contributors to install them, add to its `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "piotr-plugins": { "source": { "source": "github", "repo": "PiotrPieprzyk/claude-plugins" } }
  },
  "enabledPlugins": { "back@piotr-plugins": true, "front@piotr-plugins": true }
}
```

## Developing

Load a plugin straight from disk while editing it: `claude --plugin-dir ./back`.
Validate before pushing: `claude plugin validate .`.
