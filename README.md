# skills

User-scope skills for [Kimi Code](https://github.com/MoonshotAI/kimi-cli).

Each skill is a directory with a `SKILL.md`: frontmatter `name` +
`description` (the description alone decides when the agent loads the skill),
then the body.

## Skills

- **property-testing** — property-based tests in Go with rapid: what property
  to assert (roundtrips, oracles, metamorphic relations) and how to shape
  rapid's biased generators.

## Installation

```sh
git clone git@github.com:tsdkv/skills.git ~/code/skills
mkdir -p ~/.kimi-code/skills
ln -s ~/code/skills/property-testing ~/.kimi-code/skills/property-testing
```

Or symlink everything at once:

```sh
for s in ~/code/skills/*/; do
  ln -s "$s" ~/.kimi-code/skills/"$(basename "$s")"
done
```

Symlinking (not copying) keeps the installed skills in sync with the repo.
New sessions pick up the skills automatically.
