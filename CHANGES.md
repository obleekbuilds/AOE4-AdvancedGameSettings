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

## 6. Entity table audit against real game data

**File:** `assets/scar/helpers/ags_blueprints.scar` (`AGS_ENTITY_TABLE`)

Triggered by an in-game fatal SCAR error: a Templar player with the Fortified
settlement setting crashed on `Cannot find "building_defense_palisade_tem"` —
the real blueprint is `building_defense_palisade_wall_tem`. The entity-table
blocks inherited for the newer civilizations (section 5) were built by
copy-pasting an older civ's block and guessing the suffix, so the fallback
hardening from section 3 never fires: the table key *exists*, it just names a
blueprint that doesn't. Every such fetch is a fatal SCAR error.

All ~1,100 blueprint names referenced by the mod were extracted and checked
against the name tables of the game's data archives (`Attrib.sga`, `Xp3Data.sga`,
`Data.sga`, campaign archives, `ReferenceAttributes.sga`). 97 names did not
exist — every one in the blocks for Japanese, Sengoku Daimyo, Byzantine,
Macedonian dynasty, Jin, Lancaster, Templar, Golden Horde, and the campaign
Ayyubid/Crusader civs. Any AGS option touching those entries (Fortified start,
Early Market, No Dock, Starting Keep, statewars starting cities, Regicide king
spawn, ...) crashed for those civs. Fixes, each verified to exist in the
archives:

* **Templar** — real names verified: fortress (their keep), `building_market_tem`,
  `building_barracks_tem`, `building_stable_tem`, `building_archery_range_tem`,
  `building_siege_workshop_tem`, `building_dock_tem`, `building_monastery_tem`,
  `building_tech_blacksmith_tem`, palisade wall.
* **Jin** — uses `building_<name>_<age>_jin` naming (`building_keep_3_jin`,
  `building_barracks_1_jin`, ...). Their Machine Workshop replaces both the
  archery range and siege workshop, so both keys point at
  `building_machine_workshop_2_jin`.
* **Lancaster** — `building_unit_barracks_lan`-style names, `building_tech_blacksmith_lan`.
* **Japanese / Sengoku Daimyo** — dropped the guessed `_control` infix; monks are
  `unit_monk_buddhist_3_jpn` / `unit_monk_ikko_3_jpn_ha_sen`; monastery is the
  Buddhist temple; the blacksmith key maps to the Forge, which replaces the
  Mining Camp + Blacksmith (`building_econ_mining_camp_jpn`).
* **Byzantine / Macedonian dynasty** — monk is `unit_monk_2_byz`, not `_3_`; the
  copy-pasted English `town_center_landmark` (Westminster) entries were dropped.
* **Golden Horde** — keep is the same borrowed `building_defense_keep_control_nov`
  base Mongols use; scout is `unit_scout_1_mon_ha_gol` (this variant has a real
  tier-1 scout — its Khan starts at tier 2); outpost is their Fortified Outpost;
  the nonexistent packed-TC/packed-house entries were dropped (resolver falls
  back to base Mongol).
* **Campaign Ayyubid / Crusader** — names drop `_control` and unit tier numbers
  (`unit_villager_ayy`, `unit_monk_cru`, ...); neither civ has a king unit, so
  the `king` keys were removed and Regicide falls back through the resolver.

Keys a civ genuinely lacks are now *omitted* (not guessed) so the section 3
fallback chain handles them. After the fix, a re-run of the archive sweep finds
zero referenced blueprint names missing from game data.

## 7. Player colour enforcement resurrected (nearest-colour matching)

**File:** `assets/scar/gameplay/ags_color_maintainer.scar`

Confirmed broken in-game (2026-08-23): lobby-picked colours were ignored —
everyone rendered with default slot-order colours (player 1 blue, player 2
red, ...). The module maps each player's picked colour to a palette slot via
a hardcoded `"#AARRGGBB"` → slot table; the game changed its palette hexes
(again), every lookup missed, and the inherited nil-guard skipped every
player. This was the accepted "stale colour table" limitation; it is now
materially fixed rather than tolerated:

