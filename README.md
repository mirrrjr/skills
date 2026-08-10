# skills

A personal collection of [Agent Skills](https://zed.dev/docs/ai/skills) —
reusable `SKILL.md` instruction packages for AI coding agents such as
[Claude Code](https://claude.com/claude-code) and [Zed](https://zed.dev).

Each skill lives in its own folder and follows the standard Agent Skills
format: a `SKILL.md` file with YAML frontmatter (`name`, `description`)
plus markdown instructions, optionally paired with a `references/` folder
for supporting material the agent loads on demand.

More skills will be added here over time.

## Skills

| Skill | Description |
|---|---|
| [`en-uz-tech-translate`](./en-uz-tech-translate) | Translates English programming/software-engineering documentation into Uzbek, preserving Markdown structure, code blocks, and technical accuracy. |

<!--## Repository structure

```
.
├── en-uz-tech-translate
│   ├── references
│   │   └── glossary.md
│   └── SKILL.md
├── LICENSE
└── README.md
```

- `SKILL.md` — the skill's instructions and metadata. This is what the
  agent reads to decide when and how to use the skill.
- `references/` — supporting files (e.g. terminology decisions) the
  agent reads only when the skill is active, to keep the base context
  small.-->

## Installing

### Claude Code

Copy the skill folder into one of Claude Code's skill directories:

```bash
# Personal — available in every project
cp -r en-uz-tech-translate ~/.claude/skills/

# Project-local — available only in the current repo
cp -r en-uz-tech-translate .claude/skills/
```

Restart your Claude Code session, then verify it loaded:

```bash
/skills
```

To install straight from this repo:

```bash
git clone https://github.com/mirrrjr/skills.git
cp -r skills/en-uz-tech-translate ~/.claude/skills/
```

### Claude.ai

Zip the skill folder (the zip must contain the folder itself, not just
`SKILL.md`), then go to **Settings → Customize → Skills → Upload**.

```bash
zip -r en-uz-tech-translate.zip en-uz-tech-translate/
```

### Zed

Zed supports Agent Skills natively.

- **Global skill** (available in every project): copy the folder into
  Zed's global skills directory, or use **Settings → AI → Skills → Create
  Skill** and paste the contents in.
- **Project-local skill**: clone this repo into your project, or copy a
  skill folder into your project's skills directory. Zed only loads
  project-local skills from **trusted** worktrees — a freshly cloned
  project won't show its skills in the catalog or slash commands until
  you mark it as trusted.

Once installed, open the **Skills Manager** from the Agent Panel
(`cmd-alt-l` / `ctrl-alt-l`) to confirm the skill is listed, or invoke it
directly with its slash command.

## License

See [LICENSE](./LICENSE).
