# Golem MCP Server - TDD Implementation Plan

**Bounty**: #1926 - Add MCP (Model Context Protocol) server support to golem-cli
**Branch**: bounty/mcp-server-issue-1926-tdd
**Strategy**: Strict Test-Driven Development (Red-Green-Refactor)
**Started**: 2025-11-06

## TDD Principles

### Red-Green-Refactor Cycle (STRICT)
1. **RED**: Write a failing test that defines desired behavior
2. **GREEN**: Write MINIMAL code to make the test pass
3. **REFACTOR**: Improve code quality while keeping tests green
4. **COMMIT**: After each GREEN phase with descriptive message

### Rules
- ❌ **NO implementation code before tests exist**
- ❌ **NO "placeholder" tests that just panic**
- ✅ **Real assertions with expected vs actual comparisons**
- ✅ **Tests must FAIL initially (verify RED state)**
- ✅ **Smallest possible code increment to reach GREEN**
- ✅ **Commit after every complete cycle**

## Bounty Requirements

From issue #1926:
1. MCP server that exposes golem-cli commands as tools
2. Dynamic tool generation from Clap command metadata
3. Security filtering (exclude sensitive commands)
4. HTTP server with SSE streaming support
5. Integration tests proving end-to-end functionality
6. Documentation

## TDD Implementation Phases

---

## Phase 1: Project Structure & Dependencies

### Goal
Set up Rust project structure for TDD with proper test organization.

### Tasks
1. Add dependencies to Cargo.toml (rmcp, rmcp-actix-web, actix-web)
2. Create test module structure
3. Verify tests can compile and run (no tests yet)

### Commit Strategy
- Add dependencies → commit
- Create test module structure → commit

**No implementation code in Phase 1, only scaffolding**

---

## Phase 2: MCP Server Initialization (TDD Cycle 1)

### RED Phase - Write Failing Tests

Create `cli/golem-cli/tests/mcp_server_initialization.rs`:

```rust
#[test]
fn test_server_validates_port_number() {
    // Should accept valid ports 1024-65535
    assert!(is_valid_port(8080));
    assert!(is_valid_port(1024));
    assert!(is_valid_port(65535));

    // Should reject invalid ports
    assert!(!is_valid_port(0));
    assert!(!is_valid_port(80)); // < 1024 requires root
    assert!(!is_valid_port(70000)); // > 65535
}

#[test]
fn test_server_creates_with_valid_config() {
    let config = ServerConfig::new(8080);
    assert_eq!(config.port, 8080);
    assert_eq!(config.host, "127.0.0.1");
}

#[test]
fn test_server_handler_implements_mcp_traits() {
    // Verify ServerHandler implements required rmcp traits
    fn is_server_handler<T: rmcp::ServerHandler>() {}
    is_server_handler::<GolemMcpServer>();
}
```

**Verification**: Run `cargo test` → All tests FAIL (no implementation exists)

### GREEN Phase - Minimal Implementation

Create `cli/golem-cli/src/mcp_server/mod.rs`:
```rust
pub mod config;
pub mod server;

pub use server::GolemMcpServer;

pub fn is_valid_port(port: u16) -> bool {
    port >= 1024 && port <= 65535
}
```

Create `cli/golem-cli/src/mcp_server/config.rs`:
```rust
pub struct ServerConfig {
    pub port: u16,
    pub host: String,
}

impl ServerConfig {
    pub fn new(port: u16) -> Self {
        Self {
            port,
            host: "127.0.0.1".to_string(),
        }
    }
}
```

Create `cli/golem-cli/src/mcp_server/server.rs`:
```rust
use rmcp::ServerHandler;

pub struct GolemMcpServer;

impl ServerHandler for GolemMcpServer {
    // Minimal stub implementation
}
```

**Verification**: Run `cargo test` → All tests PASS

### REFACTOR Phase
- Add documentation comments
- Extract constants (MIN_PORT, MAX_PORT)
- Improve error messages

### COMMIT
```
TDD GREEN: Phase 2 - MCP server initialization

RED: Created 3 failing tests for port validation and config
GREEN: Minimal implementation passes all tests
REFACTOR: Added constants and documentation

Tests: 3 passed
```

---

## Phase 3: MCP Protocol - Initialize Method (TDD Cycle 2)

### RED Phase - Write Failing Tests

Add to `cli/golem-cli/tests/mcp_server_protocol.rs`:

