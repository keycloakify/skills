# Keycloakify Agent Skills

[Agent Skills](https://agentskills.io) that teach AI coding agents (Claude Code, Cursor, Codex, Copilot, Windsurf and others) how to work with the Keycloakify ecosystem.

| Skill | What it covers |
|---|---|
| [`keycloakify`](skills/keycloakify/SKILL.md) | Building, customizing, testing and deploying Keycloak themes (login, account, email) with React, Angular or Svelte |

## Install

Install all skills:

```bash
npx skills add https://github.com/keycloakify/skills
```

Install one skill:

```bash
npx skills add keycloakify/skills --skill keycloakify
```

Install for a specific agent, globally:

```bash
npx skills add keycloakify/skills --skill keycloakify -a claude-code -g
```

List the available skills without installing:

```bash
npx skills add keycloakify/skills --list
```

## Repository layout

```
skills/
  <skill-name>/
    SKILL.md          # frontmatter (name, description) + core instructions
    references/*.md   # detailed docs, loaded by the agent only when needed
```

## Contributing

- Keep each `SKILL.md` under about 500 lines. Move details into `references/`.
- The frontmatter `name` must match the folder name. Use lowercase letters, digits and hyphens only.
- The `description` drives when an agent loads the skill. Say what the skill does and when to use it, and include the keywords users will type.
- Check facts against the official docs ([docs.keycloakify.dev](https://docs.keycloakify.dev)).

## License

MIT
