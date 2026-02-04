# Implementation Tasks: Core Functionality

**Version:** 1.0
**Status:** Ready for Execution
**Created:** 2025-02-04
**Based on:** spec.md v1.0, plan.md v1.0

---

## Task Execution Guidelines

### Legend
- **[P]** = Parallel (no dependencies with other tasks marked [P])
- **TDD** = Test-Driven Development (write test first, then implement)
- **Depends on:** = Task must be completed first

### TDD Workflow (Mandatory)
1. **Red**: Write failing test
2. **Green**: Write minimal implementation to pass test
3. **Refactor**: Clean up while keeping tests green

### Task Format
```
ID: PHASE-NUMBER
Title: Brief description
File: path/to/file.go
Type: [TEST | IMPLEMENTATION]
TDD: [CYCLE-1 | CYCLE-2 | ...]
Estimate: Small/Medium/Large
```

---

## Phase 1: Foundation Layer

**Goal:** Build bottom-layer packages with no external dependencies

### 1.1 Parser Package (`internal/parser`)

#### Task 1.1.1 [P]
**Title:** Create parser package types definition
**File:** `internal/parser/types.go`
**Type:** IMPLEMENTATION
**TDD:** N/A (data structures only)
**Estimate:** Small

**Description:**
Define core data structures for URL parsing:
- `ResourceType` enum with `String()` method
- `GitHubURL` struct with `IsValid()` method

**Acceptance Criteria:**
- [ ] `ResourceType` has 3 constants: Issue, PullRequest, Discussion
- [ ] `String()` returns correct string for each type
- [ ] `GitHubURL` struct has Type, Owner, Repo, Number fields
- [ ] `IsValid()` validates non-empty required fields

---

#### Task 1.1.2
**Title:** Write table-driven tests for URL parsing
**File:** `internal/parser/url_test.go`
**Type:** TEST
**TDD:** CYCLE-1 (RED)
**Estimate:** Medium
**Depends on:** 1.1.1

**Description:**
Write comprehensive table-driven tests for `ParseURL()` function covering:
- Valid Issue URLs
- Valid PR URLs
- Valid Discussion URLs (repo-level)
- Valid Discussion URLs (org-level)
- Invalid URLs (missing protocol)
- Invalid URLs (not github.com)
- Invalid URLs (malformed structure)
- Edge cases (http vs https, www prefix, trailing slashes)

**Test Structure:**
```go
func TestParseURL(t *testing.T) {
    tests := []struct {
        name    string
        url     string
        want    *GitHubURL
        wantErr bool
        errMsg  string
    }{
        // 15+ test cases
    }
    // ... table-driven test implementation
}
```

**Acceptance Criteria:**
- [ ] Test file compiles (will fail initially - CYCLE-1 RED)
- [ ] 15+ test cases covering all scenarios
- [ ] Error message validation for invalid URLs
- [ ] Deep equality validation for valid URLs

---

#### Task 1.1.3
**Title:** Implement ParseURL function
**File:** `internal/parser/url.go`
**Type:** IMPLEMENTATION
**TDD:** CYCLE-1 (GREEN)
**Estimate:** Medium
**Depends on:** 1.1.2

**Description:**
Implement `ParseURL(urlStr string) (*GitHubURL, error)` function using `net/url` package:
- Parse URL structure
- Validate GitHub domain
- Extract owner, repo, number
- Detect resource type from path pattern
- Return wrapped errors with context

**Error Handling:**
- Missing protocol: `fmt.Errorf("invalid URL format: %w", err)`
- Not github.com: `fmt.Errorf("not a GitHub URL: %s", url)`
- Invalid pattern: `fmt.Errorf("unsupported URL pattern: %s", url)`

**Acceptance Criteria:**
- [ ] All tests from 1.1.2 pass (CYCLE-1 GREEN)
- [ ] Uses `fmt.Errorf` with `%w` for error wrapping
- [ ] No global variables
- [ ] Function is pure (no side effects)

---

### 1.2 Config Package (`internal/config`)

#### Task 1.2.1 [P]
**Title:** Create config package data structures
**File:** `internal/config/config.go` (partial)
**Type:** IMPLEMENTATION
**TDD:** N/A (data structures only)
**Estimate:** Small

**Description:**
Define `Config` struct with fields:
- `URL` string
- `OutputPath` string
- `EnableReactions` bool
- `EnableUserLinks` bool
- `Token` string

