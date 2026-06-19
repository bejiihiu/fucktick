# Project Description

FuckTick is an experimental downstream fork of Folia.

The effective stack is:

```text
Paper -> Folia -> FuckTick
```

Folia is not skipped, replaced, or treated as a historical footnote. FuckTick imports Folia as the direct upstream runtime model, then layers its own experiments on top.

## Purpose

Folia already removes the old assumption that a Minecraft server has one global main thread. FuckTick explores the next layer of that idea: what if plugin-owned execution also gets a controlled runtime boundary?

The current focus is the plugin runtime:

- one management thread for plugin lifecycle orchestration;
- one dedicated worker thread per plugin;
- command and tab-complete execution through plugin-owned threads;
- AsyncScheduler tasks routed into the owning plugin runtime;
- conservative event routing;
- diagnostics for queue pressure, blocked callers, callback timeouts, and lifecycle state.

The next runtime layer is the FuckTick Compute Broker:

- bounded compute workers for server-owned CPU-heavy work;
- short owner-thread snapshot phases;
- validated owner-thread apply phases;
- stale result handling;
- diagnostics for queued, running, completed, applied, stale, cancelled, failed, and rejected compute tasks.

The long-term direction also includes per-player execution experiments, wider compute-broker call-site coverage, and stricter tools for finding unsafe cross-context access.

## Non-Goals

FuckTick is not trying to become "Folia but with a funnier name." It is also not trying to make every Bukkit plugin magically safe.

Non-goals:

- pretending there is a single global main thread;
- mapping legacy sync scheduler calls to random regions without an ownership anchor;
- making Folia or PaperMC responsible for FuckTick-specific bugs;
- hiding unsafe world/entity/chunk access behind compatibility hacks.

## Compatibility Statement

FuckTick can break Folia-compatible plugins. That is expected while this runtime is experimental.

The target is still practical compatibility with real Folia-supported plugins, not a research toy that only boots an empty server. The project is tested on an SMP environment with a 30+ Folia-supported plugin stack, and the currently visible local plugin stack is documented in [docs/smp-plugin-stack.md](docs/smp-plugin-stack.md).

Compatibility claims should be treated as observed behavior for this fork, not as guarantees from Folia upstream.

## Where To Read More

- [README](README.md)
- [Plugin runtime design](docs/fucktick-plugin-runtime.md)
- [Compute broker design](docs/fucktick-compute-broker.md)
- [SMP plugin stack](docs/smp-plugin-stack.md)
- [Region logic notes](REGION_LOGIC.md)