```rust
#[tokio::test]
async fn test_initialize_returns_server_info() {
    let server = GolemMcpServer::new();
    let request = InitializeRequest {
        protocol_version: "2024-11-05".to_string(),
        capabilities: Capabilities::default(),
        client_info: ClientInfo {
            name: "test-client".to_string(),
            version: "1.0".to_string(),
        },
    };

    let result = server.initialize(request).await.unwrap();

    assert_eq!(result.protocol_version, "2024-11-05");
    assert_eq!(result.server_info.name, "golem-cli");
    assert!(result.capabilities.tools.is_some());
}

#[tokio::test]
async fn test_initialize_rejects_incompatible_protocol() {
    let server = GolemMcpServer::new();
    let request = InitializeRequest {
        protocol_version: "1.0.0".to_string(), // Old version
        // ...
    };

    let result = server.initialize(request).await;
    assert!(result.is_err());
}
```

**Verification**: Run `cargo test` → Tests FAIL (initialize not implemented)

### GREEN Phase - Minimal Implementation

Update `cli/golem-cli/src/mcp_server/server.rs`:
```rust
impl ServerHandler for GolemMcpServer {
    async fn initialize(&self, request: InitializeRequest) -> Result<InitializeResult> {
        if request.protocol_version != "2024-11-05" {
            return Err(Error::UnsupportedProtocol);
        }

        Ok(InitializeResult {
            protocol_version: "2024-11-05".to_string(),
            server_info: ServerInfo {
                name: "golem-cli".to_string(),
                version: env!("CARGO_PKG_VERSION").to_string(),
            },
            capabilities: ServerCapabilities {
                tools: Some(ToolsCapability {}),
                ..Default::default()
            },
        })
    }
}
```

**Verification**: Run `cargo test` → Tests PASS

### COMMIT
```
TDD GREEN: Phase 3 - MCP initialize protocol

RED: 2 failing tests for initialize method
GREEN: Implements protocol initialization with version check
```

---

## Phase 4: Tool Discovery (TDD Cycle 3)

### RED Phase - Write Failing Tests

Add to `cli/golem-cli/tests/mcp_server_tools.rs`:

```rust
#[tokio::test]
async fn test_list_tools_returns_golem_commands() {
    let server = GolemMcpServer::new();
    let tools = server.list_tools(None).await.unwrap();

    // Should expose at least component and worker commands
    assert!(tools.tools.iter().any(|t| t.name == "component list"));
    assert!(tools.tools.iter().any(|t| t.name == "worker list"));
    assert!(tools.tools.len() > 10); // Should have many commands
}

#[tokio::test]
async fn test_tool_has_valid_schema() {
    let server = GolemMcpServer::new();
    let tools = server.list_tools(None).await.unwrap();

    let component_list = tools.tools.iter()
        .find(|t| t.name == "component list")
        .expect("component list tool should exist");

    assert!(!component_list.description.is_empty());
    assert!(component_list.input_schema.is_object());
}

#[tokio::test]
async fn test_sensitive_commands_are_filtered() {
    let server = GolemMcpServer::new();
    let tools = server.list_tools(None).await.unwrap();

    // Should NOT expose sensitive commands
    assert!(!tools.tools.iter().any(|t| t.name.starts_with("profile")));
    assert!(!tools.tools.iter().any(|t| t.name.starts_with("cloud token")));
    assert!(!tools.tools.iter().any(|t| t.name.starts_with("cloud account grant")));
}
```

**Verification**: Run `cargo test` → Tests FAIL (list_tools not implemented)

### GREEN Phase - Minimal Implementation

Create `cli/golem-cli/src/mcp_server/tools.rs`:
```rust
use clap::CommandFactory;
use crate::command::GolemCommand;

pub fn discover_tools() -> Vec<Tool> {
    let app = GolemCommand::command();
    let mut tools = Vec::new();

    for subcommand in app.get_subcommands() {
        let name = subcommand.get_name().to_string();

        // Filter sensitive commands
        if is_sensitive_command(&name) {
            continue;
        }

        tools.push(Tool {
            name: name.clone(),
            description: subcommand.get_about()
                .unwrap_or_default()
                .to_string(),
            input_schema: generate_schema(subcommand),
        });
    }

    tools
}

fn is_sensitive_command(name: &str) -> bool {
    const SENSITIVE_PREFIXES: &[&str] = &[
        "profile",
        "cloud token",
        "cloud account grant",
        "cloud project grant",
        "cloud project policy",
    ];

    SENSITIVE_PREFIXES.iter().any(|prefix| name.starts_with(prefix))
}
```

Update `server.rs`:
```rust
impl ServerHandler for GolemMcpServer {
    async fn list_tools(&self, _request: Option<PaginatedRequestParam>) -> Result<ListToolsResult> {
        Ok(ListToolsResult {
            tools: discover_tools(),
            next_cursor: None,
        })
    }
}
```

**Verification**: Run `cargo test` → Tests PASS

