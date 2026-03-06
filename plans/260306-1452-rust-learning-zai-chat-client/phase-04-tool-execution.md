# Phase 4: Tool Execution

## Overview

**Priority:** P1
**Status:** pending
**Duration:** 4 weeks
**Goal:** Implement tool execution framework with shell commands, function calling

## Context Links

- [Phase 3: Memory & Persistence](./phase-03-memory-persistence.md)
- ZeroClaw tools module (for reference)
- ZeroClaw provider traits (for Tool trait design)

## Key Insights

- Tool trait enables extensibility
- `Box<dyn Tool>` for dynamic dispatch
- Safe command execution is critical (escaping, validation)
- Concurrent execution with `tokio::spawn`
- Z.AI supports function calling (OpenAI-compatible)

## Architecture

```
src/
├── ... (existing modules)
├── tools/
│   ├── mod.rs           # Tools module root
│   ├── traits.rs        # Tool trait definition
│   ├── registry.rs      # Tool registry
│   ├── executor.rs      # Tool executor
│   ├── shell.rs         # Shell command tool
│   ├── web_search.rs    # Web search tool
│   └── file_ops.rs      # File operations
└── agent/
    ├── mod.rs           # Agent module root
    ├── loop.rs          # Agent reasoning loop
    └── context.rs       # Agent context/state
```

## Requirements

### Functional
- Define tools with JSON schema
- Execute shell commands safely
- Web search integration
- File read/write operations
- Function calling with Z.AI API
- Tool result handling

### Non-Functional
- Tool execution timeout
- Safe command sanitization
- Concurrent tool support
- Clear error messages

## Implementation Steps

### Step 1: Add Dependencies

```toml
# Cargo.toml additions
[dependencies]
# ... existing dependencies ...
async-trait = "0.1"
futures = "0.3"
jsonschema = "0.17"  # For tool input validation
dirs = "5"           # For home directory resolution
```

### Step 2: Tool Trait

```rust
// src/tools/traits.rs
use async_trait::async_trait;
use serde::{Deserialize, Serialize};
use serde_json::Value;
use std::collections::HashMap;

/// Tool input schema (JSON Schema format)
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ToolSchema {
    pub name: String,
    pub description: String,
    pub parameters: Value, // JSON Schema
}

/// Tool execution result
#[derive(Debug, Clone)]
pub struct ToolResult {
    pub success: bool,
    pub output: String,
    pub error: Option<String>,
}

/// Tool trait for extensibility
#[async_trait]
pub trait Tool: Send + Sync {
    /// Tool name (unique identifier)
    fn name(&self) -> &str;

    /// Tool description for AI
    fn description(&self) -> &str;

    /// JSON Schema for input parameters
    fn parameters_schema(&self) -> Value;

    /// Generate complete tool schema for API
    fn schema(&self) -> ToolSchema {
        ToolSchema {
            name: self.name().to_string(),
            description: self.description().to_string(),
            parameters: self.parameters_schema(),
        }
    }

    /// Execute the tool with given parameters
    async fn execute(&self, params: HashMap<String, Value>) -> ToolResult;

    /// Validate input against schema
    fn validate(&self, params: &HashMap<String, Value>) -> Result<(), String> {
        // Optional: Add JSON Schema validation
        Ok(())
    }

    /// Is this tool safe to run automatically?
    fn is_safe(&self) -> bool {
        false
    }

    /// Requires user confirmation?
    fn requires_confirmation(&self) -> bool {
        true
    }
}
```

### Step 3: Tool Registry

```rust
// src/tools/registry.rs
use super::traits::{Tool, ToolSchema};
use std::collections::HashMap;
use std::sync::Arc;

pub struct ToolRegistry {
    tools: HashMap<String, Arc<dyn Tool>>,
}

impl ToolRegistry {
    pub fn new() -> Self {
        Self {
            tools: HashMap::new(),
        }
    }

    pub fn register<T: Tool + 'static>(&mut self, tool: T) {
        self.tools.insert(tool.name().to_string(), Arc::new(tool));
    }

    pub fn get(&self, name: &str) -> Option<Arc<dyn Tool>> {
        self.tools.get(name).cloned()
    }

    pub fn list(&self) -> Vec<&str> {
        self.tools.keys().map(|s| s.as_str()).collect()
    }

    pub fn schemas(&self) -> Vec<ToolSchema> {
        self.tools.values().map(|t| t.schema()).collect()
    }

    pub fn to_openai_tools(&self) -> Vec<serde_json::Value> {
        self.tools
            .values()
            .map(|t| {
                serde_json::json!({
                    "type": "function",
                    "function": {
                        "name": t.name(),
                        "description": t.description(),
                        "parameters": t.parameters_schema()
                    }
                })
            })
            .collect()
    }
}

impl Default for ToolRegistry {
    fn default() -> Self {
        Self::new()
    }
}
```

