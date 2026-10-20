# Specification: Rust Universal Agent Session Extractor (`agent-extractor-rs`)

## Status: Draft
- **Target Implementation Language:** Rust (2024 edition)
- **Primary Deliverables:** Standalone CLI & Core Library (`agent-extractor-rs`)
- **Storage Targets:**
  - Relational DBs: SQLite, PostgreSQL, MySQL
  - Knowledge Base: Obsidian Vault (Markdown + Frontmatter + Wikilinks)
- **Primary Consumers:** Personal Long-term Memory, Dreaming/Consolidation Engine, Agent Session Cloud

---

## 1. Objectives and Scope

1. **Extract Coding Agent Sessions:** Discover, decompress, and parse session histories across 17+ coding agents (Codex, Claude Code, Cursor, Antigravity, OpenCode, Copilot CLI, Devin, Pi, Kimi, Grok, Hermes, OpenClaw, Qwen, fx, Cline, DeepSeek Harness, Droid).
2. **High Performance & Low Resource Footprint:** Single compiled Rust binary with streaming JSON decoders, native zstd decompression, and zero-allocation parsing where possible.
3. **Pluggable Architecture:** A trait-based adapter interface (`AgentAdapter`) allowing new agent formats to be added without modifying core sync or database logic.
4. **Structured for Memory & Dreaming:** Capture not just plain text, but the structural components required for cognitive agent architectures:
   - User Intent & Prompt Progression
   - Chain-of-Thought (CoT) / Reasoning traces (`<thinking>` blocks)
   - Tool Execution Graphs (commands, arguments, exit codes, file modifications)
   - Code Diffs & Mutations
   - Session Milestones and Summary Nodes
5. **Multi-Target Persistence:**
   - Unified Relational Storage (SQLite, PostgreSQL, MySQL) via `sqlx`.
   - Native Obsidian Vault export for human-readable browsing, bidirectional graph linking, and local knowledge graphs.

---

## 2. High-Level Architecture

```
+-----------------------------------------------------------------------------------------+
|                                    Input Layer (Agents)                                 |
|                                                                                         |
|  [Codex CLI]          [Claude Code]        [Cursor IDE]        [Antigravity / Others]   |
|  JSONL & .jsonl.zst   Nested JSONL logs    vscdb (SQLite)      JSONL brain transcripts  |
+--------------------------------------------+--------------------------------------------+
                                             |
                                             v
+-----------------------------------------------------------------------------------------+
|                       agent-extractor-rs Core (Rust Engine)                             |
|                                                                                         |
|  +-----------------------------------------------------------------------------------+  |
|  | Discovery & Stream Decoders                                                       |  |
|  | - Parallel fs walk (jwalk / ignore)                                               |  |
|  | - Zero-copy JSON streaming (serde_json / simd-json)                               |  |
|  | - Native streaming decompression (zstd-safe / flate2)                             |  |
|  | - Embedded SQLite reader (rusqlite / sqlx-sqlite)                                 |  |
|  +-----------------------------------------+-----------------------------------------+  |
|                                            |                                            |
|                                            v                                            |
|  +-----------------------------------------------------------------------------------+  |
|  | Adapter Registry (AgentAdapter Trait)                                             |  |
|  | - CodexAdapter, ClaudeAdapter, CursorAdapter, AntigravityAdapter, OpenCodeAdapter |  |
|  +-----------------------------------------+-----------------------------------------+  |
|                                            |                                            |
|                                            v                                            |
|  +-----------------------------------------------------------------------------------+  |
|  | Canonical IR Normalizer                                                           |  |
|  | - SessionMetadata, TurnTimeline, ReasoningTraces, ToolExecutionGraph, CodeDiffs |  |
|  +-----------------------------------------+-----------------------------------------+  |
+--------------------------------------------+--------------------------------------------+
                                             |
                     +-----------------------+-----------------------+
                     |                                               |
                     v                                               v
+------------------------------------------+    +------------------------------------------+
|      Relational DB Sinks (sqlx)          |    |          Obsidian Vault Sink             |
|                                          |    |                                          |
|  [SQLite]    [PostgreSQL]    [MySQL]     |    |  [Vault Root]                            |
|  Local       Cloud / Host    Cloud / Host|    |   ├── Sessions/ (Daily & Per-Agent)      |
|  Embedded    Production      Production  |    |   ├── Projects/ (Repo-linked MOCs)       |
|                                          |    |   ├── Memory/ (Extracted Concepts)       |
|                                          |    |   └── Dreaming/ (Synthesized Replays)    |
+------------------------------------------+    +------------------------------------------+
```

