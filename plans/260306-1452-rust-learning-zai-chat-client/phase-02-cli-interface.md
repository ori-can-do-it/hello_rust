# Phase 2: CLI Interface

## Overview

**Priority:** P1
**Status:** pending
**Duration:** 2 weeks
**Goal:** Build an interactive REPL-style CLI with commands and history

## Context Links

- [Phase 1: Minimal Chat Client](./phase-01-minimal-chat-client.md)
- ZeroClaw CLI patterns (for reference)

## Key Insights

- `rustyline` provides readline-like editing (arrow keys, history)
- `clap` for command-line argument parsing
- State machine pattern for REPL
- Keep commands simple and discoverable

## Architecture

```
src/
├── main.rs              # Entry point, CLI setup
├── lib.rs
├── client/              # (from Phase 1)
├── config/              # (from Phase 1)
├── cli/
│   ├── mod.rs           # CLI module root
│   ├── repl.rs          # REPL loop implementation
│   ├── commands.rs      # Command definitions
│   └── history.rs       # Command history management
└── chat/
    ├── mod.rs           # Chat module root
    └── conversation.rs   # Message management
```

## Requirements

### Functional
- Interactive REPL loop
- Command parsing (/, !, @ prefixes)
- Arrow key history navigation
- Multi-line input support
- Clear screen command
- Model switching
- Exit command

### Non-Functional
- Responsive input (< 50ms latency)
- Graceful Ctrl+C handling
- Clean terminal output

## Implementation Steps

### Step 1: Add Dependencies

```toml
# Cargo.toml additions
[dependencies]
# ... existing dependencies ...
clap = { version = "4", features = ["derive"] }
rustyline = "14"
ctrlc = "3"
colored = "2"
```

### Step 2: Chat Module

```rust
// src/chat/conversation.rs
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ChatMessage {
    pub role: Role,
    pub content: String,
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub enum Role {
    User,
    Assistant,
    System,
}

impl Role {
    pub fn as_str(&self) -> &'static str {
        match self {
            Role::User => "user",
            Role::Assistant => "assistant",
            Role::System => "system",
        }
    }
}

#[derive(Debug, Default)]
pub struct Conversation {
    messages: Vec<ChatMessage>,
}

impl Conversation {
    pub fn new() -> Self {
        Self { messages: Vec::new() }
    }

    pub fn add_user_message(&mut self, content: String) {
        self.messages.push(ChatMessage {
            role: Role::User,
            content,
        });
    }

    pub fn add_assistant_message(&mut self, content: String) {
        self.messages.push(ChatMessage {
            role: Role::Assistant,
            content,
        });
    }

    pub fn messages(&self) -> &[ChatMessage] {
        &self.messages
    }

    pub fn clear(&mut self) {
        self.messages.clear();
    }

    pub fn last_user_message(&self) -> Option<&str> {
        self.messages
            .iter()
            .rev()
            .find(|m| m.role == Role::User)
            .map(|m| m.content.as_str())
    }
}
```

```rust
// src/chat/mod.rs
pub mod conversation;
pub use conversation::{ChatMessage, Conversation, Role};
```

### Step 3: Command Definitions

```rust
// src/cli/commands.rs
use std::collections::HashMap;

#[derive(Debug, Clone)]
pub enum Command {
    Help,
    Clear,
    Exit,
    Model(String),
    System(String),
    History,
    Unknown(String),
}

impl Command {
    pub fn parse(input: &str) -> Self {
        let input = input.trim();

        if !input.starts_with('/') {
            return Command::Unknown(input.to_string());
        }

        let parts: Vec<&str> = input[1..].splitn(2, ' ').collect();
        let cmd = parts.get(0).unwrap_or(&"");
        let args = parts.get(1).unwrap_or(&"");

        match *cmd {
            "help" | "h" | "?" => Command::Help,
            "clear" | "c" => Command::Clear,
            "exit" | "quit" | "q" => Command::Exit,
            "model" | "m" => Command::Model(args.to_string()),
            "system" | "s" => Command::System(args.to_string()),
            "history" => Command::History,
            _ => Command::Unknown(input.to_string()),
        }
    }

    pub fn help_text() -> String {
        r#"Available Commands:
  /help, /h, /?     Show this help message
  /clear, /c        Clear the screen
  /exit, /quit, /q  Exit the program
  /model, /m <name> Switch model (e.g., /model glm-4)
  /system, /s <msg> Set system message
  /history          Show conversation history

Type your message to chat with the AI.
"#
        .to_string()
    }
}
```