* Exact hex match is only a fast path. A miss now falls through to
  **nearest-reference matching on numeric RGB** (from `Player_GetUIColour`'s
  `{r,g,b}`) against nine hue anchors (blue/red/yellow/green/turquoise/
  purple/orange/pink/grey — the palette's stable slot order). A palette
  *tweak* can no longer disable enforcement; even a palette *redesign*
  degrades to the perceptually closest slot instead of doing nothing.
* `AGS_ColorMaintainer_ForceColor` now calls the engine's current
  `Game_SetPlayerColour` first; the two functions the mod used are marked
  DEPRECATED in scardocs (kept as belt-and-braces — the base game still
  calls them too).
* Each player's read colour string and chosen slot are printed to the log,
  so one test match documents the current palette hexes for the fast-path
  table.

Needs in-game verification that slot indices still map 1=blue ... 9=grey
(the deprecation notes and current base-game usage say they do).

## 8. Full-mod audit: 30 verified defects fixed before release

A systematic audit of every reachable scar file, cross-checked against the
current game's own script sources (extracted from the game archives), the
official SCARDOC signatures, and the game-data name tables. Every finding
below was independently verified against those references before being
fixed; two candidate findings that did not survive verification were left
alone. Fatal crashes first:

* **Team Solidarity** (`conditions/ags_team_solidarity.scar`) — every
  solidarity elimination crashed on an undefined global `owner`; now names
  the defeated teammate it failed to protect.
* **Wonder destruction** (`conditions/ags_wonder.scar`) — passed the raw
  entity handle into a `%d` format (fatal), credited the wonder's own owner
  as its destroyer, and used the conquest notification table; all three
  fixed (numeric UWID, base-game killer resolution, wonder table).
* **Villager-count trimming** (`startconditions/ags_start_settled.scar`,
  `ags_start_nomadic.scar`) — lowering starting villagers below the map
  default destroyed squads with a forward index over a live-shrinking
  SGroup, running out of bounds (fatal); now iterates in reverse like the
  base game.
* **Replay stat viewer** (`specials/ags_utilities.scar` +
  `conditions/ags_conquest.scar`) — fetching conquest stats crashed when
  the Conquest module had self-unregistered; degrades to zero counts, both
  at the call site and inside the counter.
* **Objective sentinels** (`coreconditions/ags_conditions_objectives.scar`)
  — the "OBJECTIVE WAS NOT DEFINED" guards warned and then fell through
  into fatal engine calls anyway; they now actually return. The score kill
  handler also gained the base game's scartype guard on `context.killer`.

Features that silently did nothing or misbehaved:

* Score timer lobby choices 100/105 minutes had no handler branch — a
  score match set to 100 ended at the default 60
  (`ags_global_settings.scar`).
* `AGS_RemoveUpgrade`'s guard was inverted (copy-paste of the apply
  helper), so Technology Age's entire per-civ conflict-correction table
  was dead code (`helpers/ags_blueprints.scar`). The Golden Horde also got
  its (missing, now data-verified) correction list, and Lancaster's
  upgrade list — a verbatim English copy — was replaced with its real
  `_lan` unit lines.
* Score timer expiry mid-treaty eliminated players during peace (or
  silently lost the resolution); now retries after the treaty ends
  (`conditions/ags_score.scar`). Treaty also never actually disabled
  sacred-site capture: Religious and Culture now re-apply the treaty state
  once their site tables exist.
* Conquest: killer was resolved from the victim (destroyer always shown as
  the victim's own owner); Capital=OFF still tracked capitals via the
  Landmarks toggle; an empty enemy list produced a 0/0 NaN progress bar.
  Annihilation's capital-vs-landmark classification could leave a
  resource-dead player unbeatable (`conditions/ags_annihilation.scar`).
* Religious/Culture polish: victory countdown could stay paused forever
  (missing forced `Timer_Resume`, world-owned-instigator guard too broad),
  site flash toggled off when asked to flash twice, and several FFA
  notifications showed "Ally ..." for enemy-vs-enemy actions
  (`conditions/ags_religious.scar`, `ags_culture.scar`).
* Regicide re-defeated already-eliminated players during king cleanup;
  surrender's network handler gained the base game's already-eliminated
  guard; tributes are now funds-checked in the simulation before gifting
  (racing clicks could previously drive resources negative); dynamic
  diplomacy no longer fires a false reverse relation cue
  (`ags_regicide.scar`, `ags_surrender.scar`, `diplomacy/*`).
* Variant-civ coverage: Golden Horde now gets the Mongol handling in team
  balance, population capacity, AI stone vision, tournament nomad wood,
  stone-tribute exclusion, and Fortified-start palisades (its own, not
  Rus); Zhu Xi's Legacy gets the Chinese tournament nerf; Sultanate of
  Tughlaq skips the generic tech grant like base Delhi; Order of the
  Dragon gets its prelate in settled/nomadic starts and its TC landmark
  under TC Restrictions (now table-driven); Jeanne d'Arc actually spawns
  for her civ in scripted starts.
* Ending Age rework (`gameplay/ags_ending_age.scar`): the Imperial cap no
  longer strips Abbasid's imperial-tier wings (off-by-one vs Relic's
  logic); the Ayyubid variant's own wing family is now removed (resolved
  lazily and verified against game data); civs with no real
  construction-menu tokens (all variants, Jin, Lancaster, Templar) degrade
  cleanly to the engine-side age cap instead of targeting nonexistent
  menu names — the prefix allowlist matches the game data exactly
  (Ottomans genuinely have no such menus).
