# Agent Skills Library

My collection of agent skills for AI assistants.

## Install a Skill

```bash
npx skills add ryanb58/agentskills --skill <skillname>
```

For example: `npx skills add ryanb58/agentskills --skill porkbun`

## Skill Structure

```
skills/
├── skill-name/
│   ├── SKILL.md          # Required: Entry point with frontmatter
│   └── references/       # Optional: Additional docs
│       └── api.md
```

## Available Skills

| Skill | Description | Install |
|-------|-------------|---------|
| [agent-browser](./skills/agent-browser) | Web research automation using agent-browser - headless browser for AI agents with compact text output | `npx skills add ryanb58/agentskills --skill agent-browser` |
| [aws-cost-optimizer](./skills/aws-cost-optimizer) | AWS cost optimization using CLI - identify waste, unattached resources, and savings opportunities | `npx skills add ryanb58/agentskills --skill aws-cost-optimizer` |
| [gh-issues-optimizer](./skills/gh-issues-optimizer) | GitHub Issues optimization using gh CLI - analyze, organize, and improve issue management | `npx skills add ryanb58/agentskills --skill gh-issues-optimizer` |
| [sql-optimization](./skills/sql-optimization) | Expert-level SQL query and data model optimization based on use-the-index-luke.com principles | `npx skills add ryanb58/agentskills --skill sql-optimization` |
| [pgcli-data-modeler](./skills/pgcli-data-modeler) | PostgreSQL database model analysis using pgcli - read-only exploration of schemas, tables, relationships, views, and stored procedures | `npx skills add ryanb58/agentskills --skill pgcli-data-modeler` |
| [porkbun](./skills/porkbun) | Porkbun API interface for DNS, domain registration, SSL, and URL forwarding - uses a locally cached OpenAPI spec with user-approved updates | `npx skills add ryanb58/agentskills --skill porkbun` |

## Contributing

1. Create a new directory under `skills/`
2. Add a `SKILL.md` with proper frontmatter
3. Test with `npx skills add ./skills/your-skill`


## Resources

- [Agent Skills Spec](https://agentskills.io)
- [Skills CLI](https://github.com/vercel-labs/skills)
