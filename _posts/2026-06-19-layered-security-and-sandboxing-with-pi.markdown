---
layout: post
title: "Layered Security and Sandboxing with Pi"
author: Mike Rodriguez
date: 2026-06-19
tags: [pi, coding-agent, security, sandboxing, layered-defense, guardrails, agent-safety, typescript]
---

# Introduction

Giving an agentic AI access to a local terminal and filesystem is necessary to
take advantage of their capabilities, but it introduces significant security
risks. A runaway shell command, accidental deletion, or token-bleeding output
loop are risks that must be accounted for.

To mitigate these risks using the Pi coding agent (see:
[https://github.com/earendil-works/pi](https://github.com/earendil-works/pi)),
I have configured a layered defense model. This includes using containerized
sandboxing, user-in-the-loop permission gating, directory scoping, and runtime
circuit breakers via extensions.

<!-- prettier-ignore -->
* TOC
{:toc}

---

# The Layers

Our security setup is structured as a series of layers, moving from hard OS-level
enforcement up to soft human and economic oversight:

1. **Enforcement (Hard Sandbox)**: OS-level kernel boundaries. Network calls
   and filesystem writes are blocked by the OS kernel, preventing unauthorized
   actions even if the agent bypasses upper-level checks.
2. **Oversight (Permission Gate)**: Human-in-the-loop approvals. Prompts the
   user before executing potentially destructive commands (e.g., `sudo`, `rm
-rf`) or modifying files.
3. **Governance (Scoped Access & Protected Paths)**: Directory boundaries.
   Prevents the agent from wandering outside the project root or reading
   sensitive credentials (like `.env` or `~/.ssh`).
4. **Resilience (Circuit Breaker & Truncation)**: Economic and state guards.
   Prevents infinite tool execution loops and context-window token bleed.

---

# Step 1: Hard Sandboxing (Unified OS Isolation)

The most important of the layers is a **Hard Sandbox** package. It uses the
`@anthropic-ai/sandbox-runtime` library to invoke OS-level containerization:
**Seatbelt** (`sandbox-exec`) on macOS, and **Bubblewrap** (`bwrap`) on Linux.

### The Evolution: Overriding vs. Unified Mutation

The official Pi codebase provides a basic sandbox extension example (see
[packages/coding-agent/examples/extensions/sandbox/index.ts](https://github.com/earendil-works/pi/blob/v0.79.8/packages/coding-agent/examples/extensions/sandbox/index.ts)).
However, that example has design limitations:

1. It overrides the built-in `bash` tool by registering a custom tool
   implementation (`pi.registerTool`).
2. It only sandboxes commands executed via standard shell invocations, failing
   to catch commands run by third-party extensions or advanced execution
   adapters (such as [`context-mode`](https://github.com/mksglu/context-mode) execution tools).

To solve this, my public
[`packages/sandbox`](https://github.com/mrrodriguez/pi-agent-recipes/tree/main/packages/sandbox)
package implements a more unified strategy. Instead of replacing the `bash`
tool, it listens to Pi's `tool_call` event and mutates the command arguments
_before_ they are sent to any executor.

This makes the sandbox tool-agnostic. It intercepts standard `bash` commands as
well as, e.g., context-mode commands (`ctx_execute` and `ctx_execute_file` when the
language is `"shell"`).

- **Source Code**:
  [`packages/sandbox/index.ts`](https://github.com/mrrodriguez/pi-agent-recipes/blob/main/packages/sandbox/index.ts)
- **Documentation & Setup**:
  [`packages/sandbox/README.md`](https://github.com/mrrodriguez/pi-agent-recipes/blob/main/packages/sandbox/README.md)

### Sandbox Configuration File (`sandbox.json`)

The global sandbox configuration defined in `~/.pi/agent/sandbox.json` controls
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
    "denyRead": ["~/.ssh", "~/.aws", "~/.gnupg"],
    "allowWrite": [".", "/tmp", "~/Projects"],
    "denyWrite": [".env", ".env.*", "*.pem", "*.key"]
  }
}
```

---

# Step 2: Safety Oversight (Permission Gating)

While the Hard Sandbox enforces strict system limits, the **Permission Gate**
provides a softer, interactive user authorization layer. It allows me to
explicitly approve or deny actions before they run, preventing accidental
filesystem overwrites or command execution.

Like the sandbox, this gate is "context-mode aware," capturing potentially
dangerous commands in both standard `bash` and other tool call executors. It also
features:

- **Trust Edits Mode**: Toggleable via `/trust-edits` or `/trust-edits on/off`.
  When enabled, the agent can write or modify files without triggering prompts,
  but destructive terminal commands remain gated.
- **Monitored Tool Status**: Displayed using `/permissions`.

- **Source Code**:
  [`extensions/permission-gate.ts`](https://github.com/mrrodriguez/pi-agent-recipes/blob/main/extensions/permission-gate.ts)

---

# Step 3: Governance (Scoped Access & Protected Paths)

To keep the agent focused on the code context and prevent it from reading
sensitive system configuration, I load two extensions from my public
[pi-agent-recipes](https://github.com/mrrodriguez/pi-agent-recipes) repository.

### 1. Scoped Access (`scoped-access.ts`)

The
[`scoped-access.ts`](https://github.com/mrrodriguez/pi-agent-recipes/blob/main/extensions/scoped-access.ts)
extension restricts file-related tool calls to the session's launch directory
by default. If the agent attempts to read, write, or list files outside this
directory, or if it issues a `bash` command containing absolute or relative
paths referencing external files, the user is prompted to authorize the access.

- **Mid-session adjustments**: If the agent needs access to a sibling
  directory, you can add it to the allowed list using the `/scope add <path>`
  command or inspect active paths with `/scope list`.

### 2. Protected Paths (`protected-paths.ts`)

To protect highly sensitive configuration,
[`protected-paths.ts`](https://github.com/mrrodriguez/pi-agent-recipes/blob/main/extensions/protected-paths.ts)
explicitly blocks the agent from touching paths containing credentials or
system components.

Even if the agent is granted CWD access, this extension intercepts and blocks
tool execution targeting directories like `.git/`, `node_modules/`, system
roots (`/etc/`, `/var/`, `/usr/`), and environment files (`.env`,
`.env.local`).

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

To prevent this, the
[`extensions/cost-protections.ts`](https://github.com/mrrodriguez/pi-agent-recipes/blob/main/extensions/cost-protections.ts)
extension acts as a circuit breaker. If the agent runs tools consecutively for
a specified threshold (e.g., 16 turns) without sending user-facing text, the
agent stops and requires manual approval to continue.

### 3. Context Truncation Guard (`smart-truncation.ts`)

Large tool outputs (e.g., listing a large directory or printing build logs) can
flood the context window, degrading agent performance and increasing costs.

The
[`extensions/smart-truncation.ts`](https://github.com/mrrodriguez/pi-agent-recipes/blob/main/extensions/smart-truncation.ts)
extension intercepts tool results after they run. If the text output exceeds a
specific character threshold (e.g., 15,000 characters), it transparently
truncates the middle of the payload, leaving the head and tail intact. This
prevents token bleed while keeping relevant context visible.

---

# Pipeline Ordering (Why Sequencing Matters)

Pi loads extensions alphabetically by default. This default behavior can
introduce dependency order issues in security extension composition pipelines.
For instance, if the `sandbox` extension runs first, it will wrap the shell
command string in `bwrap` or `sandbox-exec` commands. When the `scoped-access`
or `permission-gate` extensions execute next, they will see the compiled
sandbox wrapper string instead of the raw, human-readable shell command. This
makes regex matching and terminal notification prompts unreadable.

To establish a deterministic pipeline, this setup separates the files:

- **Pre-flight Safety Gates** (`scoped-access.ts`, `permission-gate.ts`, etc.)
  live in the global auto-discovery directory `~/.pi/agent/extensions/`. They are
  loaded and run first, inspecting commands prior to sandboxing.
- **The Enforcement Sandbox** is configured as a standalone package located in
  a separate directory (`packages/sandbox`), bypassing auto-discovery.
  - It is then explicitly registered at the **end** of the `packages` list in
    `settings.json`:

```json
{
  "packages": [
    "/path/to/pi-mcp-adapter",
    "/path/to/context-mode",
    "/path/to/pi-agent-recipes/packages/sandbox"
  ]
}
```

Because Pi processes the explicitly defined `packages` array after resolving
auto-discovered extensions, this guarantees that **pre-flight checks run
first** on clean strings, while the **hard sandbox runs last**, wrapping the
finalized command right before execution.

---

# Conclusion

By structuring agent configuration as a layered defense, you can get the safety
and isolation similar to a containerized environment, combined with
human-in-the-loop oversight for file changes and command execution. This setup
lets you run complex AI workflows on your machine with less worry about
destructive mistakes, security leaks, and/or unexpected runaway costs.
