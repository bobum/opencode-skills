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
| `composition-patterns` | React composition patterns that scale |
| `component-testing` | Vitest patterns for testing React components |
| `e2e-testing` | Playwright patterns for end-to-end testing |

### .NET & Backend

| Skill | Description |
|-------|-------------|
| `ef-migrations` | EF Core migration patterns and best practices |
| `authorization` | JWT/Azure AD B2C authorization patterns |
| `integration-testing` | SQLite in-memory integration testing patterns |

### API & Testing

| Skill | Description |
|-------|-------------|
| `mock-json-api` | Stateless mock API patterns with scenario-based testing |

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

Some skills are adapted from:
- [vercel-labs/agent-skills](https://github.com/vercel-labs/agent-skills) - React best practices and composition patterns (MIT License)

## License

MIT