### COMMIT
```
TDD GREEN: Phase 4 - Tool discovery from Clap metadata

RED: 3 failing tests for tool discovery and security filtering
GREEN: Dynamic tool generation with sensitive command filtering
```

---

## Phase 5: Tool Execution (TDD Cycle 4)

### RED Phase - Write Failing Tests

Add to `cli/golem-cli/tests/mcp_server_execution.rs`:

```rust
#[tokio::test]
async fn test_call_tool_executes_command() {
    let server = GolemMcpServer::new();
    let request = CallToolRequest {
        name: "component list".to_string(),
        arguments: json!({"component_name": ""}),
    };

    let result = server.call_tool(request).await.unwrap();

    assert!(result.content.len() > 0);
    assert!(matches!(result.content[0], Content::Text(_)));
    assert!(!result.is_error);
}

#[tokio::test]
async fn test_call_tool_returns_error_for_invalid_args() {
    let server = GolemMcpServer::new();
    let request = CallToolRequest {
        name: "component list".to_string(),
        arguments: json!({"invalid_arg": "value"}),
    };

    let result = server.call_tool(request).await.unwrap();
    assert!(result.is_error);
}

#[tokio::test]
async fn test_call_tool_rejects_sensitive_command() {
    let server = GolemMcpServer::new();
    let request = CallToolRequest {
        name: "profile add".to_string(),
        arguments: json!({}),
    };

    let result = server.call_tool(request).await;
    assert!(result.is_err());
}
```

**Verification**: Run `cargo test` → Tests FAIL (call_tool not implemented)

### GREEN Phase - Minimal Implementation

Create `cli/golem-cli/src/mcp_server/executor.rs`:
```rust
use std::process::Command;

pub fn execute_command(tool_name: &str, args: &serde_json::Value) -> Result<String> {
    // Build command from tool name
    let parts: Vec<&str> = tool_name.split_whitespace().collect();

    let mut cmd = Command::new("golem-cli");
    for part in parts {
        cmd.arg(part);
    }

    // Add arguments from JSON
    for (key, value) in args.as_object().unwrap_or(&serde_json::Map::new()) {
        if !value.is_null() && value.as_str().unwrap_or("") != "" {
            cmd.arg(format!("--{}", key));
            if let Some(v) = value.as_str() {
                cmd.arg(v);
            }
        }
    }

    let output = cmd.output()?;

    if output.status.success() {
        Ok(String::from_utf8_lossy(&output.stdout).to_string())
    } else {
        Err(Error::CommandFailed(
            String::from_utf8_lossy(&output.stderr).to_string()
        ))
    }
}
```

Update `server.rs`:
```rust
impl ServerHandler for GolemMcpServer {
    async fn call_tool(&self, request: CallToolRequest) -> Result<CallToolResult> {
        // Security check
        if is_sensitive_command(&request.name) {
            return Err(Error::Forbidden);
        }

        match execute_command(&request.name, &request.arguments) {
            Ok(output) => Ok(CallToolResult {
                content: vec![Content::Text(TextContent {
                    text: output,
                })],
                is_error: false,
            }),
            Err(e) => Ok(CallToolResult {
                content: vec![Content::Text(TextContent {
                    text: format!("Error: {}", e),
                })],
                is_error: true,
            }),
        }
    }
}
```

**Verification**: Run `cargo test` → Tests PASS

### COMMIT
```
TDD GREEN: Phase 5 - Tool execution via subprocess

RED: 3 failing tests for command execution and error handling
GREEN: Subprocess execution with security validation
```

---

## Phase 6: Integration Tests (TDD Cycle 5)

### RED Phase - Write Failing E2E Tests

Create `cli/golem-cli/tests/mcp_server_integration.rs`:

```rust
use reqwest::blocking::Client;

fn start_test_server(port: u16) -> std::process::Child {
    Command::new("cargo")
        .args(&["run", "--", "--serve", &port.to_string()])
        .spawn()
        .expect("Failed to start server")
}

fn wait_for_server(port: u16, timeout_secs: u64) -> Result<(), String> {
    let start = Instant::now();
    let client = Client::new();

    while start.elapsed() < Duration::from_secs(timeout_secs) {
        if let Ok(response) = client
            .post(format!("http://localhost:{}/mcp", port))
            .json(&json!({
                "jsonrpc": "2.0",
                "method": "initialize",
                "params": {
                    "protocolVersion": "2024-11-05",
                    "capabilities": {},
                    "clientInfo": {"name": "test", "version": "1.0"}
                }
            }))
            .send()
        {
            if response.status().is_success() {
                return Ok(());
            }
        }
        thread::sleep(Duration::from_millis(100));
    }

    Err("Server failed to start".to_string())
}

#[test]
fn test_mcp_server_starts_and_responds() {
    let port = 8090;
    let mut server = start_test_server(port);

    wait_for_server(port, 60).expect("Server should start");

    let client = Client::new();
    let response = client
        .post(format!("http://localhost:{}/mcp", port))
        .json(&json!({
            "jsonrpc": "2.0",
            "method": "tools/list",
            "params": {}
        }))
        .send()
        .expect("Request should succeed");

    assert!(response.status().is_success());

    server.kill().unwrap();
}

#[test]
fn test_mcp_server_exposes_tools() {
    // Full E2E test: Start server, list tools, call tool, verify output
    // ...
}

#[test]
fn test_mcp_server_filters_sensitive_commands() {
    // E2E test: Verify sensitive commands not in tools/list
    // ...
}
```

