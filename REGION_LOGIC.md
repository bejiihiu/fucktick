# Region Logic Notes

FuckTick inherits Folia's regionized ticking model.

The baseline region logic is still Folia's model: loaded chunks are grouped into independent regions, and those regions tick independently and in parallel while preserving ownership rules. If you need the upstream explanation, read the official Folia documentation:

- [Folia overview](https://docs.papermc.io/folia/reference/overview/)
- [Folia region logic](https://docs.papermc.io/folia/reference/region-logic/)

## What FuckTick Changes

FuckTick does not currently replace the Folia regionizer. The fork builds experiments around the runtime that sits next to it:

- plugin-owned work is isolated on dedicated plugin threads;
- server-owned heavy computation can be moved through Compute Broker workers when it has an explicit snapshot/apply boundary;
- owner-bound world, block, chunk, entity, and player state must still be touched from the correct Folia owner context;
- event routing is conservative by default;
- unknown or unsafe events stay owner-only;
- future experiments may add per-player execution domains and wider compute-broker call-site coverage, but those must still respect region ownership.

The important distinction:

```text
Folia decides who owns region/world/entity state.
FuckTick decides how plugin-owned execution is isolated before it reaches that state.
Compute Broker decides where bounded CPU work runs after owner state has been snapshotted.
```

## Practical Rule

If code needs live world state, use the Folia scheduler that matches ownership:

- `RegionScheduler` for block, chunk, and location work;
- `EntityScheduler` for entity and player work;
- `GlobalRegionScheduler` for global server-owned work;
- `AsyncScheduler` or plugin-owned execution for non-tick work that does not touch owner-bound state.

If there is no location, chunk, entity, or other ownership anchor, FuckTick should not invent one. A wrong region is worse than no offload.
