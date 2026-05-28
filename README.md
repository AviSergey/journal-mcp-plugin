# journal-mcp-plugin

A [Claude Code plugin](https://code.claude.com/docs/en/plugins.md) that
bundles the [journal-mcp](https://github.com/avisergey/journal-mcp)
server together with a small `recent-digest` skill — a minimal tutorial
example of how plugins package an MCP server + a custom skill.

## What's inside

```
journal-mcp-plugin/
├── .claude-plugin/
│   ├── plugin.json              ← plugin manifest
│   └── marketplace.json         ← single-plugin marketplace catalog (lets users install from Git)
├── .mcp.json                    ← how to spawn the journal MCP server
├── skills/
│   └── recent-digest/
│       └── SKILL.md             ← auto-triggered skill body + frontmatter
└── README.md
```

Why both `plugin.json` and `marketplace.json` live in the same `.claude-plugin/`:
Claude Code can only Git-install plugins **through a marketplace catalog**. The
simplest way to ship a single-plugin repo is to make the same repo serve as a
one-entry marketplace — that's exactly what `marketplace.json` does. Its single
entry has `"source": "./"`, pointing back at this same directory where
`plugin.json` lives.

This plugin demonstrates two things:

1. **Bundling an existing stdio MCP server** via `.mcp.json` — the server
   itself lives in a separate repo (`journal-mcp`) and is pulled in here
   only by reference.
2. **A skill that auto-triggers by description** — when the user asks
   to "summarize my recent notes", "review my notes", and similar
   phrasings, Claude Code loads the skill body and follows its
   instructions, orchestrating MCP resource reads under the hood.

## What's in the plugin

### MCP server — `journal`

Personal learning journal stored in SQLite + FTS5. Exposes:

- **Tools**: `add_note`, `search_notes`, `delete_note`, `ask_my_notes`
- **Resources**: `notes://recent`, `notes://by-tag/{tag}`, `notes://{id}` (paginated `list`)
- **Prompts**: `daily_review`, `quiz_me` (with elicitation)
- **Completions** for tag/topic, **tool annotations**, **structured output** on all tools

Full feature list and source: see the [`journal-mcp` README](https://github.com/avisergey/journal-mcp).

### Skill — `/journal-mcp-plugin:recent-digest`

Auto-triggers on phrasings like:

- "summarize my recent notes"
- "what have I been journaling about lately"
- "show me my latest entries"
- "review my notes"

Reads `notes://recent` (10 most-recent notes, 150-char previews), groups
them by tag, renders a structured markdown digest plus one reflection
prompt. Deliberately does **not** frame this as a "weekly" digest — the
resource carries no timestamps, so any time-window claim would be made up.
Read-only — never writes or deletes notes.

## Install

### Option A — Bundled MCP server from TestPyPI (no local checkout needed)

This is what `.mcp.json` ships with by default. `uvx` will fetch the
`journal-mcp-demo-avisergey` package from TestPyPI on first run and
cache it.

Install via the bundled marketplace (works straight from this Git repo):

```
# Inside Claude Code:
/plugin marketplace add avisergey/journal-mcp-plugin
/plugin install journal-mcp-plugin@journal-mcp-plugin
/reload-plugins
```

The first command registers this repo as a marketplace (Claude Code reads
`.claude-plugin/marketplace.json`). The second installs the plugin listed
inside that marketplace — the `name@marketplace` form is required; the
form `/plugin install github.com/...` does **not** exist.

For local development of the plugin itself (no Git, no marketplace):

```bash
claude --plugin-dir /absolute/path/to/journal-mcp-plugin
```

No extra MCP setup needed either way — the journal server is auto-installed
by `uvx` on first invocation.

### Option B — Local checkout of the MCP server (for hacking on the server)

If you've cloned `journal-mcp` locally and want the plugin to use
**your local copy** (so server-side changes are picked up on each
restart), edit `.mcp.json` to point at the source tree:

```json
{
  "journal": {
    "type": "stdio",
    "command": "uv",
    "args": [
      "run",
      "--project", "/absolute/path/to/journal-mcp",
      "python", "-m", "journal_mcp"
    ]
  }
}
```

Then install / reload the plugin the same way:

```bash
claude --plugin-dir /absolute/path/to/journal-mcp-plugin
# Inside Claude Code:
/reload-plugins
```

Use this mode when you're iterating on tools, resources, or prompts in
the MCP server itself.

## Verify it works

After installing, inside Claude Code:

```
/mcp                                       # journal should show "connected"
@notes://recent                            # static resource read
save a note "test from plugin"             # tool call: add_note
summarize my recent notes                  # should auto-fire the skill
```

The last one is the interesting check: if the skill auto-triggers, you
should see Claude reading `notes://recent` and producing a grouped
markdown digest with a reflection prompt at the end. If it doesn't fire,
the skill description likely needs a phrasing closer to what you typed —
edit `skills/recent-digest/SKILL.md` frontmatter and `/reload-plugins`.

## Avoid double-registering the MCP server

If you already have a `journal` MCP server registered globally (e.g. you ran
`claude mcp add ... journal ...` while building it), the plugin's bundled
`.mcp.json` will conflict — both registrations show up in `/mcp` and you can't
tell which one your calls hit. Remove the standalone registration before
installing the plugin:

```bash
claude mcp list                            # see all registered servers
claude mcp remove journal                  # drop the user-/project-scope one
```

Then start Claude Code and install the plugin (see Option A above for the
full sequence):

```
/plugin marketplace add avisergey/journal-mcp-plugin
/plugin install journal-mcp-plugin@journal-mcp-plugin
/mcp                                       # only the plugin's `journal` remains
```

If `claude mcp remove journal` reports nothing was removed, the entry may live
in a different scope; try `claude mcp remove --scope user journal` or
`--scope project journal`. You can also inspect/edit the JSON directly in
`~/.claude.json` (user scope) or the project's `.mcp.json`.

## What this plugin teaches (the meta-point)

| Concept | Where to look |
|---|---|
| Plugin manifest (required fields, namespacing) | [`.claude-plugin/plugin.json`](.claude-plugin/plugin.json) |
| **Marketplace catalog** living in the same repo as the plugin (single-plugin marketplace pattern) | [`.claude-plugin/marketplace.json`](.claude-plugin/marketplace.json) |
| Bundling a stdio MCP server | [`.mcp.json`](.mcp.json) |
| Skill anatomy: frontmatter + markdown body | [`skills/recent-digest/SKILL.md`](skills/recent-digest/SKILL.md) |
| Skill **auto-trigger** by `description` (not slash invocation) | the `description` field in SKILL.md frontmatter |
| Skill **namespace** via plugin name | `/journal-mcp-plugin:recent-digest` |
| Skill that **orchestrates MCP primitives** without server changes | the steps in SKILL.md body |
| Two install paths: published package vs local source | this README, [`.mcp.json`](.mcp.json) |

## Distribution

- **Personal / team:** push to a Git repo, then users run
  `/plugin marketplace add <user>/journal-mcp-plugin` followed by
  `/plugin install journal-mcp-plugin@journal-mcp-plugin`.
- **Local dev:** `claude --plugin-dir /absolute/path/...` and reload with
  `/reload-plugins`. No marketplace registration needed for this path.
- **Community marketplace:** validate via `claude plugin validate`, then
  submit at [platform.claude.com/plugins/submit](https://platform.claude.com/plugins/submit).

## License

MIT.
