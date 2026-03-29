---
name: ado-pr-reviewer
description: >
  Performs a full AI code review on open pull requests in any Azure DevOps project,
  posting inline line-level comments and a work item verification summary directly into the PR.
  Use this skill whenever the user says "review my PR", "review all open PRs", "check the PR",
  or asks Claude to look at code changes in Azure DevOps. Works with any org, project, language,
  or team after a one-time setup. Always use this skill for any PR review request, even if the
  user just says "review it".
---

# Generic PR Reviewer Skill

## ⚙️ ONE-TIME SETUP (required before first use)

Before using this skill, the team lead or admin must fill in the configuration below.
Replace every `<PLACEHOLDER>` with real values. Delete this setup section when done.

```
PROJECT CONTEXT
───────────────
Org name:          <YOUR_ORG>              e.g. contoso
Project name:      <YOUR_PROJECT>          e.g. MyProject
Repo name(s):      <YOUR_REPO_NAME>        e.g. MyProject Backend
Repo GUID(s):      <YOUR_REPO_GUID>        e.g. a1ebb735-050f-4741-82be-d260341748ff
Main branches:     <YOUR_BRANCHES>         e.g. main, develop, staging
Language/stack:    <YOUR_STACK>            e.g. C# / .NET 8, TypeScript / Node, Java / Spring

TEAM
────
Team members:      <NAMES AND ROLES>       e.g. Alice (backend), Bob (frontend)
Lead reviewer:     <LEAD NAME & EMAIL>     e.g. Ahmed (ahmed@company.com)
Review culture:    <DESCRIBE TONE>         e.g. Short and direct, use backticks for code names,
                                               use ==> for renames, no preamble

LANGUAGE-SPECIFIC RULES
────────────────────────
Add your team's code style rules below in the "Language Rules Checklist" section.
Examples are provided for C# — replace or extend them for your stack.
```

---

## Overview

This skill reviews pull requests in any Azure DevOps project. It:
1. Fetches open PRs and reads actual file contents
2. Verifies linked work items are correctly implemented
3. Checks code against security, performance, logic, and style rules
4. Posts precise inline comments anchored to the exact file and line
5. Votes **Waiting for Author** when issues are found

---

## Step-by-Step Workflow

### 1. Discover Project Context (first time only)

If project/repo details are not yet filled in above, discover them automatically:

```
ado:core_list_projects
```

Then for each project:
```
ado:repo_list_repos_by_project
  project: <project name>
```

Use the returned names and GUIDs to fill in the setup section above.

---

### 2. Fetch PRs Assigned to You as Reviewer

Only fetch PRs where the current user is a reviewer:

```
ado:repo_list_pull_requests_by_repo_or_project
  project: <YOUR_PROJECT>
  status: Active
  i_am_reviewer: true
```

If no PRs are returned, inform the user:
```
No open PRs are currently assigned to you as a reviewer.
```

If the user specifies a PR number directly, skip to step 2.5.

### 2.5. Pre-Flight Checks (run before reviewing each PR)

For each PR returned, do ALL of the following checks before proceeding:

#### A — Already reviewed by you?

Fetch existing threads on the PR:

```
ado:repo_list_pull_request_threads
  pullRequestId: <id>
  repositoryId: <repo GUID>
  project: <YOUR_PROJECT>
```

Scan the threads for any comments already posted by the current user (match by author display name or email).

**If you have already posted comments on this PR**, show the user:
```
⚠️ PR #<id> — "<title>"
You have already reviewed this PR (found <N> existing comment threads from you).

Options:
  A) Re-review — I will skip any lines that already have your comments
  B) Skip this PR — move to the next one
  C) Review anyway — post all comments including potential duplicates

Reply A, B, or C.
```
Wait for user's choice before continuing.

#### B — Duplicate line check (applied during step 6)

Before posting ANY comment, load the existing threads (fetched above) and build a **"already commented" index**:

```
alreadyCommented = Set of (filePath + ":" + lineNumber)
  for each existing thread where author == currentUser
    add thread.filePath + ":" + thread.rightFileStartLine
```

When about to post a comment at `filePath:line`:
- If already in `alreadyCommented` → **SKIP — do not post duplicate**
- If not in index → post normally

