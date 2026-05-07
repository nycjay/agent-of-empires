---
name: add-agent
description: Add support for a new AI coding agent to Agent of Empires. Use this skill when someone wants to integrate a new agent CLI (like claude, kiro-cli, gemini, etc.) into the aoe session manager. Covers the full workflow from research through implementation, testing, and documentation. Also use when updating or improving existing agent support. Trigger on phrases like "add a new agent", "support for X agent", "integrate X CLI", "new tool support", "agent detection", "hook installation", "status detection for X", or any mention of adding/registering a coding assistant binary to aoe.
---

# Add Agent Support

Guide for adding a new AI coding agent to Agent of Empires (aoe).

## Before Starting

Read `docs/development/adding-agents.md` for the full reference. This skill walks you through the process interactively.

## Workflow

### Phase 1: Research the Agent

Gather this information about the target agent before writing any code:

1. **Binary name** — What command launches it?
2. **Detection** — How to check if installed? (usually `which <binary>`)
3. **YOLO flag** — Flag to skip permission prompts? (e.g., `--trust-all-tools`, `--dangerously-skip-permissions`)
4. **Resume flag** — Flag to resume a prior session? (e.g., `--resume`, `--session-id`)
5. **Hook support** — Does it have lifecycle hooks? What events? What format?
6. **Config directory** — Where does it store config? (e.g., `~/.kiro`, `~/.claude`)
7. **Install command** — How do users install it?

If the agent has hooks, determine the format by reading its docs:
- Does it use JSON, YAML, or TOML?
- What are the event names? (case matters: `PreToolUse` vs `preToolUse`)
- What's the structure? (nested `{hooks: [{type, command}]}` vs flat `{command}`)

### Phase 2: Implementation

Follow this order. Each step has a verification checkpoint.

#### Step 1: Agent Definition (`src/agents.rs`)

Add an `AgentDef` entry to the `AGENTS` array. Key fields:

| Field | Notes |
|-------|-------|
| `name` | Canonical lowercase name (e.g., `"kiro"`) |
| `binary` | The executable name (e.g., `"kiro-cli"`) |
| `aliases` | Alternative strings for `resolve_tool_name` matching |
| `detection` | Usually `DetectionMethod::Which("binary")` |
| `yolo` | `Some(YoloMode::CliFlag("--flag"))` or `EnvVar` or `AlwaysYolo` |
| `hook_config` | `Some(...)` only if the agent uses the same JSON format as Claude/Cursor/Gemini. Otherwise `None`. |
| `resume_strategy` | `Flag("--resume-id")`, `FlagPair{...}`, `Subcommand("resume")`, or `Unsupported` |
| `host_only` | `true` if sandbox/worktree not supported |
| `install_hint` | One-line install command |

**Checkpoint:** `cargo build` succeeds.

#### Step 2: Status Detection (`src/tmux/status_detection.rs`)

Add a detection function. For hook-based agents, this is a stub:

```rust
pub fn detect_myagent_status(_content: &str) -> Status {
    Status::Idle
}
```

For pane-parsing agents, examine terminal content for running/waiting/idle signals.

**Checkpoint:** `cargo test --lib status_detection` passes.

#### Step 3: Hook Installation (if applicable)

If the agent supports hooks but uses a non-Claude format:

1. Add a `MYAGENT_HOOKS` constant with `(event_name, status)` pairs in `src/hooks/mod.rs`
2. Write `install_myagent_hooks()` and `uninstall_myagent_hooks()`
3. Wire into `install_agent_status_hooks()` in `src/session/instance.rs`
4. Add the tool name to `status_hook_env_prefix()` in `src/session/instance.rs`
5. Add uninstall call in `uninstall_all_hooks()` in `src/hooks/mod.rs`

**Checkpoint:** `cargo test --lib myagent` passes, hooks file is created on session start.

#### Step 4: Container Config (`src/session/container_config.rs`)

Add an `AgentConfigMount` entry. Decide:
- What to skip (sessions, cache, logs, sandbox)
- What to recursively copy (plugins, config subdirs)
- What to preserve across refreshes (credentials)

**Checkpoint:** `cargo test --lib container_config` passes.

#### Step 5: Dockerfile (`docker/Dockerfile`)

Add installation commands. Check if PATH is already set before adding redundant ENV lines.

#### Step 6: Tests

Update these tests in `src/agents.rs`:
- `test_get_agent_known`
- `test_agent_names`
- `test_resolve_tool_name`
- `test_settings_index_roundtrip`
- `test_send_keys_enter_delay`
- `test_install_hint_lookup`

Add a test for the detection stub in `src/tmux/status_detection.rs`.

If hook-based, add to `test_status_hook_env_prefix_includes_hermes` in `src/session/instance.rs`.

**Checkpoint:** `cargo test --lib -- agents kiro container_config status_hook_env_prefix` all pass.

#### Step 7: Documentation

Update these files:
- `README.md` — features list + FAQ
- `docs/index.md` — supported agents list
- `docs/guides/sandbox.md` — sandbox overview + image table
- `docker/Dockerfile.dev` — comment listing inherited agents

### Phase 3: Verification

Run the full check:

```bash
cargo fmt --check
cargo clippy -- -D warnings
cargo test --lib -- agents <youragent> container_config status_hook_env_prefix
cargo build
./target/debug/aoe agents  # verify detection
```

For hook-based agents, also verify hooks fire:
1. Start a session: `./target/debug/aoe add --cmd <binary> /tmp/test && ./target/debug/aoe session start <name>`
2. Send a message: `./target/debug/aoe send <name> "hello"`
3. Check status file: `cat /tmp/aoe-hooks/*/status`

### Phase 4: Commit

Use conventional commit format: `feat: add <Agent> agent support`

Reference the GitHub issue if one exists: `Closes #NNN`

## Keeping This Skill Updated

When you add a new agent and discover that the process has changed (new fields in `AgentDef`, new files to update, new test patterns), update both:

1. **This skill** (`/.kiro/skills/add-agent/SKILL.md`) — update the workflow steps
2. **The reference doc** (`docs/development/adding-agents.md`) — update the detailed guide

Signs that an update is needed:
- A new field was added to `AgentDef` struct
- A new file needs to be modified when adding agents (e.g., a new config system)
- The hook installation pattern changed
- New tests were added that all agents must satisfy
- The Dockerfile structure changed

After updating, verify the skill is still accurate by checking it against the most recently added agent.
