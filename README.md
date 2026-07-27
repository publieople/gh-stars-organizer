# gh-stars-organizer

> `npx skills add publieople/gh-stars-organizer`

An AI agent skill that organizes your GitHub starred repositories into categorized [GitHub Lists](https://github.com/stars).

## Why?

You've starred 800+ repos. Finding "that one Docker tool" or "that Python library for video editing" is impossible. This skill lets your AI agent read, analyze, and categorize all your stars into browsable Lists — using nothing but `gh` CLI and GitHub's native Lists feature.

## What it does

- Reads your starred repos via GitHub GraphQL API
- AI analyzes each repo's name, description, language, and topics
- Creates categorized Lists (🤖 AI-Agents, 🛠️ Coding-Tools, etc.)
- Assigns repos to the right list
- Handles pagination, rate limits, and context compression

## Install

```bash
npx skills add publieople/gh-stars-organizer
```

Then tell your agent: "Organize my GitHub stars."

## Requirements

- `gh` CLI installed and authenticated
- `gh` token must have `user` scope (run `gh auth refresh -h github.com -s user`)

## Skill structure

```
skills/gh-stars-organizer/
└── SKILL.md          # The skill instructions for AI agents
```

## License

MIT