**Acceptance Criteria:**
- [ ] All required fields present
- [ ] Field types match spec
- [ ] Exported struct with exported fields

---

#### Task 1.2.2
**Title:** Write table-driven tests for config loading
**File:** `internal/config/config_test.go`
**Type:** TEST
**TDD:** CYCLE-1 (RED)
**Estimate:** Medium
**Depends on:** 1.2.1

**Description:**
Write table-driven tests for `LoadConfig()` covering:
- Positional argument (URL)
- Optional positional argument (output file)
- `-enable-reactions` flag
- `-enable-user-links` flag
- `-h` help flag (should return special error)
- `-v` version flag (should return special error)
- Invalid flags
- Missing URL argument
- GITHUB_TOKEN environment variable

**Test Structure:**
```go
func TestLoadConfig(t *testing.T) {
    tests := []struct {
        name      string
        args      []string
        envToken  string
        want      *Config
        wantErr   bool
        errMsg    string
    }{
        // 12+ test cases
    }
    // ... table-driven test implementation
}
```

**Acceptance Criteria:**
- [ ] Test file compiles (will fail initially - CYCLE-1 RED)
- [ ] 12+ test cases covering all scenarios
- [ ] Mock environment variable setting

---

#### Task 1.2.3
**Title:** Implement LoadConfig function
**File:** `internal/config/config.go` (complete)
**Type:** IMPLEMENTATION
**TDD:** CYCLE-1 (GREEN)
**Estimate:** Medium
**Depends on:** 1.2.2

**Description:**
Implement `LoadConfig(args []string) (*Config, error)` using `flag` package:
- Parse CLI flags
- Read GITHUB_TOKEN from environment
- Validate URL is provided
- Return special error for help/version (exit code 0)
- Validate flag combinations

**Error Handling:**
- Missing URL: `fmt.Errorf("URL argument is required")`
- Invalid flags: use flag package error handling

**Acceptance Criteria:**
- [ ] All tests from 1.2.2 pass (CYCLE-1 GREEN)
- [ ] Uses `os.Getenv("GITHUB_TOKEN")` for token
- [ ] No `--token` flag exists (security requirement)
- [ ] Help/version return special error types

---

## Phase 2: GitHub Fetcher Layer

**Goal:** Implement GitHub API client with real integration tests

### 2.1 GitHub Types Definition

#### Task 2.1.1
**Title:** Create GitHub package data types
**File:** `internal/github/types.go`
**Type:** IMPLEMENTATION
**TDD:** N/A (data structures only)
**Estimate:** Medium
**Depends on:** None (can parallel with Phase 1)

**Description:**
Define all data structures for GitHub API:
- `Actor` struct (Login, URL)
- `Reactions` struct (8 reaction types)
- `Comment` struct (all fields from plan.md)
- `Issue` struct (all fields from plan.md)
- `PullRequest` struct (all fields from plan.md)
- `Discussion` struct (all fields from plan.md)

**Acceptance Criteria:**
- [ ] All structs match plan.md specification
- [ ] Field tags for JSON/GraphQL unmarshaling
- [ ] Time fields use `time.Time`
- [ ] Pointer types for optional fields

---

#### Task 2.1.2 [P]
**Title:** Create GitHub client struct and constructor
**File:** `internal/github/client.go` (partial)
**Type:** IMPLEMENTATION
**TDD:** N/A (data structures only)
**Estimate:** Small
**Depends on:** 2.1.1

**Description:**
Define `Client` struct and constructor:
- `Client` struct with fields: token, httpClient, timeout
- `NewClient(token string, timeout time.Duration) *Client`
- `SetToken(token string)` method

**Acceptance Criteria:**
- [ ] Client struct created
- [ ] Constructor initializes http.Client with timeout
- [ ] Token stored for later use in Authorization header
- [ ] No global variables

---

### 2.2 Issue Fetcher

#### Task 2.2.1
**Title:** Write integration test for FetchIssue
**File:** `internal/github/issue_test.go`
**Type:** TEST
**TDD:** CYCLE-1 (RED)
**Estimate:** Medium
**Depends on:** 2.1.2

