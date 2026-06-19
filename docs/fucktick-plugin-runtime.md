# FuckTick Plugin Runtime

FuckTick's plugin runtime is built around one blunt idea: plugin-owned code should run in a predictable plugin-owned execution context, while world, chunk, entity, player, and region state must still follow Folia ownership.

This is not an attempt to make Bukkit, Paper, or Folia APIs magically thread-safe. That would be fake safety, and fake safety is how you get silent world corruption. FuckTick instead separates plugin computation from owner-bound mutation and makes the active runtime context visible.

## Short Model

```text
callback / command / task
        |
        v
plugin-owned execution
        |
        v
decision / result / effect
        |
        v
FuckTick routing
        |
        v
Folia owner context applies the change
```

The practical rule:

```text
plugin thread computes
owner thread mutates
```

If code needs live world state, it must enter the right Folia owner context. If there is no `Location`, `Chunk`, `Entity`, or other ownership anchor, the runtime must not pretend it can safely guess one.

## Why This Exists

Folia removes the old single-main-thread server model. Commands, events, scheduler callbacks, and plugin work can be reached from different region contexts. Legacy "just run it synchronously on the main thread" logic does not map cleanly to Folia because there is no single global main thread.

FuckTick adds a controlled plugin runtime:

- `PluginFUCKLOADER` owns plugin lifecycle orchestration;
- every plugin gets a dedicated `FuckTick Plugin Thread - <plugin>`;
- plugin commands and tab-complete run on the plugin thread with blocking result semantics;
- plugin `AsyncScheduler` work is routed to the owning plugin runtime;
- `RegionScheduler`, `EntityScheduler`, and `GlobalRegionScheduler` remain Folia owner-context schedulers;
- plugin logger output includes plugin, thread id, and runtime context.

## Lifecycle

`PluginFUCKLOADER` has a single management executor named `PluginFUCKLOADER`. `onLoad()` runs there so load-time work is serialized and does not mix with region ticking.

`onEnable()` and `onDisable()` run on the plugin's dedicated worker thread. The runtime marks those calls as `plugin-lifecycle`, which lets compatibility checks distinguish startup work from ordinary callbacks.

Runtime states:

```text
CREATED
LOADING
LOADED
ENABLING
ENABLED
DRAINING
DISABLING
DISABLED
FAILED
TERMINATED
```

Normal flow:

```text
CREATED -> LOADING -> LOADED -> ENABLING -> ENABLED
ENABLED -> DRAINING -> DISABLING -> DISABLED -> TERMINATED
```

Failure flow:

```text
ENABLED -> FAILED -> DISABLING -> DISABLED -> TERMINATED
```

While a runtime is draining, it stops accepting normal callbacks and cancels queued callback work. The classloader should not be considered safe to unload until the plugin thread has stopped or the shutdown timeout has been reached.

Startup lifecycle has a longer timeout via `fucktick.plugin.maxLifecycleTimeMs` because real plugins can do heavier initialization. Ordinary plugin callbacks use shorter callback timeouts.

## Queues And Reentrant Work

The runtime separates lifecycle work from ordinary callback work. Lifecycle work has its own queue, while callback work uses a bounded queue controlled by `fucktick.plugin.maxQueuedTasksPerPlugin`.

If plugin code running on its own plugin thread submits another task to the same runtime, FuckTick runs it inline. Without this, a plugin can deadlock itself by waiting for a task that is queued behind the currently running task.

Queue pressure is intentionally plugin-local:

```text
Plugin ExamplePlugin exceeded max queued tasks.
queued=10000
limit=10000
```

One plugin should not be able to turn the whole server into an unbounded task landfill.

## Commands And Tab Completion

Plugin commands run through a blocking call into the plugin thread:

```text
caller context
        |
        v
PluginFUCKLOADER.callPluginBlocking(...)
        |
        v
FuckTick Plugin Thread - <plugin>
        |
        v
boolean command result
```

Tab completion follows the same model and returns `List<String>`. The caller waits because the Bukkit command contract expects a concrete result, not a future.

