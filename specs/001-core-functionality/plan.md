# Technical Implementation Plan: Core Functionality

**Version:** 1.0
**Status:** Draft
**Created:** 2025-02-04
**Author:** Chief Architect (AI Agent)

---

## 1. Technical Context Summary

### 1.1 Project Overview
**issue2md** is a command-line tool that converts GitHub Issues, Pull Requests, and Discussions into Markdown format. The tool aims to help developers archive technical discussions, create blog content from issues, and preserve valuable GitHub conversations offline.

### 1.2 Technology Stack

| Component | Technology | Justification |
|-----------|-----------|---------------|
| **Language** | Go >= 1.24 | Required by project constitution; excellent for CLI tools |
| **Web Framework** | `net/http` (std lib) | Follows "simplicity first" principle; no external web framework needed |
| **GitHub API Client** | `google/go-github` v68+ | Official GitHub client library; supports both REST v3 and GraphQL v4 |
| **Markdown Processing** | Standard library + string manipulation | Avoid unnecessary dependencies; Markdown generation is straightforward string formatting |
| **Data Storage** | None | Real-time API fetching; no persistence required |
| **Testing** | `testing` (std lib) | Table-driven tests; integration tests with real GitHub API |
| **CLI Parsing** | `flag` (std lib) | Standard library sufficient for current requirements |

### 1.3 External Dependencies (go.mod)
```
module github.com/liangxiaoman/issue2md

go 1.24

require (
    github.com/google/go-github/v68 v68.0.0
    github.com/google/uuid v1.6.0 // For GraphQL request IDs
)
```

**Note:** `google/go-github` is the only external dependency beyond Go standard library. This aligns with constitution principle 1.2 (standard library first) and 1.3 (avoid over-engineering).

---

## 2. Constitutional Compliance Review

### 2.1 Article I: Simplicity First

| Principle | Compliance Status | Evidence |
|-----------|-------------------|----------|
| **1.1 YAGNI** | ✅ COMPLIANT | Implementation strictly follows `spec.md`; no extra features planned |
| **1.2 Standard Library First** | ✅ COMPLIANT | Using `net/http`, `flag`, `testing`, `fmt`, `strings`; only external deps are `go-github` (essential) |
| **1.3 Anti-Over-Engineering** | ✅ COMPLIANT | Simple functions over complex interfaces; no inheritance hierarchies; data structures are plain structs |

**Verdict:** FULLY COMPLIANT

### 2.2 Article II: Test-First Imperative

| Principle | Compliance Status | Evidence |
|-----------|-------------------|----------|
| **2.1 TDD Cycle** | ✅ COMPLIANT | Implementation order in Section 7 enforces Red-Green-Refactor |
| **2.2 Table-Driven Tests** | ✅ COMPLIANT | All unit tests will use table-driven style (see Section 8) |
| **2.3 No Mocks** | ✅ COMPLIANT | Integration tests with real GitHub API; no mock objects |

**Verdict:** FULLY COMPLIANT

### 2.3 Article III: Clarity and Explicitness

| Principle | Compliance Status | Evidence |
|-----------|-------------------|----------|
| **3.1 Error Handling** | ✅ COMPLIANT | All errors wrapped with `fmt.Errorf("context: %w", err)`; explicit error propagation |
| **3.2 No Global Variables** | ✅ COMPLIANT | All dependencies via struct fields or function parameters; config loaded from env vars explicitly |

**Verdict:** FULLY COMPLIANT

### 2.4 Overall Constitutional Verdict

**🟢 FULLY COMPLIANT** - This technical implementation plan adheres to all principles in `constitution.md`.

---

## 3. Project Structure Refinement

### 3.1 Directory Tree

