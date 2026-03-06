# Phase 3: Memory & Persistence

## Overview

**Priority:** P1
**Status:** pending
**Duration:** 2 weeks
**Goal:** Save conversations to SQLite, resume sessions, export data

## Context Links

- [Phase 2: CLI Interface](./phase-02-cli-interface.md)
- ZeroClaw memory module (for reference)

## Key Insights

- SQLite is perfect for local CLI app - zero config
- `rusqlite` crate is the standard
- Design schema before implementing
- Use migrations for future-proofing

## Architecture

```
src/
├── ... (existing modules)
├── memory/
│   ├── mod.rs           # Memory module root
│   ├── store.rs         # SQLite backend
│   └── types.rs         # Message, Conversation types
└── session/
    ├── mod.rs           # Session module root
    └── manager.rs       # Session lifecycle
```

## Database Schema

```sql
-- conversations table
CREATE TABLE conversations (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    title TEXT NOT NULL,
    model TEXT NOT NULL,
    system_prompt TEXT,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL
);

-- messages table
CREATE TABLE messages (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    conversation_id INTEGER NOT NULL,
    role TEXT NOT NULL,
    content TEXT NOT NULL,
    created_at TEXT NOT NULL,
    FOREIGN KEY (conversation_id) REFERENCES conversations(id)
);

-- Index for faster message lookup
CREATE INDEX idx_messages_conversation ON messages(conversation_id);
```

## Requirements

### Functional
- Auto-save conversations after each exchange
- List past conversations
- Resume previous session
- Delete conversations
- Export to JSON/Markdown

### Non-Functional
- Query time < 100ms
- Safe concurrent access
- Data integrity

## Implementation Steps

### Step 1: Add Dependencies

```toml
# Cargo.toml additions
[dependencies]
# ... existing dependencies ...
rusqlite = { version = "0.31", features = ["bundled"] }
chrono = { version = "0.4", features = ["serde"] }
uuid = { version = "1", features = ["v4"] }
```

### Step 2: Memory Types

```rust
// src/memory/types.rs
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct StoredConversation {
    pub id: i64,
    pub title: String,
    pub model: String,
    pub system_prompt: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct StoredMessage {
    pub id: i64,
    pub conversation_id: i64,
    pub role: String,
    pub content: String,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ConversationWithMessages {
    pub conversation: StoredConversation,
    pub messages: Vec<StoredMessage>,
}

impl StoredMessage {
    pub fn from_chat_message(conversation_id: i64, msg: &crate::chat::ChatMessage) -> Self {
        Self {
            id: 0, // Set by database
            conversation_id,
            role: msg.role.as_str().to_string(),
            content: msg.content.clone(),
            created_at: Utc::now(),
        }
    }
}
```

```rust
// src/memory/mod.rs
pub mod store;
pub mod types;

pub use store::MemoryStore;
pub use types::*;
```

### Step 3: SQLite Store

