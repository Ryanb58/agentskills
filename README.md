# Agent Skills Library

My collection of agent skills for AI assistants.

## Install a Skill

```bash
npx skills add <skill-path>
```

### Install Individual Skills

```bash
# Install a specific skill
npx skills add ./skills/my-skill

# Install from GitHub
npx skills add https://github.com/your-org/agentskills/tree/main/skills/skill-name

# List available skills without installing
npx skills add . --list

# Install all skills
npx skills add . --all
```

## Skill Structure

```
skills/
├── skill-name/
│   ├── SKILL.md          # Required: Entry point with frontmatter
│   └── references/       # Optional: Additional docs
│       └── api.md
```

## Contributing

1. Create a new directory under `skills/`
2. Add a `SKILL.md` with proper frontmatter
3. Test with `npx skills add ./skills/your-skill`

## Resources

- [Agent Skills Spec](https://agentskills.io)
- [Skills CLI](https://github.com/vercel-labs/skills)
