# FuckTick Compute Broker

FuckTick Compute Broker is the internal runtime for server-owned heavy computation. It exists to move expensive CPU work out of Folia owner threads without pretending that live world, chunk, entity, player, or Bukkit state is thread-safe.

The short rule:

```text
owner thread snapshots
compute worker calculates
owner thread validates and applies
```

The broker does not replace Folia schedulers. It sits beside them. `RegionScheduler`, `EntityScheduler`, `GlobalRegionScheduler`, and owner-context server code still decide where live state can be read or mutated.

## Runtime Model

The flow is intentionally split into three phases:

```text
Folia owner context
        |
        v
short immutable snapshot
        |
        v
FuckTick Compute Worker - <n>
        |
        v
ComputeResult with generation marker
        |
        v
Folia owner context validates and applies
```

The worker phase may sort, plan, format, serialize from immutable data, score candidates, and build result objects. It may not touch live owner-bound state.

The apply phase is the only place where the result may affect live server state, and even then it must pass the task's generation/deadline/cancellation checks first.

## Main Pieces

The server patch stack adds these internal classes under `io.papermc.paper.fucktick.compute`:

- `FuckTickComputeBroker`
- `ComputeWorkerPool`
- `ComputeTask`
- `ComputeSnapshot`
- `ComputeResult`
- `ComputeGeneration`
- `ComputeApplyHandle`
- `ComputeCancellationToken`
- `ComputeDiagnostics`
- `ComputeBudgetPolicy`
- `ComputeTaskHandle`

The worker pool is bounded, priority-aware, and owned by the broker. Worker threads are named:

```text
FuckTick Compute Worker - <n>
```

The broker tracks submitted, queued, running, completed, applied, stale, cancelled, failed, and rejected tasks. It also records queue time, compute time, and apply delay totals for diagnostics.

`ComputeTaskHandle` exposes both terminal state and applied result futures. Most code should submit work and continue. Rare synchronous callers may use the bounded wait helper:

```java
handle.awaitResultOrFallback(timeout, unit, fallback)
```

That path has a timeout, cancellation, a deadlock diagnostic, and a nested-wait guard. It is used for command output where the caller contract wants a concrete string immediately.

## Workload Models

The first implementation includes typed snapshot/result models for the workloads described in the original design:

- pathfinding planning;
- explosion precompute;
- chunk save serialization payload preparation;
- scoreboard, tablist, and placeholder rendering;
- diagnostics and large command output formatting;
- routing metadata warmup;
- location and block collision snapshot data.

These models are deliberately boring data carriers. That is the point. Workers should receive plain immutable data, not live objects that can sneak back into world ownership.

The current runtime wiring is intentionally conservative:

- `/watchcat` diagnostics submit `DIAGNOSTIC_FORMAT` tasks and fall back on timeout.
- `/watchcat plugin <name> routing` submits `ROUTING_METADATA_WARMUP` tasks and falls back on timeout.
- successful plugin enable submits background routing metadata warmup from listener and command snapshots.
- chunk save NBT serialization is routed through `CHUNK_SAVE_SERIALIZE` after `SerializableChunkData.copyOf(...)` has already captured owner-thread state.
- a dedicated compute IO worker pool exists for payload-style write work that must not occupy generic CPU workers.

Pathfinding and explosion engines exist as snapshot-only workload implementations and tests. They are not blindly wired into vanilla NMS live paths yet, because current NMS pathfinding and explosion code still performs live `ServerLevel`, collision, event, and mutation work inside the same call chain. That boundary needs a separate snapshot adapter before it is safe to move real mob navigation or explosion application off owner threads.

## Budget Policy

Compute Broker is not an unbounded executor.

Current budget knobs cover:

```text
maxQueuedComputeTasks
maxQueuedComputeTasksPerWorld
maxQueuedComputeTasksPerPlugin
maxPathfindingTasksPerTick
maxExplosionPrecomputeTasksPerTick
maxChunkSerializationTasks
maxComputeTimeMs
maxApplyDelayMs
```

