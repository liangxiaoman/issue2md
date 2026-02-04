# spec: 001 - Core Functionality

**Version:** 1.0
**Status:** Draft
**Created:** 2025-02-04

---

## User Story

### CLI Version (MVP)
作为开发者或技术写作者，我希望能够通过命令行快速将 GitHub Issue/PR/Discussion 转换为 Markdown 文件，以便：
- 归档重要的技术讨论
- 将 Issue 内容转化为博客素材
- 离线保存有价值的 GitHub 对话

### Web Version (Future)
作为非技术用户，我希望通过网页界面输入 GitHub URL 并下载 Markdown 文件，无需安装任何命令行工具。

---

## Functional Requirements

### 1. URL Recognition
- **MUST**: 自动识别 URL 类型，无需用户手动指定
- **Supported URL Patterns**:
  - Issue: `https://github.com/{owner}/{repo}/issues/{number}`
  - Pull Request: `https://github.com/{owner}/{repo}/pull/{number}`
  - Discussion: `https://github.com/{orgs}/{owner}/teams/{team}/discussions/{number}` 或 `https://github.com/{owner}/{repo}/discussions/{number}`

### 2. Content Conversion

#### 2.1 Common Elements (All Types)
- Title
- Author (with optional link to GitHub profile)
- Created timestamp (ISO 8601)
- Current state (Open/Closed/Merged)
- Body content
- All comments (chronological order)
- Reactions (optional, via flag)

#### 2.2 Type-Specific Elements

**Issue**
- Labels (as comma-separated list)
- Milestone (if exists)
- All comments + issue comments

**Pull Request**
- Labels
- Milestone
- Merge status (Merged/Closed/Open)
- PR description
- **ALL** comments (PR comments + review comments, flattened by timestamp)
- **DO NOT** include diff or commit history

**Discussion**
- Category
- Answer marker (if a comment is marked as answer)
- All replies

### 3. Comment Handling
- **Sorting**: Chronological order (oldest first)
- **PR Review Comments**: Flatten with regular comments, sort by timestamp
- **Discussion Answer**: Mark with special indicator (e.g., `✅ **Accepted Answer**`)

### 4. Authentication
- **MUST**: Support public repositories without authentication
- **OPTIONAL**: Read `GITHUB_TOKEN` environment variable for authenticated requests
- **MUST NOT**: Provide `--token` CLI flag (prevents leakage in shell history)

### 5. Command-Line Interface

#### 5.1 Syntax
```bash
issue2md [flags] <url> [output_file]
```

#### 5.2 Arguments
- `url`: GitHub Issue/PR/Discussion URL (required)
- `output_file`: Output file path (optional, default: stdout)

#### 5.3 Flags
- `-enable-reactions`: Include reaction counts (👍❤️等) in output
- `-enable-user-links`: Render usernames as links to GitHub profiles
- `-h, --help`: Show help message
- `-v, --version`: Show version information

#### 5.4 Usage Examples
```bash
# Output to stdout
issue2md https://github.com/owner/repo/issues/123

# Output to file
issue2md https://github.com/owner/repo/issues/123 output.md

# With reactions and user links
issue2md -enable-reactions -enable-user-links https://github.com/owner/repo/pull/456 pr.md

# Authenticated request
GITHUB_TOKEN=ghp_xxx issue2md https://github.com/owner/repo/discussions/78
```

### 6. Output Format

#### 6.1 YAML Frontmatter
```yaml
---
title: "[Feature] Add new authentication flow"
url: "https://github.com/owner/repo/issues/123"
author: "johndoe"
author_url: "https://github.com/johndoe"
created_at: "2025-01-15T10:30:00Z"
state: "open"
type: "issue"
labels: ["enhancement", "authentication"]
milestone: "v2.0"
reactions_enabled: true
---
```

