# AI Coding Agent Skills

A collection of reusable, practical, and tool-compatible skills for AI coding agents.

The goal of this repository is to provide engineering rules, patterns, conventions, and best practices that can be consistently applied by different AI-assisted development tools.

These skills are designed to help AI agents produce code that is:

- Clean and maintainable
- Consistent and predictable
- Testable
- Production-oriented
- Easy to review and evolve
- Less prone to common architectural and implementation mistakes

## Supported AI Coding Agents

The skills are designed to be compatible with multiple AI coding environments, including:

- OpenCode
- Claude Code
- Cursor
- GitHub Copilot
- Other AI coding agents that support project-level instructions, rules, or skills

The engineering knowledge is intentionally kept independent from any specific AI tool.

Tool-specific configuration should reference the same skill instead of duplicating its engineering rules.

## Repository Structure

Each skill is stored in its own directory:

```text
skills/
├── nestjs-clean-code-solid/
│   ├── SKILL.md
│   ├── README.md
│   └── ...
│
├── another-skill/
│   ├── SKILL.md
│   └── ...
│
└── ...