**Verification**: Run `cargo test` → Tests FAIL (no HTTP server implementation)

### GREEN Phase - Add HTTP Server

Create `cli/golem-cli/src/mcp_server/http.rs`:
```rust
use actix_web::{web, App, HttpServer};
use rmcp_actix_web::create_mcp_endpoint;

pub async fn serve(port: u16) -> std::io::Result<()> {
    let server = GolemMcpServer::new();

    HttpServer::new(move || {
        App::new()
            .route("/mcp", web::post().to(
                create_mcp_endpoint(server.clone())
            ))
    })
    .bind(("127.0.0.1", port))?
    .run()
    .await
}
```

Update `cli/golem-cli/src/command.rs`:
```rust
#[derive(Parser)]
pub struct GolemCommand {
    #[arg(long, help = "Start MCP server on specified port")]
    pub serve: Option<u16>,

    #[command(subcommand)]
    pub command: Option<Command>,
}
```

Update `cli/golem-cli/src/main.rs`:
```rust
#[tokio::main]
async fn main() {
    let cmd = GolemCommand::parse();

    if let Some(port) = cmd.serve {
        mcp_server::http::serve(port).await.unwrap();
        return;
    }

    // ... existing CLI logic
}
```

**Verification**: Run `cargo test --test mcp_server_integration` → Tests PASS

### COMMIT
```
TDD GREEN: Phase 6 - HTTP server with integration tests

RED: 3 failing E2E tests for HTTP server
GREEN: Actix-web HTTP server with MCP endpoint

Tests: 15 total (12 unit + 3 integration)
```

---

## Phase 7: GitHub Actions CI Workflow

### Create `.github/workflows/mcp-server-tests.yml`

```yaml
name: MCP Server Tests

on:
  push:
    branches:
      - 'bounty/**'
  pull_request:

jobs:
  mcp-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable

      - name: Run all tests
        run: cargo test --package golem-cli

      - name: Run integration tests
        run: cargo test --package golem-cli --test mcp_server_integration
```

### COMMIT
```
TDD: Add CI workflow for MCP server tests

Validates all tests pass in CI environment
```

---

## Verification Checklist

Before considering TDD complete:

- [ ] All tests follow RED → GREEN → REFACTOR cycle
- [ ] Every commit message indicates TDD phase (RED/GREEN/REFACTOR)
- [ ] No implementation code exists without corresponding tests
- [ ] All tests have real assertions (no placeholder `panic!` tests)
- [ ] Integration tests prove E2E functionality
- [ ] CI workflow validates tests in clean environment
- [ ] Code coverage > 80% for MCP module
- [ ] Security tests validate sensitive command filtering

## Success Criteria

**Tests:**
- ✅ Unit tests: 12+ tests covering individual functions
- ✅ Integration tests: 3+ tests covering E2E scenarios
- ✅ All tests pass locally (`cargo test`)
- ✅ All tests pass in CI

**Implementation:**
- ✅ MCP server exposes 70+ golem-cli commands as tools
- ✅ Dynamic tool generation from Clap metadata
- ✅ Security filtering blocks 5+ sensitive command prefixes
- ✅ HTTP server with SSE streaming support
- ✅ --serve flag to start MCP server

**Documentation:**
- ✅ TDD-PLAN.md (this file)
- ✅ TDD-PROGRESS.md (tracking each cycle)
- ✅ README updates explaining MCP usage
- ✅ Code comments explaining design decisions

## Timeline Estimate

- Phase 1: 30 minutes (setup)
- Phase 2: 1 hour (initialization TDD)
- Phase 3: 1 hour (protocol TDD)
- Phase 4: 2 hours (tool discovery TDD)
- Phase 5: 2 hours (tool execution TDD)
- Phase 6: 2 hours (integration tests TDD)
- Phase 7: 30 minutes (CI setup)

**Total: ~9 hours of focused TDD development**

---

**Next Step**: Begin Phase 1 by adding dependencies to Cargo.toml
