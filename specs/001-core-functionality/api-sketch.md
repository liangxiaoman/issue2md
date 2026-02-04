# API Sketch: Core Functionality

**Version:** 1.0
**Status:** Draft
**Last Updated:** 2025-02-04

本文档描述 `internal/` 下各包的核心接口设计，作为后续开发的参考。

---

## Package: `internal/parser`

**职责**: URL 解析与类型识别

### Type: `ResourceType`
```go
type ResourceType int

const (
    ResourceTypeIssue ResourceType = iota
    ResourceTypePullRequest
    ResourceTypeDiscussion
)

func (rt ResourceType) String() string
```

### Type: `GitHubURL`
```go
// GitHubURL 表示解析后的 GitHub URL
type GitHubURL struct {
    Type        ResourceType
    Owner       string
    Repo        string
    Number      int
    // Discussion 特有字段（仅当 Type == ResourceTypeDiscussion 时有意义）
    Organization string // for org-level discussions
    Team         string // for team discussions
}

// IsValid 验证 URL 是否有效
func (u *GitHubURL) IsValid() bool
```

### Function: `ParseURL`
```go
// ParseURL 解析 GitHub URL 并返回结构化信息
//
// Examples:
//   - https://github.com/golang/go/issues/12345
//   - https://github.com/owner/repo/pull/456
//   - https://github.com/org/community/discussions/78
func ParseURL(urlStr string) (*GitHubURL, error)
```

---

## Package: `internal/github`

**职责**: GitHub API 交互（数据获取）

### Type: `Client`
```go
// Client 封装 GitHub API 客户端
type Client struct {
    token      string // Personal Access Token (可选)
    httpClient *http.Client
    timeout    time.Duration
}

// NewClient 创建新的 GitHub API 客户端
// token 为空字符串时，使用未认证模式
func NewClient(token string, timeout time.Duration) *Client

// SetToken 设置或更新认证 token
func (c *Client) SetToken(token string)
```

### Type: `Issue`
```go
// Issue 表示 GitHub Issue 数据
type Issue struct {
    Title        string
    URL          string
    Body         string
    State        string // "open", "closed"
    Author       Actor
    CreatedAt    time.Time
    ClosedAt     *time.Time
    Labels       []string
    Milestone    *string
    Comments     []Comment
    Reactions    *Reactions // 可选
}

// FetchIssue 获取 Issue 数据（包括评论）
func (c *Client) FetchIssue(ctx context.Context, owner, repo string, number int) (*Issue, error)
```

### Type: `PullRequest`
```go
// PullRequest 表示 GitHub PR 数据
type PullRequest struct {
    Title        string
    URL          string
    Body         string
    State        string // "open", "closed", "merged"
    Author       Actor
    CreatedAt    time.Time
    MergedAt     *time.Time
    ClosedAt     *time.Time
    Labels       []string
    Milestone    *string
    Comments     []Comment // 包括 PR comments 和 review comments（已扁平化）
    Reactions    *Reactions // 可选
}

// FetchPullRequest 获取 PR 数据（包括评论和 review comments）
// 返回的 Comments 已经按时间戳排序，包含所有类型的评论
func (c *Client) FetchPullRequest(ctx context.Context, owner, repo string, number int, includeReactions bool) (*PullRequest, error)
```

### Type: `Discussion`
```go
// Discussion 表示 GitHub Discussion 数据
type Discussion struct {
    Title       string
    URL         string
    Body        string
    State       string // "open", "closed"
    Author      Actor
    CreatedAt   time.Time
    Category    string
    Answer      *Comment // 被标记为答案的评论（如果有）
    Comments    []Comment
    Reactions   *Reactions // 可选
}

// FetchDiscussion 获取 Discussion 数据（包括回复）
func (c *Client) FetchDiscussion(ctx context.Context, owner, repo string, number int, includeReactions bool) (*Discussion, error)
```

### Common Types

```go
// Actor 表示 GitHub 用户
type Actor struct {
    Login string
    URL   string // GitHub profile URL
}

// Comment 表示评论内容
type Comment struct {
    Author      Actor
    Body        string
    CreatedAt   time.Time
    UpdatedAt   *time.Time
    Reactions   *Reactions // 可选
    // PR Review Comments 特有字段
    IsReviewComment bool
    FilePath        *string // 原始文件路径
    LineNumber      *int    // 原始行号
}

// Reactions 表示反应统计
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

// URL represents GitHub URL components
type URL struct {
    Type        ResourceType
    Owner       string
    Repo        string
    Number      int
}
```