---

## 3. Canonical Domain Models

```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CanonicalSession {
    pub id: String,
    pub source_agent: String,
    pub agent_version: Option<String>,
    pub model_name: Option<String>,
    pub file_path: String,
    pub cwd: Option<String>,
    pub git_repo: Option<String>,
    pub git_branch: Option<String>,
    pub git_commit: Option<String>,
    pub start_time: Option<DateTime<Utc>>,
    pub end_time: Option<DateTime<Utc>>,
    pub duration_seconds: Option<i64>,
    pub total_input_tokens: i64,
    pub total_output_tokens: i64,
    pub is_subagent: bool,
    pub parent_session_id: Option<String>,
    pub turns: Vec<CanonicalTurn>,
    pub diffs: Vec<CanonicalDiff>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum TurnRole {
    User,
    Assistant,
    System,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CanonicalTurn {
    pub turn_id: String,
    pub turn_index: i32,
    pub role: TurnRole,
    pub content: Option<String>,
    pub reasoning_content: Option<String>, // Reasoning/CoT trace essential for dreaming
    pub timestamp: Option<DateTime<Utc>>,
    pub tool_calls: Vec<CanonicalToolCall>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CanonicalToolCall {
    pub call_id: String,
    pub tool_name: String,
    pub tool_input: String,
    pub tool_output: Option<String>,
    pub is_error: bool,
    pub duration_ms: Option<i32>,
    pub timestamp: Option<DateTime<Utc>>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CanonicalDiff {
    pub diff_id: String,
    pub file_path: String,
    pub change_type: String, // "create", "modify", "delete"
    pub diff_content: String,
    pub timestamp: Option<DateTime<Utc>>,
}
```

---

## 4. Extensible Adapter Contract

```rust
use std::path::{Path, PathBuf};
use anyhow::Result;

pub struct AdapterCapabilities {
    pub supports_streaming_deltas: bool,
    pub records_token_costs: bool,
    pub records_thinking_traces: bool,
}

pub trait AgentAdapter: Send + Sync {
    /// Agent identifier, e.g. "codex", "claude", "cursor"
    fn name(&self) -> &'static str;

    /// Return capabilities of this format
    fn capabilities(&self) -> AdapterCapabilities;

    /// Discover candidate session paths on host filesystem
    fn discover(&self, override_root: Option<&Path>) -> Result<Vec<PathBuf>>;

    /// Fast timestamp check to prevent redundant parsing
    fn needs_sync(&self, path: &Path, last_mtime_secs: i64) -> bool;

    /// Parse full session into canonical representation
    fn parse(&self, path: &Path) -> Result<CanonicalSession>;
}
```

---

## 5. Generic Relational Database Target (SQLite / PostgreSQL / MySQL)

The DDL is designed to run interchangeably across SQLite, PostgreSQL, and MySQL without syntax collisions:

