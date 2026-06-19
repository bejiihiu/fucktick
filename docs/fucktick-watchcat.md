# FuckTick WatchCat

WatchCat is the internal FuckTick runtime monitor. It lives on its own `FuckTick WatchCat` thread and watches FuckTick-owned execution targets instead of pretending every JVM thread can be safely controlled.

The default heartbeat timeout is `10_000ms`. If a target does not answer in time, WatchCat records an incident, captures the target thread dump when a thread id is available, and then runs the target's safe recovery path.

## Targets

WatchCat v1 monitors:

- `PluginFUCKLOADER` management execution;
- every live `FuckTick Plugin Thread - <plugin>`;
- every `FuckTick Compute Worker - <n>`;
- the FuckTick compute IO worker pool;
- the server watchdog bridge.

It does not monitor arbitrary JVM threads or unrelated Folia region threads. Folia ownership rules still apply.

## Recovery Model

WatchCat never uses `Thread.stop`.

Plugin runtime recovery:

```text
timeout -> dump -> cancel queued callbacks -> interrupt -> control heartbeat -> restart only if old thread stopped -> server escalation if unrecoverable
```

Compute worker recovery:

```text
timeout -> dump -> cancel running compute task token -> interrupt -> control heartbeat -> restart only if old worker stopped -> server escalation if unrecoverable
```

Server escalation requests a safe shutdown first and falls back to the existing close path if the server does not stop.

## Commands

```text
/watchcat status
/watchcat targets
/watchcat incidents
/watchcat threads
/watchcat target <id> dump
/watchcat target <id> ping
/watchcat target <id> recover
/watchcat plugin <name> dump
/watchcat plugin <name> queue
/watchcat plugin <name> routing
/watchcat compute status
/watchcat compute workers
/watchcat compute queue
/watchcat compute tasks
/watchcat compute task <id>
```

`/meow` is an alias for `/watchcat`.

## Settings

System properties:

```text
fucktick.watchcat.enabled=true
fucktick.watchcat.heartbeatPayload=Meow <3
fucktick.watchcat.heartbeatTimeoutMs=10000
fucktick.watchcat.heartbeatIntervalMs=10000
fucktick.watchcat.recoveryWaitMs=1000
fucktick.watchcat.incidentLimit=128
```

Keep heartbeat and recovery timeouts conservative. If a plugin or compute workload regularly needs longer than the default to respond to a control heartbeat, first investigate the blocking path instead of only raising the timeout.