```
issue2md/
├── cmd/
│   ├── issue2md/
│   │   └── main.go                 # CLI entry point
│   └── issue2mdweb/
│       └── main.go                 # Web entry point (placeholder for future)
│
├── internal/
│   ├── config/
│   │   ├── config.go               # Config struct and loader
│   │   └── config_test.go          # Table-driven tests
│   │
│   ├── parser/
│   │   ├── url.go                  # URL parsing logic
│   │   ├── url_test.go             # Table-driven tests
│   │   └── types.go                # ResourceType enum, GitHubURL struct
│   │
│   ├── github/
│   │   ├── client.go               # GitHub API client
│   │   ├── client_test.go          # Integration tests
│   │   ├── issue.go                # Issue fetcher
│   │   ├── pull_request.go         # PR fetcher
│   │   ├── discussion.go           # Discussion fetcher (GraphQL)
│   │   ├── types.go                # Common data structures
│   │   └── reactions.go            # Reactions helper
│   │
│   ├── converter/
│   │   ├── converter.go            # Main converter struct
│   │   ├── converter_test.go       # Table-driven tests
│   │   ├── issue.go                # Issue → Markdown
│   │   ├── pull_request.go         # PR → Markdown
│   │   ├── discussion.go           # Discussion → Markdown
│   │   ├── frontmatter.go          # YAML frontmatter generation
│   │   ├── comments.go             # Comment formatting
│   │   └── helpers.go              // Timestamp/user/reactions formatting
│   │
│   └── cli/
│       ├── app.go                  # Application orchestrator
│       ├── app_test.go             # Integration tests
│       └── output.go               # Stdout/file output
│
├── web/
│   ├── templates/
│   │   └── index.html              # Web UI (future)
│   └── static/
│       └── style.css               # Styles (future)
│
├── specs/
│   └── 001-core-functionality/
│       ├── spec.md                 # Requirements
│       ├── api-sketch.md           # Interface sketches
│       └── plan.md                 # This file
│
├── .claude/
│   ├── constitution.md             # Project constitution
│   └── CLAUDE.md                   # Project context
│
├── Makefile                        # Build automation
├── go.mod                          # Go modules
├── go.sum                          # Dependency checksums
└── README.md                       # Project documentation
```

### 3.2 Package Responsibilities & Dependencies

#### 3.2.1 `internal/config`
**Responsibility:** Configuration management from CLI flags and environment variables

**Dependencies:** None (std lib only)

**Exports:**
- `Config` struct
- `LoadConfig(args []string) (*Config, error)`

**Used By:** `cmd/issue2md`, `internal/cli`

---

#### 3.2.2 `internal/parser`
**Responsibility:** URL parsing and resource type identification

**Dependencies:** None (std lib only)

**Exports:**
- `ResourceType` enum
- `GitHubURL` struct
- `ParseURL(urlStr string) (*GitHubURL, error)`

**Used By:** `internal/cli`, `internal/github`

**Key Design:** Pure functions with no side effects; easily testable in isolation

---

#### 3.2.3 `internal/github`
**Responsibility:** All GitHub API interactions (data fetching)

**Dependencies:**
- `github.com/google/go-github/v68` (REST API v3)
- `context` (std lib)
- `net/http` (std lib)

**Exports:**
- `Client` struct with timeout and token
- `Issue`, `PullRequest`, `Discussion` data types
- `Comment`, `Actor`, `Reactions` common types
- `FetchIssue()`, `FetchPullRequest()`, `FetchDiscussion()` methods

**Used By:** `internal/cli`, `internal/converter`

**Key Design:**
- Client uses REST v3 for Issues/PRs
- Client uses GraphQL v4 for Discussions (not available in REST v3)
- All methods accept `context.Context` for cancellation
- Reactions fetched only when `includeReactions=true` (API optimization)

---

#### 3.2.4 `internal/converter`
**Responsibility:** Convert GitHub data structures to Markdown

**Dependencies:**
- `internal/github` (data types)
- `fmt`, `strings`, `time` (std lib)

