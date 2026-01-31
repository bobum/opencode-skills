# OpenCode Skills

Reusable skills and agent prompts for AI-assisted development. These skills work with [OpenCode](https://github.com/opencode-ai/opencode), Claude Code, and other AI coding assistants that support skill files.

## What are Skills?

Skills are SKILL.md files that provide domain-specific knowledge and patterns to AI coding assistants. They help the AI understand your project's conventions, best practices, and workflows.

## Available Skills

### Git & GitHub Workflow

| Skill | Description |
|-------|-------------|
| `branch-cleanup` | Post-merge cleanup workflow for branches |
| `issue-management` | GitHub issue hierarchy (Initiative → Epic → Issue → PR) |
| `pr-workflow` | Standardized PR creation with proper issue linking |
| `doc-sync` | Keep documentation in sync with code changes |

### React & Frontend

| Skill | Description |
|-------|-------------|
| `react-best-practices` | Performance optimization patterns from Vercel Engineering |
| `react-patterns` | Modern React patterns: hooks, composition, TypeScript |
| `react-state-management` | Redux Toolkit, Zustand, Jotai, React Query patterns |
| `composition-patterns` | React composition patterns that scale |
| `component-testing` | Vitest patterns for testing React components |
| `e2e-testing` | Playwright patterns for end-to-end testing |
| `e2e-testing-patterns` | Comprehensive Playwright/Cypress testing patterns |

### .NET & Backend

| Skill | Description |
|-------|-------------|
| `dotnet-architect` | Expert .NET patterns: ASP.NET Core, EF Core, Dapper, caching |
| `ef-migrations` | EF Core migration patterns and best practices |
| `authorization` | JWT/Azure AD B2C authorization patterns |
| `integration-testing` | SQLite in-memory integration testing patterns |
| `cqrs-implementation` | Command Query Responsibility Segregation patterns |

### API & Database

| Skill | Description |
|-------|-------------|
| `api-design-principles` | REST and GraphQL API design best practices |
| `mock-json-api` | Stateless mock API patterns with scenario-based testing |
| `postgres-best-practices` | Postgres optimization from Supabase (indexing, RLS, queries) |
| `sql-optimization-patterns` | Query optimization, EXPLAIN analysis, indexing |

### TypeScript & Security

| Skill | Description |
|-------|-------------|
| `typescript-advanced-types` | Generics, conditional types, mapped types, utility types |
| `owasp-security` | OWASP Top 10 prevention patterns with code examples |

### Agent Configuration

| Skill | Description |
|-------|-------------|
| `honest-agent` | Configure AI agents for honest, objective feedback |

## Installation

### Option 1: Copy individual skills

Copy the skill folders you need into your project's `.opencode/skills/` directory:

```bash
cp -r path/to/opencode-skills/.opencode/skills/react-best-practices your-project/.opencode/skills/
```

### Option 2: Clone the entire repo

```bash
git clone https://github.com/bobum/opencode-skills.git
```

Then symlink or copy the skills you need.

## Usage

Once skills are in your project's `.opencode/skills/` directory, AI assistants that support skill files will automatically use them when relevant.

For example, when writing React components, the `react-best-practices` skill will guide the AI to use performance-optimized patterns.

## Skill Structure

Each skill is a directory containing a `SKILL.md` file:

```
.opencode/skills/
├── react-best-practices/
│   └── SKILL.md
├── ef-migrations/
│   └── SKILL.md
└── mock-json-api/
    └── SKILL.md
```

## Attribution

Skills are adapted from:
- [vercel-labs/agent-skills](https://github.com/vercel-labs/agent-skills) - React best practices and composition patterns (MIT License)
- [sickn33/antigravity-awesome-skills](https://github.com/sickn33/antigravity-awesome-skills) - .NET, TypeScript, API, SQL, CQRS patterns
- [hoodini/ai-agents-skills](https://github.com/hoodini/ai-agents-skills) - OWASP security, honest-agent configuration
- [Supabase](https://supabase.com) - Postgres best practices

## License

MIT
