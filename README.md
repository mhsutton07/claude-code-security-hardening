# Claude Code Security Hardening — Credential Protection & Safe Command Allowlist

## Purpose

This document grounds an LLM to implement security-hardened permissions for Claude Code (`claude` CLI) on a new environment. It configures `~/.claude/settings.json` with an explicit allow/deny model that:
- Prevents Claude from reading or writing credential files, SSH keys, `.env` files, and secrets
- Blocks destructive shell commands (rm -rf, sudo, git push, npm publish)
- Enables unattended use for pipeline automation (Fabrica, CI) without wildcard `Bash(*)`

This applies globally to:
- Claude Code CLI (`claude` command)
- Claude Code VS Code extension (reads the same global settings file)
- Any subprocess Claude Code instance spawned by an orchestrator (e.g., OpenClaw)

---

## Prerequisites

- Claude Code CLI installed: `npm install -g @anthropic-ai/claude-code` (or current package name)
- `~/.claude/` directory exists (created automatically on first Claude Code run)

---

## Implementation

### 1. Locate or create the settings file

```
~/.claude/settings.json
```

On Windows: `C:\Users\<USERNAME>\.claude\settings.json`

If the file already exists, merge the `permissions` block into it — do not overwrite existing `hooks`, `model`, or other keys.

### 2. Add the permissions block

The full target `settings.json` structure (merge with any existing content):

```json
{
  "permissions": {
    "allow": [
      "Read",
      "Glob",
      "Grep",
      "LS",
      "Edit",
      "MultiEdit",
      "Write(src/**)",
      "Write(tests/**)",
      "Write(docs/**)",
      "Write(workspace/**)",
      "Bash(npm run *)",
      "Bash(npm test *)",
      "Bash(npm install *)",
      "Bash(npx *)",
      "Bash(node *)",
      "Bash(python *)",
      "Bash(git status)",
      "Bash(git diff *)",
      "Bash(git log *)",
      "Bash(git add *)",
      "Bash(git commit *)",
      "Bash(git checkout *)",
      "Bash(git fetch *)",
      "Bash(git pull *)",
      "Bash(git branch *)",
      "Bash(openclaw *)",
      "Bash(claude *)",
      "Bash(codex *)",
      "Bash(ls *)",
      "Bash(cat *)",
      "Bash(echo *)",
      "mcp__*"
    ],
    "deny": [
      "Read(**/.env*)",
      "Read(**/.dev.vars*)",
      "Read(**/*.pem)",
      "Read(**/*.key)",
      "Read(**/secrets/**)",
      "Read(**/credentials/**)",
      "Read(**/.aws/**)",
      "Read(**/.ssh/**)",
      "Read(**/config/database.yml)",
      "Read(**/config/credentials.json)",
      "Read(**/.npmrc)",
      "Read(**/.pypirc)",
      "Write(**/.env*)",
      "Write(**/secrets/**)",
      "Write(**/.ssh/**)",
      "Write(.github/workflows/*)",
      "Bash(rm -rf *)",
      "Bash(sudo *)",
      "Bash(git push *)",
      "Bash(npm publish *)",
      "Bash(curl * | sh)",
      "Bash(wget *)",
      "Bash(chmod *)"
    ],
    "defaultMode": "acceptEdits"
  }
}
```

---

## Key Decisions

| Rule | Reason |
|------|--------|
| `Bash(git push *)` in DENY | Pushing to remote is a deliberate human action. An unattended pipeline should commit but never push. |
| `Read(**/.env*)` in DENY | `.env` files contain live credentials. Claude reading them during grep/search is a leak vector. |
| `Write(.github/workflows/*)` in DENY | CI/CD pipeline files are high-risk; accidental modification can trigger builds or expose secrets. |
| `Bash(curl * | sh)` in DENY | Pipe-to-shell pattern is a remote code execution vector. |
| `mcp__*` in ALLOW | MCP tools (GitHub, Supabase, etc.) need to fire unattended in pipeline sub-agents. |
| `defaultMode: "acceptEdits"` | File edits are auto-accepted; only shell commands prompt when not explicitly listed. |

### Why no `Bash(*)` wildcard

A wildcard allows unattended execution of any shell command — including `rm -rf`, `sudo`, and `git push`. The explicit allow list covers the full set of commands needed for Fabrica pipeline sub-agents while the deny list acts as a hard floor. Deny always wins over allow on conflict.

---

## Verification

After writing the file, verify deny rules are active:

```bash
# From a Claude Code session, attempt to read a credential file.
# This should be blocked:
claude "read the file .env in the current directory"

# Expected response: permission denied / tool blocked
```

Check that pipeline commands are not blocked by testing a benign allowed command:

```bash
claude "run git status"
# Expected: executes without a permission prompt
```

---

## Adjusting for Your Environment

- If your project writes to directories other than `src/`, `tests/`, `docs/`, `workspace/`: add `Write(yourdir/**)` to the allow list.
- If you need `git push` in automated flows: move it from deny to allow and accept the risk.
- If you run additional CLI tools (e.g., `cargo`, `go`, `make`): add `Bash(cargo *)`, `Bash(go *)`, etc. to the allow list.
- Deny rules take precedence — you cannot allow a pattern that is also denied.

---

## Project-Level Override

You can also create a `.claude/settings.json` inside a specific project directory to override the global settings for that project only. Project-level settings merge with (and can tighten) global settings.

```
your-project/
└── .claude/
    └── settings.json   ← project-level overrides global
```
