# Phase 1: Minimal Chat Client

## Overview

**Priority:** P1
**Status:** pending
**Duration:** 2 weeks
**Goal:** Build a CLI app that sends a message to Z.AI and prints the response

## Context Links

- [Brainstorm Report](../brainstorm-rust-learning-zai-chat-client.md)
- [ZeroClaw GLM Provider](https://github.com/zeroclaw-labs/zeroclaw/blob/main/src/providers/glm.rs)

## Key Insights

- Z.AI uses OpenAI-compatible API format
- JWT token generation required for authentication
- Simple HTTP POST to `/chat/completions` endpoint
- Keep it minimal - no streaming, no history yet

## Architecture

```
hello_rust/
├── Cargo.toml              # Dependencies
├── .env                     # ZAI_API_KEY=id.secret
├── .env.example            # Template
├── src/
│   ├── main.rs             # Entry point
│   ├── lib.rs              # Library root
│   ├── client/
│   │   ├── mod.rs          # Client module
│   │   ├── http.rs         # HTTP client
│   │   └── auth.rs         # JWT token generation
│   └── config/
│       ├── mod.rs          # Config module
│       └── settings.rs     # Environment config
```

## Requirements

### Functional
- Accept message via CLI argument
- Authenticate with Z.AI using JWT
- Send message to Z.AI API
- Print response to stdout
- Handle errors gracefully

### Non-Functional
- Response time < 5 seconds
- Clear error messages
- Secure API key handling

## Implementation Steps

### Step 1: Setup Project Structure

```bash
# Initialize project (already done, just verify)
cargo init --name hello_rust

# Create module structure
mkdir -p src/client src/config
touch src/lib.rs
touch src/client/mod.rs src/client/http.rs src/client/auth.rs
touch src/config/mod.rs src/config/settings.rs
```

### Step 2: Add Dependencies

```toml
# Cargo.toml
[package]
name = "hello_rust"
version = "0.1.0"
edition = "2021"

[dependencies]
tokio = { version = "1", features = ["full"] }
reqwest = { version = "0.12", features = ["json"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
anyhow = "1"
thiserror = "1"
dotenvy = "0.15"
base64 = "0.22"
hmac = "0.12"
sha2 = "0.10"
```

### Step 3: Configuration Module

```rust
// src/config/settings.rs
use anyhow::Result;
use std::env;

#[derive(Debug, Clone)]
pub struct Settings {
    pub api_key: String,
    pub base_url: String,
    pub model: String,
}

impl Settings {
    pub fn from_env() -> Result<Self> {
        dotenvy::dotenv().ok();

        Ok(Self {
            api_key: env::var("ZAI_API_KEY")
                .map_err(|_| anyhow::anyhow!("ZAI_API_KEY not set"))?,
            base_url: env::var("ZAI_BASE_URL")
                .unwrap_or_else(|_| "https://api.z.ai/api/paas/v4".to_string()),
            model: env::var("ZAI_MODEL")
                .unwrap_or_else(|_| "glm-5".to_string()),
        })
    }
}
```

```rust
// src/config/mod.rs
pub mod settings;
pub use settings::Settings;
```

### Step 4: JWT Authentication

```rust
// src/client/auth.rs
use anyhow::{Result, bail};
use base64::engine::general_purpose::URL_SAFE_NO_PAD;
use base64::Engine;
use hmac::{Hmac, Mac};
use sha2::Sha256;
use std::time::{SystemTime, UNIX_EPOCH};

type HmacSha256 = Hmac<Sha256>;

/// JWT token for Z.AI authentication
pub struct JwtToken {
    token: String,
    expires_at: u64,
}

impl JwtToken {
    /// Generate JWT from API key (format: "id.secret")
    pub fn generate(api_key: &str) -> Result<Self> {
        let parts: Vec<&str> = api_key.split('.').collect();
        if parts.len() != 2 {
            bail!("Invalid API key format. Expected: id.secret");
        }

        let id = parts[0];
        let secret = parts[1];

        let now = SystemTime::now()
            .duration_since(UNIX_EPOCH)?
            .as_secs();

        let exp = now + 210; // 3.5 minutes

        // Header
        let header = r#"{"alg":"HS256","typ":"JWT","sign_type":"SIGN"}"#;
        let header_b64 = URL_SAFE_NO_PAD.encode(header);

        // Payload
        let payload = format!(
            r#"{{"api_key":"{}","exp":{},"timestamp":{}}}"#,
            id, exp, now
        );
        let payload_b64 = URL_SAFE_NO_PAD.encode(&payload);

        // Sign
        let signing_input = format!("{}.{}", header_b64, payload_b64);
        let signature = sign_hmac(secret, &signing_input)?;
        let sig_b64 = URL_SAFE_NO_PAD.encode(&signature);

        Ok(Self {
            token: format!("{}.{}.{}", header_b64, payload_b64, sig_b64),
            expires_at: exp,
        })
    }

    pub fn token(&self) -> &str {
        &self.token
    }

    pub fn is_expired(&self) -> bool {
        let now = SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .map(|d| d.as_secs())
            .unwrap_or(0);
        now >= self.expires_at
    }
}

fn sign_hmac(secret: &str, data: &str) -> Result<Vec<u8>> {
    let mut mac = HmacSha256::new_from_slice(secret.as_bytes())?;
    mac.update(data.as_bytes());
    Ok(mac.finalize().into_bytes().to_vec())
}
```

```rust
// src/client/mod.rs
pub mod auth;
pub mod http;

pub use auth::JwtToken;
pub use http::ZaiClient;
```

### Step 5: HTTP Client

```rust
// src/client/http.rs
use crate::config::Settings;
use anyhow::{Result, bail};
use reqwest::Client;
use serde::{Deserialize, Serialize};
use std::time::Duration;

use super::auth::JwtToken;

pub struct ZaiClient {
    client: Client,
    settings: Settings,
    token: Option<JwtToken>,
}

#[derive(Serialize)]
struct ChatRequest {
    model: String,
    messages: Vec<Message>,
    #[serde(skip_serializing_if = "Option::is_none")]
    temperature: Option<f64>,
}

#[derive(Serialize)]
struct Message {
    role: String,
    content: String,
}

#[derive(Deserialize)]
struct ChatResponse {
    choices: Vec<Choice>,
}

#[derive(Deserialize)]
struct Choice {
    message: ResponseMessage,
}

#[derive(Deserialize)]
struct ResponseMessage {
    content: String,
}

impl ZaiClient {
    pub fn new(settings: Settings) -> Result<Self> {
        let client = Client::builder()
            .timeout(Duration::from_secs(30))
            .build()?;

        Ok(Self {
            client,
            settings,
            token: None,
        })
    }

    pub async fn chat(&mut self, message: &str) -> Result<String> {
        // Refresh token if needed
        if self.token.is_none() || self.token.as_ref().unwrap().is_expired() {
            self.token = Some(JwtToken::generate(&self.settings.api_key)?);
        }

        let request = ChatRequest {
            model: self.settings.model.clone(),
            messages: vec![Message {
                role: "user".to_string(),
                content: message.to_string(),
            }],
            temperature: Some(0.7),
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

        chat_response
            .choices
            .first()
            .map(|c| c.message.content.clone())
            .ok_or_else(|| anyhow::anyhow!("No response from Z.AI"))
    }
}
```

### Step 6: Main Entry Point

```rust
// src/main.rs
use anyhow::Result;
use hello_rust::config::Settings;
use hello_rust::client::ZaiClient;

#[tokio::main]
async fn main() -> Result<()> {
    let settings = Settings::from_env()?;
    let mut client = ZaiClient::new(settings)?;

    let args: Vec<String> = std::env::args().collect();

    if args.len() < 2 {
        eprintln!("Usage: {} <message>", args[0]);
        std::process::exit(1);
    }

    let message = args[1..].join(" ");
    println!("Sending: {}", message);

    match client.chat(&message).await {
        Ok(response) => println!("\nResponse: {}", response),
        Err(e) => eprintln!("Error: {}", e),
    }

    Ok(())
}
```

```rust
// src/lib.rs
pub mod client;
pub mod config;
```

### Step 7: Environment File

```bash
# .env.example
ZAI_API_KEY=your_api_key_here
ZAI_BASE_URL=https://api.z.ai/api/paas/v4
ZAI_MODEL=glm-5
```

```bash
# .env (create from example, add your key)
cp .env.example .env
# Edit .env and add your actual API key
```

## Testing

```bash
# Build
cargo build

# Run with message
cargo run -- "Hello, how are you?"

# Run with release optimizations
cargo run --release -- "What is Rust?"
```

## Related Code Files

### To Create
- `src/lib.rs`
- `src/client/mod.rs`
- `src/client/http.rs`
- `src/client/auth.rs`
- `src/config/mod.rs`
- `src/config/settings.rs`
- `.env.example`

### To Modify
- `Cargo.toml` - Add dependencies
- `src/main.rs` - Update with chat client logic
- `.gitignore` - Add `.env`

## Todo Checklist

- [ ] Update Cargo.toml with dependencies
- [ ] Create config module (settings.rs, mod.rs)
- [ ] Implement JWT authentication (auth.rs)
- [ ] Implement HTTP client (http.rs)
- [ ] Update main.rs with CLI entry point
- [ ] Create .env.example
- [ ] Add .env to .gitignore
- [ ] Test with real Z.AI API key
- [ ] Handle error cases gracefully

## Success Criteria

- [ ] `cargo build` compiles without errors
- [ ] `cargo run -- "Hello"` returns a response
- [ ] Error messages are clear and helpful
- [ ] API key is loaded from environment
- [ ] JWT token is generated correctly
- [ ] Response is printed to stdout

## Security Considerations

- API key must NOT be committed to git
- .env file in .gitignore
- JWT token expires after 3.5 minutes
- Use HTTPS only (reqwest default)

## Risk Assessment

| Risk | Impact | Mitigation |
|------|--------|------------|
| JWT generation errors | High | Test with real API key early |
| API rate limits | Medium | Start with simple requests |
| Network errors | Medium | Add timeout and retry later |

## Next Steps

After completing Phase 1:
1. Move to [Phase 2: CLI Interface](./phase-02-cli-interface.md)
2. Add interactive REPL
3. Implement command history