```sql
-- 1. Master Sessions Table
CREATE TABLE IF NOT EXISTS agent_sessions (
    session_id          VARCHAR(128) PRIMARY KEY,
    source_agent        VARCHAR(64) NOT NULL,
    agent_version       VARCHAR(64),
    model_name          VARCHAR(128),
    file_path           VARCHAR(1024) NOT NULL,
    cwd                 VARCHAR(1024),
    git_repo            VARCHAR(256),
    git_branch          VARCHAR(128),
    git_commit          VARCHAR(64),
    start_time          TIMESTAMP NULL,
    end_time            TIMESTAMP NULL,
    duration_seconds    INTEGER,
    turn_count          INTEGER DEFAULT 0,
    total_input_tokens  BIGINT DEFAULT 0,
    total_output_tokens BIGINT DEFAULT 0,
    is_subagent         BOOLEAN DEFAULT FALSE,
    parent_session_id   VARCHAR(128),
    synced_mtime        BIGINT NOT NULL,
    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX IF NOT EXISTS idx_sessions_source ON agent_sessions(source_agent);
CREATE INDEX IF NOT EXISTS idx_sessions_repo ON agent_sessions(git_repo);
CREATE INDEX IF NOT EXISTS idx_sessions_time ON agent_sessions(start_time);

-- 2. Turns Table (User Goals, Prompts, and Agent Reasoning)
CREATE TABLE IF NOT EXISTS session_turns (
    turn_id             VARCHAR(128) PRIMARY KEY,
    session_id          VARCHAR(128) NOT NULL,
    turn_index          INTEGER NOT NULL,
    role                VARCHAR(32) NOT NULL,
    content             TEXT,
    reasoning_content   TEXT,
    timestamp           TIMESTAMP NULL,
    FOREIGN KEY (session_id) REFERENCES agent_sessions(session_id) ON DELETE CASCADE
);

CREATE INDEX IF NOT EXISTS idx_turns_session ON session_turns(session_id);

-- 3. Tool Calls (Execution & Interaction Graph)
CREATE TABLE IF NOT EXISTS session_tool_calls (
    call_id             VARCHAR(128) PRIMARY KEY,
    session_id          VARCHAR(128) NOT NULL,
    turn_id             VARCHAR(128),
    tool_name           VARCHAR(128) NOT NULL,
    tool_input          TEXT,
    tool_output         TEXT,
    is_error            BOOLEAN DEFAULT FALSE,
    duration_ms         INTEGER,
    timestamp           TIMESTAMP NULL,
    FOREIGN KEY (session_id) REFERENCES agent_sessions(session_id) ON DELETE CASCADE
);

CREATE INDEX IF NOT EXISTS idx_tools_session ON session_tool_calls(session_id);
CREATE INDEX IF NOT EXISTS idx_tools_name ON session_tool_calls(tool_name);

-- 4. Code Diffs & Modified Files
CREATE TABLE IF NOT EXISTS session_diffs (
    diff_id             VARCHAR(128) PRIMARY KEY,
    session_id          VARCHAR(128) NOT NULL,
    file_path           VARCHAR(1024) NOT NULL,
    change_type         VARCHAR(32) NOT NULL,
    diff_content        TEXT,
    timestamp           TIMESTAMP NULL,
    FOREIGN KEY (session_id) REFERENCES agent_sessions(session_id) ON DELETE CASCADE
);

CREATE INDEX IF NOT EXISTS idx_diffs_session ON session_diffs(session_id);

-- 5. Memory & Dreaming Bridge Tables
CREATE TABLE IF NOT EXISTS memory_nodes (
    node_id             VARCHAR(128) PRIMARY KEY,
    session_id          VARCHAR(128),
    category            VARCHAR(64) NOT NULL, -- 'preference', 'convention', 'solution'
    summary             TEXT NOT NULL,
    confidence_score    DOUBLE PRECISION DEFAULT 1.0,
    extracted_at        TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (session_id) REFERENCES agent_sessions(session_id) ON DELETE SET NULL
);

CREATE TABLE IF NOT EXISTS dream_clusters (
    cluster_id          VARCHAR(128) PRIMARY KEY,
    title               VARCHAR(256) NOT NULL,
    synthesized_insight TEXT NOT NULL,
    session_count       INTEGER DEFAULT 0,
    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

## 6. Obsidian Vault Target Architecture

The Obsidian Target serializes sessions into an organized, hyperlinked knowledge base suited for human navigation, Dataview queries, Canvas visualizations, and local LLM graph-RAG.

### 6.1 Vault Folder Hierarchy

```text
ObsidianVault/
├── 00-Inbox/                     # Newly synced unreviewed sessions
├── 10-Sessions/                  # Canonical session notes
│   ├── Codex/
│   │   └── 2026-10-20-codex-fix-pricing.md
│   ├── Claude/
│   │   └── 2026-10-20-claude-refactor-sync.md
│   ├── Cursor/
│   └── Antigravity/
├── 20-Projects/                  # Map of Content (MOC) per repository
│   ├── agent-sessions.md         # Links to all agent-sessions sessions
│   └── macmem-rust.md
├── 30-Memory/                    # Extracted developer and project memories
│   ├── Concepts/                 # Architecture concepts (e.g. [[Zstandard Parsing]])
│   └── Preferences/              # User preferences (e.g. [[Prefer Rust over Python]])
└── 40-Dreaming/                  # Offline synthesized reflections & clusters
    ├── Replays/                  # Multi-agent comparative solution notes
    └── Insights/                 # Consolidated patterns from dreaming worker