* Presentation/notification plumbing: game-over stingers referenced a
  nonexistent `stinger_eliminated` movie and captured their HUD-restore
  callback before it was defined (HUD never came back); the countdown
  music duration key was misspelled; `Obj_CreatePopup`'s third argument is
  now `""` (a data-template override per SCARDOC, not a state); the
  objective "OR" template selection was inverted
  (`coreconditions/*`).
* Localization packaging: the Traditional Chinese CSV was named with dots
  (`Advanced.Game.Settings_zh-tw.csv`) so the localization loader could
  never find it — renamed to the loader's convention; Russian was never
  registered in the locdb (its CSV was dead weight) — registered, its two
  malformed rows repaired, and the 14 missing newest rows translated.

## 9. Nomad/Scattered vs AI: AI players are now settled by script

**File:** `assets/scar/ai/ags_ai.scar` (`AGS_AI_ApplyAdjustment_Nomad`)

Confirmed in-game (and present in upstream unchanged): in Nomadic and
Scattered starts the mod disables construction for AI players ("AI might
not be able to manage certain nomad conditions") — but nothing ever gave
them a town center or re-enabled building. AI opponents spent the entire
match bricked with a handful of villagers and never developed.

The construction lockout stays (the AI genuinely cannot perform the
settle-and-place-TC dance), but 60 seconds in — the same window in which
players receive their trickled starting resources — each AI player now
receives its civilization's capital town center at its start position
(force-constructed, same spawn path the Settled start uses, which is known
to work with AI), construction is re-enabled, and the AI's planner is
nudged to adopt the new base. The AI effectively plays a Settled start on
a 60-second delay, which is the accepted trade-off: a functioning opponent
with a small settling head start instead of a vegetable. If a town center
already exists by then (mode script or takeover), the grant is skipped.

## 10. Hotfix: treaty + sacred-site crash (regression from section 8)

**Files:** `conditions/ags_religious.scar`, `conditions/ags_culture.scar`

Section 8's "treaty never disables sacred-site capture" fix re-applied the
treaty state after the site tables are populated — which made upstream's
`Entity_EnableStrategicPoint(site, false)` code path execute for the first
time ever (it had always run on an empty table). The engine fatals when
`Player_GetStrategicPointCaptureProgress` is called on a disabled point, so
any match with Treaty enabled and the Religious (or City-States Culture)
condition active crashed on the first objective tick (~1 second in).

Fix, verified in a live match (3-minute treaty, Religious on, crash-free
through the treaty and its end transition): the periodic objective tick and
the strategic-point-change handler in both files now early-return while
capture is treaty-disabled (victory must not progress during peace anyway),
and the OnPlay re-apply only touches the engine when a treaty is actually
holding — a no-treaty match makes no new engine calls at all.

Note for maintainers: `EssenceEditor.exe -rtm --build_mod <path.aoe4mod>`
performs a full headless build in seconds — no GUI, no renderer needed.

## Known limitations

* All civ tables are still hardcoded snapshots of game data; future patches
  degrade them gracefully now, but full support for a new civilization still
  requires adding table entries (see the "Adding New Civilization Variants"
  notes in Sachtleben's fork history). The colour-to-slot table is no longer
  on this list — nearest-colour matching (section 7) keeps colour enforcement
  working across palette changes.
* The `AGS_CIV_PREFIXES` trailing-underscore fix for `chinese_ha_01`, `french_ha_01`
  and `hre_ha_01` follows the pattern of every other variant entry but the real
  in-game construction menu names could not be verified outside the game — test the
  Ending Age setting with Zhu Xi's Legacy / Jeanne d'Arc / Order of the Dragon.
* `assets/scar/winconditions/ags_tournament_nomad.rdo` contains a leftover
  placeholder options section (`m_key="Error"` with zero-GUID localization refs) —
  harmless, inherited from upstream, worth cleaning up someday.
* The Observer/replay UI (`assets/scar/replay/`) only knows the original 8 civ
  flag images, but it is disabled in `ags_cardinal.scar` and unreachable.