**Exports:**
- `Converter` struct with functional options
- `ConvertIssue()`, `ConvertPR()`, `ConvertDiscussion()` methods
- `WithReactions()`, `WithUserLinks()` option functions

**Used By:** `internal/cli`

**Key Design:**
- Functional options pattern for flexible configuration
- Pure functions: same input → same output
- No side effects; only returns formatted strings

---

#### 3.2.5 `internal/cli`
**Responsibility:** Application orchestration and main flow control

**Dependencies:**
- `internal/config` (configuration)
- `internal/parser` (URL parsing)
- `internal/github` (API client)
- `internal/converter` (Markdown generation)
- `os`, `fmt` (std lib)

**Exports:**
- `App` struct
- `New(cfg *config.Config) (*App, error)`
- `Run(ctx context.Context) error`

**Used By:** `cmd/issue2md`

**Key Design:**
- Dependency injection via constructor
- Clear separation of concerns
- All dependencies explicit

---

### 3.3 Dependency Graph (DAG)

```
cmd/issue2md (main)
    ↓
internal/cli (App)
    ↓
    ├─→ internal/config (Config)
    ├─→ internal/parser (ParseURL)
    ├─→ internal/github (Client, Fetch*)
    │       ↓
    │   internal/converter (Converter)
    └─→ internal/converter (Converter)
            ↓
        internal/github (data types)
```

**Key Property:** No circular dependencies; clear layer separation

---

## 4. Core Data Structures

### 4.1 Parser Types (`internal/parser/types.go`)

```go
package parser

// ResourceType represents the type of GitHub resource
type ResourceType int

const (
    ResourceTypeIssue ResourceType = iota
    ResourceTypePullRequest
    ResourceTypeDiscussion
)

// String returns the string representation
func (rt ResourceType) String() string {
    switch rt {
    case ResourceTypeIssue:
        return "issue"
    case ResourceTypePullRequest:
        return "pull_request"
    case ResourceTypeDiscussion:
        return "discussion"
    default:
        return "unknown"
    }
}

// GitHubURL represents a parsed GitHub URL
type GitHubURL struct {
    Type  ResourceType
    Owner string
    Repo  string
    Number int
}
```

### 4.2 GitHub API Types (`internal/github/types.go`)

```go
package github

import "time"

// Actor represents a GitHub user
type Actor struct {
    Login string
    URL   string // Profile URL
}

// Reactions represents reaction counts
type Reactions struct {
    ThumbsUp   int
    ThumbsDown int
    Laugh      int
    Hooray     int
    Confused   int
    Heart      int
    Eyes       int
    Rocket     int
}

// Comment represents a generic comment
type Comment struct {
    Author            Actor
    Body              string
    CreatedAt         time.Time
    UpdatedAt         *time.Time
    Reactions         *Reactions
    IsReviewComment   bool    // PR-specific
    FilePath          *string // PR-specific: original file path
    LineNumber        *int    // PR-specific: original line number
    IsAnswer          bool    // Discussion-specific
}

// Issue represents a GitHub Issue
type Issue struct {
    Title     string
    URL       string
    Body      string
    State     string // "open", "closed"
    Author    Actor
    CreatedAt time.Time
    ClosedAt  *time.Time
    Labels    []string
    Milestone *string
    Comments  []Comment
    Reactions *Reactions
}

// PullRequest represents a GitHub Pull Request
type PullRequest struct {
    Title     string
    URL       string
    Body      string
    State     string // "open", "closed", "merged"
    Author    Actor
    CreatedAt time.Time
    MergedAt  *time.Time
    ClosedAt  *time.Time
    Labels    []string
    Milestone *string
    Comments  []Comment // Flattened: PR + review comments
    Reactions *Reactions
}

// Discussion represents a GitHub Discussion
type Discussion struct {
    Title     string
    URL       string
    Body      string
    State     string // "open", "closed"
    Author    Actor
    CreatedAt time.Time
    Category  string
    Answer    *Comment // The accepted answer, if any
    Comments  []Comment
    Reactions *Reactions
}
```