#### 6.2 Markdown Body Structure
```markdown
# [Feature] Add new authentication flow

> **Author:** [@johndoe](https://github.com/johndoe)
> **Created:** 2025-01-15 10:30 UTC
> **Status:** 🟢 Open
> **Labels:** enhancement, authentication
> **Milestone:** v2.0

## Description

This is the main issue body...

## Comments

### @alice - 2025-01-15 14:20 UTC

This looks great! One suggestion...

**Reactions:** 👍 2, 🎉 1

### @bob - 2025-01-15 16:45 UTC

I agree with @alice. What about...?

**Reactions:** 👀 1, ❤️ 3

### @johndoe - 2025-01-16 09:10 UTC

✅ **Accepted Answer**

Good points! I'll implement...

**Reactions:** 🚀 5
```

### 7. Media Handling
- **Images**: Preserve original GitHub URLs, no local download
- **Attachments**: Preserve original links

---

## Non-Functional Requirements

### 1. Architecture
- **Decoupling**: Business logic MUST be separated from CLI layer
  - `internal/github/`: GitHub API client
  - `internal/converter/`: Markdown conversion logic
  - `cmd/issue2md/`: CLI entry point
- **Rationale**: Enable future Web version reuse

### 2. Error Handling
- **Invalid URL**: Clear error message to stderr, exit code 1
- **Resource Not Found (404)**: Clear error message, exit code 1
- **API Rate Limit**: Forward GitHub API error message to user
- **Network Errors**: Display error context, exit code 1
- **No Retry Logic**: Keep CLI lightweight, let GitHub handle retries

### 3. Performance
- **Timeout**: 30 second default timeout for API requests
- **Memory Efficiency**: Stream output where possible, don't load entire payload in memory if not needed

### 4. Code Quality (Constitution Compliance)
- **Test-First**: All features start with failing tests
- **Table-Driven Tests**: Preferred testing style
- **No Mocks**: Use integration tests with real GitHub API where possible
- **Explicit Error Handling**: All errors MUST be handled with `fmt.Errorf("...: %w", err)`
- **No Global Variables**: All dependencies via explicit injection

---

## Acceptance Criteria

### Test Cases

#### TC001: URL Recognition
- [ ] Parse Issue URL correctly
- [ ] Parse PR URL correctly
- [ ] Parse Discussion URL correctly
- [ ] Reject invalid URL with clear error

#### TC002: Issue Conversion
- [ ] Convert open issue to Markdown
- [ ] Convert closed issue to Markdown
- [ ] Include all labels in frontmatter
- [ ] Include milestone if present
- [ ] Preserve markdown formatting in body

#### TC003: PR Conversion
- [ ] Convert open PR to Markdown
- [ ] Convert merged PR to Markdown
- [ ] Flatten all comments (PR + review) by timestamp
- [ ] Include merge state in frontmatter
- [ ] NOT include diff or commit history

#### TC004: Discussion Conversion
- [ ] Convert discussion to Markdown
- [ ] Mark accepted answer with special indicator
- [ ] Include discussion category

#### TC005: Comment Handling
- [ ] Sort comments chronologically
- [ ] Include comment timestamps
- [ ] Include comment authors
- [ ] Preserve code blocks in comments

#### TC006: Optional Features
- [ ] `-enable-reactions` flag includes reaction counts
- [ ] `-enable-user-links` flag renders profile links
- [ ] Without flags, reactions and links are omitted

#### TC007: Output
- [ ] Default output to stdout
- [ ] Write to file when output_file specified
- [ ] File is created/overwritten successfully

#### TC008: Authentication
- [ ] Works without GITHUB_TOKEN (public repo)
- [ ] Uses GITHUB_TOKEN when set
- [ ] Rejects private repo without token

#### TC009: Error Handling
- [ ] Clear error on invalid URL
- [ ] Clear error on 404
- [ ] Clear error on network failure
- [ ] Exit code 1 on all errors

#### TC010: Edge Cases
- [ ] Handle issue with no comments
- [ ] Handle issue with no body
- [ ] Handle deleted comments
- [ ] Handle very long comment threads (100+ comments)
- [ ] Handle special characters in title/body

---

## Output Format Examples

### Example 1: Issue with Reactions and User Links

**Input:**
```bash
issue2md -enable-reactions -enable-user-links https://github.com/golang/go/issues/12345
```