---

### 3. Fetch PR Details + Source Files

```
ado:repo_get_pull_request_by_id
  pullRequestId: <id>
  repositoryId: <repo name or GUID>
  project: <YOUR_PROJECT>
  includeWorkItemRefs: true
```

Then list changed files:

```
ado:repo_list_directory
  repositoryId: <repo GUID>
  path: /
  recursive: true
  recursionDepth: 4
  version: <sourceRefName branch>
  versionType: Branch
```

Then read file contents using:
```
ado:search_code
  searchText: <class or function name>
  project: [<YOUR_PROJECT>]
  repository: [<YOUR_REPO_NAME>]
```

---

### 4. Verify Linked Work Items

For each work item ID in `workItemRefs`:

```
ado:wit_get_work_item
  id: <work item id>
```

**Cross-check the work item against the code:**

| Check | What to look for |
|-------|-----------------|
| **Title alignment** | Does the PR title/description match the work item? |
| **Acceptance criteria** | Does the code implement everything described in the work item? |
| **Scope creep** | Does the PR touch things unrelated to the work item? |
| **Missing implementation** | Is anything from the work item absent from the code? |
| **No linked work item** | If `workItemRefs` is empty, flag it |

**Post a PR-level comment** (no file/line) with the work item result:

✅ All good:
```
✅ Work Item #<id> — "<title>"
All acceptance criteria appear to be addressed in this PR.
```

⚠️ Issues found:
```
⚠️ Work Item #<id> — "<title>"

Issues found:
- <specific gap or mismatch>
- <specific gap or mismatch>

💡 Make sure the following is addressed before merge: <what is missing>
```

⚠️ No work item linked:
```
⚠️ No work item linked to this PR

💡 Link the relevant work item so reviewers can verify the implementation matches the requirements.
```

---

### 5. Analyze Code for Violations

Check every changed file against **all checklists** below. Note the exact line number for each violation.

---

### 6. Post Inline Comments

For each violation, post:

```
ado:repo_create_pull_request_thread
  pullRequestId: <id>
  repositoryId: <repo GUID>
  project: <YOUR_PROJECT>
  filePath: /path/to/file
  rightFileStartLine: <line>
  rightFileEndLine: <line>
  rightFileStartOffset: 1
  rightFileEndOffset: <end char>
  content: <formatted comment — see Comment Format>
  status: Active
```

---

### 7. Ask for Approval Before Voting

After ALL inline comments have been posted, **STOP** and present a review summary to the user before casting any vote. Do NOT call `ado:repo_vote_pull_request` yet.

Present this summary in chat:

```
---
✅ Review complete for PR #<id> — "<title>"

📋 Summary:
  • <N> violations found  (🔴 <x> blockers, 🟡 <y> warnings, style <z>)
  • Work item: ✅ / ⚠️ <status>
  • All inline comments have been posted to the PR.

🗳️ Recommended vote: <WaitingForAuthor | ApprovedWithSuggestions | Approved>
   Reason: <one sentence explaining why>

Shall I cast this vote on the PR?
Reply YES to confirm, or tell me to change the vote before I proceed.
---
```

**Wait for the user's explicit confirmation before proceeding.**

- If the user says **"yes"** / **"confirm"** / **"go ahead"** → cast the vote (step 8)
- If the user says **"change it to Approved"** or similar → cast the vote they specified instead
- If the user says **"no"** / **"don't vote"** → skip voting entirely and confirm no vote was cast

---

### 8. Vote — Cast After Confirmation

Only after the user confirms, cast the vote:

```
ado:repo_vote_pull_request
  pullRequestId: <id>
  repositoryId: <repo GUID>
  vote: <confirmed vote>
```

| Outcome | Recommended vote |
|---------|-----------------|
| Any 🔴 blockers found | `WaitingForAuthor` |
| Minor 🟡 warnings / style only, no blockers | `ApprovedWithSuggestions` |
| Completely clean PR | `Approved` |

After voting, confirm to the user:
```
🗳️ Vote cast: <vote> on PR #<id>
The author will be notified.
```

---

---

## Pre-Post Comment Review Mode

Before posting any comment to the PR, the user can choose how they want to review them first. 
**Always ask the user at the start of the review:**

