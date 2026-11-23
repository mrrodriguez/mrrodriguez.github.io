---
layout: post
title: "Layered Security and Sandboxing with Pi"
author: Mike Rodriguez
date: 2026-06-19
last_modified_at: 2026-11-22
tags:
  [
    pi,
    coding-agent,
    security,
    sandboxing,
    layered-defense,
    guardrails,
    agent-safety,
    typescript,
  ]
---

# Introduction

Giving an agentic AI access to a local terminal and filesystem is necessary to
take advantage of their capabilities, but it introduces significant security
risks. A runaway shell command, accidental deletion, credential exfiltration, or
an excessive token output loop are risks that must be accounted for.

To mitigate these risks using the Pi coding agent (see:
[https://github.com/earendil-works/pi](https://github.com/earendil-works/pi)),
I have configured a layered defense model. This includes OS-level sandboxing
and file-tool policy enforcement, user-in-the-loop permission gating,
read-only/read-write directory scoping bridged live to the kernel sandbox,
zero-plaintext Keychain credential resolution, and runtime circuit breakers via
extensions.

<!-- prettier-ignore -->
* TOC
{:toc}

---

# The Layers

Our security setup is structured as a series of layers, moving from hard OS-level
enforcement up to soft human and economic oversight:

1. **Enforcement (Hard Sandbox & Policy Gate)**: OS-level kernel boundaries for
   shell commands plus matching `sandbox.json` filesystem enforcement for
   built-in non-shell file tools. Network calls and filesystem writes are
   blocked by the OS kernel, preventing unauthorized actions even if the agent
   bypasses upper-level checks.
2. **Oversight (Permission Gate)**: Human-in-the-loop approvals. Prompts the
   user before executing potentially destructive commands (e.g., `sudo`, `rm
-rf`, `chmod 777`) or modifying files.
3. **Governance (Scoped Access & Protected Paths)**: Directory boundaries and
   hardcoded credential/config blocks. Prevents the agent from wandering outside
   the project root (while supporting live `read-only` and `read-write` scope
   expansion) or touching sensitive credentials and live Pi runtime configs.
4. **Resilience (Circuit Breaker & Truncation)**: Economic and state guards.
   Prevents infinite silent tool execution loops and context-window token bloat.

---

# Step 1: Hard Sandboxing (Unified OS Isolation & File Policy)

The most important of the layers is a **Hard Sandbox** package. It uses the
`@anthropic-ai/sandbox-runtime` library to invoke OS-level containerization:
**Seatbelt** (`sandbox-exec`) on macOS, and **Bubblewrap** (`bwrap`) on Linux.

### The Evolution: Unified Mutation & Policy Interception

Earlier versions of Pi shipped an `examples/extensions/sandbox` extension that
replaced the built-in `bash` tool via `pi.registerTool`, and later Pi 1.5.0+
removed that example in favor of VM/container documentation. On macOS, however,
full Linux VMs and Docker break host macOS Keychain integration, Apple Silicon
Metal inference (`ds4`), local language tool chains such as Clojure/Babashka nREPLs,
and native host toolchains.

To retain lightweight host-native isolation without replacing built-in tools, my
public
[`packages/sandbox`](https://github.com/mrrodriguez/pi-agent-recipes/tree/main/packages/sandbox)
package implements a **Unified Mutation & Policy Interceptor** on Pi's
`tool_call` event:

1. **Shell Isolation**: Wraps `bash` command strings in Seatbelt (`sandbox-exec`)
   or Bubblewrap (`bwrap`) right before execution, composing cleanly with
   upstream pre-flight extensions.
2. **Built-in File Tool Policy**: Enforces the same `sandbox.json` filesystem
   rules (`denyRead`, `denyWrite`, `allowWrite`, plus active session `/scope`
   grants) on built-in non-shell file tools (`read`, `ls`, `grep`, `find`,
   `write`, `edit`, `replace`).
3. **Live Scoped Write Expansion (`pi.events`)**: Listens to
   `sandbox:allow-write`, `sandbox:revoke-write`, and `sandbox:allow-write-once`
   events emitted by `scoped-access.ts`. Cross-repo write approvals update
   `SandboxManager` live in memory (or per command via `customConfig`) without
   restarting Pi.

_Note on `context-mode`_: An earlier version of this setup also intercepted
`context-mode` shell execution tools (`ctx_execute`, `ctx_execute_file`). I have
since retired `context-mode` from my plugin chain: Pi's native output caps,
structured compaction, `smart-truncation.ts`, and `delegate_to_local` cover the
workflow without the ~2.5K system-prompt token overhead or unsandboxed polyglot
(`javascript`/`python`) and `ctx_batch_execute` bypass paths.

- **Source Code**:
  [`packages/sandbox/index.ts`](https://github.com/mrrodriguez/pi-agent-recipes/blob/main/packages/sandbox/index.ts)
- **Documentation & Setup**:
  [`packages/sandbox/README.md`](https://github.com/mrrodriguez/pi-agent-recipes/blob/main/packages/sandbox/README.md)

### Sandbox Configuration File (`sandbox.json`)

The global sandbox configuration defined in `~/.pi/agent/sandbox.json` (and
optionally overridden per-project in `<cwd>/.pi/sandbox.json`) controls
filesystem reads, writes, and allowed external network domains:

```json
{
  "enabled": true,
  "network": {
    "allowedDomains": [
      "npmjs.org",
      "*.npmjs.org",
      "registry.npmjs.org",
      "registry.yarnpkg.com",
      "pypi.org",
      "*.pypi.org",
      "github.com",
      "*.github.com",
      "api.github.com",
      "raw.githubusercontent.com"
    ],
    "deniedDomains": [],
    "allowLocalBinding": true
  },
  "filesystem": {
    "denyRead": [
      "~/.ssh",
      "~/.aws",
      "~/.gnupg",
      "~/.pi/agent/auth.json",
      "~/Library/Keychains"
    ],
    "allowWrite": [".", "/tmp", "/private/var/folders", "/var/folders"],
    "denyWrite": [
      ".env",
      "/**/.env",
      "/**/.env.*",
      "/**/*.pem",
      "/**/*.key",
      "~/.pi/agent/auth.json",
      "~/.pi/agent/models.json",
      "~/.pi/agent/settings.json",
      "~/.pi/agent/sandbox.json",
      "~/.pi/agent/mcp.json",
      "~/.pi/agent/extensions"
    ]
  }
}
```

A few important hardening details in this policy:

- **No broad parent-directory write grant**: Instead of allowlisting a broad
  parent directory containing all of your repositories (such as a `~/Projects`
  or `~/code` workspace root), default kernel writes stay locked to the current
  working directory (`.`) and macOS temporary directories (`/tmp`,
  `/private/var/folders`, `/var/folders`). If a session needs to edit a sibling
  repository, write access is granted dynamically on demand via
  `scoped-access.ts`.
- **Depth-independent `denyWrite` globs**: Using `/**/`-rooted globs (`/**/.env`,
  `/**/.env.*`, `/**/*.pem`, `/**/*.key`, alongside `.env` for Linux `bwrap`)
  prevents nested writes from slipping past root-anchored patterns.
- **Host Pi config tampering protection**: Live runtime configs under
  `~/.pi/agent/` (`auth.json`, `models.json`, `settings.json`, `sandbox.json`,
  `mcp.json`, `extensions`) are write-protected so a compromised tool call
  cannot weaken the sandbox or inject extensions.

### Zero-Plaintext Credentials (`auth.json`)

Rather than exporting API keys into `process.env` (where any child process can
read them), credentials can be configured in `~/.pi/agent/auth.json` (`0600`
permissions) using Pi's native `!command` resolution with any CLI secret
manager. For example, demonstrating with macOS Keychain:

```json
{
  "deepseek": {
    "type": "api_key",
    "key": "!security find-generic-password -a \"$USER\" -s DEEPSEEK_API_KEY -w"
  }
}
```

Pi's `AuthStorage` resolves and caches these values inside the host process
without exporting them to `process.env`, while Seatbelt `denyRead` and
`protected-paths.ts` block child processes and built-in file tools from reading
`~/.pi/agent/auth.json` or querying `~/Library/Keychains`.

---

# Step 2: Safety Oversight (Permission Gating)

While the Hard Sandbox enforces strict system limits, the **Permission Gate**
provides a softer, interactive user authorization layer. It allows me to
explicitly approve or deny actions before they run, preventing accidental
filesystem overwrites or destructive command execution.

It monitors `bash` for dangerous patterns (`rm -rf`, `sudo`, `chmod`/`chown`
`777`) as well as mutating file tools (`write`, `edit`, `replace`), and
features:

- **Trust Edits Mode**: Toggleable via `/trust-edits` or `/trust-edits on/off`.
  When enabled, the agent can write or modify files without triggering prompts,
  but destructive terminal commands remain gated.
- **Monitored Tool Status**: Displayed using `/permissions`.

- **Source Code**:
  [`extensions/permission-gate.ts`](https://github.com/mrrodriguez/pi-agent-recipes/blob/main/extensions/permission-gate.ts)

---

# Step 3: Governance (Scoped Access & Protected Paths)

To keep the agent focused on the code context and prevent it from touching
sensitive credentials or live runtime config, I load two extensions from my
public [pi-agent-recipes](https://github.com/mrrodriguez/pi-agent-recipes)
repository.

### 1. Scoped Access (`scoped-access.ts`)

The
[`scoped-access.ts`](https://github.com/mrrodriguez/pi-agent-recipes/blob/main/extensions/scoped-access.ts)
extension restricts file-related tool calls and `bash` commands to the
session's launch directory by default, deduplicating multi-path `bash` commands
so you only get prompted once per out-of-scope directory.

Crucially, it separates **read-only** from **read-write** approvals (both
one-shot and session-scoped) and bridges write grants and revocations directly
to `packages/sandbox` over `pi.events`. Approving a directory as `read-only`
silences repeated read prompts while keeping the OS sandbox write-locked;
approving `read-write` expands Seatbelt/Bubblewrap `allowWrite` live without
restarting Pi (and revoking it tightens the kernel lock again immediately).

See [`docs/scoped-access.md`](https://github.com/mrrodriguez/pi-agent-recipes/blob/main/docs/scoped-access.md)
for prompt options and `/scope` command usage.

### 2. Protected Paths (`protected-paths.ts`)

[`protected-paths.ts`](https://github.com/mrrodriguez/pi-agent-recipes/blob/main/extensions/protected-paths.ts)
provides standalone, zero-config guardrails across built-in file tools with no
dependency on `sandbox.json`. Even if the agent is granted CWD or expanded
directory access, it blocks secret/key filenames (`.env`, `.env.*`, `*.pem`,
`*.key`) at any depth, sensitive credential stores and system roots (`.ssh/`,
`.aws/`, `.gnupg/`, `.pi/agent/auth.json`, `Library/Keychains/`, `/etc/`, etc.),
and writes to live host Pi configuration under `~/.pi/agent/`.

---

# Step 4: Destructive Action & Economic Circuit Breakers

To guard against runaway cost loops and destructive state changes, I have
configured session state and cost protections.

### 1. Confirm Destructive Session Actions (`confirm-destructive.ts`)

The
[`confirm-destructive.ts`](https://github.com/mrrodriguez/pi-agent-recipes/blob/main/extensions/confirm-destructive.ts)
extension hooks into Pi's lifecycle events to prompt for confirmation before
state-clearing actions:

- **`session_before_switch`**: Prompts the user before clearing message history
  (e.g., executing a new session) or switching sessions when the current session
  has unsaved user inputs.
- **`session_before_fork`**: Warns before branching the session state from an
  earlier historical event.

### 2. Economic Circuit Breaker (`cost-protections.ts`)

In agentic workflows, an agent can get stuck in a "silent execution loop,"
running tools continuously without outputting text to the user. This can drain
API tokens unexpectedly fast.

To prevent this,
[`extensions/cost-protections.ts`](https://github.com/mrrodriguez/pi-agent-recipes/blob/main/extensions/cost-protections.ts)
acts as an opt-in circuit breaker: if the agent runs tools consecutively past a
configurable silent-turn threshold without sending user-facing text, the session
pauses and requires manual approval to continue.

### 3. Context Truncation Guard (`smart-truncation.ts`)

Large tool outputs (e.g., listing a large directory or printing build logs) can
flood the context window, degrading agent performance and increasing costs.

[`extensions/smart-truncation.ts`](https://github.com/mrrodriguez/pi-agent-recipes/blob/main/extensions/smart-truncation.ts)
intercepts tool results after they run. When text output exceeds a configurable
character threshold, it transparently truncates the middle of the payload,
leaving the head and tail intact and instructing the agent to use targeted
line-range reads if the omitted section is needed.

See the [pi-agent-recipes README](https://github.com/mrrodriguez/pi-agent-recipes#7-cost-protections--truncation-cost-protectionsts-and-smart-truncationts)
for configuration flags and environment variables for these guards.

---

# Pipeline Ordering & Sub-Agent Inheritance

Pi loads extensions alphabetically by default. This default behavior can
introduce dependency order issues in security extension composition pipelines.
For instance, if the `sandbox` extension runs first, it will wrap the shell
command string in `bwrap` or `sandbox-exec` commands. When the `scoped-access`
or `permission-gate` extensions execute next, they will see the compiled
sandbox wrapper string instead of the raw, human-readable shell command. This
makes regex matching and terminal notification prompts unreadable.

To establish a deterministic pipeline, this setup separates the files:

- **Pre-flight Safety Gates** (`protected-paths.ts`, `scoped-access.ts`,
  `permission-gate.ts`, etc.) live in the global auto-discovery directory
  `~/.pi/agent/extensions/` (synced from `pi-agent-recipes` and `pi-dotfiles`
  via `sync-pi.sh`). They are loaded and run first, inspecting clean command
  strings and file paths prior to sandboxing.
- **The Enforcement Sandbox** is configured as a standalone package located in
  a separate directory (`packages/sandbox`), bypassing auto-discovery.
  - It is then explicitly registered at the **end** of the `packages` list in
    `settings.json` (with Pi's built-in MCP support removing the need for
    `pi-mcp-adapter`, and `context-mode` retired):

```json
{
  "packages": ["/path/to/pi-agent-recipes/packages/sandbox"]
}
```

Because Pi processes the explicitly defined `packages` array after resolving
auto-discovered extensions, this guarantees that **pre-flight checks run
first** on clean strings, while the **hard sandbox runs last**, enforcing
`sandbox.json` on built-in file tools and wrapping finalized `bash` commands
right before execution.

### Inheriting the Pipeline in Delegated Sub-Agents

If you also use [`delegate-local.ts`](https://github.com/mrrodriguez/pi-agent-recipes/blob/main/extensions/delegate-local.ts)
to offload tasks to a local worker model, the worker does not bypass this
pipeline. It registers `delegate_to_local` with `exposure: "model-only"` (to
prevent recursive delegation) and proxies every worker tool call through Pi's
`ctx.executeTool()`, ensuring local sub-agents pass through the exact same
`protected-paths`, `scoped-access`, `permission-gate`, `smart-truncation`, and
`packages/sandbox` layers as the orchestrator.

---

# Conclusion

By structuring agent configuration as a layered defense, you can get the safety
and isolation similar to a containerized environment, combined with
human-in-the-loop oversight for file changes and command execution, live
cross-repo scope management, and zero-plaintext credential handling. This setup
lets you run complex AI workflows on your machine with less worry about
destructive mistakes, security leaks, and/or unexpected runaway costs.