### Step 4: Shell Tool

```rust
// src/tools/shell.rs
use super::traits::{Tool, ToolResult};
use async_trait::async_trait;
use serde_json::{json, Value};
use std::collections::HashMap;
use std::time::Duration;
use tokio::process::Command;
use tokio::time::timeout;

pub struct ShellTool {
    allowed_commands: Vec<String>,
    timeout_seconds: u64,
}

impl ShellTool {
    pub fn new() -> Self {
        Self {
            allowed_commands: vec![
                "ls".to_string(),
                "pwd".to_string(),
                "echo".to_string(),
                "cat".to_string(),
                "head".to_string(),
                "tail".to_string(),
                "grep".to_string(),
                "find".to_string(),
                "which".to_string(),
            ],
            timeout_seconds: 30,
        }
    }

    pub fn with_allowed_commands(commands: Vec<String>) -> Self {
        Self {
            allowed_commands: commands,
            timeout_seconds: 30,
        }
    }

    fn is_allowed(&self, command: &str) -> bool {
        // Extract base command
        let base = command.split_whitespace().next().unwrap_or("");
        self.allowed_commands.iter().any(|c| c == base)
    }
}

impl Default for ShellTool {
    fn default() -> Self {
        Self::new()
    }
}

#[async_trait]
impl Tool for ShellTool {
    fn name(&self) -> &str {
        "shell"
    }

    fn description(&self) -> &str {
        "Execute shell commands. Use for file system operations and system queries."
    }

    fn parameters_schema(&self) -> Value {
        json!({
            "type": "object",
            "properties": {
                "command": {
                    "type": "string",
                    "description": "The shell command to execute"
                }
            },
            "required": ["command"]
        })
    }

    async fn execute(&self, params: HashMap<String, Value>) -> ToolResult {
        let command = params
            .get("command")
            .and_then(|v| v.as_str())
            .unwrap_or("");

        // Security check
        if !self.is_allowed(command) {
            return ToolResult {
                success: false,
                output: String::new(),
                error: Some(format!(
                    "Command not allowed. Allowed: {:?}",
                    self.allowed_commands
                )),
            };
        }

        // Execute with timeout
        let result = timeout(
            Duration::from_secs(self.timeout_seconds),
            Command::new("sh").arg("-c").arg(command).output(),
        )
        .await;

        match result {
            Ok(Ok(output)) => {
                let stdout = String::from_utf8_lossy(&output.stdout).to_string();
                let stderr = String::from_utf8_lossy(&output.stderr).to_string();

                if output.status.success() {
                    ToolResult {
                        success: true,
                        output: stdout,
                        error: None,
                    }
                } else {
                    ToolResult {
                        success: false,
                        output: stdout,
                        error: Some(stderr),
                    }
                }
            }
            Ok(Err(e)) => ToolResult {
                success: false,
                output: String::new(),
                error: Some(format!("Failed to execute: {}", e)),
            },
            Err(_) => ToolResult {
                success: false,
                output: String::new(),
                error: Some(format!("Command timed out after {}s", self.timeout_seconds)),
            },
        }
    }

    fn requires_confirmation(&self) -> bool {
        true // Always ask before running shell commands
    }
}
```

### Step 5: File Operations Tool