```rust
// src/memory/store.rs
use super::types::*;
use crate::chat::{ChatMessage, Conversation, Role};
use anyhow::Result;
use chrono::Utc;
use rusqlite::{Connection, Row};
use std::path::Path;

pub struct MemoryStore {
    conn: Connection,
}

impl MemoryStore {
    pub fn new<P: AsRef<Path>>(path: P) -> Result<Self> {
        let conn = Connection::open(path)?;
        let store = Self { conn };
        store.initialize()?;
        Ok(store)
    }

    pub fn in_memory() -> Result<Self> {
        let conn = Connection::open_in_memory()?;
        let store = Self { conn };
        store.initialize()?;
        Ok(store)
    }

    fn initialize(&self) -> Result<()> {
        self.conn.execute_batch(
            r#"
            CREATE TABLE IF NOT EXISTS conversations (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                title TEXT NOT NULL,
                model TEXT NOT NULL,
                system_prompt TEXT,
                created_at TEXT NOT NULL,
                updated_at TEXT NOT NULL
            );

            CREATE TABLE IF NOT EXISTS messages (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                conversation_id INTEGER NOT NULL,
                role TEXT NOT NULL,
                content TEXT NOT NULL,
                created_at TEXT NOT NULL,
                FOREIGN KEY (conversation_id) REFERENCES conversations(id) ON DELETE CASCADE
            );

            CREATE INDEX IF NOT EXISTS idx_messages_conversation ON messages(conversation_id);
            "#,
        )?;
        Ok(())
    }

    pub fn create_conversation(
        &self,
        title: &str,
        model: &str,
        system_prompt: Option<&str>,
    ) -> Result<i64> {
        let now = Utc::now().to_rfc3339();
        self.conn.execute(
            "INSERT INTO conversations (title, model, system_prompt, created_at, updated_at)
             VALUES (?1, ?2, ?3, ?4, ?5)",
            (title, model, system_prompt, &now, &now),
        )?;

        Ok(self.conn.last_insert_rowid())
    }

    pub fn add_message(&self, conversation_id: i64, role: &str, content: &str) -> Result<i64> {
        let now = Utc::now().to_rfc3339();
        self.conn.execute(
            "INSERT INTO messages (conversation_id, role, content, created_at)
             VALUES (?1, ?2, ?3, ?4)",
            (conversation_id, role, content, &now),
        )?;

        // Update conversation's updated_at
        self.conn.execute(
            "UPDATE conversations SET updated_at = ?1 WHERE id = ?2",
            (&now, conversation_id),
        )?;

        Ok(self.conn.last_insert_rowid())
    }

    pub fn list_conversations(&self, limit: i64) -> Result<Vec<StoredConversation>> {
        let mut stmt = self.conn.prepare(
            "SELECT id, title, model, system_prompt, created_at, updated_at
             FROM conversations
             ORDER BY updated_at DESC
             LIMIT ?1",
        )?;

        let rows = stmt.query_map([limit], |row| {
            Ok(StoredConversation {
                id: row.get(0)?,
                title: row.get(1)?,
                model: row.get(2)?,
                system_prompt: row.get(3)?,
                created_at: row.get::<_, String>(4)?.parse().unwrap(),
                updated_at: row.get::<_, String>(5)?.parse().unwrap(),
            })
        })?;

        let mut conversations = Vec::new();
        for row in rows {
            conversations.push(row?);
        }
        Ok(conversations)
    }

    pub fn get_conversation(&self, id: i64) -> Result<Option<StoredConversation>> {
        let mut stmt = self.conn.prepare(
            "SELECT id, title, model, system_prompt, created_at, updated_at
             FROM conversations WHERE id = ?1",
        )?;

        let mut rows = stmt.query([id])?;

        if let Some(row) = rows.next()? {
            Ok(Some(StoredConversation {
                id: row.get(0)?,
                title: row.get(1)?,
                model: row.get(2)?,
                system_prompt: row.get(3)?,
                created_at: row.get::<_, String>(4)?.parse().unwrap(),
                updated_at: row.get::<_, String>(5)?.parse().unwrap(),
            }))
        } else {
            Ok(None)
        }
    }

    pub fn get_messages(&self, conversation_id: i64) -> Result<Vec<StoredMessage>> {
        let mut stmt = self.conn.prepare(
            "SELECT id, conversation_id, role, content, created_at
             FROM messages
             WHERE conversation_id = ?1
             ORDER BY created_at ASC",
        )?;

        let rows = stmt.query_map([conversation_id], |row| {
            Ok(StoredMessage {
                id: row.get(0)?,
                conversation_id: row.get(1)?,
                role: row.get(2)?,
                content: row.get(3)?,
                created_at: row.get::<_, String>(4)?.parse().unwrap(),
            })
        })?;

        let mut messages = Vec::new();
        for row in rows {
            messages.push(row?);
        }
        Ok(messages)
    }

    pub fn delete_conversation(&self, id: i64) -> Result<()> {
        self.conn.execute("DELETE FROM conversations WHERE id = ?1", [id])?;
        Ok(())
    }

    pub fn export_conversation(&self, id: i64, format: ExportFormat) -> Result<String> {
        let conversation = self.get_conversation(id)?
            .ok_or_else(|| anyhow::anyhow!("Conversation not found"))?;
        let messages = self.get_messages(id)?;

        match format {
            ExportFormat::Json => {
                let data = ConversationWithMessages {
                    conversation,
                    messages,
                };
                Ok(serde_json::to_string_pretty(&data)?)
            }
            ExportFormat::Markdown => {
                let mut md = format!("# {}\n\n", conversation.title);
                md.push_str(&format!("Model: {}\n\n", conversation.model));
                if let Some(sp) = &conversation.system_prompt {
                    md.push_str(&format!("**System:** {}\n\n", sp));
                }
                md.push_str("---\n\n");

                for msg in messages {
                    let role = match msg.role.as_str() {
                        "user" => "**You**",
                        "assistant" => "**AI**",
                        _ => "**System**",
                    };
                    md.push_str(&format!("{}: {}\n\n", role, msg.content));
                }

                Ok(md)
            }
        }
    }
}

pub enum ExportFormat {
    Json,
    Markdown,
}
```

