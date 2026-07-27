---
name: gh-stars-organizer
description: Organize GitHub starred repositories into categorized Lists using AI. The agent reads your stars, analyzes them, and creates GitHub Lists with smart categorization — turning 800+ unorganized stars into browsable, searchable collections.
---

# GitHub Stars Organizer

Use AI to automatically organize your GitHub starred repositories into categorized [GitHub Lists](https://github.com/stars) — making 800+ unorganized stars browsable and searchable in minutes.

## When to Use

- "I have too many starred repos and can't find anything"
- "Organize my GitHub stars"
- "Categorize my starred repositories"
- "Clean up my GitHub stars into lists"

## Prerequisites

This skill uses `gh` CLI + GraphQL. The agent should verify:

```
gh auth status
```

If the user's token lacks the `user` scope (needed for List management), guide them:

```
gh auth refresh -h github.com -s user
```

## Core Workflow

### Step 1: Discover current Lists

```graphql
query {
  viewer {
    lists(first: 20) {
      nodes { id name description }
    }
  }
}
```

### Step 2: Fetch starred repos (100 per batch)

```graphql
query($cursor: String) {
  viewer {
    starredRepositories(first: 100, orderBy: {field: STARRED_AT, direction: DESC}, after: $cursor) {
      pageInfo { hasNextPage endCursor }
      edges {
        node {
          id
          nameWithOwner
          description
          primaryLanguage { name }
          repositoryTopics(first: 5) { nodes { topic { name } } }
          stargazerCount
        }
      }
    }
  }
}
```

Run via: `gh api graphql -f query='...'`

### Step 3: AI analyzes each batch

For each repository, the agent reads `nameWithOwner`, `description`, `primaryLanguage`, and `repositoryTopics` to decide the best category.

**Classification guidelines:**
- Create 12-18 lists max — too many becomes noise
- Categorize by **what the repo is** (not what language it's written in)
- Prefer existing lists when they fit; create new ones sparingly
- A repo can be in ONE list (GitHub limitation)
- Common categories: AI-Agents, Coding-Tools, Automation, Dev-Tools, Libraries, Apps, Docs, Game-Dev, Frontend-UI

**Context compression reminder:** After each batch of 100 repos, prompt: "Batch N done. Use /compact or /new before the next batch to avoid context overflow."

### Step 4: Create Lists

```graphql
mutation {
  createUserList(input: {
    name: "🤖 AI-Agents",
    description: "Agent frameworks, multi-agent systems, orchestration",
    isPrivate: false
  }) {
    list { id name }
  }
}
```

### Step 5: Assign repos to Lists

```graphql
mutation {
  updateUserListsForItem(input: {
    itemId: "R_kgDOOIt0fw",
    listIds: ["UL_kwDOBV8csM4AhGre"]
  }) {
    clientMutationId
  }
}
```

Rate limit: ~3 requests/second. 100 repos ≈ 35 seconds.

### Step 6: Report results

After classification, show the user a summary:

```
🤖 AI-Agents:     22 repos
🛠️ Coding-Tools:  31 repos
📦 Libraries:     69 repos
...
Total: 200 repos in 14 lists
View at: https://github.com/<username>?tab=stars
```

## Correction Mechanism

If the user says a repo is in the wrong list:

```
The user wants to move <repo-name> out of <current-list> into <target-list>.

1. Remove: updateUserListsForItem(itemId: "<id>", listIds: [<target_list_id>])
2. This REPLACES the assignment — GitHub only allows one list per item
```

## Deleting Lists

```graphql
mutation {
  deleteUserList(input: { listId: "UL_xxx" }) { clientMutationId }
}
```

## Cron Automation (optional)

For agents that support cron (Hermes, ClawdBot), schedule daily updates:

```bash
# Run every day at 3 AM
cron: "0 3 * * *"
prompt: "Fetch newly starred repos (since last run) and categorize them into existing Lists. Only process repos starred in the last 24 hours."
```

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `INSUFFICIENT_SCOPES` error | Run `gh auth refresh -h github.com -s user` |
| `gh` not installed | `brew install gh` / `winget install GitHub.cli` / `apt install gh` |
| Rate limited | Add 500ms delay between mutations |
| `You have already starred this repository` | Normal — no action needed |
