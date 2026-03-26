# ClaudeWorksHub Plugin Marketplace

Claude Code plugin marketplace maintained by [ClaudeWorksHub](https://github.com/ClaudeWorksHub).

## Available Plugins

| Plugin | Description | Version |
|--------|-------------|---------|
| [agora](https://github.com/ClaudeWorksHub/agora) | Multi-AI role debate for high-quality proposals | 2.1.0 |

## Installation

```bash
# 1. Add marketplace
/plugin marketplace add ClaudeWorksHub/claude-plugins

# 2. Install a plugin
/plugin install agora@claudeworkshub

# 3. Activate
/reload-plugins
```

## Update

```bash
/plugin marketplace update claudeworkshub
/plugin install agora@claudeworkshub
/reload-plugins
```

---

## For Plugin Developers

### How to publish a plugin to this marketplace

#### Prerequisites

- Your plugin repo is in the [ClaudeWorksHub](https://github.com/ClaudeWorksHub) organization
- The [claude-plugins-publisher](https://github.com/apps/claude-plugins-publisher) GitHub App is installed on your repo

#### Step 1: Register your plugin

Add your plugin entry to `.claude-plugin/marketplace.json` in this repo via PR:

```json
{
  "name": "your-plugin-name",
  "source": {
    "source": "url",
    "url": "https://github.com/ClaudeWorksHub/your-plugin.git",
    "sha": "<commit-sha>"
  },
  "description": "Short description",
  "version": "1.0.0",
  "author": { "name": "Your Name", "email": "you@example.com" },
  "homepage": "https://github.com/ClaudeWorksHub/your-plugin",
  "license": "MIT",
  "category": "productivity",
  "tags": ["tag1", "tag2"]
}
```

#### Step 2: Add the publish workflow

Copy this file to your plugin repo as `.github/workflows/publish.yml`:

```yaml
name: Publish to Marketplace

on:
  push:
    tags:
      - 'v*'

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Extract version and validate
        id: meta
        run: |
          VERSION="${GITHUB_REF_NAME#v}"
          PLUGIN_VERSION=$(jq -r '.version' .claude-plugin/plugin.json)
          if [ "$PLUGIN_VERSION" != "$VERSION" ]; then
            echo "::error::Tag version (${VERSION}) does not match plugin.json (${PLUGIN_VERSION})"
            exit 1
          fi
          echo "version=${VERSION}" >> "$GITHUB_OUTPUT"
          echo "sha=${GITHUB_SHA}" >> "$GITHUB_OUTPUT"

      - name: Generate App token
        id: app-token
        uses: actions/create-github-app-token@v1
        with:
          app-id: ${{ secrets.APP_ID }}
          private-key: ${{ secrets.APP_PRIVATE_KEY }}
          owner: ClaudeWorksHub
          repositories: claude-plugins

      - name: Trigger marketplace update
        run: |
          gh api repos/ClaudeWorksHub/claude-plugins/dispatches \
            -f event_type=plugin-update \
            -f "client_payload[plugin]=<your-plugin-name>" \
            -f "client_payload[version]=${{ steps.meta.outputs.version }}" \
            -f "client_payload[sha]=${{ steps.meta.outputs.sha }}"
        env:
          GH_TOKEN: ${{ steps.app-token.outputs.token }}
```

> Replace `<your-plugin-name>` with your plugin's `name` in marketplace.json.

#### Step 3: Release

```bash
# 1. Update version in .claude-plugin/plugin.json
# 2. Commit and push
git commit -am "v1.1.0: your changes"
git push

# 3. Tag and push — marketplace updates automatically
git tag v1.1.0
git push origin v1.1.0
```

The publish workflow validates the tag, generates a short-lived App token, and triggers this repo's sync workflow. Marketplace is updated within seconds.

### Architecture

```
Plugin repo                          claude-plugins (this repo)
───────────                          ──────────────────────────
push tag v1.1.0
    │
    ▼
publish.yml
    │  actions/create-github-app-token
    │  (org-level GitHub App, no PATs)
    │
    ▼  repository_dispatch
    │
    └──────────────────────────► sync-plugins.yml
                                    │  updates marketplace.json
                                    │  commits & pushes
                                    ▼
                                 Users run /plugin install
```

- **No PATs or manual tokens** — uses a GitHub App installed on the org
- **Event-driven** — instant updates on tag push, no polling
- **Scalable** — each plugin repo independently triggers updates

## License

MIT
