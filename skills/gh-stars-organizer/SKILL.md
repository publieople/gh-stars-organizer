---
name: gh-stars-organizer
description: Organize starred repos into GitHub Lists with AI reasoning — agent examines each repo individually, never regex rules.
---

# GitHub Stars Organizer

Use AI reasoning to categorize starred repos into GitHub Lists. The agent examines each repo's name, description, and topics to understand what it IS, then assigns the best-fit category.

**CRITICAL: Never use regex rules as primary classifier.** The agent MUST reason about each repo individually. Regex is only acceptable as a fast first-pass bulk pre-sort if the user has 500+ repos and doesn't want to wait.

## Prerequisites

Verify `gh auth status`. If `user` scope missing: `gh auth refresh -h github.com -s user`

## Core Principle: Agent Reasoning Per Repo

For every repo, the agent reads `nameWithOwner`, `description`, `language`, and `topics`, then asks itself: **"What is this project?"** — not "what keywords match?"

- A repo named `anthropics/claude-code` with desc "agentic coding tool" → 🤖 AI-Agents
- A repo named `fatedier/frp` with desc "reverse proxy to expose local server" → 🏠 Self-Hosted  
- A repo named `harry0703/MoneyPrinterTurbo` with desc "一键生成短视频" → ⚡ Automation
- A repo named `torvalds/linux` → 🔧 Dev-Tools (infrastructure, not a library)

## Workflow

### 1. Discovery

Pull existing lists:
```graphql
query { viewer { lists(first: 30) { nodes { id name } } } }
```

### 2. Fetch repos (small batches: 30-50 per batch)

GraphQL pull (from starred list or a specific list):
```graphql
query {
  viewer {
    starredRepositories(first: 50, orderBy: {field: STARRED_AT, direction: DESC}, after: null) {
      pageInfo { hasNextPage endCursor }
      edges {
        node {
          id
          nameWithOwner
          description
          primaryLanguage { name }
          repositoryTopics(first: 5) { nodes { topic { name } } }
        }
      }
    }
  }
}
```

Querying a specific list (MUST use `... on Repository` — items is union type):
```graphql
query {
  node(id: "UL_xxx") {
    ... on UserList {
      items(first: 100, after: null) {
        pageInfo { hasNextPage endCursor }
        edges { node { ... on Repository { id nameWithOwner description primaryLanguage { name } repositoryTopics(first: 5) { nodes { topic { name } } } } } }
      }
    }
  }
}
```

### 3. AI classifies each batch

For each batch of 30-50 repos:
1. Read all repo metadata (name, desc, lang, topics)
2. For each repo, reason about its category — understand what the project IS
3. Assign the best matching existing list, or create a new one
4. Execute mutations immediately (don't batch all to the end)

**Category guide (reason, don't regex-match):**
- 🤖 **AI-Agents**: agent frameworks, MCP, tool-use, agentic systems, AI assistants
- 🌐 **AI-Gateways**: LLM API proxies, model routers, provider aggregators
- 🐟 **AstrBot**: QQ bots, OneBot, NapCat, mirai, QQ group tools
- 🛠️ **Coding-Tools**: CLI tools, editors, formatters, terminal utilities, shell plugins
- 📋 **LLM-Skills**: agent skills, SKILL.md repos, prompt templates, cursorrules
- ⚡ **Automation**: scrapers, automation scripts, n8n, data pipelines, 自动化工具
- 🏠 **Self-Hosted**: Docker compose stacks, self-hosted services, NAS tools, proxy tools (frp)
- 📦 **Libraries-SDKs**: SDKs, API wrappers, npm/pypi packages, language bindings
- 🔧 **Dev-Tools**: Git tools, CI/CD, build systems, devops, debuggers, system tools
- 🎨 **Frontend-UI**: UI component libraries, CSS frameworks, design systems
- 🧠 **AI-ML-Research**: papers, benchmarks, datasets, research code
- 📱 **Apps-Clients**: Desktop/mobile apps, browser extensions, GUI tools, user-facing software
- 💾 **Data-Storage**: databases, cache systems, storage engines
- 🎮 **Game-Dev**: game engines, mods, game development tools
- 📚 **Docs-Learning**: tutorials, awesome lists, documentation, learning resources
- 🎨 **ComfyUI**: ComfyUI nodes and workflows
- 🧠 **BCI-Neuro**: brain-computer interfaces, neuroscience
- 🤖 **LLMs-Models**: model weights, TTS, diffusion models, 3D/video generation, training repos

### 4. Mutations

Create list:
```graphql
mutation { createUserList(input: {name: "🤖 AI-Agents", description: "...", isPrivate: false}) { list { id } } }
```

Assign (one call moves a repo between lists):
```graphql
mutation { updateUserListsForItem(input: {itemId: "R_xxx", listIds: ["UL_xxx"]}) { clientMutationId } }
```

### 5. Review pass (MANDATORY after initial classification)

After all repos are classified, check each list for obvious misclassifications. Particularly:
- Any list with 200+ repos likely has false positives — pull its contents and re-examine 30 at a time
- Lists with single-digit counts may be too granular — consider merging

### 6. Report

```graphql
query { viewer { lists(first: 30) { nodes { name items: items(first: 1) { totalCount } } } } }
```

## Execution Strategy

- **Small batches**: 30-50 repos per AI reasoning batch. Large batches cause context issues and timeout
- **Mutations inline**: assign each repo as you classify it, don't queue all to the end
- **0.15s delay** between mutations to avoid rate limiting
- **Use execute_code**, not delegate_task — subagents time out on this workload
- If context fills mid-classification, pause, report progress, and ask to continue

## Correction

User says "X doesn't belong in Y":
1. Find the repo ID from the list contents
2. `updateUserListsForItem(itemId: "ID", listIds: ["TARGET_LIST_ID"])` — replaces previous assignment

## Troubleshooting

| Problem | Fix |
|---------|-----|
| INSUFFICIENT_SCOPES | `gh auth refresh -h github.com -s user` |
| List too large after pass 1 | Run Pass 2 AI reclassification of that list |
| Subagent timeouts | Use execute_code directly, 30-50 per batch |
| Rate limited | sleep(0.15) between mutations |
| `... on Repository` required | items field is union type, must query via inline fragment |