This is one of the riskiest parts of the runtime. A caller can wait for a plugin thread while the plugin thread waits for an owner context. Blocking paths therefore have timeouts and diagnostics with `possibleDeadlock=true`.

The timeout is not a magic fix. It is a circuit breaker and a useful log line instead of a silent hang.

## AsyncScheduler

Folia `AsyncScheduler` plugin work is routed into the plugin runtime:

```text
plugin async task -> plugin queue -> FuckTick Plugin Thread - <plugin>
```

This makes plugin-owned async behavior more predictable. A plugin that is not internally synchronized at least does not have its own callbacks scattered across unrelated pool threads by default.

Tasks submitted through `AsyncScheduler` are queued even when the caller is already on that plugin's thread. This preserves the scheduler contract that `runNow` returns after scheduling instead of turning startup background checks into lifecycle-blocking work. Explicit reentrant `PluginFUCKLOADER` submissions still run inline when plugin code waits on its own submitted task.

This still does not make world access safe. Async/plugin-thread work must not directly mutate live world, chunk, block, entity, or player state. Use the right Folia scheduler or the compute -> effect -> apply model.

## Owner Schedulers Stay Owner Schedulers

FuckTick does not replace Folia's owner schedulers with plugin threads.

- `RegionScheduler` owns block, chunk, and location work.
- `EntityScheduler` owns entity and player work.
- `GlobalRegionScheduler` owns global server work.
- Async/plugin-owned execution is for work that does not need live owner-bound state.

If a legacy call has no ownership anchor, FuckTick should route it conservatively or leave it alone. Guessing the wrong region is worse than not offloading.

## Event Routing

Event routing is intentionally conservative:

```text
Event
 |
 v
EventRoutingRegistry
 |
 v
RoutingOverrideTable
 |
 v
GeneratedFuckTickEventRoutingTable
 |
 v
ConservativeRoutingPolicy
 |
 v
RoutingPlan
```

Unknown events stay owner-only:

```text
unknown event -> OWNER_ONLY
```

The current override table lives in `META-INF/fucktick-event-routing-overrides.yml`. It exists for events where package-name heuristics are not enough. Examples in the current stack:

- `BlockBreakEvent` is region owner-bound;
- legacy async chat-style events can be plugin-local;
- `ServerListPingEvent` is global and result-required.

The generated table currently uses broad rules:

- `.block.` -> region owner-bound;
- `.entity.` -> entity owner-bound;
- `.world.` and `.chunk.` -> region owner-bound;
- selected async server/player events -> plugin-local;
- server events -> global-safe;
- everything else -> unknown owner-bound.

This is not the final router. It is the safe first step: do not violate Folia ownership while the coverage expands.

## Compute -> Effect -> Apply

The long-term model for owner-bound behavior is:

```text
owner context creates safe input/snapshot
        |
        v
plugin thread computes decision
        |
        v
plugin returns result/effect
        |
        v
runtime applies effect in owner context
```

Example:

```text
BlockBreakEvent on region thread
        |
        v
safe snapshot
        |
        v
plugin listener computes
        |
        v
CancelEventEffect / RunRegionEffect / SendMessageEffect
        |
        v
effect applier applies legally
```

The API patch stack already contains the first pieces: `PluginEffect`, `PluginEffectDispatcher`, effect appliers, routing metadata, and snapshot records. Block snapshots store material as a key string instead of carrying live Bukkit object state.

## Context Guard And Logging

`FuckTickPluginContext` stores the current runtime context in a `ThreadLocal`.

Important contexts:

- `PluginFUCKLOADER`;
- `plugin-lifecycle`;
- `plugin-thread`;
- `region-owner`;
- `global-owner`;
- `async`.

`plugin.getLogger()` messages are prefixed like this:

```text
[plugin:<name>] [tid:<threadId>] [ctx:<context>] message
```

The thread id comes from `Thread.currentThread().threadId()`, not from the thread name.

Diagnostic commands:

```text
/fucktick plugins threads
/fucktick plugin <name> dump
/fucktick plugin <name> queue
/fucktick plugin <name> routing
```