### 4.3 Config Types (`internal/config/config.go`)

```go
package config

// Config represents application configuration
type Config struct {
    // Input
    URL string

    // Output
    OutputPath string // Empty = stdout

    // Feature flags
    EnableReactions bool
    EnableUserLinks bool

    // Authentication
    Token string // From GITHUB_TOKEN env var
}
```

### 4.4 Converter Types (`internal/converter/converter.go`)

```go
package converter

import "github.com/liangxiaoman/issue2md/internal/github"

// Option configures the Converter
type Option func(*Converter)

// Converter converts GitHub resources to Markdown
type Converter struct {
    enableReactions bool
    enableUserLinks bool
}

// NewConverter creates a new Converter with options
func NewConverter(options ...Option) *Converter {
    c := &Converter{
        enableReactions: false,
        enableUserLinks: false,
    }
    for _, opt := range options {
        opt(c)
    }
    return c
}

// WithReactions enables reaction counts in output
func WithReactions() Option {
    return func(c *Converter) {
        c.enableReactions = true
    }
}

// WithUserLinks enables GitHub profile links for usernames
func WithUserLinks() Option {
    return func(c *Converter) {
        c.enableUserLinks = true
    }
}

// ConvertIssue converts an Issue to Markdown
func (c *Converter) ConvertIssue(issue *github.Issue) (string, error)

// ConvertPullRequest converts a PullRequest to Markdown
func (c *Converter) ConvertPullRequest(pr *github.PullRequest) (string, error)

// ConvertDiscussion converts a Discussion to Markdown
func (c *Converter) ConvertDiscussion(discussion *github.Discussion) (string, error)
```

### 4.5 CLI Types (`internal/cli/app.go`)

```go
package cli

import (
    "context"
    "github.com/liangxiaoman/issue2md/internal/config"
    "github.com/liangxiaoman/issue2md/internal/github"
)

// App represents the CLI application
type App struct {
    config   *config.Config
    ghClient *github.Client
}

// New creates a new App instance
func New(cfg *config.Config) (*App, error) {
    client := github.NewClient(cfg.Token, 30*time.Second)
    return &App{
        config:   cfg,
        ghClient: client,
    }, nil
}

// Run executes the main application flow
func (a *App) Run(ctx context.Context) error
```

---

## 5. Interface Design

### 5.1 Public Interfaces (Package Exports)

#### 5.1.1 `internal/parser`

```go
// ParseURL parses a GitHub URL and returns structured information
func ParseURL(urlStr string) (*GitHubURL, error)

// IsValid validates if the parsed URL is valid
func (u *GitHubURL) IsValid() bool

// String returns the string representation of ResourceType
func (rt ResourceType) String() string
```

**Design Principles:**
- Pure function (no side effects)
- Early validation and clear errors
- Simple return types

---

#### 5.1.2 `internal/github`

```go
// NewClient creates a new GitHub API client
func NewClient(token string, timeout time.Duration) *Client

// SetToken updates the authentication token
func (c *Client) SetToken(token string)

// FetchIssue retrieves an issue with comments
func (c *Client) FetchIssue(ctx context.Context, owner, repo string, number int, includeReactions bool) (*Issue, error)

// FetchPullRequest retrieves a PR with all comments (flattened)
func (c *Client) FetchPullRequest(ctx context.Context, owner, repo string, number int, includeReactions bool) (*PullRequest, error)

// FetchDiscussion retrieves a discussion with replies
func (c *Client) FetchDiscussion(ctx context.Context, owner, repo string, number int, includeReactions bool) (*Discussion, error)
```

**Design Principles:**
- Context support for cancellation
- Explicit error wrapping
- Reactions fetched on-demand (performance optimization)
- PR comments already flattened (single source of truth)

---