### Step 4: REPL Implementation

```rust
// src/cli/repl.rs
use crate::chat::Conversation;
use crate::client::ZaiClient;
use crate::config::Settings;
use crate::cli::commands::Command;
use anyhow::Result;
use colored::Colorize;
use rustyline::error::ReadlineError;
use rustyline::history::DefaultHistory;
use rustyline::{CompletionType, Config, EditMode, Editor};

pub struct Repl {
    client: ZaiClient,
    conversation: Conversation,
    editor: Editor<(), DefaultHistory>,
    running: bool,
    model: String,
    system_prompt: Option<String>,
}

impl Repl {
    pub fn new(settings: Settings) -> Result<Self> {
        let client = ZaiClient::new(settings.clone())?;
        let config = Config::builder()
            .history_ignore_space(true)
            .completion_type(CompletionType::List)
            .edit_mode(EditMode::Emacs)
            .build();

        let mut editor = Editor::with_config(config)?;
        editor.load_history(".zai_history").ok();

        Ok(Self {
            client,
            conversation: Conversation::new(),
            editor,
            running: true,
            model: settings.model,
            system_prompt: None,
        })
    }

    pub fn run(&mut self) -> Result<()> {
        self.print_welcome();

        while self.running {
            match self.editor.readline("> ") {
                Ok(line) => {
                    let line = line.trim();
                    if line.is_empty() {
                        continue;
                    }

                    // Add to history
                    self.editor.add_history_entry(line)?;

                    // Process input
                    self.process_input(line)?;
                }
                Err(ReadlineError::Interrupted) => {
                    println!("\nUse /exit to quit.");
                }
                Err(ReadlineError::Eof) => {
                    println!("\nGoodbye!");
                    break;
                }
                Err(err) => {
                    eprintln!("Error: {}", err);
                    break;
                }
            }
        }

        self.editor.save_history(".zai_history")?;
        Ok(())
    }

    fn process_input(&mut self, input: &str) -> Result<()> {
        let command = Command::parse(input);

        match command {
            Command::Help => {
                println!("{}", Command::help_text());
            }
            Command::Clear => {
                print!("\x1B[2J\x1B[1;1H"); // ANSI clear screen
            }
            Command::Exit => {
                println!("Goodbye!");
                self.running = false;
            }
            Command::Model(name) => {
                if name.is_empty() {
                    println!("Current model: {}", self.model.cyan());
                } else {
                    self.model = name.trim().to_string();
                    println!("Switched to model: {}", self.model.cyan());
                }
            }
            Command::System(prompt) => {
                self.system_prompt = if prompt.is_empty() {
                    None
                } else {
                    Some(prompt.trim().to_string())
                };
                println!("System prompt updated.");
            }
            Command::History => {
                println!("Conversation history:");
                for msg in self.conversation.messages() {
                    let role = match msg.role {
                        crate::chat::Role::User => "You".green(),
                        crate::chat::Role::Assistant => "AI".cyan(),
                        crate::chat::Role::System => "System".yellow(),
                    };
                    println!("{}: {}", role, msg.content);
                }
            }
            Command::Unknown(msg) => {
                self.send_message(&msg)?;
            }
        }

        Ok(())
    }

    fn send_message(&mut self, message: &str) -> Result<()> {
        self.conversation.add_user_message(message.to_string());

        print!("{} ", "Thinking...".yellow());
        std::io::Write::flush(&mut std::io::stdout())?;

        match self.client.chat(message).await {
            Ok(response) => {
                print!("\r{}", " ".repeat(15)); // Clear "Thinking..."
                print!("\r");
                println!("{}: {}", "AI".cyan(), response);
                self.conversation.add_assistant_message(response);
            }
            Err(e) => {
                print!("\r");
                eprintln!("{}: {}", "Error".red(), e);
            }
        }

        Ok(())
    }

    fn print_welcome(&self) {
        println!(
            r#"
╔════════════════════════════════════════╗
║       Z.AI Chat Client v0.1.0          ║
║   Type /help for available commands    ║
╚════════════════════════════════════════╝
"#
        );
        println!("Model: {}", self.model.cyan());
        println!();
    }
}
```

