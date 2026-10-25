# skills

User-scope agent skills in the SKILL.md format.

Each skill is a directory with a `SKILL.md`: frontmatter `name` +
`description` (the description alone decides when the agent loads the skill),
then the body.

## Skills

- **property-testing** — property-based tests in Go with rapid: what property
  to assert (roundtrips, oracles, metamorphic relations) and how to shape
  rapid's biased generators.

## Installation

Clone, then symlink the skills you want into your agent's skills directory.
Symlinking (not copying) keeps installed skills in sync with the repo.

```sh
git clone git@github.com:tsdkv/skills.git ~/code/skills
```

Kimi Code:

```sh
mkdir -p ~/.kimi-code/skills
ln -s ~/code/skills/property-testing ~/.kimi-code/skills/
```

Claude Code:

```sh
mkdir -p ~/.claude/skills
ln -s ~/code/skills/property-testing ~/.claude/skills/
```

Codex CLI:

```sh
mkdir -p ~/.codex/skills
ln -s ~/code/skills/property-testing ~/.codex/skills/
```

Or symlink every skill into one agent:

```sh
for s in ~/code/skills/*/; do
  ln -s "$s" ~/.kimi-code/skills/"$(basename "$s")"
done
```

New sessions pick up the skills automatically.
