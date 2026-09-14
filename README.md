[CreativeGuard Website & Downloads](https://brendanwang.github.io/CreativeGuard/)

# CreativeGuard — Minecraft Forge 1.20.1

A server mod for Creative/privileged item provenance, durable audit logs, staff alerts, investigation, and explicitly confirmed quarantine. The same JAR optionally supplies client badges and tooltips. No other mod, database driver, or external service is required.

## Install

1. Use **Minecraft 1.20.1**, **Forge 47.4.0 or a later 47.x release**, and **Java 17**.
2. Put `creativeguard-1.0.0.jar` in the **server's `mods/` folder** and restart the server.
3. Run `/creativeaudit status` as an operator. Default staff access is operator level 2; quarantine requires level 3.
4. Optionally put the same JAR in each player's Forge 1.20.1 `mods/` folder for badges and hover tooltips. Players without CreativeGuard can join; logging, commands, and chat work without the companion. Other installed mods may have their own client requirements.

Settings are generated at `<world>/serverconfig/creativeguard-server.toml`. Client appearance is in `config/creativeguard-client.toml`. Place a server config in `defaultconfigs/` to use it for new worlds. `/creativeaudit reload` reloads the live file; queue size and retention apply after restart.

## Creation-only logging (1.0.1)

`general.creations_only=true` is the new default, including when upgrading an existing server config. New Creative stacks, quantity increases, clones, Creative-origin drops, and privileged item grants are recorded. Moves, swaps, splits, pickups, deletions, gamemode changes, and recipe previews no longer generate routine audit/chat messages. Administrative rollback records remain enabled. Set this option to `false` to opt back into the broader audit events.

Creative slot packets are balanced by item identifier and NBT, ignoring our own marker. The detector waits two quiet server ticks before reporting positive quantity changes, and allows ten seconds for a cursor pickup debit to be matched by placement. This avoids treating each source/destination slot update as a new grant. Vanilla provides no authoritative client cursor gesture: very delayed/reordered packets or deleting and then recreating the same item within that window remain ambiguous. Only positive net additions are reported. Existing provenance travels with ordinary movements.

To upgrade, stop the server, **replace** the 1.0.0 JAR with 1.0.1, and restart. Do not leave both versions in `mods/`. No config reset is needed.

## Included

- Validated Creative slot assignments, edits, deletions, drops, server container clones, and actual gamemode transitions.
- `/give`, `/item` (replace/from/modify), and `/loot` player/container/world paths through vanilla command execution.
- Hidden, namespaced NBT provenance; save/load, copy, split, pickup/drop, and normal container persistence.
- Player container click tracking, hopper transfer hooks, and nonsimulated Forge `ItemStackHandler` inserts/extracts.
- Crafting/trading/anvil/smithing/grindstone/stonecutter output lineage, furnace processing, and brewing. Preview records are labeled as previews and excluded from rollback.
- Asynchronous JSONL logging with durable acknowledgments, a bounded queue, crash-tail preservation/recovery, retention, indexed searches, filtered exports, and visible storage health.
- Per-family audiences (`ALL`, `OPS_ONLY`, `PERMISSION_NODE`, `NONE`), rate limits, trusted UUIDs, Forge permission nodes, and item/tag/count/action/NBT alert rules.
- Optional client badges across vanilla item decoration paths (including hotbar and container screens), configurable appearance, staff-only server-supplied tooltip detail, and server-controlled public visibility.
- Preview/confirmation quarantine of matching, unambiguous stacks in online inventories and currently open containers, with recovery files.
- A small optional Java adapter API for other mods (`CreativeGuardApi`).

## Important behavior and coverage boundaries

This implementation makes compatibility-oriented choices from the specification. It does **not** claim every possible mod integration or optional future feature is implemented. See [the requirement coverage matrix](docs/COVERAGE.md).

**Vanilla Creative gesture ambiguity:** menu grabs, pick-block, and some Creative screen clones produce indistinguishable slot packets. Positive net additions are recorded as `CREATIVE_CREATE` with `gesture=UNKNOWN`, never guessed as a particular mouse action. Server container `ClickType.CLONE` is recorded as `CLONE_STACK`. Paired move/split/swap updates do not create a record.

**Stack policy:** the base policy is `REJECT_MERGE`, using vanilla NBT equality. Marked/unmarked stacks and distinct origins remain separate; splits and stacks from the same origin still stack normally. This changes stackability but avoids globally rewriting item comparison and transfer semantics. Furnace accumulations and adapter-declared merges use conservative `MIXED` lineage. Global `CONSERVATIVE_TAINT` merging and exact per-unit lineage are not provided.

**Mod compatibility:** ordinary `ItemStack` serialization, standard container screens, vanilla handler subclasses, and Forge `ItemStackHandler` paths work generically. Mods that strip NBT, replace targeted vanilla methods, use virtual storage, perform custom recipes, or override these hooks may need an adapter. No modpack-wide compatibility claim is made. Required hook conflicts fail at startup rather than silently disabling audit coverage.

**Privacy:** item NBT contains public provenance IDs, origin/state/schema/action, and up to eight parent IDs. Names, UUID attribution, timestamps, command contents, and detailed lineage remain in staff-protected server logs. Operators/mods can still edit NBT; markers are evidence, not authentication. A modified client can read public marker data even if the server hides the badge.

**Commands/functions:** standard `Commands.performCommand` execution is tracked, including console/command-block invocations and nested `/execute` paths. Initiator is the enclosing command source; transformed `/execute as` identity is not independently reconstructed. Function/datapack or mod APIs that bypass that execution entry point need the adapter API. `/data`, Creative block placement/breaking, arbitrary machine APIs, fluid/energy conversions, and offline item grants are not silently inferred.

**No-client UI:** vanilla clients cannot render custom item badges or append arbitrary tooltips from hidden NBT. They receive configured chat messages and can use permitted commands; the companion is needed for hover indicators.

## Commands

Examples (player filters accept a name or UUID; timestamps are UTC ISO 8601):

```text
/creativeaudit status
/creativeaudit recent
/creativeaudit recent 20 since=2026-09-06T00:00:00Z page=2
/creativeaudit player Steve limit=20
/creativeaudit item minecraft:diamond player=Steve limit=20
/creativeaudit session <session-uuid> page=1
/creativeaudit suspicious limit=20
/creativeaudit inspect <event-or-provenance-uuid>
/creativeaudit export player=Steve since=2026-09-06T00:00:00Z
/creativeaudit reload
/creativeaudit rollback <origin-event-uuid> --preview
/creativeaudit confirm <preview-token> <reason>
```

Query filters: `player`, `item`, `session`, `action`, `since`, `suspicious`, `limit` (1–100), `page` (1–10000). Exports apply matching filters to all retained rows; pagination only affects displayed queries. Inspection shows up to 20 related records; use export for a complete external investigation. Pending writes are not included in queries. Player UUIDs are preferable for renamed players and target searches.

Permission nodes are registered with Forge's permission API:

| Node | Default |
| --- | --- |
| `creativeguard.audit.staff` | Operator level 2 |
| `creativeguard.audit.detail` | Operator level 2 |
| `creativeguard.audit.rollback` | Operator level 3 |
| `creativeguard.bypass` | Nobody (grant explicitly through a permission provider) |

Trusted actors suppress alerts/broadcasts while retaining audit records by default. `retain_trusted_audit=false` disables their records but still marks their items. `OPS_ONLY` accepts ops or the staff node; `PERMISSION_NODE` follows only the node. Forge permission providers can customize all nodes.

Alert rule format:

```text
name|severity|comma-separated item IDs or #tags|minimum count|comma-separated actions or *|optional NBT substring
rare-items|HIGH|minecraft:elytra,minecraft:beacon,#forge:storage_blocks/netherite|1|*
large-grants|MEDIUM|*|64|GIVE_COMMAND,LOOT_COMMAND,CREATIVE_CREATE
```

## Audit storage and recovery

- `<world>/creativeguard/audit/YYYY-MM-DD.jsonl`: immutable event rows after each forced batch write.
- `<world>/creativeguard/audit/exports/`: filtered JSONL exports.
- `<world>/creativeguard/quarantine/`: prepared SNBT recovery files and applied-result markers.
- `*.partial-*`: preserved incomplete crash tails; these are not interpreted as complete audit events.

Writes use a dedicated worker; queues hold at most the configured capacity plus one batch of 128. When full, rejected events are explicitly reported as **unpersisted**. No finite in-memory queue can preserve unlimited events during an outage. A hard crash can lose unflushed events; the item marker may exist before its log is durable. Audit failure never deletes or duplicates gameplay items.

Secondary indexes map event/player/item/action/session/provenance/alert/day keys to daily files and rebuild at startup. They are bounded to 200,000 distinct keys; a saturated index safely falls back to disk scans. Detailed queries and exports run on a separate bounded worker. Retention removes expired daily files at startup (0 disables pruning). Markers survive retention; authorized inspection then reports history unavailable. Schema 1 is additive; unknown marker schemas remain visibly marked and are rejected by quarantine.

Quarantine always requires a fresh preview and explicit token/reason. It checks permissions, durable history, storage health, exact item state, pure lineage, and active locations again before changing items. It does not load chunks, edit offline players, search every world entity, recursively open shulker contents, or follow descendants automatically. A maximum of 1,024 candidate stacks is allowed. A missing log never authorizes removal.

**Recovery is manual.** A `.snbt` file is persisted before any removal, and `.applied` is written afterward. World saves and audit/recovery files cannot commit atomically. A crash or custom handler failure can leave a prepared or partially applied operation. Inspect current inventories, audit results, and both files before restoring anything; blindly replaying a prepared file could duplicate items. Keep the server's audit/quarantine directory restricted to administrators.

## Build and verification

```sh
# JAVA_HOME must point to a Java 17 installation
./gradlew test runGameTestServer build
```

Windows: `gradlew.bat test runGameTestServer build`.

Output: `build/libs/creativeguard-1.0.1.jar`. Forge GameTests run an isolated development server in `run/`. Tests exercise actual dedicated-server hooks and normal Survival behavior; [VALIDATION.md](docs/VALIDATION.md) records the verified scope and remaining manual matrix. Test structures/classes are included for optional Forge GameTest runs and do nothing in normal gameplay.

Development uses the official Forge 1.20.1 MDK, Mojang mappings, Java 17, and Sponge Mixin. Runtime dependencies are supplied by Forge; no bundled database, additional runtime mod, global item comparator overwrite, or external network sink is used. Official references: [Forge installation/development](https://docs.minecraftforge.net/en/1.20.1/gettingstarted/), [logical and physical sides](https://docs.minecraftforge.net/en/1.20.x/concepts/sides/), [Forge events](https://docs.minecraftforge.net/en/1.20.1/concepts/events/).