```
How would you like to review comments before I post them?

  A) Bulk checklist — I'll show you all comments at once as a numbered list.
     You tick ✅ the ones to post and ✗ the ones to skip, then I post them all.

  B) One by one — I'll show you each comment individually and wait for your
     approval before moving to the next one.

  C) Post all automatically — Skip review, post everything directly.

Reply A, B, or C.
```

### Mode A — Bulk Checklist

After analysis, present ALL findings as a numbered checklist before posting anything:

```
📋 Review complete — here are all the comments I found for PR #<id>:

 #  | File & Line                                      | Issue
----|--------------------------------------------------|---------------------------
 1  | ClaimTests.cs : 10                               | typo — TpsFramwork ==> TpsFramework
 2  | ClaimTests.cs : 52                               | use ShouldNotBe not Assert.NotEqual
 3  | ClaimsRepositoryTests.cs : 19                    | PartyId ==> partyId
 4  | ClaimsRepositoryTests.cs : 232                   | use [] instead of new() { }
 5  | DependsOnAttribute.cs : 1                        | DependsOn should not be in UnitTests
 6  | MockNafathCacheService.cs : 1                    | 🔴 mock boundary leak
 7  | TpsTestBaseWIthDbCleaner.cs : 1                  | WIth ==> With
 8  | ⚠️ Work Item #194002                              | No linked work item found

Which comments do you want me to post?
Reply with numbers separated by commas (e.g. "1,2,4,7"), "all", or "none".
```

Wait for the user's reply, then post only the selected comments.

### Mode B — One by One

Show each comment individually and wait for approval before the next:

```
Comment 1 of 8:

📄 File: ClaimTests.cs — Line 10
──────────────────────────────────
typo — `TpsFramwork` ==> `TpsFramework`

💡 Find and replace all usages of this alias name.

Post this comment? [YES / skip / stop]
```

- **YES** → post it, move to next
- **skip** → skip it silently, move to next
- **stop** → stop the whole review, don't post anything further

### Mode C — Automatic (default if user says "just post everything")

Skip the review step entirely. Post all comments immediately as found.

---

---

## Precise Comment Positioning

Every inline comment MUST be anchored to the exact location in the file.
Use all four positioning fields — never leave offsets as 1/1 unless the issue truly spans the whole line.

### Required fields for every inline comment

```
filePath:              /path/to/file.cs          ← full path from repo root
rightFileStartLine:    <line number>              ← 1-based, first line of the issue
rightFileEndLine:      <line number>              ← same as start for single-line
rightFileStartOffset:  <char position>            ← 1-based char offset where issue starts
rightFileEndOffset:    <char position>            ← char offset where issue ends
```

### How to calculate offsets

When you read file contents via `ado:search_code`, count characters from the start of the line:

```
Line 19: "        Guid PartyId = existingParty?.Id ?? Guid.Parse(..."
         123456789012345678901234
                   ^             ^
                   offset 9      offset 21 → "Guid PartyId"
```

So the comment for "PartyId ==> partyId" would be:
```
rightFileStartLine:   19
rightFileEndLine:     19
rightFileStartOffset: 14    ← start of "PartyId"
rightFileEndOffset:   20    ← end of "PartyId"
```

### Positioning rules by issue type

| Issue type | How to position |
|-----------|----------------|
| **Rename / typo** | Highlight the exact wrong word — start/end offsets wrap the bad identifier |
| **Missing blank line** | Point to the line after the `}` — offset 1 to end of line |
| **Wrong keyword** (`var`, `!=`) | Highlight the exact keyword |
| **File-scope namespace** | Line 1, full line (offset 1 to end) |
| **Entire file issue** (naming, structure) | Line 1, offset 1 to end of first line |
| **Multi-line block** | Set startLine/endLine to span the full block |
| **Specific method call** | Highlight just the method name, not the full call |
| **Work item summary** | No file/line — PR-level comment (omit all position fields) |

### Example: Highlighting a specific method name

```python
# Code: line 63 reads: "    var result = await _repo.GetClaimsAsync(request);"
# Issue: don't use var

rightFileStartLine:   63
rightFileEndLine:     63
rightFileStartOffset: 5     ← "var" starts at col 5
rightFileEndOffset:   7     ← "var" ends at col 7
```