**Output:**
```yaml
---
title: "proposal: go2: add generics"
url: "https://github.com/golang/go/issues/12345"
author: "gopher"
author_url: "https://github.com/gopher"
created_at: "2025-01-15T10:30:00Z"
state: "closed"
type: "issue"
labels: ["proposal", "go2"]
milestone: "Go1.20"
reactions_enabled: true
---

# proposal: go2: add generics

> **Author:** [@gopher](https://github.com/gopher)
> **Created:** 2025-01-15 10:30 UTC
> **Status:** 🔴 Closed
> **Labels:** proposal, go2
> **Milestone:** Go1.20

## Description

This proposal describes adding generics to Go...

## Comments

### @dev1 - 2025-01-15 14:20 UTC

This looks great! One suggestion...

**Reactions:** 👍 2, 🎉 1

### @dev2 - 2025-01-15 16:45 UTC

I agree with @dev1. What about...?

**Reactions:** 👀 1, ❤️ 3
```

### Example 2: Pull Request (Flattened Comments)

**Input:**
```bash
issue2md https://github.com/owner/repo/pull/456 pr.md
```

**Output:**
```yaml
---
title: "Fix authentication bug in login flow"
url: "https://github.com/owner/repo/pull/456"
author: "contributor"
author_url: "https://github.com/contributor"
created_at: "2025-01-20T09:15:00Z"
state: "merged"
type: "pull_request"
labels: ["bugfix"]
reactions_enabled: false
---

# Fix authentication bug in login flow

> **Author:** [@contributor](https://github.com/contributor)
> **Created:** 2025-01-20 09:15 UTC
> **Status:** 🟣 Merged
> **Labels:** bugfix

## Description

This PR fixes the token expiration issue...

## Comments

### @reviewer1 - 2025-01-20 10:00 UTC

Main comment on the PR

### @reviewer2 - 2025-01-20 11:30 UTC

Review comment on code (originally on file `auth.go:123`)

LGTM! Just one question...

### @contributor - 2025-01-20 13:45 UTC

Good catch! Updated...

### @maintainer - 2025-01-20 15:00 UTC

Thanks! Merging.
```

### Example 3: Discussion with Answer

**Input:**
```bash
issue2md https://github.com/org/community/discussions/78 discussion.md
```

**Output:**
```yaml
---
title: "Best practices for error handling?"
url: "https://github.com/org/community/discussions/78"
author: "newbie"
author_url: "https://github.com/newbie"
created_at: "2025-01-25T08:00:00Z"
state: "open"
type: "discussion"
category: "q-and-a"
reactions_enabled: false
---

# Best practices for error handling?

> **Author:** [@newbie](https://github.com/newbie)
> **Created:** 2025-01-25 08:00 UTC
> **Status:** 🟢 Open
> **Category:** Q&A

## Description

What's the recommended pattern for...?

## Comments

### @senior1 - 2025-01-25 09:00 UTC

You should use... (not accepted)

### @expert - 2025-01-25 10:30 UTC

✅ **Accepted Answer**

The best practice is to...

### @newbie - 2025-01-25 11:00 UTC

Thank you @expert!
```

---

## Implementation Notes

### Dependencies
- Go 1.24+
- GitHub REST API v3
- Standard library only (no external dependencies for MVP)

### GitHub API Endpoints
- Issues: `GET /repos/{owner}/{repo}/issues/{issue_number}`
- Pull Requests: `GET /repos/{owner}/{repo}/pulls/{pull_number}`
- Discussions: `GET /repos/{owner}/{repo}/discussions/{discussion_number}` (requires GraphQL)
- Comments: Various endpoints per resource type

### Rate Limits
- Unauthenticated: 60 requests/hour
- Authenticated: 5000 requests/hour

### Future Enhancements (Post-MVP)
- [ ] Web interface
- [ ] Custom templates
- [ ] Batch mode (multiple URLs)
- [ ] Output filtering (by date, author, etc.)
- [ ] Local image download
- [ ] JSON output format
- [ ] GitLab/Jira support
