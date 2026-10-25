# darktide-mods-boss-health-fix
Boss Health Fix is a small defensive workaround for a crash in the vanilla boss health bar. It has been reported on non-host clients when several bosses die or despawn in quick succession, for example in late Mortis Trials waves.

Up to Darktide 1.13.2 the crash reads:

```text
.../ui/hud/elements/boss_health/hud_element_boss_health.lua:258:
Cannot access property "current_health_percent" on destroyed object of type HuskHealthExtension
```

Hotfix 1.13.3 only partly fixed it. Since then the same situation crashes with:

```text
.../ui/hud/elements/boss_health/hud_element_boss_health.lua:279:
Cannot access property "boss_is_depleted_interrupter" on destroyed object of type BossExtension
```

The mod prevents both. It does not replace the boss health bar. Before each vanilla update it removes boss entries whose health data no longer exists, and then lets the vanilla update run unchanged. Living bosses, dying bosses during their death animation, the two-bar layout and the vanilla boss queue behave exactly as before. There are no settings.

## What goes wrong

The boss health bar keeps references to each displayed boss's extensions and only checks that the boss *unit* still exists before reading them. When a boss dies, the game removes its extensions but keeps the unit around as a ragdoll. If the boss is still listed at that point, the next update reads a destroyed extension and the game crashes.

Up to 1.13.2 that was the cached health extension. Since 1.13.3 the bar looks up the health and toughness extensions fresh every frame, but it still reads the cached boss extension, so the same stale entry now crashes on the destroyed `BossExtension` instead.

Since Darktide 1.13 the bar can hold more bosses than it shows (two visible bars plus a queue). Each time a boss's encounter ends, the game takes the next boss from the queue; if that boss is already dead, it is dropped and a bar slot stays free while other bosses are still queued. If one of those queued bosses then dies without a death animation on a client, its own `boss_encounter_end` event can promote it into the free slot while its extensions are being removed. This queue handling is unchanged in 1.13.4. The host clears its health state earlier, which is why Solo Play does not seem to be affected. This is the most likely cause, worked out from the game's source code and reproduced against it in a test harness; a crash without mods has not been confirmed in game yet.

## What the mod does

- Hooks `HudElementBossHealth.update` and runs before the original update and before any other mod's `hook_safe` on it.
- Treats a boss entry as stale only when the boss is no longer in `HEALTH_ALIVE` **and** its `health_system` extension is gone. A boss that has just been killed and is still playing its death animation keeps its bar, as in vanilla.
- The game always removes a boss's boss extension together with its health extension, so the same check also catches entries whose `BossExtension` was destroyed.
- Removes stale entries from the visible bars and from the boss queue. When the last visible bar is removed, the boss HUD is hidden, the same way vanilla hides it.
- Never promotes queued bosses itself; that stays with the game's own `boss_encounter_end` handling.
- Never wraps the vanilla update in `pcall`, so unrelated errors are still reported.

Every removal is logged as a warning (`Removing stale active boss HUD target ...` / `Removing stale queued boss HUD target ...`), so you can check in the console log whether the workaround was needed.

## Compatibility

No special integration is needed. The mod only removes invalid entries before the boss health bar updates. Healthbars, NumericUI and Recolor Boss Health Bars extend the bar through `hook_safe`, which DMF always runs after the original update and after every regular hook, so they see the cleaned-up state. The position of Boss Health Fix in `mod_load_order.txt` does not matter for them.

Load order could only matter for a mod that wraps the boss health bar's `update` with a regular `mod:hook` and reads a boss's health before calling the original. No such mod is known.

## Testing

Solo Play is not a reliable test. Join someone else's game as a client and play Mortis Trials up to the late waves with several monstrosities alive at once. Disabling the mod in the mod options turns the hook off, which gives you a vanilla control run.

## Removal

This mod is meant to be temporary. The bug has been reported to Fatshark: [Client crash in Mortis Trials when boss HUD accesses destroyed HuskHealthExtension](https://forums.fatsharkgames.com/t/client-crash-in-mortis-trials-when-boss-hud-accesses-destroyed-huskhealthextension/126442). Hotfix 1.13.3 only fixed the health extension part, so keep the mod enabled until the boss queue handling is fixed as well. After that, disable or uninstall it; it stores no settings or data.