### Step 4: Session Manager

```rust
// src/session/manager.rs
use crate::chat::Conversation;
use crate::memory::{MemoryStore, StoredConversation};
use anyhow::Result;

pub struct SessionManager {
    store: MemoryStore,
    current_conversation_id: Option<i64>,
}

impl SessionManager {
    pub fn new(store: MemoryStore) -> Self {
        Self {
            store,
            current_conversation_id: None,
        }
    }

    pub fn start_new_conversation(
        &mut self,
        title: &str,
        model: &str,
        system_prompt: Option<&str>,
    ) -> Result<i64> {
        let id = self.store.create_conversation(title, model, system_prompt)?;
        self.current_conversation_id = Some(id);
        Ok(id)
    }

    pub fn load_conversation(&mut self, id: i64) -> Result<Conversation> {
        let stored = self.store.get_conversation(id)?
            .ok_or_else(|| anyhow::anyhow!("Conversation not found"))?;
        let messages = self.store.get_messages(id)?;

        let mut conversation = Conversation::new();
        for msg in messages {
            let role = match msg.role.as_str() {
                "user" => crate::chat::Role::User,
                "assistant" => crate::chat::Role::Assistant,
                _ => crate::chat::Role::System,
            };
            conversation.add_message(crate::chat::ChatMessage {
                role,
                content: msg.content,
            });
        }

        self.current_conversation_id = Some(id);
        Ok(conversation)
    }

    pub fn save_message(&self, role: &str, content: &str) -> Result<()> {
        if let Some(id) = self.current_conversation_id {
            self.store.add_message(id, role, content)?;
        }
        Ok(())
    }

    pub fn list_recent(&self, limit: i64) -> Result<Vec<StoredConversation>> {
        self.store.list_conversations(limit)
    }

    pub fn current_id(&self) -> Option<i64> {
        self.current_conversation_id
    }

    pub fn export_current(&self, format: crate::memory::store::ExportFormat) -> Result<String> {
        let id = self.current_conversation_id
            .ok_or_else(|| anyhow::anyhow!("No active conversation"))?;
        self.store.export_conversation(id, format)
    }

    pub fn delete_current(&mut self) -> Result<()> {
        if let Some(id) = self.current_conversation_id {
            self.store.delete_conversation(id)?;
            self.current_conversation_id = None;
        }
        Ok(())
    }
}
```

```rust
// src/session/mod.rs
pub mod manager;
pub use manager::SessionManager;
```

### Step 5: Update Commands

```rust
// Add to src/cli/commands.rs
#[derive(Debug, Clone)]
pub enum Command {
    // ... existing commands ...
    New(String),                    // /new <title>
    Load(i64),                      // /load <id>
    List,                           // /list
    Delete,                         // /delete
    Export(String),                 // /export json|md
}

impl Command {
    pub fn parse(input: &str) -> Self {
        // ... existing parsing ...
        match *cmd {
            // ... existing matches ...
            "new" | "n" => Command::New(args.to_string()),
            "load" | "l" => Command::Load(args.parse().unwrap_or(0)),
            "list" | "ls" => Command::List,
            "delete" | "del" => Command::Delete,
            "export" | "ex" => Command::Export(args.to_string()),
            _ => Command::Unknown(input.to_string()),
        }
    }

    pub fn help_text() -> String {
        r#"Available Commands:
  /help, /h, /?     Show this help message
  /clear, /c        Clear the screen
  /exit, /quit, /q  Exit the program
  /model, /m <name> Switch model
  /system, /s <msg> Set system message
  /history          Show conversation history

Session Commands:
  /new, /n <title>  Start new conversation
  /load, /l <id>    Load conversation by ID
  /list, /ls        List recent conversations
  /delete, /del     Delete current conversation
  /export, /ex <fmt> Export (json or md)

Type your message to chat with the AI.
"#
        .to_string()
    }
}
```