**Description:**
Write integration test for `FetchIssue()` using real GitHub API:
- Use public repository (e.g., golang/go #1)
- Test with reactions enabled/disabled
- Validate all returned fields
- Skip in short mode: `if testing.Short() { t.Skip() }`

**Test Cases:**
- Fetch open issue
- Fetch closed issue
- Fetch issue with no comments
- Fetch issue with reactions

**Acceptance Criteria:**
- [ ] Test file compiles (will fail initially - CYCLE-1 RED)
- [ ] Uses real GitHub API (no mocks)
- [ ] Validates issue structure
- [ ] Validates comment structure
- [ ] Tests reactions when enabled

---

#### Task 2.2.2
**Title:** Implement FetchIssue method
**File:** `internal/github/issue.go`
**Type:** IMPLEMENTATION
**TDD:** CYCLE-1 (GREEN)
**Estimate:** Medium
**Depends on:** 2.2.1

**Description:**
Implement `FetchIssue()` using `google/go-github` library:
```go
func (c *Client) FetchIssue(ctx context.Context, owner, repo string, number int, includeReactions bool) (*Issue, error)
```

**Implementation Steps:**
1. Call `client.Issues.Get()`
2. Call `client.Issues.ListComments()` with pagination
3. If `includeReactions`, call `client.Reactions.ListIssueReactions()`
4. Transform github.Issue to internal.Issue
5. Transform github.IssueComment to internal.Comment
6. Return wrapped errors

**Error Handling:**
- 404: `fmt.Errorf("issue not found (404): %w", err)`
- Rate limit: `fmt.Errorf("API rate limit exceeded: %w", err)`
- Generic: `fmt.Errorf("fetch issue %s/%s#%d: %w", owner, repo, number, err)`

**Acceptance Criteria:**
- [ ] All tests from 2.2.1 pass (CYCLE-1 GREEN)
- [ ] Uses `fmt.Errorf` with `%w` for error wrapping
- [ ] Handles pagination for comments
- [ ] Context cancellation supported

---

### 2.3 Pull Request Fetcher

#### Task 2.3.1
**Title:** Write integration test for FetchPullRequest
**File:** `internal/github/pull_request_test.go`
**Type:** TEST
**TDD:** CYCLE-1 (RED)
**Estimate:** Medium
**Depends on:** 2.1.2

**Description:**
Write integration test for `FetchPullRequest()`:
- Use real PR from public repository
- Test comment flattening (PR comments + review comments)
- Validate chronological sorting
- Test merged/closed/open states

**Test Cases:**
- Fetch open PR
- Fetch merged PR
- Fetch closed PR
- Verify comment flattening and sorting

**Acceptance Criteria:**
- [ ] Test file compiles (will fail initially - CYCLE-1 RED)
- [ ] Uses real GitHub API
- [ ] Validates PR structure
- [ ] Validates comment flattening logic

---

#### Task 2.3.2
**Title:** Implement FetchPullRequest method
**File:** `internal/github/pull_request.go`
**Type:** IMPLEMENTATION
**TDD:** CYCLE-1 (GREEN)
**Estimate:** Large
**Depends on:** 2.3.1

**Description:**
Implement `FetchPullRequest()` with comment flattening:
```go
func (c *Client) FetchPullRequest(ctx context.Context, owner, repo string, number int, includeReactions bool) (*PullRequest, error)
```

**Implementation Steps:**
1. Call `client.PullRequests.Get()`
2. Call `client.PullRequests.ListComments()` (PR comments)
3. Call `client.PullRequests.ListComments()` with `ListOptions` for review comments
4. Merge both comment lists
5. Sort by `CreatedAt` timestamp
6. Transform to internal.PullRequest

**Flattening Logic:**
```go
allComments := append(prComments, reviewComments...)
sort.Slice(allComments, func(i, j int) bool {
    return allComments[i].CreatedAt.Before(allComments[j].CreatedAt)
})
```

**Error Handling:**
- Same pattern as FetchIssue

**Acceptance Criteria:**
- [ ] All tests from 2.3.1 pass (CYCLE-1 GREEN)
- [ ] Comments are properly flattened
- [ ] Comments are chronologically sorted
- [ ] Review comments marked with `IsReviewComment = true`

---

### 2.4 Discussion Fetcher (GraphQL)

#### Task 2.4.1
**Title:** Write integration test for FetchDiscussion
**File:** `internal/github/discussion_test.go`
**Type:** TEST
**TDD:** CYCLE-1 (RED)
**Estimate:** Medium
**Depends on:** 2.1.2

**Description:**
Write integration test for `FetchDiscussion()` using GraphQL:
- Use real discussion from public repository
- Test discussion with answer
- Test discussion without answer

**Test Cases:**
- Fetch discussion with answer
- Fetch discussion without answer
- Validate category field

**Acceptance Criteria:**
- [ ] Test file compiles (will fail initially - CYCLE-1 RED)
- [ ] Uses GraphQL API
- [ ] Validates answer detection

---

#### Task 2.4.2
**Title:** Implement FetchDiscussion method (GraphQL)
**File:** `internal/github/discussion.go`
**Type:** IMPLEMENTATION
**TDD:** CYCLE-1 (GREEN)
**Estimate:** Large
**Depends on:** 2.4.1

**Description:**
Implement `FetchDiscussion()` using GitHub GraphQL API:
```go
func (c *Client) FetchDiscussion(ctx context.Context, owner, repo string, number int, includeReactions bool) (*Discussion, error)
```

**Implementation Steps:**
1. Define GraphQL query (see plan.md appendix)
2. Use `github.com/shurcooL/githubv4` or go-github GraphQL client
3. Execute query with variables
4. Parse response into internal.Discussion
5. Mark answer comment with `IsAnswer = true`

**GraphQL Query:** (from plan.md appendix)

**Error Handling:**
- Same pattern as FetchIssue
- GraphQL-specific errors handled

**Acceptance Criteria:**
- [ ] All tests from 2.4.1 pass (CYCLE-1 GREEN)
- [ ] Uses GraphQL API correctly
- [ ] Answer comment properly marked
- [ ] Comments include `IsAnswer` flag

---

## Phase 3: Markdown Converter Layer

**Goal:** Convert GitHub data structures to Markdown format

### 3.1 Converter Core

#### Task 3.1.1
**Title:** Create converter package core structures
**File:** `internal/converter/converter.go` (partial)
**Type:** IMPLEMENTATION
**TDD:** N/A (data structures only)
**Estimate:** Small
**Depends on:** Phase 2 complete

**Description:**
Define `Converter` struct with functional options:
- `Converter` struct: enableReactions, enableUserLinks fields
- `Option` type: `func(*Converter)`
- `NewConverter(options ...Option) *Converter`
- `WithReactions() Option`
- `WithUserLinks() Option`

**Acceptance Criteria:**
- [ ] Functional options pattern implemented
- [ ] Defaults: reactions=false, userLinks=false

---

#### Task 3.1.2
**Title:** Write table-driven tests for helper functions
**File:** `internal/converter/helpers_test.go`
**Type:** TEST
**TDD:** CYCLE-1 (RED)
**Estimate:** Medium
**Depends on:** 3.1.1

**Description:**
Write table-driven tests for helper functions:
- `formatTimestamp(t time.Time) string`
- `formatUser(c *Converter, actor github.Actor) string`
- `getStateEmoji(state string) string`
- `formatReactions(r *github.Reactions) string`

**Test Cases:**
- Timestamp formatting with various timezones
- User formatting with/without links
- State emoji for open/closed/merged
- Reactions with various combinations

**Acceptance Criteria:**
- [ ] Test file compiles (will fail initially - CYCLE-1 RED)
- [ ] Table-driven tests for all helpers
- [ ] Edge cases covered (empty reactions, nil times)

---

#### Task 3.1.3
**Title:** Implement helper functions
**File:** `internal/converter/helpers.go`
**Type:** IMPLEMENTATION
**TDD:** CYCLE-1 (GREEN)
**Estimate:** Medium
**Depends on:** 3.1.2

**Description:**
Implement helper functions:
- `formatTimestamp()`: format as "2006-01-02 15:04 UTC"
- `formatUser()`: return "@username" or "[@username](url)"
- `getStateEmoji()`: return 🟢/🔴/🟣
- `formatReactions()`: return "👍 2, ❤️ 3" or empty string

**Acceptance Criteria:**
- [ ] All tests from 3.1.2 pass (CYCLE-1 GREEN)
- [ ] Functions are pure (no side effects)
- [ ] Proper UTC timezone handling

---

#### Task 3.1.4
**Title:** Write test for frontmatter generation
**File:** `internal/converter/frontmatter_test.go`
**Type:** TEST
**TDD:** CYCLE-1 (RED)
**Estimate:** Medium
**Depends on:** 3.1.1

**Description:**
Write test for `generateFrontmatter()` using different resource types:
- Issue frontmatter
- PR frontmatter
- Discussion frontmatter

**Test Cases:**
- Issue with labels and milestone
- PR with merge state
- Discussion with category
- Verify YAML format

**Acceptance Criteria:**
- [ ] Test file compiles (will fail initially - CYCLE-1 RED)
- [ ] Validates YAML structure
- [ ] Validates all required fields

---

#### Task 3.1.5
**Title:** Implement frontmatter generation
**File:** `internal/converter/frontmatter.go`
**Type:** IMPLEMENTATION
**TDD:** CYCLE-1 (GREEN)
**Estimate:** Medium
**Depends on:** 3.1.4

**Description:**
Implement `generateFrontmatter(data interface{}) (string, error)`:
- Use `encoding/json` with custom YAML marshal
- Or simple string template (avoid YAML library dependency)
- Include all fields from spec

**Acceptance Criteria:**
- [ ] All tests from 3.1.4 pass (CYCLE-1 GREEN)
- [ ] Valid YAML frontmatter format
- [ ] Handles nil/optional fields correctly

---

### 3.2 Issue Converter

#### Task 3.2.1
**Title:** Write test for ConvertIssue
**File:** `internal/converter/issue_test.go`
**Type:** TEST
**TDD:** CYCLE-1 (RED)
**Estimate:** Medium
**Depends on:** 3.1.5

**Description:**
Write table-driven test for `ConvertIssue()`:
- Open issue with reactions
- Closed issue without reactions
- Issue with no comments
- Issue with many comments

**Test Cases:**
- Verify markdown structure
- Verify frontmatter
- Verify comment format
- Verify reactions when enabled

**Acceptance Criteria:**
- [ ] Test file compiles (will fail initially - CYCLE-1 RED)
- [ ] Uses mock github.Issue data
- [ ] Validates output format

---

#### Task 3.2.2
**Title:** Implement ConvertIssue method
**File:** `internal/converter/issue.go`
**Type:** IMPLEMENTATION
**TDD:** CYCLE-1 (GREEN)
**Estimate:** Medium
**Depends on:** 3.2.1

**Description:**
Implement `ConvertIssue(issue *github.Issue) (string, error)`:
1. Generate frontmatter
2. Format header with author, date, state, labels
3. Format body
4. Format each comment
5. Return complete markdown

**Acceptance Criteria:**
- [ ] All tests from 3.2.1 pass (CYCLE-1 GREEN)
- [ ] Output matches spec format
- [ ] Respects enableReactions flag
- [ ] Respects enableUserLinks flag

---

### 3.3 Pull Request Converter

#### Task 3.3.1
**Title:** Write test for ConvertPullRequest
**File:** `internal/converter/pull_request_test.go`
**Type:** TEST
**TDD:** CYCLE-1 (RED)
**Estimate:** Medium
**Depends on:** 3.1.5

**Description:**
Write table-driven test for `ConvertPullRequest()`:
- Merged PR
- Open PR
- Closed PR
- PR with review comments

**Acceptance Criteria:**
- [ ] Test file compiles (will fail initially - CYCLE-1 RED)
- [ ] Validates PR-specific fields
- [ ] Validates merge state display

---

#### Task 3.3.2
**Title:** Implement ConvertPullRequest method
**File:** `internal/converter/pull_request.go`
**Type:** IMPLEMENTATION
**TDD:** CYCLE-1 (GREEN)
**Estimate:** Medium
**Depends on:** 3.3.1

**Description:**
Implement `ConvertPullRequest(pr *github.PullRequest) (string, error)`:
- Similar structure to ConvertIssue
- PR-specific state handling (merged/closed/open)
- All comments already flattened

**Acceptance Criteria:**
- [ ] All tests from 3.3.1 pass (CYCLE-1 GREEN)
- [ ] Merged/closed/open states correct
- [ ] Review comments formatted properly

---

### 3.4 Discussion Converter

#### Task 3.4.1
**Title:** Write test for ConvertDiscussion
**File:** `internal/converter/discussion_test.go`
**Type:** TEST
**TDD:** CYCLE-1 (RED)
**Estimate:** Medium
**Depends on:** 3.1.5

**Description:**
Write table-driven test for `ConvertDiscussion()`:
- Discussion with answer
- Discussion without answer
- Discussion with category

**Acceptance Criteria:**
- [ ] Test file compiles (will fail initially - CYCLE-1 RED)
- [ ] Validates answer marker
- [ ] Validates category display

---

#### Task 3.4.2
**Title:** Implement ConvertDiscussion method
**File:** `internal/converter/discussion.go`
**Type:** IMPLEMENTATION
**TDD:** CYCLE-1 (GREEN)
**Estimate:** Medium
**Depends on:** 3.4.1

**Description:**
Implement `ConvertDiscussion(discussion *github.Discussion) (string, error)`:
- Similar to Issue converter
- Mark answer comment with "✅ **Accepted Answer**"
- Include category in metadata

**Acceptance Criteria:**
- [ ] All tests from 3.4.1 pass (CYCLE-1 GREEN)
- [ ] Answer comment properly marked
- [ ] Category included in output

---

## Phase 4: CLI Assembly Layer

**Goal:** Integrate all packages into working CLI application

### 4.1 CLI Application

#### Task 4.1.1
**Title:** Write integration test for CLI App
**File:** `internal/cli/app_test.go`
**Type:** TEST
**TDD:** CYCLE-1 (RED)
**Estimate:** Large
**Depends on:** Phase 3 complete

**Description:**
Write integration test for `App.Run()`:
- Test Issue URL → fetch → convert → output
- Test PR URL → fetch → convert → output
- Test Discussion URL → fetch → convert → output
- Test invalid URL error handling
- Test 404 error handling
- Test output to stdout vs file

**Test Cases:**
- End-to-end issue conversion
- End-to-end PR conversion
- End-to-end discussion conversion
- Error cases

**Acceptance Criteria:**
- [ ] Test file compiles (will fail initially - CYCLE-1 RED)
- [ ] Tests full integration flow
- [ ] Uses real GitHub API (skip in short mode)

---

#### Task 4.1.2
**Title:** Implement App orchestrator
**File:** `internal/cli/app.go`
**Type:** IMPLEMENTATION
**TDD:** CYCLE-1 (GREEN)
**Estimate:** Large
**Depends on:** 4.1.1

**Description:**
Implement `App` struct and `Run()` method:
```go
type App struct {
    config   *config.Config
    ghClient *github.Client
    converter *converter.Converter
}

func New(cfg *config.Config) (*App, error) {
    // Initialize all dependencies
}

func (a *App) Run(ctx context.Context) error {
    // 1. Parse URL
    // 2. Fetch data (switch on resource type)
    // 3. Convert to markdown
    // 4. Output (stdout or file)
}
```

**Acceptance Criteria:**
- [ ] All tests from 4.1.1 pass (CYCLE-1 GREEN)
- [ ] Dependency injection via constructor
- [ ] Context cancellation supported
- [ ] All errors wrapped with context

---

#### Task 4.1.3
**Title:** Implement output handling
**File:** `internal/cli/output.go`
**Type:** IMPLEMENTATION
**TDD:** N/A (helper function)
**Estimate:** Small
**Depends on:** 4.1.2

**Description:**
Implement output logic:
- Write to stdout if OutputPath is empty
- Write to file if OutputPath is set
- Handle file write errors

**Acceptance Criteria:**
- [ ] Stdout output works
- [ ] File output works
- [ ] Errors properly wrapped

---

### 4.2 Main Entry Point

#### Task 4.2.1
**Title:** Create main.go with error handling
**File:** `cmd/issue2md/main.go`
**Type:** IMPLEMENTATION
**TDD:** N/A (entry point)
**Estimate:** Small
**Depends on:** 4.1.2

**Description:**
Implement `main()` function:
1. Call `config.LoadConfig(os.Args[1:])`
2. Call `cli.New(cfg)`
3. Call `app.Run(context.Background())`
4. Handle errors with exit code 1
5. Handle help/version with exit code 0

**Error Handling:**
```go
if err != nil {
    fmt.Fprintf(os.Stderr, "Error: %v\n", err)
    os.Exit(1)
}
```

**Acceptance Criteria:**
- [ ] Binary compiles
- [ ] Errors go to stderr
- [ ] Exit codes: 0 (success), 1 (error)

---

## Phase 5: Build Infrastructure

### 5.1 Build System

#### Task 5.1.1 [P]
**Title:** Create go.mod file
**File:** `go.mod`
**Type:** IMPLEMENTATION
**TDD:** N/A
**Estimate:** Small
**Depends on:** None

**Description:**
Initialize Go module with dependencies:
```
module github.com/liangxiaoman/issue2md

go 1.24

require (
    github.com/google/go-github/v68 v68.0.0
    github.com/google/uuid v1.6.0
)
```

**Acceptance Criteria:**
- [ ] `go mod tidy` succeeds
- [ ] Module path correct

---

#### Task 5.1.2 [P]
**Title:** Create Makefile
**File:** `Makefile`
**Type:** IMPLEMENTATION
**TDD:** N/A
**Estimate:** Small
**Depends on:** None

**Description:**
Create Makefile with targets:
- `make build`: Build binary
- `make test`: Run all tests
- `make test-integration`: Run integration tests
- `make test-coverage`: Generate coverage report
- `make clean`: Clean build artifacts
- `make run`: Build and run

**Acceptance Criteria:**
- [ ] All targets work
- [ ] Uses standard Go tools

---

### 5.2 Documentation

#### Task 5.2.1
**Title:** Create README.md
**File:** `README.md`
**Type:** IMPLEMENTATION
**TDD:** N/A
**Estimate:** Medium
**Depends on:** Phase 4 complete

**Description:**
Create comprehensive README with:
- Project description
- Installation instructions
- Usage examples
- Build instructions
- Development setup

**Acceptance Criteria:**
- [ ] Matches spec examples
- [ ] Clear installation guide
- [ ] Usage examples work

---

#### Task 5.2.2
**Title:** Create LICENSE file
**File:** `LICENSE`
**Type:** IMPLEMENTATION
**TDD:** N/A
**Estimate:** Small
**Depends on:** None

**Description:**
Choose and add open-source license (MIT recommended).

**Acceptance Criteria:**
- [ ] License file present
- [ ] Copyright year correct

---

## Task Dependency Graph

```
Phase 1: Foundation
├── 1.1.1 [P] ────┐
├── 1.1.2 ────────→ 1.1.3
├── 1.2.1 [P] ────┐
└── 1.2.2 ────────→ 1.2.3

Phase 2: GitHub Fetcher
├── 2.1.1 [P] ────→ 2.1.2 [P] ────┐
├── 2.2.1 ──────────────────────→ 2.2.2
├── 2.3.1 ──────────────────────→ 2.3.2
└── 2.4.1 ──────────────────────→ 2.4.2

Phase 3: Converter
├── 3.1.1 ────→ 3.1.2 ──→ 3.1.3 ────┐
│            └→ 3.1.4 ──→ 3.1.5 ────┤
├────────────────────────────────→ 3.2.1 ──→ 3.2.2
├────────────────────────────────→ 3.3.1 ──→ 3.3.3
└────────────────────────────────→ 3.4.1 ──→ 3.4.2

Phase 4: CLI Assembly
├── 4.1.1 ──→ 4.1.2 ──→ 4.1.3
└── 4.2.1 (depends on 4.1.2)

Phase 5: Infrastructure
├── 5.1.1 [P]
├── 5.1.2 [P]
├── 5.2.1 (depends on Phase 4)
└── 5.2.2 [P]
```

---

## Execution Strategy

### Parallel Execution Opportunities
1. **Phase 1 Start:** 1.1.1, 1.2.1, 5.1.1, 5.1.2, 5.2.2 can all start in parallel
2. **Phase 2 Start:** 2.1.1 can start in parallel with Phase 1 (no dependencies)
3. **After Phase 1:** 1.1.3 and 1.2.3 can run in parallel

### Suggested Execution Order
1. **Week 1:** Phase 1 + Phase 2 (Foundation + Data Layer)
2. **Week 2:** Phase 3 + Phase 4 (Converter + CLI)
3. **Week 2 End:** Phase 5 (Infrastructure)

### Testing Strategy
- **Unit Tests:** Run with `make test` (fast, no external deps)
- **Integration Tests:** Run with `make test-integration` (requires network)
- **Coverage:** Generate with `make test-coverage`

---

## Total Task Count

- **Phase 1:** 6 tasks (2 test + 2 implementation + 2 parallel setup)
- **Phase 2:** 8 tasks (4 test + 4 implementation)
- **Phase 3:** 12 tasks (6 test + 6 implementation)
- **Phase 4:** 4 tasks (1 test + 3 implementation)
- **Phase 5:** 5 tasks (0 test + 5 implementation)

**Total: 35 atomic tasks**

---

**End of Task List**

This task list is ready for AI execution. Each task is atomic, test-driven (where applicable), and has clear acceptance criteria.