### Offset counting tips

- Count from **1**, not 0
- **Spaces count** — leading whitespace is part of the offset
- **Tabs count as 1** character each
- If you cannot determine exact offsets from the code snippet, use offset 1 to end-of-identifier as a safe fallback — but always try to be precise
- For a whole-line issue, use offset 1 to the length of the line content

---

## Comment Format (MUST follow for every comment)

Every inline comment must use this two-part structure:

```
<issue statement>

💡 <suggestion tip>
```

**Rules:**
- **Issue line** — short, direct, in your team's voice. No preamble. Use backticks for code/file names.
- **Tip line** — starts with `💡`. Shows the corrected code or explains exactly what to do.
- **One blank line** between issue and tip.
- **Omit the tip** only if no concrete fix exists (e.g. pure whitespace formatting).
- **One thread per violation** — never combine two issues in one comment.
- **Severity prefix** — use `🔴` for blockers, `🟡` for warnings, no prefix for style.

### Comment Examples by Category

**Rename / typo:**
```
`getUserDta` ==> `getUserData`

💡 Rename the method and update all call sites.
```

**Security:**
```
🔴 security: API key hardcoded in source

💡 Move to environment variable or secrets manager and rotate this value immediately.
```

**Performance:**
```
🟡 performance: `.ToList()` before `.Where()` loads everything into memory

💡 Apply the filter before materializing: `.Where(x => x.IsActive).ToList()`
```

**Logic / correctness:**
```
🔴 logic: null reference — `.FirstOrDefault()` result used without null check

💡 Add a null guard before use: `if (result is null) return NotFound();`
```

**Work item mismatch:**
```
⚠️ Work Item #1234 — "Add pagination to claims list"

Issues found:
- Endpoint returns unbounded list with no page size limit
- `pageNumber` parameter is accepted but never passed to the repository

💡 Make sure the following is addressed before merge: implement actual pagination logic in the handler.
```

**Style (no tip needed):**
```
format — closing `)` indentation is inconsistent
```

---

## Universal Rules Checklist

These apply to **any language or stack**. Check them on every PR.

### 🔴 Security
- [ ] No hardcoded secrets, passwords, API keys, tokens, or connection strings in code
- [ ] No sensitive data (IDs, tokens, PII) in logs
- [ ] All endpoints/routes have authorization/authentication checks
- [ ] External input (from APIs, users, webhooks) is validated before use
- [ ] No credentials, `.env` files, or secret configs committed to the repo
- [ ] No internal IDs or sensitive fields unnecessarily exposed in response DTOs/models
- [ ] No open CORS policies or wildcard permissions in production config

### 🟡 Performance
- [ ] No filtering/sorting after loading all records into memory — push to DB/query layer
- [ ] No N+1 query patterns — fetch related data in one query where possible
- [ ] No blocking synchronous calls inside async methods
- [ ] Large result sets must be paginated — no unbounded list returns from endpoints
- [ ] No repeated calls for the same data within one request — cache or batch
- [ ] No `Thread.Sleep` or equivalent blocking delays in async code

### 🔴 Logic & Correctness
- [ ] No null reference risks — check all `.FirstOrDefault()`, optional chaining, nullable types
- [ ] No silent failures — `try/catch` must log or re-throw, not swallow exceptions silently
- [ ] No dead code paths — conditions that can never be true/false
- [ ] All edge cases handled — empty collections, zero values, null optionals, invalid IDs
- [ ] Status/enum values used in conditions match the intended business logic
- [ ] No off-by-one errors in comparisons (`>` vs `>=`, date ranges, pagination)
- [ ] Async patterns correct — no `async void`, no `.Result`/`.GetAwaiter().GetResult()`

### Style & Quality (universal)
- [ ] No commented-out dead code left in the codebase
- [ ] No unused imports/dependencies
- [ ] No `TODO` or `FIXME` comments merged without a linked work item
- [ ] Method/function names clearly describe what they do
- [ ] No overly long methods — split into smaller focused functions if > ~40 lines
- [ ] PR title and description match what the code actually does

---

## Language Rules Checklist