```

### 6.2 Session Markdown Note Format

Each session produces a rich markdown file with structured frontmatter and internal wikilinks:

````markdown
---
id: "ses_01a0cc26_codex"
aliases: ["Codex: Fix pricing calculation bug"]
type: session
agent: codex
agent_version: "0.162.1"
model: "gpt-5.6-sol-pro"
date: 2026-10-20T14:00:00Z
repo: "agent-sessions"
branch: "main"
cwd: "/Users/sfuser/develop/python/agent-sessions"
tokens_in: 45210
tokens_out: 3200
duration: "00:14:22"
is_subagent: false
parent_session: null
tags:
  - agent/codex
  - project/agent-sessions
  - type/debugging
---

# Session: Fix pricing calculation bug

- **Project:** [[Projects/agent-sessions|agent-sessions]]
- **Agent:** [[Agents/Codex|Codex CLI]] (`gpt-5.6-sol-pro`)
- **Timestamp:** [[2026-10-20]] 14:00:00

---

## 🎯 User Intent / Goal
> [!quote] User Prompt
> Fix the negative manifest prices in the pricing review and verify that Kimi moonshot-ai prefix costs are populated.

---

## 🧠 Reasoning & Decisions (Chain-of-Thought)
> [!note] Agent Reasoning
> The manifest parser was accepting negative prices because the validation regex allowed leading minus signs.
> Kimi source strings need normalisation before looking up token list prices in `prices-v2.json`.

---

## 🛠️ Tool Execution & Actions
- **Executed Command:** `python3 -m pytest tests/test_pricing.py`
  - *Exit Code:* 0
  - *Duration:* 240ms
- **File Read:** `AgentSessions/Telemetry/PricingTable.swift`
- **File Edit:** `AgentSessions/Telemetry/PricingCalculator.swift`

---

## 📝 Code Diffs
> [!example]- Diff: `AgentSessions/Telemetry/PricingCalculator.swift`
> ```diff
> - guard price >= 0 else { return nil }
> + guard price > 0, !price.isNaN else { return nil }
> ```

---

## 💡 Extracted Memory Candidates
- [[Memory/Concepts/Pricing Validation Rule]]: Negative prices in manifests indicate invalid upstream probe dumps.
````

---

## 7. Rust CLI Surface (`agent-extractor`)

```bash
# 1. Discover all detectable agent sessions on the machine
agent-extractor discover

# 2. Sync to a local SQLite database
agent-extractor sync --target sqlite:///Users/sfuser/.agent-sessions/memory.db

# 3. Sync to a PostgreSQL database (Cloud / Remote)
agent-extractor sync --target postgres://user:password@localhost:5432/agent_cloud

# 4. Sync to an Obsidian Vault directory
agent-extractor sync --target obsidian:///Users/sfuser/Documents/MyKnowledgeVault

# 5. Dual Sync: Mirror to PostgreSQL and generate Obsidian notes concurrently
agent-extractor sync \
  --target postgres://user:password@localhost:5432/agent_cloud \
  --target obsidian:///Users/sfuser/Documents/MyKnowledgeVault

# 6. Continuous daemon watch mode (streaming newly active sessions)
agent-extractor watch --target sqlite:///Users/sfuser/.agent-sessions/memory.db --interval 2s
```

---

## 8. Implementation Milestones

1. **Phase 1: Project Scaffold & Core IR (`agent-extractor-core`)**
   - Initialize Rust cargo workspace.
   - Define canonical structs (`CanonicalSession`, `CanonicalTurn`, `CanonicalToolCall`, `CanonicalDiff`).
   - Define `AgentAdapter` and `SessionSink` traits.
2. **Phase 2: Adapters Implementation**
   - Codex adapter (JSONL and streaming zstd reader).
   - Claude Code adapter (JSONL with workflow/subagent hierarchy).
   - Cursor adapter (direct SQLite reader from workspaceStorage).
   - Antigravity adapter (brain transcript parser).
3. **Phase 3: Database Storage Sinks (`sqlx`)**
   - Universal schema migrations for SQLite, PostgreSQL, and MySQL.
   - Batch transactional upserts with mtime-based change detection.
4. **Phase 4: Obsidian Vault Sink**
   - Template engine for Obsidian markdown generation.
   - Automatic generation of Project MOCs and daily wikilinks.
   - Tag taxonomy and callout formatting.
5. **Phase 5: CLI & Streaming Watcher**
   - Commands: `discover`, `sync`, `watch`.
   - Parallel multi-source discovery using Rayon.