#### 5.1.3 `internal/converter`

```go
// NewConverter creates a new converter with options
func NewConverter(options ...Option) *Converter

// WithReactions enables reaction counts
func WithReactions() Option

// WithUserLinks enables user profile links
func WithUserLinks() Option

// ConvertIssue converts an Issue to Markdown
func (c *Converter) ConvertIssue(issue *github.Issue) (string, error)

// ConvertPullRequest converts a PR to Markdown
func (c *Converter) ConvertPullRequest(pr *github.PullRequest) (string, error)

// ConvertDiscussion converts a Discussion to Markdown
func (c *Converter) ConvertDiscussion(discussion *github.Discussion) (string, error)
```

**Design Principles:**
- Functional options pattern (flexibility)
- Pure functions (testability)
- Single responsibility per method

---

#### 5.1.4 `internal/config`

```go
// LoadConfig loads configuration from args and environment
func LoadConfig(args []string) (*Config, error)
```

**Design Principles:**
- Single entry point
- Environment variable reading encapsulated
- Validation happens early

---

#### 5.1.5 `internal/cli`

```go
// New creates a new application instance
func New(cfg *config.Config) (*App, error)

// Run executes the main application flow
func (a *App) Run(ctx context.Context) error
```

**Design Principles:**
- Dependency injection via constructor
- Context-based cancellation
- Clear error propagation

---

### 5.2 Internal Helper Interfaces

#### 5.2.1 Markdown Generation Helpers (`internal/converter/helpers.go`)

```go
// generateFrontmatter creates YAML frontmatter
func (c *Converter) generateFrontmatter(data interface{}) (string, error)

// formatComment formats a single comment
func (c *Converter) formatComment(comment github.Comment) string

// formatReactions formats reaction counts
func (c *Converter) formatReactions(r *github.Reactions) string

// formatTimestamp formats a time as UTC
func formatTimestamp(t time.Time) string

// formatUser formats a username (with optional link)
func (c *Converter) formatUser(actor github.Actor) string

// getStateEmoji returns emoji for state
func getStateEmoji(state string) string
```

**Note:** These are unexported (lowercase) - internal use only

---

## 6. API Interaction Strategy

### 6.1 REST API v3 (Issues & Pull Requests)

**Library:** `github.com/google/go-github/v68`

#### 6.1.1 Issue Fetching

```go
// REST Endpoint: GET /repos/{owner}/{repo}/issues/{number}
issue, _, err := client.Issues.Get(ctx, owner, repo, number)

// Comments: GET /repos/{owner}/{repo}/issues/{number}/comments
comments, _, err := client.Issues.ListComments(ctx, owner, repo, number, nil)

// Reactions: GET /repos/{owner}/{repo}/issues/{number}/reactions
reactions, _, err := client.Reactions.ListIssueReactions(ctx, owner, repo, number)
```

#### 6.1.2 Pull Request Fetching

```go
// REST Endpoint: GET /repos/{owner}/{repo}/pulls/{number}
pr, _, err := client.PullRequests.Get(ctx, owner, repo, number)

// PR Comments: GET /repos/{owner}/{repo}/pulls/{number}/comments
prComments, _, err := client.PullRequests.ListComments(ctx, owner, repo, number, nil)

// Review Comments: GET /repos/{owner}/{repo}/pulls/{number}/reviews
reviews, _, err := client.PullRequests.ListReviews(ctx, owner, repo, number, nil)

// Merge both comment lists and sort by timestamp
```

