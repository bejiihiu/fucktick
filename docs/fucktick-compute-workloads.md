# FuckTick Compute Workloads

This page documents the workload side of FuckTick Compute Broker: what is wired into runtime today, what exists as a snapshot-only engine, and what must stay on Folia owner threads until the snapshot/apply boundary is explicit.

The rule is still the same:

```text
snapshot only on owner context
compute only from immutable data
apply only on the owning context
```

If a workload cannot follow that rule, it does not get moved just because it is expensive.

## Current Runtime Wiring

| Workload | Compute task type | Current entry point | Owner/live boundary |
| --- | --- | --- | --- |
| Diagnostics formatting | `DIAGNOSTIC_FORMAT` | `/watchcat compute ...`, plugin dump, queue, routing views | Caller gathers counters and short sections, worker formats text, caller sends output. |
| Routing metadata warmup | `ROUTING_METADATA_WARMUP` | `/watchcat plugin <name> routing` and plugin enable warmup | Server snapshots listener class names and plugin command labels, worker validates metadata strings. |
| Chunk save NBT serialization | `CHUNK_SAVE_SERIALIZE` | `NewChunkHolder.saveChunk(...)` | Region/chunk owner creates `SerializableChunkData`, worker runs `chunkData.write()`, Moonrise region IO writes the resulting `CompoundTag`. |
| Payload write helper | dedicated compute IO worker | `FuckTickComputeIoWorkerPool.writeChunkPayload(...)` | Generic compute workers never block on this payload write path. |

Diagnostics and routing have timeout fallbacks because command callers expect immediate output. Chunk save has a correctness fallback: if compute submission or execution fails, the old scheduler-side serialization path is used instead of dropping the save.

## Snapshot Engines

The workload package also contains engines for candidate work:

- `EntityPathfindingSnapshot` -> `PathfindingPlanResult`
- `ExplosionSnapshot` -> `ExplosionPlanResult`
- `ScoreboardRenderSnapshot` -> `ScoreboardRenderResult`
- `ChunkSaveSnapshot` -> `ChunkSavePayloadResult`

These are not placeholders. They execute on immutable input, handle cancellation, and are covered by tests. They are also not a license to call live Bukkit, NMS, or Folia APIs from a compute worker.

## Why Pathfinding Is Not Directly Wired Yet

The current vanilla pathfinding path still reads live `Level`, chunk, collision, entity, and event state while building the path. Moving that call as-is would only move the bug to another thread.

Safe wiring needs a separate adapter:

```text
entity owner thread snapshots navigation input
compute worker builds a candidate path from snapshot data
entity owner thread checks generation and installs path
```

Until that adapter exists, the compute pathfinding engine remains available for snapshot-backed planning and tests, while live NMS navigation keeps its owner-thread semantics.

## Why Explosion Is Not Directly Wired Yet

Explosion code is even more sensitive. The calculation path reads live block state, fluid state, chunk caches, entity exposure, Bukkit events, and then mutates blocks/entities.

The safe split is:

```text
region owner snapshots resistance/collision/candidate data
compute worker builds an explosion plan
region/entity owner contexts apply effects
```

Multi-region explosions need explicit region partitioning. One random owner thread must not mutate every affected region.

## Scoreboard And Placeholder Rendering

The compute workload can render scoreboard-style templates from snapshot strings. Real plugin integrations like TAB and PlaceholderAPI still need their own snapshot layer because many placeholder providers read plugin-owned or player-owned live state.

The safe shape is:

```text
owner or plugin runtime snapshots player/plugin values
compute worker formats strings
network/owner context sends packets or updates live scoreboard objects
```

The worker may format strings. It must not read live `Player`, `Scoreboard`, or plugin objects.

## Backpressure

Workload priority matters:

```text
CRITICAL_SAVE > HIGH_ENTITY_AI > NORMAL_WORLD_COMPUTE > LOW_DIAGNOSTICS > BACKGROUND_WARMUP
```

Chunk save serialization is critical and has a fallback. Routing warmup is background work and can be rejected under pressure. Diagnostics use bounded waits and fallback formatting so a command does not become an unbounded region-thread stall.

## Verification

The focused test suite exercises:

- worker thread naming;
- owner-context apply;
- stale result discard;
- cancellation before run;
- priority displacement;
- type-specific budget rejection;
- immutable snapshot copying;
- live-state access guard diagnostics;
- result futures and bounded wait fallback;
- dedicated IO worker payload writes;
- all candidate workload engines.

The broader acceptance run is still `gradlew check` after patch regeneration. A passing focused suite means the compute layer is coherent; it does not prove every future live workload adapter is safe.
