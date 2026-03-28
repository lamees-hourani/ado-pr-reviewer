# ado-pr-reviewer

> An Agent Skill for Claude that automatically reviews Azure DevOps pull requests — posting inline comments, verifying work items, checking for security and logic issues, and voting on PRs.

![Claude Skill](https://img.shields.io/badge/Claude-Agent%20Skill-blue)
![Azure DevOps](https://img.shields.io/badge/Azure%20DevOps-MCP-0078D7?logo=azure-devops)
![License](https://img.shields.io/badge/License-Apache%202.0-green)

---

## ✨ What it does

Once installed, just say **"review my open PRs"** in Claude Desktop and it will:

1. 🔍 **Fetch open PRs** from your Azure DevOps project via the ADO MCP server
2. 📋 **Verify linked work items** — checks that all acceptance criteria are implemented
3. 🔴 **Flag security issues** — hardcoded secrets, missing auth, sensitive data in logs
4. 🟡 **Flag performance issues** — N+1 queries, unbounded lists, blocking async calls
5. 🔴 **Flag logic errors** — null refs, silent catch blocks, wrong enums, async void
6. ✏️ **Post inline comments** anchored to the exact file and line — just like a human reviewer
7. 🗳️ **Ask for your approval** before casting a vote (WaitingForAuthor / ApprovedWithSuggestions / Approved)

Every comment uses a clean two-part format:

```
don't use `var`, use static type instead

💡 `List<Claim> results = new(...);` instead of `var results = new List<Claim>(...);`
```

---

## 🚀 Quick Start

### Prerequisites

| Requirement | Version | Link |
|-------------|---------|------|
| Claude Desktop | Latest | [claude.ai/download](https://claude.ai/download) |
| Node.js | 20+ | [nodejs.org](https://nodejs.org) |
| Azure DevOps MCP | Latest | [@azure-devops/mcp](https://github.com/microsoft/azure-devops-mcp) |

### Step 1 — Configure the Azure DevOps MCP server

Open `%APPDATA%\Claude\claude_desktop_config.json` (Windows) or `~/Library/Application Support/Claude/claude_desktop_config.json` (Mac) and add:

```json
{
  "mcpServers": {
    "ado": {
      "command": "npx",
      "args": ["-y", "@azure-devops/mcp", "<your-org>"]
    }
  }
}
```

Replace `<your-org>` with your Azure DevOps organization name (found in your URL: `dev.azure.com/<your-org>`).

### Step 2 — Install the skill

**Via Claude Code plugin (recommended):**
```bash
/plugin install ado-pr-reviewer@lamees-hourani
```

**Manual install:**
```bash
# Clone and copy to your skills folder
git clone https://github.com/lamees-hourani/ado-pr-reviewer
cp -r ado-pr-reviewer ~/.claude/skills/ado-pr-reviewer
```

### Step 3 — Configure your project

Fill in the setup block at the top of `SKILL.md` with your org, project, and repo details. This takes 2 minutes and only needs to be done once.

### Step 4 — Restart Claude Desktop and test

```
list my Azure DevOps projects
```

A browser will open asking you to sign in with your Microsoft account. After that, you're ready.

---

## 💬 Usage

| What you say | What happens |
|-------------|-------------|
| `review my open PRs` | Reviews all active PRs in your project |
| `review PR #123` | Reviews a specific PR by number |
| `YES` / `confirm` | Casts the recommended vote after review |
| `change it to Approved` | Casts Approved instead |
| `don't vote` | Skips voting for this review |

---

## 📋 What gets checked

### 🔴 Security
- Hardcoded secrets, passwords, API keys in code
- Sensitive data (IDs, tokens, PII) in logs
- Missing authorization checks on endpoints
- Unvalidated external API input
- `.env` files or credentials committed to the repo

### 🟡 Performance
- N+1 query patterns
- Filtering after loading all records into memory
- Blocking synchronous calls inside async methods
- Unbounded lists returned from endpoints

### 🔴 Logic & Correctness
- Null references after `.FirstOrDefault()` without null checks
- Silent `catch` blocks that swallow exceptions
- `async void` methods
- Dead code paths
- Off-by-one errors in comparisons

### Style & Quality
- File-scoped namespaces, no `var`, blank lines, typos
- `record` types for DTOs, `is not` instead of `!=`
- Collection expressions `[]`, `CancellationToken` passing
- Constructor order, unused usings

### Work Item Verification
- PR linked to a work item?
- Code implements all acceptance criteria?
- Any scope creep unrelated to the work item?

---

## ⚙️ Setup Reference

The `SKILL.md` file has a setup block at the top. Fill in these values:

| Field | Example |
|-------|---------|
| Org name | `contoso` |
| Project name | `MyProject` |
| Repo name(s) | `MyProject Backend` |
| Repo GUID(s) | Found in ADO: Repo Settings → Properties |
| Main branches | `main`, `develop`, `staging` |
| Language/stack | `C# / .NET 8`, `TypeScript / Node` |

---

## 🌐 Compatible Platforms

This skill works with **Azure DevOps** (cloud and on-premises) via the [Microsoft Azure DevOps MCP server](https://github.com/microsoft/azure-devops-mcp). It does **not** work with GitHub — for GitHub PRs, see [aidankinzett/claude-git-pr-skill](https://github.com/aidankinzett/claude-git-pr-skill).

---

## 📄 License

Apache 2.0 — see [LICENSE](./LICENSE)

---

## 🤝 Contributing

PRs welcome! If you add support for new language rules, security checks, or ADO-specific patterns, please open a pull request.

Issues and feature requests: [github.com/lamees-hourani/ado-pr-reviewer/issues](https://github.com/lamees-hourani/ado-pr-reviewer/issues)
