# SMP Plugin Stack

FuckTick is tested on my SMP server. The private SMP environment targets a 30+ Folia-supported plugin stack, and this repository's local `run/plugins` directory currently exposes the JARs listed below.

This page is not a compatibility certificate. It is a living test stack: if one of these plugins breaks, the runtime has to explain why. If the runtime cannot explain why, that is also a bug.

## Current Visible Local Stack

Observed from `run/plugins` on 2026-06-18. Every JAR listed here declares `folia-supported: true` in `plugin.yml` or `paper-plugin.yml`.

| Plugin | Version in local stack | Author(s) from metadata | Public GitHub / source | Notes |
| --- | --- | --- | --- | --- |
| ArmorStandEditor | `26.1.2-51` | Wolfstorm and legacy contributors | [Wolfieheart/ArmorStandEditor](https://github.com/Wolfieheart/ArmorStandEditor) | Survival armor stand editing. |
| CoreProtect | `24.0-rc1` | Intelli | [PlayPro/CoreProtect](https://github.com/PlayPro/CoreProtect) | Block logging, lookup, rollback, restore. |
| Emotecraft | `3.3.0-a.build.150` | KomsX, dima_dencep | [KosmX/emotes](https://github.com/KosmX/emotes) | Server-side Bukkit/Paper plugin build for emotes. |
| FlectonePulse | `1.10.0` | TheFaser | [Flectone/FlectonePulse](https://github.com/Flectone/FlectonePulse) | Chat, messages, notifications, integrations. |
| GrimAC | `2.3.74-2.0-21f1534` | GrimAC | [GrimAnticheat/Grim](https://github.com/GrimAnticheat/Grim) | Simulation anticheat. |
| GSit | `3.4.2` | Gecolay | [Gecolay/GSit](https://github.com/Gecolay/GSit) | Sit, lay, crawl, pose features. |
| HeadDrop | `5.4.5` | RRS | [RRS-9747/HeadDrop](https://github.com/RRS-9747/HeadDrop) | Player and mob head drops. |
| LuckPerms | `5.5.54` | Luck | [LuckPerms/LuckPerms](https://github.com/LuckPerms/LuckPerms) | Permissions backend and group system. |
| mdMOTD | `1.1.5` | mdfnative-NotPatch | No public GitHub repository found; public listing exists on Spigot/BuiltByBit | MOTD customization. |
| PlaceholderAPI | `2.12.2` | HelpChat | [HelpChat/PlaceholderAPI](https://github.com/HelpChat/PlaceholderAPI) | Placeholder provider bridge. |
| ProtocolLib | `5.5.0-SNAPSHOT-b723ff3` | dmulloy2, comphenix | [dmulloy2/ProtocolLib](https://github.com/dmulloy2/ProtocolLib) | Minecraft protocol access layer. |
| SkinsRestorer | `15.12.3` | knat, AlexProgrammerDE, Th3Tr0LLeR, McLive, DoNotSpamPls | [SkinsRestorer/SkinsRestorer](https://github.com/SkinsRestorer/SkinsRestorer) | Offline/cracked server skin restore/change support. |
| TAB | `6.0.3` | NEZNAMY | [NEZNAMY/TAB](https://github.com/NEZNAMY/TAB) | Tablist, nametag, scoreboard-style presentation features. |
| ViaBackwards | `5.9.2-SNAPSHOT` | Matsv, kennytv, Gerrygames, creeper123123321, ForceUpdate1, EnZaXD | [ViaVersion/ViaBackwards](https://github.com/ViaVersion/ViaBackwards) | Older clients on newer servers. |
| ViaVersion | `5.9.2-SNAPSHOT` | _MylesC, creeper123123321, Gerrygames, kennytv, Matsv, EnZaXD, RK_01 | [ViaVersion/ViaVersion](https://github.com/ViaVersion/ViaVersion) | Newer clients on older/native servers. |
| Simple Voice Chat (`voicechat`) | `2.6.18` | Max Henkel, Matthew Wells | [henkelmax/simple-voice-chat](https://github.com/henkelmax/simple-voice-chat) | Proximity voice chat server plugin. |
| XSCraft | `1.0` | bejiihiu.xs | Local/private plugin, no public GitHub repository found | SMP-specific plugin. |

## Compatibility Reading

The important part is not that these plugins exist in the folder. The important part is what they stress:

- LuckPerms and PlaceholderAPI stress shared plugin state and dependency ordering.
- ProtocolLib, ViaVersion, ViaBackwards, GrimAC, and TAB stress packet, protocol, and player-facing callback paths.
- CoreProtect stresses database-backed block history and region-sensitive world interactions.
- SkinsRestorer and Simple Voice Chat stress login/session style behavior.
- FlectonePulse, mdMOTD, TAB, and PlaceholderAPI stress command, tab-complete, chat, ping, and formatting integrations.
- GSit, ArmorStandEditor, HeadDrop, and XSCraft stress entity/player/world-adjacent gameplay behavior.

That is exactly the kind of stack FuckTick needs. An empty test server lies. A real SMP with too many plugins tells the truth and then ruins your evening.

## How To Update This Page

When `run/plugins` changes:

1. Read the plugin metadata from the JAR.
2. Confirm whether `folia-supported: true` is still declared.
3. Update the version and author columns from `plugin.yml` or `paper-plugin.yml`.
4. Use the official GitHub repository when one exists.
5. If there is no public GitHub repository, say that plainly instead of inventing a link.
