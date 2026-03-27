# Publishing Guide — ado-pr-reviewer

Step-by-step instructions for publishing this skill to all platforms.

---

## Step 1 — Publish to Your GitHub (do this first)

Everything else depends on this.

1. Go to [github.com/new](https://github.com/new)
2. Repository name: `ado-pr-reviewer`
3. Description: `AI-powered Azure DevOps PR reviewer skill for Claude`
4. Set to **Public**
5. Do NOT initialise with README (you already have one)
6. Click **Create repository**

Then push the files:

```bash
cd /path/to/ado-pr-reviewer
git init
git add .
git commit -m "Initial release: ado-pr-reviewer v1.0.0"
git branch -M main
git remote add origin https://github.com/<your-username>/ado-pr-reviewer.git
git push -u origin main
```

---

## Step 2 — Submit to SkillsMP (automatic after GitHub)

SkillsMP crawls GitHub automatically. To speed it up:

1. Go to [skillsmp.com/submit](https://skillsmp.com/submit) (or search for a submit form on their site)
2. Enter your GitHub repo URL: `https://github.com/<your-username>/ado-pr-reviewer`
3. Add tags: `azure-devops`, `pull-request`, `code-review`, `ado`, `mcp`
4. Submit

Your skill will appear in search results once indexed. Getting a few GitHub stars will boost its ranking.

---

## Step 3 — Submit to anthropics/skills

This is the official Anthropic skills repository and gives the most visibility.

1. Fork [github.com/anthropics/skills](https://github.com/anthropics/skills)

2. Create a folder in the `skills/` directory:
```
skills/
└── ado-pr-reviewer/
    ├── SKILL.md
    └── README.md
```

3. Copy your `SKILL.md` and `README.md` into that folder

4. Open a Pull Request to `anthropics/skills` with:
   - **Title**: `feat: add ado-pr-reviewer — Azure DevOps PR review skill`
   - **Description**: Use the template below

### PR Description Template for anthropics/skills

```markdown
## Summary

Adds `ado-pr-reviewer` — the first Azure DevOps PR review skill for Claude.

## What it does

- Connects to Azure DevOps via the Microsoft ADO MCP server
- Reviews open pull requests and posts inline comments at the exact file/line
- Checks for security, performance, and logic issues
- Verifies linked work items are correctly implemented
- Votes on PRs (WaitingForAuthor / ApprovedWithSuggestions / Approved)
- Asks for user confirmation before casting any vote

## Why this is unique

All existing PR review skills in this repo target GitHub. This is the first skill 
that targets Azure DevOps — filling a genuine gap for enterprise teams using ADO.

## Compatibility

- Claude Desktop (required)
- Azure DevOps MCP server (@azure-devops/mcp)
- Node.js 20+
- Any Azure DevOps organization (cloud or on-premises)

## Testing

Tested on Azure DevOps org with C# / .NET backend repository.
Trigger phrase: "review my open PRs"
```

---

## Checklist Before Publishing

- [ ] `SKILL.md` frontmatter has `name: ado-pr-reviewer`
- [ ] `SKILL.md` description is clear and includes trigger phrases
- [ ] `README.md` has install instructions and usage examples
- [ ] `LICENSE` is Apache 2.0
- [ ] `plugin.json` has correct repo URL (update `<your-username>`)
- [ ] All references to specific org names (`dttmoj`, `Voluntary-Enforcement`) are removed
- [ ] Setup placeholders (`<YOUR_ORG>`, `<YOUR_PROJECT>`) are clearly marked
- [ ] Tested end-to-end on a real PR before publishing

---

## After Publishing

- Share the GitHub link with your team
- They install with: `/plugin install ado-pr-reviewer@<your-username>`
- Or manually: copy `SKILL.md` to `~/.claude/skills/ado-pr-reviewer/`
- Star your own repo to kickstart visibility on SkillsMP
