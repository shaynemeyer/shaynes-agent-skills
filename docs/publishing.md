# Publishing this marketplace

How `shaynes-agent-skills` was built and published, so it can be repeated or updated. It covers
packaging agent skills once so that both Claude Code and Kiro can install them from one GitHub
repository.

Repository: `github.com/shaynemeyer/shaynes-agent-skills`. Set up 2026-10-30.

## How the two tools install shared skills

- **Claude Code** has plugin marketplaces. A marketplace is a git repository with a
  `.claude-plugin/marketplace.json` catalog that lists plugins. A plugin is a folder with a
  `.claude-plugin/plugin.json` manifest and a `skills/` folder. Users add the marketplace once and
  then install plugins from it by name.
- **Kiro** has no marketplace for skills. It installs *powers*, and a power can contain only
  skills. A power is a folder with a `plugin.json` manifest (the Agent Plugins spec) and a
  `skills/` folder. The IDE imports a power from a public GitHub URL or a local folder. The CLI
  installs one from a local folder.

The two tools look for their manifest in different places, so one plugin folder can hold both
manifests and share a single `skills/` folder.

## Layout

```
shaynes-agent-skills/
├── .claude-plugin/
│   └── marketplace.json          Claude Code catalog
├── plugins/
│   └── feature-workflow/
│       ├── .claude-plugin/
│       │   └── plugin.json       Claude Code plugin manifest
│       ├── plugin.json           Kiro power manifest
│       └── skills/
│           ├── feature/          SKILL.md, actions/, scripts/, templates/
│           ├── cleanup/SKILL.md
│           └── research/SKILL.md
├── docs/publishing.md            this file
├── README.md                     install steps for users
└── .gitignore                    .DS_Store
```

One plugin holds all three skills, because `cleanup` and `research` read the same `context/`
files that `feature` sets up. Split them into separate plugins if someone needs one without the
others.

## 1. Create the repository

```sh
mkdir -p ~/github/shaynes-agent-skills/.claude-plugin \
         ~/github/shaynes-agent-skills/plugins/feature-workflow/.claude-plugin \
         ~/github/shaynes-agent-skills/plugins/feature-workflow/skills
cd ~/github/mcpherd
tar -cf - --exclude .DS_Store -C .kiro/skills feature cleanup research \
  | tar -xf - -C ~/github/shaynes-agent-skills/plugins/feature-workflow/skills
cd ~/github/shaynes-agent-skills && git init -b main
```

The skills were copied from `mcpherd/.kiro/skills/`, where they were first written. The `tar`
pipe leaves out macOS `.DS_Store` files, and `.gitignore` keeps new ones out.

## 2. Write the manifests

`.claude-plugin/marketplace.json`. The required fields are `name`, `owner` and `plugins`. Add
`description` too, or validation warns. The `name` is what users type after `@`, so pick one
nobody else is likely to use:

```json
{
  "name": "shaynes-agent-skills",
  "description": "Shayne's agent skills for Claude Code and Kiro: ...",
  "owner": { "name": "Shayne Meyer" },
  "plugins": [
    {
      "name": "feature-workflow",
      "source": "./plugins/feature-workflow",
      "description": "Spec-driven feature workflow ..."
    }
  ]
}
```

`plugins/feature-workflow/.claude-plugin/plugin.json`. Only `name` is required, and it must match
the entry name in `marketplace.json`:

```json
{
  "name": "feature-workflow",
  "version": "1.0.0",
  "description": "...",
  "author": { "name": "Shayne Meyer" }
}
```

`plugins/feature-workflow/plugin.json`, for Kiro. `$schema`, `name`, `version`, `description`,
`author` and `keywords` are required. `keywords` are the words that make Kiro activate the power:

```json
{
  "$schema": "https://agent-plugins.org/schemas/1.0.0/plugin.schema.json",
  "name": "feature-workflow",
  "version": "1.0.0",
  "description": "...",
  "author": { "name": "Shayne Meyer" },
  "keywords": ["feature", "spec", "specs", "audit", "parallel", "worktree", "cleanup", "research"]
}
```

Rules that cause most failed installs:

- `source` paths start at the repository root (the folder that holds `.claude-plugin/`) and can't
  contain `..`.
- The entry `name` in `marketplace.json` and the `name` in the Claude Code `plugin.json` must be
  the same.
- Set `version` in `plugin.json` only, not in the `marketplace.json` entry. If both are set,
  Claude Code silently uses the `plugin.json` value.

## 3. Adjust the skills for installed copies

Installed skills live in a plugin or power cache, not in the project. Two edits made that work:

