---
layout: post
title: "Introducing Clara Rules Explorer"
subtitle: "Analyzing and Visualizing the Session Dependency Graph"
author: Mike Rodriguez
date: 2026-06-22
tags:
  [
    clojure,
    clara-rules,
    svelte,
    sveltekit,
    rules-engines,
    rete,
    visualization,
    frontend,
    open-source,
  ]
---

# Introduction

Maintaining and understanding large production rule systems built with [Clara
Rules](https://github.com/gateless/clara-rules) can be a challenge. Tracing the
dependencies and relationships between rules and queries is important to
keeping the system maintainable. It is useful to be able to trace this rulebase
dependency graph statically. Additionally, it is useful to overlay an active
working memory state directly onto that graph.

I have created a project to address this concern called **Clara Rules
Explorer**. This is an experimental tool composed of a Clojure HTTP server
layer and a SvelteKit UI designed to introspect rulebase structures, map
dependency links, and visualize runtime working memory state. You can try a
live interactive static demo of the tool at
[metasimple.org/clara-rules-explorer](https://www.metasimple.org/clara-rules-explorer/).

<!-- prettier-ignore -->
* TOC
{:toc}

---

# The Architecture

The project is structured into two main components:

1. **Introspection Server
   ([`server/`](https://github.com/mrrodriguez/clara-rules-explorer/tree/main/server))**:

- A Clojure HTTP API server that analyzes the compiled Rete network and current
  session state.
- It exposes a series of endpoints for static analysis (rules, queries, and
  fact-type structures) as well as runtime introspection (session working
  memory state, fact instance tracking, and rule execution activity).

2. **Visual Explorer UI ([`ui/`](https://github.com/mrrodriguez/clara-rules-explorer/tree/main/ui))**:

- A web application built on SvelteKit and Svelte, styled with Bootstrap.
- It interfaces with the Clojure backend to present an interactive visualization of the rules, queries, and fact relationships.

---

# Introspection Challenge: RHS Opaqueness

In Clara Rules, the Left-Hand Side (LHS) of a rule is highly structured. The
Clara compiler parses the LHS conditions to construct the Alpha and Beta join
nodes of the Rete network. This structured layout makes it straightforward to
inspect what fact types a rule matches or joins.

However, the Right-Hand Side (RHS) is unstructured. It is arbitrary Clojure
code executed when a rule fires:

```clojure
(defrule process-customer-status
  [Customer (= ?id id) (= :premium tier)]
  [?order <- Order (= ?id customer-id) (= :pending status)]
  =>
  (insert! (map->Discount {:order-id (:id ?order) :percentage 10}))
  (retract! ?order))
```

To a static analysis tool, the Clojure expression inside the RHS is just an unstructured list of forms. Because Clojure is dynamic, determining exactly what fact types a rule's RHS might insert or retract is statically undecidable. We might see `(insert! x)` where `x` is a variable bound elsewhere, or the insertion might happen inside nested helper functions.

Without knowing what fact types are inserted or retracted by the RHS of each
rule, we cannot determine the dependency links between rules. We cannot answer
the important question: _Which rules produce the facts that satisfy this
other rule's conditions?_

---

## Bridging the Gap with Out-of-Band Annotations

Clara Rules Explorer introduces an **out-of-band annotation system** that lets
you declare the missing metadata. The server combines this metadata with static
compiler analysis to construct a complete dependency graph.

Annotations can be declared in two ways:

### Path A: In-Code Rule `:props` Map

You can attach metadata directly to the rule definition using custom `:props` keys:

```clojure
(defrule process-customer-status
  {:clara-rules/insert-types [my.app.Discount]
   :clara-rules/retract-types [my.app.Order]
   :clara-rules/notes "Processes premium order discounts."}
  [Customer (= ?id id) (= :premium tier)]
  [?order <- Order (= ?id customer-id) (= :pending status)]
  =>
  (insert! (map->Discount {:order-id (:id ?order) :percentage 10}))
  (retract! ?order))
```

> **Note**: I typically advise against explicit retractions in favor of letting the Truth Maintenance System (TMS) keep working memory state logically consistent automatically; [see my previous post for more details](http://www.metasimple.org/2017/12/23/clara-updating-facts.html).

### Path B: Sidecar EDN File

If you cannot or do not want to modify existing production rule files, you can
define annotations in a sidecar EDN file. This is loaded by the HTTP server at
startup and can be reloaded at runtime:

```edn
{"my.app/process-customer-status"
 {:clara-rules/insert-types [my.app.Discount]
  :clara-rules/retract-types [my.app.Order]
  :clara-rules/notes "Processes premium order discounts."}}
```

The server resolves these annotations using
[`clara.server.tools.graph.annotations/resolve-annotations`](https://github.com/mrrodriguez/clara-rules-explorer/blob/main/server/src/clara/server/tools/graph/annotations.clj#L88),
merges them with any existing `:props` metadata, and exposes them via the
`/v1/analysis` and `/v1/rules/:fq-name` endpoints.

---

# Mapping Working Memory State

While introspecting the static rulebase structure is useful on its own, connecting it to the active working memory state provides far more value. At runtime, we need to see what facts currently reside in working memory and trace how those active instances map back to the Rete network.

The Clojure server's
[`clara.server.tools.graph.memory`](https://github.com/mrrodriguez/clara-rules-explorer/blob/main/server/src/clara/server/tools/graph/memory.clj)
namespace handles analyzing and snapshotting the session memory. When a session
is active, it runs an inspection and groups facts by their types, mapping them
to their:

- **Origin**: The rule activation that caused the fact to be inserted.
- **Impact (Used-by)**: The rule/query activations that currently match on the
  fact.

This bridges the gap between static topology and runtime execution, letting you
trace how a specific fact propagated through the Rete network.

---

# The UI and Graceful Fallbacks

A central design goal for both the analysis server and the explorer UI is that they must remain functional even when a rulebase is only partially annotated. The server tracks rules whose RHS appears to call `insert!` or `retract!` but lack matching annotations, flagging them as `:unresolved` via the `/v1/analysis` endpoint:

```json
{
  "rule": "my.app/process-customer-status",
  "reason": "RHS likely contains insertion/retraction calls but no :clara-rules/insert-types or :clara-rules/retract-types declared.",
  "hint": "Add :clara-rules/insert-types to the rule's properties map or a sidecar annotation file."
}
```

In the UI, these rules are flagged with warning badges, prompting the rule author to track down what is missing and declare the necessary metadata.

---

# Future Directions: Automating Metadata

Manually maintaining annotations can become a chore. The out-of-band annotation
design is specifically intended to act as an integration layer for automated
metadata generation:

1. **Deterministic Rulebase Analysis**: Future updates to the server-layer (or
   to the `clara-rules` compiler itself) could incorporate macro-expansion
   analysis or static code tracing to automatically infer the types of simple
   insertions.
2. **Agentic Code Analysis**: For complex or dynamic rules where deterministic
   tools fall short, we can employ modern LLM coding agents to analyze the
   Clojure rules, inspect their RHS forms, and write the sidecar EDN annotations.

---

# Conclusion

By combining static rulebase network analysis with a flexible, out-of-band
annotation system, Clara Rules Explorer makes it possible to visualize the
complex web of rule and query dependencies and active working memory.

The project is available on GitHub at
[mrrodriguez/clara-rules-explorer](https://github.com/mrrodriguez/clara-rules-explorer),
and a live interactive static demo is hosted at
[metasimple.org/clara-rules-explorer](https://www.metasimple.org/clara-rules-explorer/).
It is in an experimental state, but the core patterns for visual dependency
tracing and runtime introspection are already showing promising results.