```rust
// src/cli/mod.rs
pub mod commands;
pub mod repl;

pub use commands::Command;
pub use repl::Repl;
```

### Step 5: Update Main

```rust
// src/main.rs
use anyhow::Result;
use clap::Parser;
use hello_rust::cli::Repl;
use hello_rust::config::Settings;

#[derive(Parser, Debug)]
#[command(name = "zai")]
#[command(about = "Z.AI Chat Client", long_about = None)]
struct Args {
    /// Message to send (non-interactive mode)
    #[arg(short, long)]
    message: Option<String>,

    /// Model to use
    #[arg(short, long, default_value = "glm-5")]
    model: String,
}

#[tokio::main]
async fn main() -> Result<()> {
    let args = Args::parse();
    let settings = Settings::from_env()?;

    if let Some(message) = args.message {
        // Non-interactive mode
        let mut client = hello_rust::client::ZaiClient::new(settings)?;
        match client.chat(&message).await {
            Ok(response) => println!("{}", response),
            Err(e) => eprintln!("Error: {}", e),
        }
    } else {
        // Interactive mode
        let repl = Repl::new(settings)?;
        repl.run()?;
    }

    Ok(())
}
```

```rust
// src/lib.rs
pub mod chat;
pub mod client;
pub mod cli;
pub mod config;
```

### Step 6: Handle Ctrl+C Gracefully

```rust
// Add to main.rs before REPL
use std::sync::atomic::{AtomicBool, Ordering};
use std::sync::Arc;

// In main():
let running = Arc::new(AtomicBool::new(true));
let r = running.clone();
ctrlc::set_handler(move || {
    r.store(false, Ordering::SeqCst);
    println!("\nExiting...");
})?;
```

## Testing

```bash
# Build
cargo build

# Interactive mode
cargo run

# Non-interactive mode
cargo run -- -m "Hello"

# Specify model
cargo run -- --model glm-4

# Test commands
/help
/model glm-4
/history
/clear
/exit
```

## Related Code Files

### To Create
- `src/cli/mod.rs`
- `src/cli/repl.rs`
- `src/cli/commands.rs`
- `src/chat/mod.rs`
- `src/chat/conversation.rs`

### To Modify
- `Cargo.toml` - Add clap, rustyline, colored, ctrlc
- `src/main.rs` - Add clap, REPL integration
- `src/lib.rs` - Export new modules

## Todo Checklist

- [ ] Add new dependencies to Cargo.toml
- [ ] Create chat module with Conversation
- [ ] Create CLI module with commands
- [ ] Implement REPL with rustyline
- [ ] Update main.rs with clap parsing
- [ ] Add graceful Ctrl+C handling
- [ ] Test interactive mode
- [ ] Test non-interactive mode
- [ ] Test all commands

## Success Criteria

- [ ] Interactive REPL starts and accepts input
- [ ] Arrow keys navigate history
- [ ] All commands work (/help, /clear, /exit, /model, /history)
- [ ] Non-interactive mode works with -m flag
- [ ] Ctrl+C exits gracefully
- [ ] Response latency acceptable

## Risk Assessment

| Risk | Impact | Mitigation |
|------|--------|------------|
| Terminal compatibility | Medium | Use rustyline for cross-platform |
| Input encoding issues | Low | Rust handles UTF-8 natively |

## Next Steps

After completing Phase 2:
1. Move to [Phase 3: Memory & Persistence](./phase-03-memory-persistence.md)
2. Add SQLite storage
3. Implement session management