- `skills/feature/SKILL.md` says paths starting `actions/`, `templates/` or `scripts/` are
  relative to the folder the skill was loaded from, which may be outside the project. The agent
  runs `sh <skill-dir>/scripts/spec-fields.sh` from the project root.
- `skills/feature/README.md` mentions installing from the marketplace next to the older symlink
  option.

## 4. Validate

```sh
claude plugin validate .
claude plugin validate plugins/feature-workflow
sh plugins/feature-workflow/skills/feature/scripts/test-spec-fields.sh
```

`validate` checks JSON syntax, required fields, naming rules, `..` in paths, and each plugin's
`plugin.json`. It doesn't check that a `source` folder exists; installing does. Kiro has no
validate command.

## 5. Publish

Create an empty `shaynes-agent-skills` repository on GitHub, then:

```sh
git add .
git commit -m "initial commit"
git remote add origin git@github.com:shaynemeyer/shaynes-agent-skills.git
git push -u origin main
```

Public or private both work for Claude Code. Kiro's "Import power from GitHub" needs a public
repository; for a private one, people install from a clone (see below).

## 6. Install and test

**Claude Code:**

```sh
claude plugin marketplace add shaynemeyer/shaynes-agent-skills
claude plugin install feature-workflow@shaynes-agent-skills
```

The `owner/repo` shorthand clones over SSH when `ssh -T git@github.com` works, and over HTTPS
otherwise. For a private repository the machine's git credentials must work without a prompt:
an SSH key loaded in `ssh-agent`, or `gh auth login` followed by `gh auth setup-git`.

Skills from a plugin carry its name as a prefix: `/feature-workflow:feature`. In a new session,
run `/feature-workflow:feature status` in a project with `context/` files. That shows whether the
skill finds `scripts/spec-fields.sh` in the cache.

Remove it with `claude plugin marketplace remove shaynes-agent-skills`, which also uninstalls its
plugins.

**Kiro:**

- IDE: Powers panel → **Add Custom Power** → **Import power from GitHub**, with the URL of the
  `plugins/feature-workflow` folder (not the repository root).
- CLI: `kiro-cli powers install ~/github/shaynes-agent-skills/plugins/feature-workflow`.

Check that the `feature` skill appears and that Kiro accepts a folder that also holds
`.claude-plugin/`. Remove it with `kiro-cli powers uninstall feature-workflow`.

## 7. Release a change

1. Edit the skills under `plugins/feature-workflow/skills/`, and run the script tests.
2. Bump `version` in both `plugin.json` files. Claude Code users stay on their cached copy until
   the version changes, so a push without a bump reaches nobody.
3. `claude plugin validate .`, then commit and push.

Users get the change when they update. In Claude Code, that's
`claude plugin marketplace update shaynes-agent-skills`, or automatic once they enable auto-update
for the marketplace in `/plugin` (it's off by default). In Kiro, it's **Check for updates** in
the IDE, or pulling the clone and installing again.

## 8. Switch a project to the installed plugin

A project that still has its own copy in `.kiro/skills/` or `.claude/skills/` should drop it once
the plugin is installed. In Kiro a workspace skill with the same name overrides the installed
one, and in Claude Code both copies would be offered. Do the removal on a branch, through a pull
request.

To have Claude Code offer the marketplace to everyone who opens a project, run this in that
project and commit the `.claude/settings.json` it writes:

```sh
claude plugin marketplace add shaynemeyer/shaynes-agent-skills --scope project
```

## 9. Keep the copies in step

The same skills also exist in `mcpherd/.kiro/skills/`, where they were written, and may be kept
in other marketplaces. Make changes here first, release them, and copy them to the others.
Compare before copying:

```sh
diff -r ~/github/shaynes-agent-skills/plugins/feature-workflow/skills \
        <other copy>/plugins/feature-workflow/skills
```

The `README.md` in `skills/feature/` differs on purpose: each names its own marketplace.

## Still to check

- Installing from GitHub in both tools, and running a skill from the installed copy.
- Whether Kiro accepts a power folder that also contains `.claude-plugin/`.

## Sources

- Claude Code: [Create a marketplace](https://code.claude.com/docs/en/plugin-marketplaces),
  [Host and maintain a marketplace](https://code.claude.com/docs/en/plugins/host-marketplace),
  [Marketplace reference](https://code.claude.com/docs/en/plugins/marketplace-reference)
- Kiro: [Agent Skills](https://kiro.dev/docs/skills/),
  [Create powers](https://kiro.dev/docs/powers/create/),
  [Install powers](https://kiro.dev/docs/powers/installation/)