These expose runtime state, thread name/id, queue sizes, running task, average and last task duration, blocked callers, and last owner context.

After a plugin enables successfully, the server snapshots the plugin's registered listener class names and command labels, then submits a background `ROUTING_METADATA_WARMUP` task to FuckTick Compute Broker. That warmup validates metadata without touching live world state and without blocking plugin enable completion.

## Compatibility Edges

The server patch stack already includes a few pragmatic compatibility rules:

- plugin lifecycle is not held under the `PaperPluginInstanceManager` monitor;
- sync event dispatch is allowed from a plugin thread when the current context is plugin-owned execution;
- recipe registration during `plugin-lifecycle` is allowed through `AsyncCatcher`;
- `CraftServer#isPrimaryThread()` can treat `plugin-lifecycle` as startup-primary for plugins that check it before the first ticks.

These are compatibility edges, not a new global main thread. They exist because real Folia-supported plugins often use old checks during startup even when their runtime behavior is otherwise Folia-aware.

## What Exists In The Patch Stack

Current patches already implement the first runtime layer:

- `fucktick-api/paper-patches/features/0001-Add-FuckTick-plugin-runtime-layer.patch` adds `PluginFUCKLOADER`, `PluginRuntime`, states, settings, snapshots, effect/routing scaffolding, logger prefixes, and unit tests.
- `fucktick-api/paper-patches/features/0002-Add-FuckTick-event-routing-overrides.patch` adds resource-driven routing overrides.
- `fucktick-api/paper-patches/features/0004-Store-block-snapshot-material-as-key.patch` simplifies block snapshots.
- `fucktick-api/paper-patches/features/0005-Use-lifecycle-timeout-for-plugin-startup.patch` separates lifecycle timeout from ordinary callback timeout.
- `fucktick-api/paper-patches/features/0006-Run-reentrant-plugin-tasks-inline.patch` handles reentrant task submission from the same plugin thread.
- `fucktick-server/paper-patches/features/0002-Wire-FuckTick-plugin-runtime.patch` wires lifecycle, commands, tab-complete, async scheduler, and diagnostics.
- `fucktick-server/paper-patches/features/0004-Avoid-plugin-manager-monitor-during-lifecycle.patch` avoids lifecycle monitor deadlocks.
- `fucktick-server/paper-patches/features/0005-Allow-plugin-thread-sync-event-dispatch.patch`, `0006-Allow-lifecycle-recipe-registration-on-plugin-thread.patch`, and `0007-Treat-plugin-lifecycle-as-startup-primary-context.patch` handle early real-world compatibility problems.
- `fucktick-server/paper-patches/features/0011-Warm-routing-metadata-after-plugin-enable.patch` queues listener and command metadata warmup through the compute broker after successful plugin enable.

## What It Does Not Promise

FuckTick does not promise that unsafe plugin code becomes safe.

It promises a more honest boundary:

```text
plugin-owned execution is isolated
Folia ownership rules are preserved
blocking paths have timeouts and diagnostics
unknown owner-bound work stays conservative
```

If plugin-thread code directly touches live world, entity, chunk, block, or player state from the wrong context, that is still a bug. The correct pattern is snapshot for computation, scheduler/effect for mutation.

## Minimum Acceptance Checks

For this runtime to be considered healthy:

- plugins from the SMP stack start without lifecycle errors;
- `onLoad()` runs on `PluginFUCKLOADER`;
- `onEnable()` and `onDisable()` run on `FuckTick Plugin Thread - <plugin>`;
- two plugins do not share the same plugin thread;
- callbacks for one plugin serialize on that plugin thread;
- commands and tab completion return results to the caller;
- `AsyncScheduler` work enters the owning plugin runtime;
- region/entity/global schedulers remain owner-context schedulers;
- unknown events stay owner-only;
- timeout diagnostics appear instead of silent hangs;
- `/fucktick plugin <name> dump|queue|routing` reports useful state.

If a Folia-compatible plugin breaks, do not immediately widen legacy scheduler behavior. First find which ownership or startup assumption failed, then add the smallest compatibility rule that preserves Folia's model.