```rust
// src/tools/file_ops.rs
use super::traits::{Tool, ToolResult};
use async_trait::async_trait;
use serde_json::{json, Value};
use std::collections::HashMap;
use std::path::Path;
use tokio::fs;
use tokio::io::AsyncWriteExt;

pub struct FileOpsTool;

#[async_trait]
impl Tool for FileOpsTool {
    fn name(&self) -> &str {
        "file_ops"
    }

    fn description(&self) -> &str {
        "Read, write, and list files. Use for file system operations."
    }

    fn parameters_schema(&self) -> Value {
        json!({
            "type": "object",
            "properties": {
                "operation": {
                    "type": "string",
                    "enum": ["read", "write", "list", "delete"],
                    "description": "The file operation to perform"
                },
                "path": {
                    "type": "string",
                    "description": "The file or directory path"
                },
                "content": {
                    "type": "string",
                    "description": "Content to write (for write operation)"
                }
            },
            "required": ["operation", "path"]
        })
    }

    async fn execute(&self, params: HashMap<String, Value>) -> ToolResult {
        let operation = params
            .get("operation")
            .and_then(|v| v.as_str())
            .unwrap_or("");

        let path = params.get("path").and_then(|v| v.as_str()).unwrap_or("");

        // Security: Don't allow paths outside current directory or home
        let path = Path::new(path);
        if path.is_absolute() && !path.starts_with(std::env::current_dir().unwrap_or_default()) {
            return ToolResult {
                success: false,
                output: String::new(),
                error: Some("Access denied: path outside allowed directories".to_string()),
            };
        }

        match operation {
            "read" => self.read_file(path).await,
            "write" => {
                let content = params.get("content").and_then(|v| v.as_str()).unwrap_or("");
                self.write_file(path, content).await
            }
            "list" => self.list_dir(path).await,
            "delete" => self.delete_file(path).await,
            _ => ToolResult {
                success: false,
                output: String::new(),
                error: Some(format!("Unknown operation: {}", operation)),
            },
        }
    }

    fn is_safe(&self) -> bool {
        false
    }

    fn requires_confirmation(&self) -> bool {
        true
    }
}

impl FileOpsTool {
    async fn read_file(&self, path: &Path) -> ToolResult {
        match fs::read_to_string(path).await {
            Ok(content) => ToolResult {
                success: true,
                output: content,
                error: None,
            },
            Err(e) => ToolResult {
                success: false,
                output: String::new(),
                error: Some(format!("Failed to read file: {}", e)),
            },
        }
    }

    async fn write_file(&self, path: &Path, content: &str) -> ToolResult {
        match fs::write(path, content).await {
            Ok(_) => ToolResult {
                success: true,
                output: format!("Successfully wrote to {}", path.display()),
                error: None,
            },
            Err(e) => ToolResult {
                success: false,
                output: String::new(),
                error: Some(format!("Failed to write file: {}", e)),
            },
        }
    }

    async fn list_dir(&self, path: &Path) -> ToolResult {
        match fs::read_dir(path).await {
            Ok mut entries) => {
                let mut files = Vec::new();
                while let Ok(Some(entry)) = entries.next_entry().await {
                    files.push(entry.file_name().to_string_lossy().to_string());
                }
                ToolResult {
                    success: true,
                    output: files.join("\n"),
                    error: None,
                }
            }
            Err(e) => ToolResult {
                success: false,
                output: String::new(),
                error: Some(format!("Failed to list directory: {}", e)),
            },
        }
    }

    async fn delete_file(&self, path: &Path) -> ToolResult {
        match fs::remove_file(path).await {
            Ok(_) => ToolResult {
                success: true,
                output: format!("Deleted {}", path.display()),
                error: None,
            },
            Err(e) => ToolResult {
                success: false,
                output: String::new(),
                error: Some(format!("Failed to delete: {}", e)),
            },
        }
    }
}
```

### Step 6: Tool Executor