### Step 6: Update REPL

```rust
// Update src/cli/repl.rs to use SessionManager
use crate::session::SessionManager;
use crate::memory::MemoryStore;

pub struct Repl {
    client: ZaiClient,
    conversation: Conversation,
    session: SessionManager,
    // ... rest of fields ...
}

impl Repl {
    pub fn new(settings: Settings) -> Result<Self> {
        let store = MemoryStore::new(".zai.db")?;
        let session = SessionManager::new(store);
        // ... rest of initialization ...
    }

    fn process_input(&mut self, input: &str) -> Result<()> {
        match Command::parse(input) {
            // ... existing commands ...
            Command::New(title) => {
                let title = if title.is_empty() {
                    "New Conversation"
                } else {
                    title.trim()
                };
                self.session.start_new_conversation(
                    title,
                    &self.model,
                    self.system_prompt.as_deref(),
                )?;
                self.conversation = Conversation::new();
                println!("Started new conversation");
            }
            Command::List => {
                let convos = self.session.list_recent(10)?;
                println!("Recent conversations:");
                for c in convos {
                    println!(
                        "  [{}] {} ({})",
                        c.id.to_string().cyan(),
                        c.title,
                        c.model
                    );
                }
            }
            Command::Load(id) => {
                match self.session.load_conversation(id) {
                    Ok(conv) => {
                        self.conversation = conv;
                        println!("Loaded conversation {}", id);
                    }
                    Err(e) => eprintln!("Error loading: {}", e),
                }
            }
            Command::Delete => {
                self.session.delete_current()?;
                self.conversation = Conversation::new();
                println!("Deleted conversation");
            }
            Command::Export(format) => {
                let fmt = match format.trim() {
                    "json" | "j" => crate::memory::store::ExportFormat::Json,
                    _ => crate::memory::store::ExportFormat::Markdown,
                };
                let output = self.session.export_current(fmt)?;
                println!("{}", output);
            }
            // ... rest ...
        }
        Ok(())
    }

    fn send_message(&mut self, message: &str) -> Result<()> {
        // ... existing code ...

        // Save to memory
        if let Some(id) = self.session.current_id() {
            self.session.save_message("user", message)?;
            self.session.save_message("assistant", &response)?;
        }

        // ... rest ...
    }
}
```

## Testing

```bash
# Build
cargo build

# Interactive testing
cargo run

# Test commands
/new My First Chat
Hello, how are you?
/list
/load 1
/export json
/delete
```

## Related Code Files

### To Create
- `src/memory/mod.rs`
- `src/memory/store.rs`
- `src/memory/types.rs`
- `src/session/mod.rs`
- `src/session/manager.rs`

### To Modify
- `Cargo.toml` - Add rusqlite, chrono, uuid
- `src/cli/commands.rs` - Add new commands
- `src/cli/repl.rs` - Integrate session manager

## Todo Checklist

- [ ] Add rusqlite, chrono dependencies
- [ ] Create memory types (StoredConversation, StoredMessage)
- [ ] Implement MemoryStore with SQLite
- [ ] Create SessionManager
- [ ] Add new commands to CLI
- [ ] Integrate session manager in REPL
- [ ] Test conversation persistence
- [ ] Test export functionality

## Success Criteria

- [ ] Conversations auto-save after each exchange
- [ ] `/list` shows recent conversations
- [ ] `/load <id>` restores previous session
- [ ] `/export json` produces valid JSON
- [ ] `/export md` produces readable Markdown
- [ ] Data persists across app restarts

## Security Considerations

- Database file permissions (user only)
- Sensitive data in messages (consider encryption for production)
- Export file handling

## Risk Assessment

| Risk | Impact | Mitigation |
|------|--------|------------|
| Database corruption | High | Use SQLite transactions |
| Schema migration needs | Medium | Plan schema carefully, add migrations later |
| Performance with many messages | Low | Index on conversation_id, pagination |

## Next Steps

After completing Phase 3:
1. Move to [Phase 4: Tool Execution](./phase-04-tool-execution.md)
2. Implement tool trait system
3. Add shell command execution