**Key Logic:** Flattening PR comments
```go
func (c *Client) FetchPullRequest(...) (*PullRequest, error) {
    // 1. Fetch PR metadata
    pr, _, err := c.client.PullRequests.Get(ctx, owner, repo, number)
    if err != nil {
        return nil, fmt.Errorf("get PR: %w", err)
    }

    // 2. Fetch PR comments
    prComments, err := c.fetchAllPRComments(ctx, owner, repo, number)
    if err != nil {
        return nil, fmt.Errorf("fetch PR comments: %w", err)
    }

    // 3. Fetch review comments
    reviewComments, err := c.fetchAllReviewComments(ctx, owner, repo, number)
    if err != nil {
        return nil, fmt.Errorf("fetch review comments: %w", err)
    }

    // 4. Merge and sort by timestamp
    allComments := append(prComments, reviewComments...)
    sort.Slice(allComments, func(i, j int) bool {
        return allComments[i].CreatedAt.Before(allComments[j].CreatedAt)
    })

    // 5. Build and return PullRequest struct
    return &PullRequest{
        Comments: allComments,
        // ... other fields
    }, nil
}
```

---

### 6.2 GraphQL API v4 (Discussions)

**Why GraphQL?** Discussions are not available in GitHub REST API v3

#### 6.2.2 GraphQL Query

```graphql
query GetDiscussion($owner: String!, $repo: String!, $number: Int!) {
    repository(owner: $owner, name: $repo) {
        discussion(number: $number) {
            title
            url
            body
            state
            author {
                login
                url
            }
            createdAt
            category {
                name
            }
            answer {
                author { login, url }
                body
                createdAt
            }
            comments(first: 100) {
                nodes {
                    author { login, url }
                    body
                    createdAt
                    isAnswer
                }
            }
        }
    }
}
```

**Implementation:**
```go
func (c *Client) FetchDiscussion(ctx context.Context, owner, repo string, number int, includeReactions bool) (*Discussion, error) {
    // Use graphql.NewClient from go-github
    // Execute query with variables
    // Parse response into Discussion struct
}
```

**Pagination Note:** For discussions with >100 comments, implement pagination using `after` cursor

---

### 6.3 Error Handling Strategy

```go
// All API calls follow this pattern:
resp, err := client.Resource.Get(ctx, owner, repo, number)
if err != nil {
    // Check for rate limit
    if resp != nil && resp.StatusCode == http.StatusForbidden {
        return nil, fmt.Errorf("API rate limit exceeded: %w", err)
    }

    // Check for not found
    if resp != nil && resp.StatusCode == http.StatusNotFound {
        return nil, fmt.Errorf("resource not found (404): %w", err)
    }

    // Generic error
    return nil, fmt.Errorf("fetch resource: %w", err)
}
```

---

## 7. Implementation Order

Following TDD principles and dependency analysis, implementation proceeds in this order:

### Phase 1: Foundation (Bottom Layer)

1. ✅ **`internal/parser`** (Week 1, Days 1-2)
   - Implement `ParseURL()` with table-driven tests
   - Test coverage: valid URLs, invalid URLs, edge cases
   - No external dependencies → easy to test

2. ✅ **`internal/config`** (Week 1, Days 2-3)
   - Implement `LoadConfig()` with table-driven tests
   - Test flag parsing, environment variable reading
   - Validate config early

### Phase 2: Data Layer

3. ✅ **`internal/github`** (Week 1, Days 3-5)
   - Implement `Client` struct
   - Implement `FetchIssue()` with integration tests
   - Implement `FetchPullRequest()` with comment flattening
   - Implement `FetchDiscussion()` with GraphQL
   - Test with real GitHub API (use test repository)

### Phase 3: Conversion Layer

4. ✅ **`internal/converter`** (Week 2, Days 1-3)
   - Implement `Converter` with functional options
   - Implement `ConvertIssue()` with table-driven tests
   - Implement `ConvertPR()` with table-driven tests
   - Implement `ConvertDiscussion()` with table-driven tests
   - Test with mock data and real API responses

### Phase 4: Application Layer

5. ✅ **`internal/cli`** (Week 2, Days 3-4)
   - Implement `App` orchestrator
   - Implement `Run()` method
   - Integration tests with all layers