```rust
// src/tools/executor.rs
use super::registry::ToolRegistry;
use super::traits::{ToolResult};
use anyhow::Result;
use std::collections::HashMap;
use std::sync::Arc;
use tokio::sync::RwLock;

pub struct ToolExecutor {
    registry: Arc<RwLock<ToolRegistry>>,
}

impl ToolExecutor {
    pub fn new(registry: ToolRegistry) -> Self {
        Self {
            registry: Arc::new(RwLock::new(registry)),
        }
    }

    pub async fn execute(
        &self,
        tool_name: &str,
        params: HashMap<String, serde_json::Value>,
    ) -> Result<ToolResult> {
        let registry = self.registry.read().await;

        let tool = registry
            .get(tool_name)
            .ok_or_else(|| anyhow::anyhow!("Tool not found: {}", tool_name))?;

        // Validate
        tool.validate(&params)?;

        // Execute
        let result = tool.execute(params).await;

        Ok(result)
    }

    pub async fn execute_with_confirmation(
        &self,
        tool_name: &str,
        params: HashMap<String, serde_json::Value>,
        confirm: bool,
    ) -> Result<ToolResult> {
        let registry = self.registry.read().await;

        let tool = registry
            .get(tool_name)
            .ok_or_else(|| anyhow::anyhow!("Tool not found: {}", tool_name))?;

        if tool.requires_confirmation() && !confirm {
            return Ok(ToolResult {
                success: false,
                output: String::new(),
                error: Some("Tool execution was not confirmed by user".to_string()),
            });
        }

        drop(registry);
        self.execute(tool_name, params).await
    }

    pub fn registry(&self) -> Arc<RwLock<ToolRegistry>> {
        self.registry.clone()
    }
}
```

### Step 7: Update Client for Function Calling

```rust
// Add to src/client/http.rs

#[derive(Serialize)]
struct ChatRequest {
    model: String,
    messages: Vec<Message>,
    #[serde(skip_serializing_if = "Option::is_none")]
    temperature: Option<f64>,
    #[serde(skip_serializing_if = "Option::is_none")]
    tools: Option<Vec<serde_json::Value>>,
}

#[derive(Deserialize)]
struct ChatResponse {
    choices: Vec<Choice>,
}

#[derive(Deserialize)]
struct Choice {
    message: ResponseMessage,
    #[serde(default)]
    finish_reason: Option<String>,
}

#[derive(Deserialize)]
struct ResponseMessage {
    content: Option<String>,
    #[serde(default)]
    tool_calls: Vec<ToolCall>,
}

#[derive(Debug, Deserialize, Clone)]
pub struct ToolCall {
    pub id: String,
    pub r#type: String,
    pub function: FunctionCall,
}

#[derive(Debug, Deserialize, Clone)]
pub struct FunctionCall {
    pub name: String,
    pub arguments: String, // JSON string
}

impl ZaiClient {
    pub async fn chat_with_tools(
        &mut self,
        message: &str,
        tools: &[serde_json::Value],
    ) -> Result<ChatResponse> {
        // ... token refresh ...

        let request = ChatRequest {
            model: self.settings.model.clone(),
            messages: vec![Message {
                role: "user".to_string(),
                content: message.to_string(),
            }],
            temperature: Some(0.7),
            tools: Some(tools.to_vec()),
        };

        let url = format!("{}/chat/completions", self.settings.base_url);

        let response = self.client
            .post(&url)
            .bearer_auth(self.token.as_ref().unwrap().token())
            .json(&request)
            .send()
            .await?;

        if !response.status().is_success() {
            let error_text = response.text().await?;
            bail!("API error: {}", error_text);
        }

        let chat_response: ChatResponse = response.json().await?;
        Ok(chat_response)
    }
}
```

### Step 8: Module Organization

```rust
// src/tools/mod.rs
pub mod executor;
pub mod file_ops;
pub mod registry;
pub mod shell;
pub mod traits;

pub use executor::ToolExecutor;
pub use registry::ToolRegistry;
pub use traits::{Tool, ToolResult, ToolSchema};
```

```rust
// src/agent/mod.rs
pub mod context;
pub mod reason_loop;

pub use context::AgentContext;
pub use reason_loop::AgentLoop;
```

### Step 9: Integrate with REPL