> **⚙️ CUSTOMIZE THIS SECTION** for your team's stack.
> Examples below are for **C# / .NET**. Replace or extend for your language.

### C# / .NET (example — keep, replace, or extend)

**Formatting**
- [ ] File-scoped namespaces — `namespace Foo.Bar;` not `namespace Foo.Bar { }`
- [ ] Blank line after `}` between methods/properties
- [ ] No extra blank lines inside class bodies
- [ ] Wrap long method calls — each parameter on its own line

**Naming & Conventions**
- [ ] No `var` — use explicit static types
- [ ] After `new`, omit the type: `List<Claim> x = new(...)` not `new List<Claim>(...)`
- [ ] Variable names start lowercase: `partyId` not `PartyId`
- [ ] Namespace must match folder path

**Modern C# Patterns**
- [ ] Use `is not` instead of `!=` for null checks
- [ ] Use collection expressions `[]` instead of `new List<T>() { }`
- [ ] DTOs with only properties should be `record` types
- [ ] Pass `CancellationToken` to all async calls that accept it
- [ ] Use `Guard` instead of manual `if (x == null) throw`

**Code Quality**
- [ ] Clean unused `using` directives
- [ ] Remove unused interceptors or middleware
- [ ] Constructor order: properties first, then constructor
- [ ] No normal constructor on `record` — use primary constructor

---

### TypeScript / Node.js (example — add if relevant)

- [ ] No `any` type — always use explicit types or generics
- [ ] Use `const` over `let` where value doesn't change
- [ ] Async functions must `await` or return the `Promise` — no floating promises
- [ ] No `console.log` left in production code — use a logger
- [ ] Use optional chaining `?.` and nullish coalescing `??` appropriately
- [ ] Interfaces for shared data shapes, not inline object types

---

### Java / Spring (example — add if relevant)

- [ ] Use `Optional` instead of returning `null`
- [ ] No field injection (`@Autowired` on fields) — use constructor injection
- [ ] All `@RestController` methods must have appropriate `@PreAuthorize` or security check
- [ ] Use `ResponseEntity<T>` for explicit HTTP status control
- [ ] No checked exceptions swallowed silently

---

## Test Rules Checklist

- [ ] New features must have corresponding tests in the PR
- [ ] Test names clearly describe the scenario: `Should_<action>_When_<condition>`
- [ ] No empty test bodies or `Assert.True(true)` placeholder tests merged
- [ ] Unit tests are independent — no shared mutable state between tests
- [ ] Integration tests clean up after themselves — no data leaked between runs
- [ ] Mocks only at the service boundary — don't mock internal implementation details
- [ ] Edge cases covered — empty inputs, null values, invalid IDs

---

## Architecture Rules Checklist

- [ ] No binary files (Word docs, PDFs, images) committed to the repo — link to external storage
- [ ] No backup files (`.bak`, `.orig`, `.old`) in source control
- [ ] Folder and file names use consistent casing convention (check existing project style)
- [ ] No local/dev connection strings or IPs in non-dev config files
- [ ] Config files for different environments are properly separated
- [ ] Dependency changes (new packages, version bumps) are intentional and documented

---

## Comment Posting Rules

1. **One thread per violation** — never combine two issues in one comment
2. **Always pin to a file and line** — PR-level comments only for work item summaries
3. **Use the two-part format** — issue + `💡` tip (if a concrete fix exists)
4. **Use severity prefix** — `🔴` for blockers, `🟡` for warnings, none for style
5. **Tip must be actionable** — show corrected code or explain exactly what to do
6. **Set `status: Active`** on all threads so they show as unresolved
7. **Vote after every review** — `WaitingForAuthor` / `ApprovedWithSuggestions` / `Approved`

---

## Adapting This Skill for Your Team

When setting up for a new team, update these sections:

| Section | What to change |
|---------|---------------|
| **Setup block** | Fill in org, project, repo, branches, stack |
| **Language Rules Checklist** | Replace C# examples with your language's rules |
| **Comment Format examples** | Add real examples from your team's past reviews |
| **Comment Style Guide** | Describe your team's review tone and voice |
| **Architecture Rules** | Add project-specific structure conventions |

Once configured, this skill works identically to a team-specific skill — just with your values baked in.
