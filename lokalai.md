# lokalai — Local LLM Coding Assistant CLI

> A production-grade, cross-platform CLI coding assistant powered entirely by local Ollama models. No API keys. No cloud. No data leaving your machine.

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Why lokalai?](#why-lokalai)
3. [Architecture](#architecture)
4. [Tech Stack](#tech-stack)
5. [Project Structure](#project-structure)
6. [CLI Commands Reference](#cli-commands-reference)
7. [Ollama API Integration](#ollama-api-integration)
8. [Streaming Implementation](#streaming-implementation)
9. [Cross-Platform Support](#cross-platform-support)
10. [Configuration System](#configuration-system)
11. [Session & History Management](#session--history-management)
12. [TUI Design](#tui-design)
13. [File Context Injection](#file-context-injection)
14. [Project Context Auto-Detection](#project-context-auto-detection)
15. [Multi-Model Routing](#multi-model-routing)
16. [Slash Commands](#slash-commands)
17. [Shell Integration](#shell-integration)
18. [Build System & GoReleaser](#build-system--goreleaser)
19. [GitHub Actions CI/CD](#github-actions-cicd)
20. [Installation Guide](#installation-guide)
21. [Cursor Prompt](#cursor-prompt)
22. [Roadmap](#roadmap)
23. [Testing Strategy & Test Cases](#testing-strategy--test-cases)
24. [Contributing](#contributing)

---

## Project Overview

**lokalai** is an open-source CLI tool that brings the experience of Gemini CLI and Qwen Code CLI to your local machine — but instead of sending your code to the cloud, it talks exclusively to models running in [Ollama](https://ollama.com) on your own hardware.

It is designed to feel fast, modern, and developer-native. You can ask it questions, have it explain errors, refactor files, write boilerplate, or just chat with your favourite local model — all from the terminal.

**Key principles:**

- **Privacy first** — All inference runs locally. Nothing is sent to any external server.
- **Model agnostic** — Works with any model available in Ollama (Llama 3, Qwen2.5-Coder, DeepSeek-Coder, Mistral, Phi-3, etc.)
- **Cross-platform** — Single binary for Windows (`.exe`) and Linux (amd64 + arm64)
- **Developer-native UX** — Syntax highlighting, markdown rendering, streaming output, file context injection
- **Extensible** — Built around a clean `LLMProvider` interface so other backends (OpenAI-compatible, LM Studio) can be added later

---

## Why lokalai?

| Feature | lokalai | Gemini CLI | Qwen Code CLI |
|---|---|---|---|
| Runs 100% locally | ✅ | ❌ | ❌ |
| No API key required | ✅ | ❌ | ❌ |
| Works offline | ✅ | ❌ | ❌ |
| Supports any Ollama model | ✅ | ❌ | ❌ |
| Cross-platform binary | ✅ | ✅ | ✅ |
| Open source | ✅ | ✅ | ✅ |
| File context injection | ✅ | ✅ | ✅ |
| Streaming output | ✅ | ✅ | ✅ |

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                      User Terminal                       │
│              (cmd.exe / PowerShell / bash / zsh)         │
└───────────────────────────┬─────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────┐
│                    lokalai Binary                        │
│                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐  │
│  │  cobra   │  │bubbletea │  │  viper   │  │glamour │  │
│  │  (cmds)  │  │  (TUI)   │  │ (config) │  │(render)│  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └───┬────┘  │
│       └─────────────┴─────────────┴─────────────┘       │
│                            │                             │
│              ┌─────────────▼──────────────┐             │
│              │       LLMProvider           │             │
│              │       (interface)           │             │
│              └─────────────┬──────────────┘             │
│                            │                             │
│              ┌─────────────▼──────────────┐             │
│              │      OllamaClient          │             │
│              │  (HTTP + SSE streaming)    │             │
│              └─────────────┬──────────────┘             │
└────────────────────────────┼────────────────────────────┘
                             │  HTTP (localhost:11434)
                             ▼
┌─────────────────────────────────────────────────────────┐
│                    Ollama Daemon                          │
│         (llama3, qwen2.5-coder, deepseek, etc.)          │
└─────────────────────────────────────────────────────────┘
```

---

## Tech Stack

| Layer | Library | Purpose |
|---|---|---|
| CLI framework | `cobra` | Commands, flags, help text |
| TUI engine | `bubbletea` | Interactive REPL, key handling |
| TUI styling | `lipgloss` | Colors, borders, layout |
| Markdown rendering | `glamour` | Code blocks, formatting |
| Config management | `viper` | YAML config + env vars |
| Terminal detection | `golang.org/x/term` | TTY check, raw mode |
| HTTP client | `net/http` (stdlib) | Ollama REST API calls |
| JSON handling | `encoding/json` (stdlib) | Request/response parsing |
| File paths | `path/filepath` (stdlib) | Cross-platform paths |
| Build & release | `goreleaser` | Multi-platform binary releases |

**Why all stdlib for HTTP/JSON?**
Ollama's API is simple REST + JSON streaming. Adding `resty` or `gorequest` would be unnecessary weight. The stdlib `net/http` with a streaming body reader is perfectly sufficient and keeps the binary lean.

---

## Project Structure

```
lokalai/
│
├── main.go                        # Entry point
│
├── cmd/                           # Cobra command definitions
│   ├── root.go                    # Root command, global flags
│   ├── ask.go                     # `lokalai ask "prompt"`
│   ├── chat.go                    # `lokalai chat` (REPL)
│   ├── models.go                  # `lokalai models`
│   ├── run.go                     # `lokalai run <file>`
│   ├── config.go                  # `lokalai config set/get`
│   ├── root_test.go               # CLI flag tests
│   ├── ask_test.go
│   ├── config_test.go
│   └── models_test.go
│
├── e2e/
│   └── cli_integration_test.go    # //go:build integration — live Ollama E2E
│
├── internal/
│   ├── testutil/
│   │   ├── ollama.go              # RequireOllama, TestModel helpers
│   │   └── fixtures/              # sample go.mod, package.json, config.yaml
│   │
│   ├── ollama/
│   │   ├── client.go              # HTTP client, API methods
│   │   ├── stream.go              # SSE/chunked stream reader
│   │   ├── models.go              # Model list, tag structs
│   │   ├── types.go               # Request/response types
│   │   ├── client_test.go         # mock HTTP unit tests
│   │   ├── stream_test.go
│   │   ├── client_integration_test.go   # //go:build integration
│   │   └── stream_integration_test.go
│   │
│   ├── ui/
│   │   ├── repl.go                # Bubbletea REPL model
│   │   ├── spinner.go             # Loading spinner component
│   │   ├── codeview.go            # Syntax-highlighted code view
│   │   ├── theme.go               # Lipgloss styles & themes
│   │   └── theme_test.go
│   │
│   ├── config/
│   │   ├── config.go              # Viper setup, defaults
│   │   ├── paths.go               # OS-aware path resolution
│   │   ├── paths_test.go
│   │   └── config_test.go
│   │
│   ├── context/
│   │   ├── file.go                # File context injection
│   │   ├── project.go             # Auto-detect project type
│   │   ├── truncate.go            # Token budget management
│   │   ├── file_test.go
│   │   ├── truncate_test.go
│   │   └── project_test.go
│   │
│   └── history/
│       ├── store.go               # Read/write history.json
│       ├── session.go             # In-memory session state
│       ├── store_test.go
│       └── session_test.go
│
├── pkg/
│   └── provider/
│       ├── interface.go           # LLMProvider interface
│       └── interface_test.go      # compile-time interface satisfaction
│
├── .goreleaser.yaml               # GoReleaser config
├── .github/
│   └── workflows/
│       ├── ci.yml                 # Test on push
│       └── release.yml            # Build & publish on tag
│
├── install.sh                     # Linux/macOS curl installer
├── Makefile                       # Dev shortcuts
├── go.mod
├── go.sum
└── README.md
```

---

## CLI Commands Reference

### `lokalai ask`

Single-shot question. Outputs the response and exits.

```bash
lokalai ask "Explain what this error means: cannot use x (type int) as type string"
lokalai ask "Write a Python function to flatten a nested list"
lokalai ask --model qwen2.5-coder:7b "Refactor this to use generics" --file main.go
```

**Flags:**

| Flag | Short | Default | Description |
|---|---|---|---|
| `--model` | `-m` | config default | Ollama model to use |
| `--file` | `-f` | — | Inject file content as context |
| `--no-stream` | — | false | Wait for full response before printing |
| `--raw` | — | false | Output plain text, no markdown rendering |
| `--max-tokens` | — | 2048 | Max tokens to generate |

---

### `lokalai chat`

Interactive REPL session with full conversation history.

```bash
lokalai chat
lokalai chat --model llama3:8b
lokalai chat --context ./src          # inject entire directory as context
```

**Inside the REPL, available slash commands:**

| Command | Description |
|---|---|
| `/model <name>` | Switch to a different Ollama model mid-session |
| `/file <path>` | Inject a file into the current context |
| `/context <dir>` | Load a project directory as context |
| `/clear` | Clear conversation history |
| `/save <name>` | Save session to `~/.lokalai/sessions/<name>.json` |
| `/load <name>` | Load a saved session |
| `/edit <file>` | Send file to LLM for editing, preview diff, confirm save |
| `/explain` | Re-explain the last response in simpler terms |
| `/retry` | Regenerate the last response |
| `/history` | Show conversation turns in this session |
| `/models` | List available Ollama models |
| `/help` | Show all slash commands |
| `/exit` | Exit the REPL |

---

### `lokalai models`

List all models currently available in your local Ollama instance.

```bash
lokalai models
lokalai models --json        # output as JSON
lokalai models --pull        # interactive model downloader (coming soon)
```

**Example output:**

```
┌─────────────────────────────┬──────────┬───────────┬────────────────────┐
│ Model                       │ Size     │ Modified  │ Family             │
├─────────────────────────────┼──────────┼───────────┼────────────────────┤
│ llama3:8b                   │ 4.7 GB   │ 2 days    │ llama              │
│ qwen2.5-coder:7b            │ 4.4 GB   │ 5 days    │ qwen2              │
│ deepseek-coder-v2:16b       │ 9.1 GB   │ 1 week    │ deepseek2          │
│ mistral:7b                  │ 4.1 GB   │ 2 weeks   │ mistral            │
│ phi3:mini                   │ 2.2 GB   │ 3 weeks   │ phi3               │
└─────────────────────────────┴──────────┴───────────┴────────────────────┘
```

---

### `lokalai run`

Analyze, explain, or transform a file.

```bash
lokalai run main.go                          # explain the file
lokalai run main.go --task "add error handling"
lokalai run main.go --task "convert to use generics"
lokalai run error.log --task "diagnose this error log"
lokalai run --diff                           # show diff before saving changes
```

---

### `lokalai config`

Manage the lokalai configuration file.

```bash
lokalai config set model qwen2.5-coder:7b
lokalai config set host http://localhost:11434
lokalai config set theme dark
lokalai config get model
lokalai config list
lokalai config reset
```

---

## Ollama API Integration

Ollama exposes a simple HTTP REST API. lokalai uses two primary endpoints:

### List Models

```
GET http://localhost:11434/api/tags
```

**Response:**
```json
{
  "models": [
    {
      "name": "qwen2.5-coder:7b",
      "modified_at": "2025-01-15T10:23:00Z",
      "size": 4368491520,
      "digest": "sha256:...",
      "details": {
        "family": "qwen2",
        "parameter_size": "7B",
        "quantization_level": "Q4_K_M"
      }
    }
  ]
}
```

### Generate (with streaming)

```
POST http://localhost:11434/api/generate
Content-Type: application/json

{
  "model": "qwen2.5-coder:7b",
  "prompt": "Write a Go function to reverse a string",
  "stream": true,
  "options": {
    "temperature": 0.7,
    "num_predict": 2048,
    "top_p": 0.9
  }
}
```

**Stream response** (one JSON object per line):
```json
{"model":"qwen2.5-coder:7b","response":"Here","done":false}
{"model":"qwen2.5-coder:7b","response":" is","done":false}
{"model":"qwen2.5-coder:7b","response":" a","done":false}
...
{"model":"qwen2.5-coder:7b","response":"","done":true,"total_duration":4523000000}
```

### Chat (multi-turn)

```
POST http://localhost:11434/api/chat
Content-Type: application/json

{
  "model": "qwen2.5-coder:7b",
  "stream": true,
  "messages": [
    { "role": "system", "content": "You are a helpful coding assistant..." },
    { "role": "user", "content": "What is a goroutine?" },
    { "role": "assistant", "content": "A goroutine is..." },
    { "role": "user", "content": "How is it different from a thread?" }
  ]
}
```

---

## Streaming Implementation

Streaming is what makes the CLI feel snappy and alive. Instead of waiting 10–30 seconds for the full response, tokens print as they are generated.

### Go Implementation

```go
// internal/ollama/stream.go

package ollama

import (
    "bufio"
    "encoding/json"
    "net/http"
)

type StreamChunk struct {
    Model    string `json:"model"`
    Response string `json:"response"`
    Done     bool   `json:"done"`
}

func (c *Client) StreamGenerate(req GenerateRequest, out chan<- string) error {
    req.Stream = true
    body, _ := json.Marshal(req)

    resp, err := http.Post(c.baseURL+"/api/generate", "application/json", bytes.NewReader(body))
    if err != nil {
        return err
    }
    defer resp.Body.Close()

    scanner := bufio.NewScanner(resp.Body)
    for scanner.Scan() {
        var chunk StreamChunk
        if err := json.Unmarshal(scanner.Bytes(), &chunk); err != nil {
            continue
        }
        if chunk.Response != "" {
            out <- chunk.Response
        }
        if chunk.Done {
            close(out)
            return nil
        }
    }
    return scanner.Err()
}
```

The channel-based approach keeps the stream reader decoupled from the UI. The TUI can drain the channel at whatever rate it renders without blocking the HTTP reader.

---

## Cross-Platform Support

### Path Handling

Always use stdlib functions — never hardcode slashes or assume `$HOME`.

```go
// internal/config/paths.go

package config

import (
    "os"
    "path/filepath"
    "runtime"
)

// ConfigDir returns the OS-appropriate config directory.
// Windows: C:\Users\<user>\AppData\Roaming\lokalai
// Linux:   /home/<user>/.lokalai
func ConfigDir() (string, error) {
    if runtime.GOOS == "windows" {
        appData := os.Getenv("APPDATA")
        if appData == "" {
            home, err := os.UserHomeDir()
            if err != nil {
                return "", err
            }
            appData = filepath.Join(home, "AppData", "Roaming")
        }
        return filepath.Join(appData, "lokalai"), nil
    }
    home, err := os.UserHomeDir()
    if err != nil {
        return "", err
    }
    return filepath.Join(home, ".lokalai"), nil
}
```

### Terminal & Color Detection

```go
// internal/ui/theme.go

package ui

import (
    "os"
    "golang.org/x/term"
)

func SupportsColor() bool {
    // Respect NO_COLOR standard (https://no-color.org)
    if os.Getenv("NO_COLOR") != "" {
        return false
    }
    // Check if stdout is a real terminal (not piped)
    return term.IsTerminal(int(os.Stdout.Fd()))
}
```

### Signal Handling (Ctrl+C on both platforms)

```go
// Works identically on Windows and Linux
sigChan := make(chan os.Signal, 1)
signal.Notify(sigChan, os.Interrupt, syscall.SIGTERM)

go func() {
    <-sigChan
    fmt.Println("\nGoodbye!")
    os.Exit(0)
}()
```

> **Note:** `syscall.SIGTERM` is not available on Windows, so use a build constraint or check `runtime.GOOS` to conditionally add it. Alternatively, use `os.Interrupt` only — it maps correctly on both platforms.

### Windows Terminal Compatibility

- **ANSI escape codes** work in Windows Terminal, PowerShell 7+, and VS Code terminal
- **cmd.exe** has limited support — use `lipgloss` with color auto-detection
- **ConPTY** (Console Pseudo Terminal) is used internally by `bubbletea` on Windows for proper raw mode input handling

---

## Configuration System

Config file lives at:
- **Linux:** `~/.lokalai/config.yaml`
- **Windows:** `%APPDATA%\lokalai\config.yaml`

### Default `config.yaml`

```yaml
# lokalai configuration

# Ollama connection
host: http://localhost:11434

# Default model to use (run `lokalai models` to see available models)
model: llama3:8b

# Fast model for lightweight tasks (autocomplete, quick questions)
fast_model: phi3:mini

# UI settings
theme: dark          # dark | light | auto
stream: true         # stream output token-by-token
render_markdown: true

# Context settings
max_context_tokens: 4096    # max tokens to use for file/project context
auto_detect_project: true   # auto-scan README, go.mod, package.json etc.

# History
save_history: true
max_history_entries: 500

# System prompt prepended to every conversation
system_prompt: |
  You are an expert coding assistant. You help developers write, debug, explain,
  and refactor code. Be concise. Prefer code examples over long explanations.
  When writing code, always include comments for non-obvious logic.
```

### Viper Setup

```go
// internal/config/config.go

package config

import (
    "github.com/spf13/viper"
)

func Init() error {
    configDir, err := ConfigDir()
    if err != nil {
        return err
    }

    viper.SetConfigName("config")
    viper.SetConfigType("yaml")
    viper.AddConfigPath(configDir)

    // Defaults
    viper.SetDefault("host", "http://localhost:11434")
    viper.SetDefault("model", "llama3:8b")
    viper.SetDefault("theme", "auto")
    viper.SetDefault("stream", true)
    viper.SetDefault("max_context_tokens", 4096)

    // Allow env overrides: LOKALAI_MODEL, LOKALAI_HOST, etc.
    viper.SetEnvPrefix("LOKALAI")
    viper.AutomaticEnv()

    if err := viper.ReadInConfig(); err != nil {
        if _, ok := err.(viper.ConfigFileNotFoundError); ok {
            return writeDefaultConfig(configDir)
        }
        return err
    }
    return nil
}
```

---

## Session & History Management

### Session Structure

```go
// internal/history/session.go

type Message struct {
    Role      string    `json:"role"`       // "user" | "assistant" | "system"
    Content   string    `json:"content"`
    Timestamp time.Time `json:"timestamp"`
    Model     string    `json:"model"`
}

type Session struct {
    ID        string    `json:"id"`
    Name      string    `json:"name"`
    Model     string    `json:"model"`
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
    Messages  []Message `json:"messages"`
}
```

### History File

Sessions are persisted as JSON at:
- **Linux:** `~/.lokalai/sessions/<session-id>.json`
- **Windows:** `%APPDATA%\lokalai\sessions\<session-id>.json`

A master index `~/.lokalai/history.json` tracks all sessions:

```json
{
  "sessions": [
    {
      "id": "2025-01-15-abc123",
      "name": "debugging goroutine leak",
      "model": "qwen2.5-coder:7b",
      "created_at": "2025-01-15T10:00:00Z",
      "updated_at": "2025-01-15T10:45:00Z",
      "message_count": 12
    }
  ]
}
```

### Context Window Management

Long conversations will exceed the model's context window. lokalai manages this automatically:

```go
// internal/context/truncate.go

// EstimateTokens is a rough approximation: 1 token ≈ 4 characters
func EstimateTokens(s string) int {
    return len(s) / 4
}

// TruncateHistory keeps the system prompt + last N turns within budget
func TruncateHistory(messages []Message, maxTokens int) []Message {
    budget := maxTokens
    result := []Message{}

    // Always keep system message
    for _, m := range messages {
        if m.Role == "system" {
            budget -= EstimateTokens(m.Content)
            result = append(result, m)
        }
    }

    // Walk backwards through conversation, keeping recent turns
    nonSystem := []Message{}
    for _, m := range messages {
        if m.Role != "system" {
            nonSystem = append(nonSystem, m)
        }
    }

    kept := []Message{}
    for i := len(nonSystem) - 1; i >= 0; i-- {
        cost := EstimateTokens(nonSystem[i].Content)
        if budget-cost < 200 { // keep 200 token buffer
            break
        }
        budget -= cost
        kept = append([]Message{nonSystem[i]}, kept...)
    }

    return append(result, kept...)
}
```

---

## TUI Design

lokalai uses [Bubble Tea](https://github.com/charmbracelet/bubbletea) (Elm architecture) for the interactive REPL.

### Model-View-Update Pattern

```
┌────────────────────────────────────────────────────┐
│                  Terminal Window                    │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │  lokalai  ●  qwen2.5-coder:7b  ●  session   │   │  ← Header bar
│  └─────────────────────────────────────────────┘   │
│                                                     │
│  You: explain the visitor pattern                   │
│                                                     │
│  Assistant:                                         │
│  The Visitor pattern lets you add operations to     │
│  objects without modifying their classes.           │
│                                                     │
│  ```go                                              │
│  type Visitor interface {                           │
│      VisitCircle(c *Circle) float64                 │
│      VisitRect(r *Rectangle) float64                │
│  }                                                  │
│  ```                                                │
│                                                     │
│  ─────────────────────────────────────────────────  │  ← Divider
│  > █                                                │  ← Input area
│  ─────────────────────────────────────────────────  │
│  [/help] [/model] [/file] [/clear]  tokens: 847    │  ← Status bar
└────────────────────────────────────────────────────┘
```

### Streaming Render

As tokens arrive from Ollama, the output area updates in real time. A spinner is shown while waiting for the first token:

```
Assistant: ⣾ thinking...
```

Then transitions to streaming text as tokens arrive. The final response is re-rendered through `glamour` for proper markdown formatting once streaming completes.

---

## File Context Injection

Inject any file's contents into the conversation context:

```bash
# Via flag
lokalai ask "What does this code do?" --file main.go

# Via slash command in REPL
/file ./internal/ollama/client.go

# Multiple files
lokalai ask "Compare these two implementations" --file v1.go --file v2.go
```

### Implementation

```go
// internal/context/file.go

func InjectFile(path string, maxTokens int) (string, error) {
    content, err := os.ReadFile(path)
    if err != nil {
        return "", fmt.Errorf("could not read file %s: %w", path, err)
    }

    ext := filepath.Ext(path)
    lang := extToLanguage(ext)  // ".go" → "go", ".py" → "python"

    snippet := string(content)
    if EstimateTokens(snippet) > maxTokens {
        // Truncate with a warning comment
        snippet = truncateToTokens(snippet, maxTokens)
        snippet += fmt.Sprintf("\n\n// ... (truncated to ~%d tokens)", maxTokens)
    }

    return fmt.Sprintf("File: %s\n```%s\n%s\n```", path, lang, snippet), nil
}
```

---

## Project Context Auto-Detection

When you run `lokalai chat` from a project directory, lokalai automatically detects the project type and injects a concise summary as the system context.

### Detected Project Types

| File Found | Project Type | Auto-context |
|---|---|---|
| `go.mod` | Go module | Module name, Go version, key dependencies |
| `package.json` | Node.js/JS | Project name, scripts, main dependencies |
| `Cargo.toml` | Rust | Crate name, edition, dependencies |
| `pyproject.toml` / `setup.py` | Python | Package name, Python version, deps |
| `pom.xml` | Java/Maven | Artifact ID, Java version, deps |
| `Dockerfile` | Container | Base image, exposed ports |
| `README.md` | Any | First 500 chars as project description |

### Example Auto-context Injected

```
System context (auto-detected):
- Project: my-api (Go module)
- Go version: 1.23
- Dependencies: gin, gorm, viper, cobra
- README: "A RESTful API server for managing user accounts..."
```

This means you can ask questions like *"Add JWT middleware"* without explaining what framework you're using.

---

## Multi-Model Routing

lokalai supports routing different types of tasks to different models automatically:

```yaml
# config.yaml
model: qwen2.5-coder:7b        # default for most tasks
fast_model: phi3:mini           # used for quick/autocomplete tasks
large_model: deepseek-coder-v2:16b  # used for complex refactoring
```

### Routing Logic

| Task type | Model used |
|---|---|
| Short question (< 20 words) | `fast_model` |
| File analysis / explain | `model` (default) |
| Refactor / rewrite file | `large_model` |
| Manual `--model` flag | Always use specified |

You can override routing at any time with `--model` flag or `/model` slash command.

---

## Slash Commands

Full reference of all slash commands available inside `lokalai chat`:

```
/model <name>         Switch model for this session
/model                Show current model

/file <path>          Inject file as context
/context <dir>        Inject project directory as context
/context clear        Remove injected context

/edit <file>          LLM edits the file; shows diff; confirms before saving
/explain              Re-explain last response in simpler terms
/retry                Regenerate last assistant response
/shorter              Ask LLM to shorten last response
/longer               Ask LLM to expand last response

/save [name]          Save session to disk
/load <name>          Load a saved session
/sessions             List saved sessions
/history              Show turns in current session
/clear                Clear conversation history (keep context)
/reset                Clear everything (history + context)

/models               List available Ollama models
/status               Show current model, tokens used, context size

/help                 Show this list
/exit                 Exit lokalai
```

---

## Shell Integration

### Pipe Support

lokalai reads from stdin when input is piped, enabling powerful shell integration:

```bash
# Explain a compiler error
go build ./... 2>&1 | lokalai ask "explain this error"

# Review git diff before committing
git diff | lokalai ask "review this diff and suggest improvements"

# Analyze log files
tail -100 /var/log/app.log | lokalai ask "find anomalies in this log"

# Generate commit message
git diff --staged | lokalai ask "write a conventional commit message for this diff"
```

### Fix Last Command

```bash
# Add to your .bashrc / .zshrc
alias fix='lokalai ask "fix this command: $(fc -ln -1 2>/dev/null || history 1)"'

# Usage: after a failed command, just type:
fix
```

### PowerShell Integration (Windows)

```powershell
# Add to $PROFILE
function fix {
    $lastCmd = (Get-History -Count 1).CommandLine
    lokalai ask "fix this PowerShell command: $lastCmd"
}
```

---

## Build System & GoReleaser

### `.goreleaser.yaml`

```yaml
# .goreleaser.yaml
version: 2

project_name: lokalai

before:
  hooks:
    - go mod tidy
    - make test-unit
    - make test-integration   # requires Ollama on release runner (see ci.yml)

builds:
  - env:
      - CGO_ENABLED=0
    goos:
      - linux
      - windows
    goarch:
      - amd64
      - arm64
    ignore:
      - goos: windows
        goarch: arm64
    ldflags:
      - -s -w
      - -X main.version={{.Version}}
      - -X main.commit={{.Commit}}
      - -X main.date={{.Date}}
    binary: "{{ if eq .Os \"windows\" }}lokalai.exe{{ else }}lokalai{{ end }}"

archives:
  - format: tar.gz
    name_template: "lokalai_{{ .Version }}_{{ .Os }}_{{ .Arch }}"
    format_overrides:
      - goos: windows
        format: zip
    files:
      - README.md
      - LICENSE

checksum:
  name_template: "checksums.txt"

release:
  github:
    owner: your-username
    name: lokalai
  draft: false
  prerelease: auto
  name_template: "lokalai v{{.Version}}"

changelog:
  sort: asc
  filters:
    exclude:
      - "^docs:"
      - "^test:"
      - "^ci:"
```

### Makefile

```makefile
# Makefile

.PHONY: build run test test-unit test-integration clean release

test-unit:
	go test ./... -v -race -count=1

test-integration:
	@echo "Requires: ollama serve + model $$(LOKALAI_TEST_MODEL or phi3:mini)"
	go test -tags=integration ./... -v -count=1

test: test-unit test-integration

test-coverage:
	go test ./... -coverprofile=coverage.out
	go test -tags=integration ./... -coverprofile=coverage-integration.out
	go tool cover -html=coverage.out

build: test
	go build -ldflags="-s -w" -o bin/lokalai ./main.go

build-windows: test
	GOOS=windows GOARCH=amd64 go build -o bin/lokalai.exe ./main.go

build-linux-arm: test
	GOOS=linux GOARCH=arm64 go build -o bin/lokalai-arm64 ./main.go

build-all: build build-windows build-linux-arm

run:
	go run ./main.go $(ARGS)

lint:
	golangci-lint run

clean:
	rm -rf bin/ dist/

release-dry:
	goreleaser release --snapshot --clean

release:
	goreleaser release --clean
```

---

## GitHub Actions CI/CD

### `.github/workflows/ci.yml` — Run on every push/PR

```yaml
name: CI

on:
  push:
    branches: [ main, dev ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest]
        go: ["1.23"]

    runs-on: ${{ matrix.os }}

    steps:
      - uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: ${{ matrix.go }}

      - name: Cache Go modules
        uses: actions/cache@v4
        with:
          path: |
            ~/.cache/go-build
            ~/go/pkg/mod
          key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}

      - name: Download dependencies
        run: go mod download

      - name: Install Ollama (Linux)
        if: runner.os == 'Linux'
        run: curl -fsSL https://ollama.com/install.sh | sh

      - name: Install Ollama (Windows)
        if: runner.os == 'Windows'
        run: winget install --id Ollama.Ollama --accept-package-agreements --accept-source-agreements

      - name: Start Ollama
        shell: bash
        run: |
          ollama serve &
          until curl -sf http://localhost:11434/api/tags; do sleep 2; done

      - name: Pull test model
        run: ollama pull phi3:mini
        env:
          LOKALAI_TEST_MODEL: phi3:mini

      - name: Run unit tests
        run: make test-unit

      - name: Run integration tests
        run: make test-integration
        env:
          LOKALAI_TEST_MODEL: phi3:mini

      - name: Build
        run: go build ./...
```

### `.github/workflows/release.yml` — Trigger on version tag

```yaml
name: Release

on:
  push:
    tags:
      - 'v*'

permissions:
  contents: write

jobs:
  goreleaser:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0   # GoReleaser needs full history for changelog

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: "1.23"

      - name: Install Ollama
        run: curl -fsSL https://ollama.com/install.sh | sh

      - name: Start Ollama
        run: |
          ollama serve &
          until curl -sf http://localhost:11434/api/tags; do sleep 2; done

      - name: Pull test model
        run: ollama pull phi3:mini
        env:
          LOKALAI_TEST_MODEL: phi3:mini

      - name: Run GoReleaser
        uses: goreleaser/goreleaser-action@v6
        with:
          distribution: goreleaser
          version: latest
          args: release --clean
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          LOKALAI_TEST_MODEL: phi3:mini
```

**To cut a new release:**

```bash
git tag v1.0.0
git push origin v1.0.0
# GitHub Actions takes it from here
```

---

## Installation Guide

### Linux / macOS — curl installer

```bash
curl -fsSL https://raw.githubusercontent.com/your-username/lokalai/main/install.sh | bash
```

The install script:
1. Detects OS and architecture
2. Downloads the right binary from GitHub Releases
3. Moves it to `/usr/local/bin/lokalai`
4. Verifies checksum
5. Prints version confirmation

### Windows — winget

```powershell
winget install your-username.lokalai
```

### Windows — Scoop

```powershell
scoop bucket add lokalai https://github.com/your-username/scoop-lokalai
scoop install lokalai
```

### Windows — Manual

1. Download `lokalai_windows_amd64.zip` from [GitHub Releases](https://github.com/your-username/lokalai/releases)
2. Extract `lokalai.exe`
3. Move to a directory in your `PATH` (e.g. `C:\Tools\`)
4. Open a new terminal and run `lokalai --version`

### Build from Source

```bash
git clone https://github.com/your-username/lokalai.git
cd lokalai
go build -o lokalai ./main.go        # Linux
go build -o lokalai.exe ./main.go    # Windows (run in PowerShell)
```

**Requirements:**
- Go 1.21 or later
- Ollama running locally (`ollama serve`)

---

## Cursor Prompt

Use this complete prompt in Cursor to scaffold the entire project:

```
Build a production-grade CLI coding assistant in Go called "lokalai" that 
connects to a local Ollama instance.

Core features:
- Interactive REPL with syntax-highlighted code output
- Auto-detect available Ollama models via GET /api/tags
- Stream responses token-by-token using Ollama's /api/generate (stream: true)
  and /api/chat for multi-turn conversations
- File context injection: read local files and append to prompt
- Session history stored per-platform (see paths below)
- Config file using viper (YAML format, see defaults below)
- Project context auto-detection (go.mod, package.json, Cargo.toml, README.md)

Tech stack:
- cobra for CLI commands
- bubbletea + lipgloss for TUI
- glamour for markdown/code rendering
- viper for config management
- golang.org/x/term for TTY detection
- Ollama REST API (default: http://localhost:11434)
- stdlib net/http for HTTP (no third-party HTTP clients)

CLI commands:
  lokalai ask "prompt"      → single-shot question, streams output
  lokalai chat              → interactive REPL with full conversation history
  lokalai models            → list available Ollama models in a table
  lokalai run <file>        → analyze/explain/edit a file
  lokalai config set/get    → manage config values

Slash commands inside REPL:
  /model <name>   /file <path>   /context <dir>   /edit <file>
  /clear   /save   /load   /retry   /explain   /models   /help   /exit

Cross-platform (Windows + Linux) requirements:
- Use os.UserHomeDir() instead of hardcoded paths
- Use filepath.Join() everywhere, never hardcode slashes
- Config dir:
    Windows → %APPDATA%\lokalai\config.yaml
    Linux   → ~/.lokalai/config.yaml
- Handle Windows terminal (cmd.exe, PowerShell, Windows Terminal) and
  Linux terminals (bash, zsh) gracefully
- Detect ANSI/color support with os.Getenv("NO_COLOR") and
  golang.org/x/term for TTY detection
- Graceful Ctrl+C on both platforms using os/signal with os.Interrupt
- Build pipeline using GoReleaser with targets:
    GOOS=windows GOARCH=amd64 → lokalai.exe
    GOOS=linux   GOARCH=amd64 → lokalai
    GOOS=linux   GOARCH=arm64 → lokalai (for ARM/Raspberry Pi)
- GitHub Actions CI/CD:
    ci.yml    → install Ollama, pull test model, make test-unit + make test-integration
              on ubuntu-latest AND windows-latest on every push
    release.yml → GoReleaser on tag push (v*); before hooks run make test-unit + test-integration

LLMProvider interface:
  Create an interface in pkg/provider/interface.go so OllamaClient
  implements it. This allows future backends (OpenAI-compatible, LM Studio)
  to be added without changing CLI code.

Context window management:
  Implement token estimation (1 token ≈ 4 chars) and auto-truncation of
  history to stay within max_context_tokens, always preserving the system
  prompt and most recent turns.

Project structure:
  lokalai/
  ├── main.go
  ├── cmd/              (root, ask, chat, models, run, config + *_test.go)
  ├── e2e/              (cli_integration_test.go — //go:build integration)
  ├── internal/
  │   ├── testutil/     (ollama.go helpers, fixtures/)
  │   ├── ollama/       (client, stream, models, types + unit/integration tests)
  │   ├── ui/           (repl, spinner, codeview, theme + theme_test.go)
  │   ├── config/       (config, paths + *_test.go)
  │   ├── context/      (file, project, truncate + *_test.go)
  │   └── history/      (store, session + *_test.go)
  ├── pkg/provider/     (LLMProvider interface + interface_test.go)
  ├── .goreleaser.yaml
  ├── .github/workflows/ci.yml
  ├── .github/workflows/release.yml
  ├── install.sh        (Linux curl installer)
  ├── Makefile
  └── README.md

Testing (Section 23 — Testing Strategy & Test Cases):
- Implement EVERY test case listed in Section 23 before marking a feature done
- Test-first: add new test rows to Section 23, write failing tests, then implement
- Unit tests: httptest.Server mocks, t.TempDir(), no live Ollama
- Integration tests: //go:build integration tag, RequireOllama(t), live API calls
- E2E tests: build binary with os/exec, run lokalai models/ask/config against live Ollama
- make test-unit       → go test ./... -v -race (no Ollama)
- make test-integration → go test -tags=integration ./... (requires ollama serve)
- make test            → unit + integration
- make build           → runs make test first, then compiles binary
- Default test model: phi3:mini (override with LOKALAI_TEST_MODEL env var)

Code quality:
- Idiomatic Go: interfaces, error wrapping with fmt.Errorf + %w
- Concurrent streaming: goroutines + channels
- No global state except viper config
- All user-facing strings in English
```

---

## Roadmap

### v1.0 — MVP
- [x] `lokalai ask` with streaming
- [x] `lokalai models`
- [x] Config system
- [x] Windows + Linux binaries via GoReleaser

### v1.1 — REPL
- [ ] `lokalai chat` interactive REPL
- [ ] Slash commands
- [ ] Session save/load
- [ ] Syntax highlighted code blocks

### v1.2 — Context
- [ ] File context injection (`--file`)
- [ ] Project auto-detection
- [ ] Context window management & truncation

### v1.3 — Power Features
- [ ] `/edit` command with diff preview
- [ ] Pipe/stdin support
- [ ] Multi-model routing
- [ ] `lokalai run <file>`

### v2.0 — Extensibility
- [ ] Plugin system for custom slash commands
- [ ] OpenAI-compatible backend (LM Studio, Jan.ai)
- [ ] MCP (Model Context Protocol) tool calls
- [ ] VSCode extension

---

## Testing Strategy & Test Cases

Every feature in lokalai must ship with tests. **`make build` runs the full suite** (unit + live Ollama integration) before compiling the binary. CI and GoReleaser use the same targets.

### Test Layers

| Layer | Requires Ollama | Purpose |
|---|---|---|
| Unit | No | Parsing, paths, truncation, config, CLI flags — use `httptest.Server` or `t.TempDir()` |
| Integration | **Yes (live)** | Real calls to `http://localhost:11434` — list models, stream generate, multi-turn chat |
| E2E CLI | Yes (live) | `os/exec` runs the built binary: `lokalai models`, `lokalai ask "hi"` |

### Local Prerequisites

Before running integration or E2E tests:

```bash
ollama serve                          # daemon must be running
ollama pull phi3:mini                 # default test model (~2 GB)
export LOKALAI_TEST_MODEL=phi3:mini   # optional override
```

Use `tinyllama` instead if disk space is tight (~600 MB). Set `LOKALAI_TEST_MODEL=tinyllama` accordingly.

### Build Tags

Integration and E2E tests live in files tagged with `//go:build integration` at the top. This keeps `go test ./...` (unit-only) fast when invoked directly, while `make test-integration` passes `-tags=integration` to include them.

### Test Helpers

```go
// internal/testutil/ollama.go

package testutil

import "testing"

// RequireOllama fails the test with a clear message if Ollama is unreachable.
// Returns the base URL (default http://localhost:11434).
func RequireOllama(t *testing.T) string

// TestModel returns LOKALAI_TEST_MODEL env var or "phi3:mini".
func TestModel() string
```

Example integration test skeleton:

```go
//go:build integration

package ollama_test

import (
    "testing"
    "github.com/your-username/lokalai/internal/testutil"
)

func TestIntegration_ListModels(t *testing.T) {
    baseURL := testutil.RequireOllama(t)
    // ... create client with baseURL, call ListModels, assert len >= 1
}
```

### Makefile Targets

| Target | Command | When to use |
|---|---|---|
| `make test-unit` | `go test ./... -v -race -count=1` | Fast feedback, no Ollama needed |
| `make test-integration` | `go test -tags=integration ./... -v -count=1` | Requires live Ollama |
| `make test` | unit + integration | Pre-commit, CI |
| `make build` | `test` then `go build` | Production binary |

### Test Case Catalog

Implement **every** test listed below. When adding a feature, add its test rows here first (test-first), then implement.

#### `internal/ollama/` — unit (mock `httptest.Server`)

| Test | Setup | Expected |
|---|---|---|
| `TestListModels_Success` | Mock `/api/tags` JSON | Returns parsed model list |
| `TestListModels_HTTPError` | Server returns 500 | Wrapped error |
| `TestListModels_InvalidJSON` | Malformed body | Error, no panic |
| `TestStreamGenerate_EmitsTokensInOrder` | NDJSON stream chunks | Channel receives `"Here"`, `" is"`, … then closes |
| `TestStreamGenerate_SkipsMalformedLines` | Mix of valid/invalid lines | Valid tokens only |
| `TestStreamGenerate_DoneClosesChannel` | Final `done:true` chunk | Channel closed, no leak |
| `TestChatRequest_SerializesMessages` | Multi-turn messages | POST body matches Ollama schema |

#### `internal/ollama/` — integration (live Ollama)

| Test | Setup | Expected |
|---|---|---|
| `TestIntegration_ListModels` | `RequireOllama(t)` | At least one model returned |
| `TestIntegration_GenerateStream` | Model from `TestModel()` | Non-empty streamed tokens, `done:true` received |
| `TestIntegration_ChatMultiTurn` | 2 user + 1 assistant turn | Coherent response string, no HTTP error |
| `TestIntegration_ModelNotFound` | Model `"nonexistent:0b"` | Error from Ollama |

#### `internal/config/`

| Test | Setup | Expected |
|---|---|---|
| `TestConfigDir_Linux` | `GOOS=linux` + temp home | `~/.lokalai` |
| `TestConfigDir_Windows` | `GOOS=windows`, `APPDATA` set | `%APPDATA%\lokalai` |
| `TestConfigDir_Windows_Fallback` | Empty `APPDATA` | Falls back to `UserHomeDir/AppData/Roaming/lokalai` |
| `TestInit_WritesDefaultConfig` | Missing config file | Creates YAML with defaults from spec |
| `TestInit_ReadsExistingConfig` | Pre-written YAML | Values loaded correctly |
| `TestEnvOverride_LOKALAI_MODEL` | `LOKALAI_MODEL=custom` | Overrides file default |

#### `internal/context/`

| Test | Setup | Expected |
|---|---|---|
| `TestEstimateTokens` | `"abcd"` (4 chars) | Returns `1` |
| `TestTruncateHistory_KeepsSystem` | System + 10 turns, low budget | System message always present |
| `TestTruncateHistory_KeepsRecent` | Budget fits last 3 turns | Oldest user turns dropped first |
| `TestTruncateHistory_200TokenBuffer` | Edge budget | Stops before exceeding buffer |
| `TestInjectFile_Success` | Temp `.go` file | Markdown fence with `go` lang tag |
| `TestInjectFile_NotFound` | Missing path | Wrapped `could not read file` error |
| `TestInjectFile_TruncatesLargeFile` | File > `maxTokens` | Truncation comment appended |
| `TestDetectGoModule` | Fixture `go.mod` | Module name + Go version in context |
| `TestDetectPackageJSON` | Fixture `package.json` | Project name + deps |
| `TestDetectUnknownDir` | Empty temp dir | Empty/minimal context, no error |

#### `internal/history/`

| Test | Setup | Expected |
|---|---|---|
| `TestSaveLoadSession_RoundTrip` | Session with 3 messages | JSON equal after reload |
| `TestHistoryIndex_AppendSession` | Two saves | Index lists both sessions |
| `TestSession_AddMessage` | Append user + assistant | `UpdatedAt` changes, count correct |

#### `internal/ui/`

| Test | Setup | Expected |
|---|---|---|
| `TestSupportsColor_NO_COLOR` | `NO_COLOR=1` | `false` |
| `TestSupportsColor_ForcedOff` | Non-TTY stdout (pipe) | `false` when not a terminal |

#### `cmd/` (Cobra, no network)

| Test | Setup | Expected |
|---|---|---|
| `TestRoot_VersionFlag` | `--version` | Prints version string |
| `TestAsk_FlagsParsed` | `-m`, `-f`, `--no-stream` | Bound to correct fields |
| `TestConfigSet_WritesValue` | `config set model x` | Viper/config file updated |
| `TestModels_JSONFlag` | `--json` | Output is valid JSON |

#### `e2e/` — integration (live Ollama + built binary)

| Test | Setup | Expected |
|---|---|---|
| `TestE2E_ModelsCommand` | `go build` then exec | Exit 0, table or JSON output |
| `TestE2E_AskCommand` | `lokalai ask "say ok"` | Exit 0, non-empty stdout |
| `TestE2E_ConfigGet` | Set then get model | Value matches |

#### `pkg/provider/`

| Test | Expected |
|---|---|
| `TestOllamaClientImplementsProvider` | Compile-time: `var _ provider.LLMProvider = (*ollama.Client)(nil)` |

### CI Notes

- **Linux (ubuntu-latest):** Install Ollama via `curl -fsSL https://ollama.com/install.sh | sh`, start daemon, pull `phi3:mini`, run `make test`.
- **Windows (windows-latest):** Install via `winget install Ollama.Ollama`, start daemon, pull model, run `make test`. Job fails clearly if Ollama cannot start — no silent skip.
- **Model cache:** Consider caching `~/.ollama/models` in GHA to avoid re-pulling on every run (~2–5 min saved).

### Developer Workflow

1. Add test case row(s) to this section **before** implementing the feature.
2. Write failing tests.
3. Implement feature until tests pass.
4. Run `make test` locally with Ollama running.
5. Run `make build` to confirm the full pipeline passes.

If Ollama is not running, `make test-unit` still works for fast iteration. `make build` and `make test-integration` require a live Ollama instance.

---

## Contributing

1. Fork the repo
2. Create a feature branch: `git checkout -b feat/my-feature`
3. Add test case row(s) to [Testing Strategy & Test Cases](#testing-strategy--test-cases) **before** implementing
4. Write tests for new behaviour (unit + integration where applicable)
5. Ensure `make test` and `make lint` pass locally with Ollama running
6. Ensure CI passes on both Linux and Windows
7. Open a PR with a clear description

**Commit convention:** `feat:`, `fix:`, `docs:`, `refactor:`, `test:`, `ci:`

---

*Built with ❤️ using Go + Ollama. 100% local. 100% private.*