6. ✅ **`cmd/issue2md`** (Week 2, Day 5)
   - Implement `main.go`
   - Add help/version flags
   - End-to-end testing

### Phase 5: Documentation & Polish

7. ✅ **README, Makefile** (Week 2, Day 5)
   - Write usage documentation
   - Add build targets
   - Final verification against spec

---

## 8. Testing Strategy

### 8.1 Table-Driven Test Template

```go
func TestParseURL(t *testing.T) {
    tests := []struct {
        name    string
        url     string
        want    *parser.GitHubURL
        wantErr bool
        errMsg  string
    }{
        {
            name: "valid issue URL",
            url:  "https://github.com/golang/go/issues/12345",
            want: &parser.GitHubURL{
                Type:   parser.ResourceTypeIssue,
                Owner:  "golang",
                Repo:   "go",
                Number: 12345,
            },
            wantErr: false,
        },
        {
            name:    "invalid URL - missing protocol",
            url:     "github.com/owner/repo/issues/123",
            want:    nil,
            wantErr: true,
            errMsg:  "invalid URL format",
        },
        {
            name:    "invalid URL - not github.com",
            url:     "https://gitlab.com/owner/repo/issues/123",
            want:    nil,
            wantErr: true,
            errMsg:  "not a GitHub URL",
        },
        // ... more test cases
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := parser.ParseURL(tt.url)

            if tt.wantErr {
                if err == nil {
                    t.Errorf("ParseURL() expected error, got nil")
                    return
                }
                // Check error message
                if tt.errMsg != "" && !strings.Contains(err.Error(), tt.errMsg) {
                    t.Errorf("ParseURL() error = %v, want contain %v", err, tt.errMsg)
                }
                return
            }

            if err != nil {
                t.Errorf("ParseURL() unexpected error: %v", err)
                return
            }

            if !reflect.DeepEqual(got, tt.want) {
                t.Errorf("ParseURL() = %+v, want %+v", got, tt.want)
            }
        })
    }
}
```

### 8.2 Integration Test Template

```go
func TestClient_FetchIssue_Integration(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test in short mode")
    }

    // Use a real, stable test repository
    client := NewClient("", 30*time.Second)

    issue, err := client.FetchIssue(context.Background(), "golang", "go", 1, false)
    if err != nil {
        t.Fatalf("FetchIssue() error = %v", err)
    }

    // Validate structure
    if issue.Title == "" {
        t.Error("Issue title is empty")
    }

    if len(issue.Comments) == 0 {
        t.Log("No comments found (this is OK for some issues)")
    }
}
```

**Note:** Integration tests marked with `+build integration` or skipped in short mode

---

## 9. Error Handling Catalog

### 9.1 Error Categories

| Category | Example | Handling |
|----------|---------|----------|
| **Invalid Input** | Malformed URL | Return error immediately with clear message |
| **Network Error** | Timeout, DNS failure | Wrap with context, return to caller |
| **API Error (404)** | Resource not found | Return specific "not found" error |
| **API Error (403)** | Rate limit exceeded | Forward GitHub error message |
| **API Error (5xx)** | GitHub server error | Return with context for retry |

### 9.2 Error Message Format

```go
// Consistent error wrapping pattern
return nil, fmt.Errorf("fetch issue: %w", err)

// With additional context
return nil, fmt.Errorf("fetch issue %s/%s#%d: %w", owner, repo, number, err)
```

---

## 10. Performance Considerations

### 10.1 HTTP Timeouts

```go
// Default timeout for all API requests
const defaultTimeout = 30 * time.Second

client := &http.Client{
    Timeout: defaultTimeout,
}
```

### 10.2 Context Cancellation

```go
// Support graceful cancellation
func (a *App) Run(ctx context.Context) error {
    // All child calls inherit ctx
    issue, err := a.ghClient.FetchIssue(ctx, owner, repo, number)
    // ...
}
```

### 10.3 Memory Efficiency