---

## Package: `internal/converter`

**职责**: 将 GitHub 数据转换为 Markdown

### Type: `Converter`
```go
// Converter 负责将 GitHub 资源转换为 Markdown
type Converter struct {
    enableReactions  bool
    enableUserLinks  bool
}

// NewConverter 创建新的转换器
func NewConverter(options ...Option) *Converter

// Option 配置转换器的函数式选项
type Option func(*Converter)

// WithReactions 启用 reactions
func WithReactions() Option

// WithUserLinks 启用用户名链接
func WithUserLinks() Option
```

### Methods: `Converter`

```go
// ConvertIssue 将 Issue 转换为 Markdown
func (c *Converter) ConvertIssue(issue *github.Issue) (string, error)

// ConvertPullRequest 将 PR 转换为 Markdown
func (c *Converter) ConvertPullRequest(pr *github.PullRequest) (string, error)

// ConvertDiscussion 将 Discussion 转换为 Markdown
func (c *Converter) ConvertDiscussion(discussion *github.Discussion) (string, error)
```

### Helper Functions (Internal Use)

```go
// generateFrontmatter 生成 YAML frontmatter
func (c *Converter) generateFrontmatter(data interface{}) (string, error)

// formatComment 格式化单条评论
func (c *Converter) formatComment(comment github.Comment, enableReactions bool) string

// formatReactions 格式化 reactions 统计
func (c *Converter) formatReactions(r *github.Reactions) string

// formatTimestamp 格式化时间戳
func formatTimestamp(t time.Time) string

// formatUser 格式化用户名（可选链接）
func (c *Converter) formatUser(actor github.Actor) string
```

---

## Package: `internal/config`

**职责**: 配置管理

### Type: `Config`
```go
// Config 应用配置
type Config struct {
    // 输入
    URL string

    // 输出
    OutputPath string // 空字符串表示输出到 stdout

    // 功能开关
    EnableReactions  bool
    EnableUserLinks  bool

    // 认证
    Token string // 从环境变量 GITHUB_TOKEN 读取
}

// LoadConfig 从命令行参数和环境变量加载配置
func LoadConfig(args []string) (*Config, error)
```

---

## Package: `internal/cli`

**职责**: 命令行界面与主流程编排

### Type: `App`
```go
// App 代表 CLI 应用
type App struct {
    config   *config.Config
    ghClient *github.Client
}

// New 创建新的应用实例
func New(cfg *config.Config) (*App, error)

// Run 执行主流程
func (a *App) Run(ctx context.Context) error
```

### Main Flow (Pseudo-code)

```go
func (a *App) Run(ctx context.Context) error {
    // 1. 解析 URL
    parsedURL, err := parser.ParseURL(a.config.URL)
    if err != nil {
        return fmt.Errorf("invalid URL: %w", err)
    }

    // 2. 创建 GitHub 客户端
    a.ghClient = github.NewClient(a.config.Token, 30*time.Second)

    // 3. 根据 URL 类型获取数据
    var markdown string
    converter := converter.NewConverter(
        converter.WithReactions(a.config.EnableReactions),
        converter.WithUserLinks(a.config.EnableUserLinks),
    )

    switch parsedURL.Type {
    case parser.ResourceTypeIssue:
        issue, err := a.ghClient.FetchIssue(ctx, parsedURL.Owner, parsedURL.Repo, parsedURL.Number)
        if err != nil {
            return fmt.Errorf("fetch issue: %w", err)
        }
        markdown, err = converter.ConvertIssue(issue)

    case parser.ResourceTypePullRequest:
        pr, err := a.ghClient.FetchPullRequest(ctx, parsedURL.Owner, parsedURL.Repo, parsedURL.Number, a.config.EnableReactions)
        if err != nil {
            return fmt.Errorf("fetch pull request: %w", err)
        }
        markdown, err = converter.ConvertPR(pr)

    case parser.ResourceTypeDiscussion:
        discussion, err := a.ghClient.FetchDiscussion(ctx, parsedURL.Owner, parsedURL.Repo, parsedURL.Number, a.config.EnableReactions)
        if err != nil {
            return fmt.Errorf("fetch discussion: %w", err)
        }
        markdown, err = converter.ConvertDiscussion(discussion)

    default:
        return fmt.Errorf("unsupported resource type: %s", parsedURL.Type)
    }

    if err != nil {
        return fmt.Errorf("convert to markdown: %w", err)
    }

    // 4. 输出结果
    return a.output(markdown)
}

// output 输出到文件或 stdout
func (a *App) output(markdown string) error {
    if a.config.OutputPath == "" {
        fmt.Println(markdown)
        return nil
    }

    return os.WriteFile(a.config.OutputPath, []byte(markdown), 0644)
}
```