When pressure is too high, the broker rejects new work or displaces lower-priority queued work for a higher-priority task. That keeps diagnostics and warmup from crowding out save or entity-AI work.

Priority classes are:

```text
CRITICAL_SAVE
HIGH_ENTITY_AI
NORMAL_WORLD_COMPUTE
LOW_DIAGNOSTICS
BACKGROUND_WARMUP
```

## Guard Rails

Compute workers mark their active task in `FuckTickComputeContext`. Guard helpers can then detect whether code is currently running in the compute runtime:

```java
FuckTickComputeContext.isComputeWorker()
FuckTickComputeContext.currentTaskType()
FuckTickComputeThreadChecks.requireNotOwnerAccess()
FuckTickComputeThreadChecks.requireImmutableSnapshot(...)
```

`AsyncCatcher` now gives compute workers a compute-specific failure message before the generic async access error:

```text
Compute task PATHFINDING attempted to access live world state from a compute worker.
Use owner-thread snapshot data and schedule validated apply instead.
```

That message is intentionally strict. If a worker needs live state, the task shape is wrong. Take a better snapshot or move that piece back into the owner context.

## Commands

The `/watchcat` command now exposes compute diagnostics:

```text
/watchcat compute status
/watchcat compute workers
/watchcat compute queue
/watchcat compute tasks
/watchcat compute task <id>
```

These commands are for runtime inspection, not for steering live task execution by hand. They show whether the broker is healthy, overloaded, stuck, or producing stale results.

## Stale Results

Stale is a normal outcome.

Examples:

- an entity dies while pathfinding is still running;
- a player changes world before a rendered UI result returns;
- a chunk unloads before a save payload ack;
- a block or explosion snapshot is no longer current;
- the apply deadline expires.

In these cases the broker marks the task stale and does not apply the result. Stale results are counted because they are useful signal, but they are not treated as crashes.

## Shutdown

The broker shuts down from the Minecraft server stop path. Queued and running work is cancelled, workers are asked to stop, and the server does not leave compute workers behind as stray non-daemon threads.

The shutdown hook lives in the Minecraft patch stack because server stop is upstream-owned code. The implementation remains FuckTick-owned through generated downstream patches.

## What This Does Not Promise

Compute Broker does not promise:

```text
Bukkit API thread safety
live world reads from worker threads
automatic async conversion of every workload
exactly identical timing for every AI or diagnostic decision
safe mutation outside the correct Folia owner context
```

It promises a safer shape:

```text
short owner-thread snapshot phase
bounded compute workers
validated owner-thread apply phase
stale result discard
diagnostics and backpressure
```

If ownership is unclear, keep the workload on the owner thread until the snapshot/apply boundary is explicit. Fast wrong code is still wrong.

## Acceptance Checks

The implementation is covered by the normal server test suite. The checks exercise worker thread naming, owner-thread apply, stale result handling, cancellation, queue pressure, priority displacement, snapshot guards, live-state access guards, graceful shutdown, workload model copying, and the metrics path.

The patch workflow deliverable is:

- `fucktick-server/paper-patches/features/0008-Add-FuckTick-compute-broker.patch`
- `fucktick-server/paper-patches/features/0009-Expand-FuckTick-compute-workloads.patch`
- `fucktick-server/paper-patches/features/0010-Route-FuckTick-diagnostics-through-compute-workloads.patch`
- `fucktick-server/paper-patches/features/0011-Warm-routing-metadata-after-plugin-enable.patch`
- `fucktick-server/minecraft-patches/features/0001-Shutdown-FuckTick-compute-broker.patch`
- `fucktick-server/minecraft-patches/features/0002-Route-chunk-save-serialization-through-compute-broke.patch`

These patches are generated from readable sources and carry the project patch author metadata.
