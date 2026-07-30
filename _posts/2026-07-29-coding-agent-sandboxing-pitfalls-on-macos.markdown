---
layout: post
title: "Coding Agent Sandboxing Pitfalls on macOS"
author: Mike Rodriguez
date: 2026-07-29
tags:
  [
    pi,
    coding-agent,
    security,
    sandboxing,
    macos,
    seatbelt,
    playwright,
    babashka,
    chromium,
  ]
---

# Introduction

In a [previous post](/2026/06/19/layered-security-and-sandboxing-with-pi.html), I outlined a layered security model for the Pi coding agent, emphasizing a "Hard Sandbox" utilizing macOS Seatbelt (`sandbox-exec`). Applying strict OS-level containerization provides an important layer of security, however, it can cause issues with various dev tools.

Recently, while using the sandboxed agent, I encountered roadblocks when attempting to run end-to-end web tests and evaluate native code. The problems and solutions outlined here are specific to macOS, encompassing browsers failing on startup, scripts failing to write temporary files, and "Operation not permitted" local network errors.

This post will describe how I resolved the Chromium/Playwright conflicts in our public [pi-agent-recipes](https://github.com/mrrodriguez/pi-agent-recipes) repository, and the strict network pattern matching rules you may need to work around.

<!-- prettier-ignore -->
* TOC
{:toc}

---

# The Chromium / Playwright Problem

When the sandboxed agent attempted to run Playwright tests, the headless Chromium browser immediately failed with an error: `bootstrap_check_in ... Permission denied (1100)`.

Even with standard flags like `--no-sandbox` or `--single-process`, the browser refused to start. The issue stems from two conflicting sandboxing mechanisms colliding on macOS.

### 1. Mach IPC Ports

Chromium's multi-process architecture (Main, GPU, Renderer, Network) relies heavily on macOS Mach IPC ports to communicate. Specifically, the browser process attempts to register a rendezvous server using ports prefixed with `org.chromium.`.

By default, the Pi Seatbelt sandbox blocks all `mach-register` and `mach-lookup` operations to prevent sandbox escapes. To fix this, we updated the `packages/sandbox` configuration in `pi-agent-recipes` to explicitly punch a hole for Chromium's IPC. We use `patch-package` to inject the following rules into the sandbox profile:

```lisp
(allow mach-register (global-name-regex #"^org\.chromium\."))
(allow mach-lookup (global-name-regex #"^org\.chromium\."))
```

_Note: If you are scripting Seatbelt profiles dynamically, be extremely careful with backslash escaping in regex strings!_

### 2. Nested Sandboxing

Even with Mach IPC allowed, Playwright requires specific configuration to run inside an existing sandbox. Chromium attempts to initialize its _own_ internal Seatbelt sandbox for its renderer processes. Because macOS does not support nested `sandbox-exec` initialization, the browser will crash trying to communicate with kernel sandboxing daemons.

To run Playwright inside the agent's sandbox, you must explicitly disable Chromium's internal sandbox in your `playwright.config.ts`:

```typescript
export default defineConfig({
  use: {
    launchOptions: {
      args: process.env.PI_CODING_AGENT
        ? ["--no-sandbox", "--disable-setuid-sandbox"]
        : undefined,
    },
  },
});
```

This combination—patching the outer sandbox to allow Mach IPC, and disabling the inner browser sandbox—allows Playwright to function seamlessly.

---

# The Temporary Directory Problem

A more subtle issue occurred when the agent attempted to execute scripts that compile or evaluate code (like `clj-nrepl-eval`). The scripts failed with `Operation not permitted` when trying to write temporary configuration files.

My local `sandbox.json` policy explicitly allowed writing to `/tmp`. However, macOS does not actually use `/tmp` (which is just a symlink to `/private/tmp`) for its default temporary directories. Instead, the OS sets the `$TMPDIR` environment variable to a uniquely generated path under `/var/folders/` (e.g., `/var/folders/kn/.../T/`).

Because `/var/folders` was not in the sandbox's `allowWrite` list, any standard tool relying on `$TMPDIR` or `java.io.tmpdir` was blocked.

To fix this, you must explicitly add the macOS folder structures to your local `~/.pi/agent/sandbox.json` file:

```json
{
  "filesystem": {
    "allowWrite": [".", "/tmp", "/private/var/folders", "/var/folders"]
  }
}
```

---

# The IP Pattern Matching Problem

Another issue with the sandbox involved local networking. When configuring the sandbox to allow local service binding (`"allowLocalBinding": true`), it permits outbound connections to `localhost`.

However, Apple's Seatbelt engine is literal when it comes to matching the host. The rule `(remote ip "localhost:*")` strictly matches pure IPv4 (`127.0.0.1`) and pure IPv6 (`::1`).

Modern network stacks, including Java, GraalVM, and Node.js, often default to using **dual-stack IPv6 sockets**. When an application attempts to connect to `127.0.0.1` using a dual-stack socket, the kernel translates the destination into an "IPv4-mapped IPv6 address," which looks like `::ffff:127.0.0.1`.

Seatbelt completely fails to recognize `::ffff:127.0.0.1` as "localhost" and aggressively blocks the connection.

### Working Around the Sandbox

The standard fix for this in the JVM ecosystem is to inject the environment variable `JAVA_TOOL_OPTIONS=-Djava.net.preferIPv4Stack=true`, forcing the application to use pure IPv4 sockets that Seatbelt recognizes. The Pi sandbox actually injects this automatically.

However, certain tools slip past this automatic fix. For example, **Babashka** (`bb`) is compiled as a GraalVM native binary. Native images intentionally ignore `JAVA_TOOL_OPTIONS` for security reasons. As a result, Babashka scripts inside the sandbox will consistently fail to connect to local ports.

Since this means that global environment variables cannot be relied upon, the most robust workaround was to create a shell wrapper script placed early in the system `$PATH` (e.g., `~/.local/bin/bb`):

```bash
#!/bin/bash
# Replace with the actual path to your original bb executable
exec /opt/homebrew/bin/bb -Djava.net.preferIPv4Stack=true "$@"
```

This intercepts all calls to the tool, forcibly injecting the IPv4 flag, and allowing the native binary to communicate within the strict bounds of the macOS sandbox.

While Babashka is just one concrete example, this dual-stack IP mismatch is a pervasive issue in strict sandboxing environments. When local connections mysteriously fail despite being allowlisted, checking how the networking stack formats its IP addresses is a good place to start.
