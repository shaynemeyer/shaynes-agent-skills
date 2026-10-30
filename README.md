# shaynes-agent-skills

Agent skills for Claude Code and Kiro, packaged as a Claude Code plugin marketplace and a Kiro power
from the same files.

## Plugins

| Plugin | Skills | What it does |
| --- | --- | --- |
| `feature-workflow` | `feature`, `cleanup`, `research` | Spec-driven feature workflow tracked in `context/`: set up the context files, split a plan into specs, audit them, load and implement one, or run up to 3 independent specs in parallel in git worktrees, then review, test, explain, complete or abandon. `cleanup` checks project housekeeping; `research` runs named research prompts from `context/research/`. |

See `plugins/feature-workflow/skills/feature/README.md` for the full workflow.

## Install in Claude Code

```sh
claude plugin marketplace add shaynemeyer/shaynes-agent-skills
claude plugin install feature-workflow@shaynes-agent-skills
```

Or inside a session: `/plugin marketplace add shaynemeyer/shaynes-agent-skills`, then `/plugin`. The
repository can be private; anyone with git access to it can add it.

Plugin skills carry the plugin name as a prefix: `/feature-workflow:feature`,
`/feature-workflow:cleanup`, `/feature-workflow:research`. Asking for a skill by what it does
("audit the specs", "run 03 and 04 in parallel") works too.

To get updates: `claude plugin marketplace update shaynes-agent-skills`.

## Install in Kiro

Kiro installs this plugin as a power. It contains only skills, no MCP servers.

- **IDE:** Powers panel → **Add Custom Power** → **Import power from GitHub**, with the URL of the
  `plugins/feature-workflow` folder (public repositories only). For a private repository, clone it
  and use **Import power from a folder** with `plugins/feature-workflow`.
- **CLI:** clone the repository, then run
  `kiro-cli powers install <clone>/plugins/feature-workflow`, or `/powers install <path>` in chat.

To update, pull the clone and install again, or use **Check for updates** in the IDE.

Kiro also loads skills copied straight into a project's `.kiro/skills/` or into
`~/.kiro/skills/`. Copy the folders under `plugins/feature-workflow/skills/` there if you prefer
that over a power.

## One copy per project

If a project also has its own copy of these skills in `.kiro/skills/` or `.claude/skills/`,
remove it after installing the plugin. In Kiro a workspace skill with the same name overrides the
installed one, and in Claude Code both copies would be offered.

## Releasing a change

1. Edit the skills under `plugins/feature-workflow/skills/`.
2. Run the script tests: `sh plugins/feature-workflow/skills/feature/scripts/test-spec-fields.sh`.
3. Bump `version` in **both** `plugins/feature-workflow/.claude-plugin/plugin.json` (Claude Code)
   and `plugins/feature-workflow/plugin.json` (Kiro), so both tools see an update.
4. Validate: `claude plugin validate .`
5. Commit and push.

## Layout

```
.claude-plugin/marketplace.json            Claude Code marketplace catalog
plugins/feature-workflow/
  .claude-plugin/plugin.json               Claude Code plugin manifest
  plugin.json                              Kiro power manifest (Agent Plugins spec)
  skills/feature/                          SKILL.md, actions/, scripts/, templates/
  skills/cleanup/SKILL.md
  skills/research/SKILL.md
```