```rust
// Add to src/cli/repl.rs
use crate::tools::{ToolExecutor, ToolRegistry, ShellTool, FileOpsTool};

impl Repl {
    pub fn new(settings: Settings) -> Result<Self> {
        // ... existing setup ...

        // Initialize tools
        let mut registry = ToolRegistry::new();
        registry.register(ShellTool::new());
        registry.register(FileOpsTool);

        let executor = ToolExecutor::new(registry);

        Ok(Self {
            // ... existing fields ...
            tool_executor: executor,
        })
    }

    // Add tool handling in send_message
    async fn handle_tool_calls(&mut self, tool_calls: &[ToolCall]) -> Result<()> {
        for call in tool_calls {
            println!("{} Tool: {}", "⚡".yellow(), call.function.name.cyan());

            let params: HashMap<String, Value> =
                serde_json::from_str(&call.function.arguments)?;

            // Ask for confirmation
            println!("Arguments: {}", serde_json::to_string_pretty(&params)?);
            print!("Execute this tool? [y/N] ");
            std::io::stdout().flush()?;

            let mut input = String::new();
            std::io::stdin().read_line(&mut input)?;

            let confirm = input.trim().to_lowercase() == "y";

            let result = self.tool_executor
                .execute_with_confirmation(&call.function.name, params, confirm)
                .await?;

            if result.success {
                println!("{} {}", "✓".green(), result.output);
            } else {
                println!("{} {}", "✗".red(), result.error.unwrap_or_default());
            }
        }
        Ok(())
    }
}
```

## Testing

```bash
# Build
cargo build

# Test tool execution
cargo run

# In REPL:
Hello, can you help me list files in the current directory?
# AI should request tool_call for shell or file_ops

# Manual tool test
!ls
!cat README.md
```

## Related Code Files

### To Create
- `src/tools/mod.rs`
- `src/tools/traits.rs`
- `src/tools/registry.rs`
- `src/tools/executor.rs`
- `src/tools/shell.rs`
- `src/tools/file_ops.rs`
- `src/agent/mod.rs`
- `src/agent/context.rs`

### To Modify
- `Cargo.toml` - Add async-trait, futures, jsonschema
- `src/client/http.rs` - Add tool calling support
- `src/cli/repl.rs` - Add tool execution handling

## Todo Checklist

- [ ] Create Tool trait with async-trait
- [ ] Implement ToolRegistry
- [ ] Implement ShellTool with safety checks
- [ ] Implement FileOpsTool
- [ ] Implement ToolExecutor
- [ ] Update client for function calling
- [ ] Integrate tools with REPL
- [ ] Add tool confirmation prompts
- [ ] Test tool execution end-to-end

## Success Criteria

- [ ] Tool trait compiles and works with async
- [ ] Registry can register and list tools
- [ ] ShellTool executes allowed commands
- [ ] FileOpsTool reads/writes files
- [ ] Z.AI can request tool calls
- [ ] User confirmation works
- [ ] Tool results display correctly

## Security Considerations

**CRITICAL**: Tool execution security

1. **Command Injection**: Sanitize all inputs, whitelist allowed commands
2. **Path Traversal**: Validate file paths, restrict to allowed directories
3. **Resource Limits**: Timeout for commands, limit output size
4. **User Confirmation**: Always ask before executing tools
5. **Audit Logging**: Log all tool executions for review

```rust
// Security patterns to implement

// 1. Command whitelisting
const ALLOWED_COMMANDS: &[&str] = &["ls", "cat", "head", "tail", "grep", "find"];

// 2. Path validation
fn validate_path(path: &Path) -> Result<()> {
    let canonical = path.canonicalize()?;
    let cwd = std::env::current_dir()?;
    if !canonical.starts_with(&cwd) {
        bail!("Path outside allowed directory");
    }
    Ok(())
}

// 3. Timeout protection
let result = timeout(Duration::from_secs(30), command.output()).await?;

// 4. Output size limit
const MAX_OUTPUT_SIZE: usize = 100_000;
```

## Risk Assessment

| Risk | Impact | Mitigation |
|------|--------|------------|
| Command injection | Critical | Whitelist, sanitization, user confirmation |
| Path traversal | High | Validate paths, restrict directories |
| Resource exhaustion | Medium | Timeout, output size limits |
| API changes | Low | Use OpenAI-compatible format |

## Next Steps

After completing Phase 4:

1. **Advanced Features** (Optional):
   - Streaming responses (SSE)
   - Multi-turn agent loops
   - Tool result caching
   - Custom tool plugins

2. **Production Ready**:
   - Configuration file support
   - Logging system
   - Error recovery
   - Performance optimization

3. **Contribute to ZeroClaw**:
   - Port improvements upstream
   - Add new tools
   - Improve documentation