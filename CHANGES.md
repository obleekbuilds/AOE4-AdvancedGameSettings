# Changes in this fork

This file documents every modification this fork makes, with the reasoning behind it.
Base: [joshuasachtleben/AOE4-AdvancedGameSettings](https://github.com/joshuasachtleben/AOE4-AdvancedGameSettings)
`main` at commit `b604d46` ("Added new Jin Dynasty DLC civilization", 2026-05-08), which is
itself a fork of the original [Woprok/AOE4-AdvancedGameSettings](https://github.com/Woprok/AOE4-AdvancedGameSettings).

## 1. Fix fatal SCAR error at treaty end / victory (`Obj_CreatePopup`)

**File:** `assets/scar/coreconditions/ags_conditions_objectives.scar` (`AGS_Objectives_Present`)

A game patch changed the engine's `Obj_CreatePopup` SCAR API from 2 to 3 arguments
(id, title, state). The mod still passed 2, so any objective popup — treaty timer
expiring, victory being declared, conquest objectives — raised
`invalid number of arguments. expected: 3 received: 2` and paused the match
([upstream issue #130](https://github.com/Woprok/AOE4-AdvancedGameSettings/issues/130)).

The call now passes `tostring(state)` as the third argument. Fix authored by
grandoth in [upstream PR #132](https://github.com/Woprok/AOE4-AdvancedGameSettings/pull/132),
confirmed working in-game by its author; cherry-picked here.

## 2. Fortified start: walls for every civilization

**File:** `assets/scar/startconditions/ags_start_fortified.scar` (`AGS_Fortified_CreateWalls`)

Upstream skipped Mongols entirely (they have no palisade blueprints), so Mongol players
got no walls in Fortified starts. Adapted from
[upstream PR #127](https://github.com/Woprok/AOE4-AdvancedGameSettings/pull/127) by Levelleor,
with a refinement: instead of giving *everyone* Rus palisades, each civilization keeps its
own palisade style, and only civs lacking palisade blueprints (Mongols, or any civilization
missing from the entity table) fall back to Rus palisades. The lookup is also nil-guarded,
so an unknown civilization can no longer crash wall creation.

## 3. Defensive hardening of per-civilization tables

The mod resolves civ-specific content through hardcoded tables keyed by civilization name.
Any civilization released after the tables were last updated used to crash the match with
a nil table index (this is why DLC civs black-screened on the abandoned original —
[upstream issue #115](https://github.com/Woprok/AOE4-AdvancedGameSettings/issues/115)).
These changes make unknown civilizations degrade a feature instead of killing the match:

* **`assets/scar/helpers/ags_blueprints.scar`** — `AGS_GetCivilizationEntity` and
  `AGS_GetCivilizationUnit` now resolve through a new helper,
  `AGS_ResolveCivilizationBlueprintName`, which falls back to the English entry
  (logging a warning via `AGS_Print`) when a civilization or blueprint key is missing.
  Wrong-looking blueprints beat a dead match.
* **`assets/scar/gameplay/ags_technology_age.scar`** — `AGS_TechnologyAge_DetermineUpgrades`
  defaults missing civ entries in `AGS_UPGRADE_TABLE` / `AGS_UPGRADE_CORRECTION_TABLE`
  to empty tables, so an unknown civilization simply receives only the common upgrades.
* **`assets/scar/gameplay/ags_ending_age.scar`** — `AGS_EndingAge_SetMaximumAge` skips
  the construction-menu restrictions for a civilization missing from `AGS_CIV_PREFIXES`
  (nothing sensible to restrict) while still applying the engine-side
  `player_max_age` cap, instead of erroring on a nil string concatenation.

## 4. Pre-existing crash bugs found by audit

A full sweep of the codebase for the same class of problems (unguarded lookups,
nil dereferences reachable in real matches) found and fixed these:

* **`assets/scar/startconditions/ags_start_settled.scar`** — `AGS_Settled_CreateSpawn`
  passed an undefined `player_civ` (a nil global) to every spawn helper; the loop
  variable was `player`. Reachable via the Nomad game mode with Settled/Fortified
  settlement: Mongol players hard-crashed on a nil blueprint, other civs would
  spawn wrong units. Now defines `local player_civ = player.raceName`.
* **`assets/scar/gameplay/ags_wonder_scale_cost.scar`** — the per-civ wonder resource
  split table only covered the 10 launch-era civs; enabling the Wonder Scale Cost
  option with any newer civ crashed on a nil index during the loading screen.
  Unknown civs now default to the standard four-resource split, and `mongol_ha_gol`
  got an explicit entry with the Mongol no-stone split.
* **`assets/scar/conditions/ags_culture.scar`** (City-States) — two guards, both
  mirroring what `ags_religious.scar` already had: the nil-squad guard was dead code
  (`not context.changeType == 7` parses as `(not changeType) == 7`, always false),
  and world-owned instigator squads (e.g. after a surrender) crashed the handlers
  when resolving their player owner. Both paths now exit cleanly.
* **`assets/scar/conditions/ags_regicide.scar`** — a world-owned attacker (wildlife
  biting the king) resolves to no player table entry; the damage notification then
  crashed on `attacker.id`. Now skipped, matching the guards in the conquest and
  wonder conditions.
* **`assets/scar/startconditions/statewars/ags_starting_cities.scar`** — maps larger
  than the biggest placement strategy (1024×1024) matched no strategy and crashed on
  a nil index; now falls back to the largest defined strategy.
* **`assets/scar/helpers/ags_blueprints.scar`** — `AGS_CIV_PREFIXES` entries for
  `chinese_ha_01`, `french_ha_01`, `hre_ha_01` were missing the trailing underscore
  every other variant entry has, so Ending Age menu restrictions built wrong menu
  names (`chi_ha_01age3` instead of `chi_ha_01_age3`) for Zhu Xi's Legacy,
  Jeanne d'Arc and Order of the Dragon. Fixed to match the `abb_ha_01_` pattern —
  **needs in-game verification** (see Known limitations). The blueprint fallback was
  also extended to chain english → mongol → abbasid so moving-building and
  house-of-wisdom keys resolve for civs missing from the entity table.

## 5. Inherited fixes (from Joshua Sachtleben's fork, not authored here)

Listed for completeness, since the original mod lacks them:

* All DLC civilizations added after the original was abandoned, through the
  Jin Dynasty (May 2026): entity, upgrade, and prefix table entries.
* Match-start fatal crash guard in `assets/scar/gameplay/ags_color_maintainer.scar`:
  the mod maps each player's UI color (hex string) to a slot through a hardcoded table
  that goes stale when the game changes its palette; unmatched colors used to pass nil
  into `Game_SetPlayerSlotColour` and fatally crash at match start. His guard skips
  color enforcement for unmatched colors. (Known limitation: if the game changes its
  color palette again, color enforcement silently does nothing for those players —
  but the match keeps running.)

## Known limitations

* The color-to-slot table (`ags_color_maintainer.scar`) and all civ tables are still
  hardcoded snapshots of game data; future patches degrade them gracefully now, but
  full support for a new civilization still requires adding table entries (see the
  "Adding New Civilization Variants" notes in Sachtleben's fork history).
* The `AGS_CIV_PREFIXES` trailing-underscore fix for `chinese_ha_01`, `french_ha_01`
  and `hre_ha_01` follows the pattern of every other variant entry but the real
  in-game construction menu names could not be verified outside the game — test the
  Ending Age setting with Zhu Xi's Legacy / Jeanne d'Arc / Order of the Dragon.
* `assets/scar/winconditions/ags_tournament_nomad.rdo` contains a leftover
  placeholder options section (`m_key="Error"` with zero-GUID localization refs) —
  harmless, inherited from upstream, worth cleaning up someday.
* The Observer/replay UI (`assets/scar/replay/`) only knows the original 8 civ
  flag images, but it is disabled in `ags_cardinal.scar` and unreachable.