- Stream Markdown output to `io.Writer` (avoid large string concatenation)
- Use pagination for large comment threads (>100 comments)
- Release HTTP responses immediately after reading

---

## 11. Security Considerations

### 11.1 Token Handling

```go
// Token only from environment variable
token := os.Getenv("GITHUB_TOKEN")

// NEVER accept token via CLI flag (prevents shell history leakage)
```

### 11.2 URL Validation

```go
// Validate URL before making API calls
if !strings.HasPrefix(url, "https://github.com/") {
    return nil, fmt.Errorf("invalid URL: must be https://github.com/")
}
```

### 11.3 No Code Execution

- Tool only reads data; no code execution
- No file writes except user-specified output path

---

## 12. Future Extensibility

### 12.1 Web Interface (Phase 2)

The current architecture supports easy addition of a web interface:

```go
// cmd/issue2mdweb/main.go
func main() {
    // Reuse internal packages
    // http.HandleFunc("/", handleForm)
    // http.ListenAndServe(":8080", nil)
}

func handleForm(w http.ResponseWriter, r *http.Request) {
    url := r.FormValue("url")

    // Reuse parser
    parsed, err := parser.ParseURL(url)

    // Reuse GitHub client
    client := github.NewClient(os.Getenv("GITHUB_TOKEN"), 30*time.Second)

    // Reuse converter
    converter := converter.NewConverter()

    // Return markdown
    fmt.Fprint(w, markdown)
}
```

### 12.2 Custom Templates (Phase 3)

```go
// future: internal/converter/template.go
type TemplateConverter struct {
    template *template.Template
}

func (tc *TemplateConverter) ConvertIssue(issue *github.Issue) (string, error) {
    // Use Go templates for user-defined layouts
}
```

---

## 13. Verification Checklist

Before implementation begins, verify:

- [ ] All constitutional principles are followed
- [ ] Technology stack is finalized
- [ ] Data structures cover all spec requirements
- [ ] Interfaces are clearly defined
- [ ] Dependency graph has no cycles
- [ ] Testing strategy is established
- [ ] Error handling approach is consistent
- [ ] Security considerations are addressed

---

## 14. Appendix

### 14.1 GraphQL Query for Discussions

```graphql
query GetDiscussion($owner: String!, $repo: String!, $number: Int!) {
    repository(owner: $owner, name: $repo) {
        discussion(number: $number) {
            title
            url
            body
            closed
            author {
                login
                url
            }
            createdAt
            category {
                name
            }
            answer {
                author { login url }
                body
                createdAt
            }
            comments(first: 100) {
                nodes {
                    author { login url }
                    body
                    createdAt
                    isAnswer
                    reactions(first: 100) {
                        nodes {
                            content
                        }
                    }
                }
                pageInfo {
                    hasNextPage
                    endCursor
                }
            }
        }
    }
}
```

### 14.2 Makefile Template

```makefile
.PHONY: test build clean run

# Go parameters
GOCMD=go
GOBUILD=$(GOCMD) build
GOTEST=$(GOCMD) test
GOCLEAN=$(GOCMD) clean

# Binary name
BINARY_NAME=issue2md

build:
	$(GOBUILD) -o $(BINARY_NAME) -v ./cmd/issue2md

test:
	$(GOTEST) -v ./...

test-integration:
	$(GOTEST) -v -tags=integration ./...

test-coverage:
	$(GOTEST) -coverprofile=coverage.out ./...
	$(GOCMD) tool cover -html=coverage.out

clean:
	$(GOCLEAN)
	rm -f $(BINARY_NAME)
	rm -f coverage.out

run:
	$(GOBUILD) -o $(BINARY_NAME) -v ./cmd/issue2md
	./$(BINARY_NAME)
```

---

**Document End**

This technical implementation plan is ready for review. Upon approval, development will proceed according to the implementation order defined in Section 7.