---

## Package: `cmd/issue2md`

**职责**: CLI 入口点

### Function: `main`
```go
func main() {
    // 1. 加载配置
    cfg, err := config.LoadConfig(os.Args[1:])
    if err != nil {
        fmt.Fprintf(os.Stderr, "Error: %v\n", err)
        os.Exit(1)
    }

    // 2. 创建应用
    app, err := cli.New(cfg)
    if err != nil {
        fmt.Fprintf(os.Stderr, "Error: %v\n", err)
        os.Exit(1)
    }

    // 3. 运行
    ctx := context.Background()
    if err := app.Run(ctx); err != nil {
        fmt.Fprintf(os.Stderr, "Error: %v\n", err)
        os.Exit(1)
    }
}
```

---

## Package: `cmd/issue2mdweb`

**职责**: Web 服务入口点（Future）

> NOTE: 此包在 MVP 阶段不实现，预留接口供后续开发

```go
// Placeholder for future web interface
func main() {
    // TODO: 启动 HTTP 服务器
    // TODO: 提供表单界面
    // TODO: 复用 internal/ 下的所有包
}
```

---

## Error Handling

所有包中的错误都应该：
1. 使用 `fmt.Errorf("context: %w", err)` 包装底层错误
2. 提供清晰的上下文信息
3. 支持错误链解包（`errors.Is` / `errors.As`）

### Example
```go
func (c *Client) FetchIssue(ctx context.Context, owner, repo string, number int) (*Issue, error) {
    url := fmt.Sprintf("https://api.github.com/repos/%s/%s/issues/%d", owner, repo, number)

    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return nil, fmt.Errorf("create request: %w", err)
    }

    if c.token != "" {
        req.Header.Set("Authorization", fmt.Sprintf("Bearer %s", c.token))
    }

    resp, err := c.httpClient.Do(req)
    if err != nil {
        return nil, fmt.Errorf("execute request: %w", err)
    }
    defer resp.Body.Close()

    if resp.StatusCode == http.StatusNotFound {
        return nil, fmt.Errorf("issue not found (404)")
    }

    if resp.StatusCode != http.StatusOK {
        return nil, fmt.Errorf("unexpected status code: %d", resp.StatusCode)
    }

    // ... parse response ...
}
```

---

## Testing Strategy

根据 `constitution.md` 要求：
1. **Test-First**: 先写测试，再写实现
2. **Table-Driven Tests**: 优先使用表格驱动测试
3. **No Mocks**: 优先集成测试，使用真实 GitHub API

### Example: `internal/parser` Table-Driven Test

```go
func TestParseURL(t *testing.T) {
    tests := []struct {
        name    string
        url     string
        want    *GitHubURL
        wantErr bool
    }{
        {
            name: "valid issue URL",
            url:  "https://github.com/golang/go/issues/12345",
            want: &GitHubURL{
                Type:  ResourceTypeIssue,
                Owner: "golang",
                Repo:  "go",
                Number: 12345,
            },
            wantErr: false,
        },
        {
            name:    "invalid URL",
            url:     "not-a-url",
            want:    nil,
            wantErr: true,
        },
        // ... more test cases
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := ParseURL(tt.url)
            if (err != nil) != tt.wantErr {
                t.Errorf("ParseURL() error = %v, wantErr %v", err, tt.wantErr)
                return
            }
            if !reflect.DeepEqual(got, tt.want) {
                t.Errorf("ParseURL() = %v, want %v", got, tt.want)
            }
        })
    }
}
```

---

## Implementation Order

基于依赖关系和测试优先原则，建议实现顺序：

1. ✅ **`internal/parser`** - 最底层，无外部依赖，易于测试
2. ✅ **`internal/config`** - 配置解析，独立性强
3. ✅ **`internal/github`** - API 客户端，可独立测试
4. ✅ **`internal/converter`** - 转换逻辑，依赖 github 包
5. ✅ **`internal/cli`** - 流程编排，依赖以上所有包
6. ✅ **`cmd/issue2md`** - 入口点，最简单

每一步都遵循：**测试先行 → 实现功能 → 重构优化**
