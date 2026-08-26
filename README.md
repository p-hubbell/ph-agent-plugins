# ph-agent-plugins

Claude Code plugin marketplace. Users add this catalog, then install individual plugins from it.

## Layout

```
.claude-plugin/marketplace.json   # catalog: name, owner, plugins
plugins/<plugin-name>/            # one directory per in-repo plugin
  .claude-plugin/plugin.json      # plugin manifest (required)
  commands/                       # optional slash commands
  skills/                         # optional skills
  agents/                         # optional agents
  hooks/                          # optional hooks
  .mcp.json                       # optional MCP servers
  README.md
```

Component directories live at the plugin root, never inside `.claude-plugin/`. Only create the ones a plugin actually uses.

Plugin sources in `marketplace.json` are relative to this repo root (for example `./plugins/my-plugin`), not to `.claude-plugin/`.

## Add this marketplace

Local path (no remote required):

```
/plugin marketplace add ~/projects/ph-agent-plugins
```

After the repo is on GitHub:

```
/plugin marketplace add <owner>/ph-agent-plugins
```

Install a listed plugin:

```
/plugin install <plugin-name>@ph-agent-plugins
```

Refresh after catalog changes:

```
/plugin marketplace update ph-agent-plugins
```

## Add a plugin to this catalog

1. Create `plugins/<name>/.claude-plugin/plugin.json` with a kebab-case `name` (required). Add `description` and `author` when you have them. Use `$schema` `https://json.schemastore.org/claude-code-plugin-manifest.json`.
2. Add skills, commands, agents, hooks, or MCP servers at the plugin root using the default directory names above.
3. Append an entry to `.claude-plugin/marketplace.json`:

```json
{
  "name": "my-plugin",
  "source": "./plugins/my-plugin",
  "description": "What it does",
  "author": { "name": "seto" },
  "category": "development"
}
```

`name` is the install slug (`/plugin install my-plugin@ph-agent-plugins`). Do not rename it after people have installed it; use `displayName` for UI labeling, or a top-level `renames` map if a rename is unavoidable.

4. Validate from this repo root:

```
claude plugin validate .
```

An empty `plugins` array is valid; the validator warns until the first plugin is listed.

For scaffolding help, install Anthropic's official toolkit:

```
/plugin install plugin-dev@claude-plugins-official
```

then run `/plugin-dev:create-plugin`.
