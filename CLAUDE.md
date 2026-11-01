# Weevee map: working notes

Read this before changing the map. Keep it short and correct it when it turns out wrong.

## The situation
The map grew by adding rule on top of rule, with later passes correcting earlier ones, and each climate adds more. A full rewrite is not planned. It gets cleaner slowly: every change follows the principles below and leaves the code it touches a little better.

## Start here (state on 2026-11-01)
Commits `0b584bc` to the latest are local on `main`, **not pushed**, and the version is still 11.0.16. The user tests next, with the Weevee mod on (the first tests ran with Lekmod Scion on, which isn't this map's usual mod).

**The test copy:** the game plays `Assets\Maps\Weevee v11 working copy`, a plain copy of the tracked `.lua` files that does not update by itself. The working folder is on the Desktop. After every edit, copy the changed files over and tell the user, or move the working folder into `Assets\Maps` and delete the copy (never have both copies' helpers disagree: `include()` picks one by bare name).

**Done, Standard only (Standard-Diagonal and other climates unchanged):**
- `WV11*` helper names, so 10.5 and 11 can be installed side by side.
- Front mountains: one column per side (10.5's column, `iW/2 - 4`), `iH * 25%` mountains. Before: five columns, 15 mountains, leftover tectonics mountains in two of the columns, and up to 4 peaks in the snow behind them.
- No forest in the center snow barrier; snow elsewhere keeps forests.
- Rivers ending on land walk on to water (`RepairDeadEndRivers`, ported from Lekmap); stuck: a one-tile lake, or near the front or a start the river is removed.
- Dead on load fixed in the start finder: a best fallback scoring 0 left a start with no coordinates. An 18-tile region triggered it.

**What the logs show so far:** the last load with Lekmod Scion on generated the whole map without a Lua error (`StartPlotSystem done`; `Dead-end rivers: 1 extended 1 steps 10`; `Standard center snow forests removed: 0`). Whether the game then opened fine is unknown: the user reports getting into games is not smooth, and suspects the mod. Not yet seen or judged by the user: the front, the rivers, the snow.

**Next:**
1. Ask how the test went with the Weevee mod. Test steps are on the Notion page "Tests" under "Weevee map 11". Any dead on load: read `Lua.log` before another game starts.
2. Open: why a start region can be as small as 18 tiles. A weak start is now possible there; if those appear, reroll the map instead.
3. Push and bump the version only when the user says so.

**Notion:** this project shares the Lekmod workspace ("New Specs", "Tests", tracker row "Weevee 11 map, Standard: ..."). Specs for this map arrive on "New Specs".

## Principles
1. **Fix the cause, don't add a correction.** Before adding a pass that strips, ensures or caps something, find what produced the wrong result and change that. If a correction is really needed (for example the stock game passes can't be changed), its comment says what it corrects.
2. **Replacing means deleting.** When a new version of something works, remove the old one in the same commit. No `_UNUSED` copies, no switches kept "just in case": git has the history.
3. **Climates differ through data where possible.** Prefer a value in `GetBarrierConfig` over another `kind ==` check spread through shared code. Climate-specific passes return early at their top, in one obvious check.
4. **Name the scope of every change.** Say which climates it affects. `IsStandardClimate()` includes Standard-Diagonal; `IsPlainStandardClimate()` doesn't. Don't change other climates by accident.
5. **West side first, mirror last.** Generate on the west half only. `MirrorPlotsAfterResourcePlacement` is the only source of the east side and the last step that edits plots.
6. **Comments explain why the code is the way it is now.** The story of earlier attempts belongs in the commit message. When you touch a comment that retells history, shorten it.
7. **Small commits, focused on why.** One topic per commit. Refactors don't change behaviour and are committed apart from changes that do.
8. **Say what was actually checked.** Syntax check, simulated outside the game, or seen in the game are different things; report which. Map changes count as done only once seen in a generated map.
9. **Clean as you go, only nearby.** While working on a function, remove dead code and duplicates next to it. Larger cleanups get their own commit, and the backlog below gets updated.

## Words we use
- **Dead on load:** the game dies while loading a new game. Usually no player got a valid start. Look in `Documents\My Games\Sid Meier's Civilization 5\Logs\Lua.log` for `Runtime Error` lines from the map's files (before starting another game, it gets overwritten); `weevee_dbg.log` next to it shows the last step reached. A Lua error in generation stops the map outright: `GenerateMap`'s retry loop doesn't catch it.

## Facts
- `Weevee - vX.Y.Z.lua` is the map script; the `WV11*.lua` files are helpers. `include()` finds files by bare name anywhere under `Assets`, so the helpers must keep the `WV11` prefix. Version bumps follow `VERSION-BUMP.txt`.
- The working folder is `Desktop\civ mods\Weevee-v11.08-Map-main` (2026-11-01). `VERSION-BUMP.txt` still says it lives in `Assets\Maps`. The game only sees a copy or link placed there.
- Civ V runs Lua 5.1: at most 200 local variables per scope (the map script has 129 at top level). Check syntax with `lupa.lua51`, not a newer Lua.
- According to an earlier comment in the script, calling a missing global during generation aborts it silently, showing up as "no valid start" rather than a `Lua.log` error.
- Files are CRLF. Keep the endings when editing with a script.
- Generation order lives in `StartPlotSystem`; pass order matters, so read it before adding a step.

## Cleanup backlog (measured 2026-11-01)
Take these when working nearby. Check each before deleting; a name could still be used in a way a text search misses.
- **Functions nothing refers to (18):** `OasisInJunglePocket`, `OasisJungleExists`, `TongueNoGoWidth`, `TongueMtnDist`, `TongueGetJungleDepth`, `CopyWestToEast`, `WastelandForceCoastalLuxuryRoles`, `ScrubWaterNearSnow_UNUSED`, `GetFrontMountainColumnsWest`, `GetFrontMountainOffsetsWest`, `GetWastelandSaltCoastPlotIndices`, `RescueUnplacedLuxuryTypes`, `ForceWastelandCoastalLuxuries`, `AddWetlandRiverDesert`, `TongueEconKind`, `EconBandBleed`, `PaintEconBands`, `PlaceMurkTundraLakeFish`.
- **Helpers no map includes:** both map scripts include `WV11MapGeneratorW8`, so `WV11MapGeneratorW.lua` and `WV11AssignStartingPlotsW.lua` look unused. The two generator files differ only in their include line.
- **`StartPlotSystem` has 81 steps**, 17 of them `Strip*` and 9 `Ensure*`, and some run twice (`EnsureLuxuryQuota`, `StripIllegalWaterResources`, `ClampPlayerStartsOffEdges`, `NudgePlayerStartsMinDist`). These are the main place where corrections pile up on corrections.
- **Climate checks:** about 180 `kind ==` / `kind ~=` checks, plus many `IsOasisClimate`, `IsBramble` and `IsTiltedMirrorAxis` calls.
- **`PlaceFrontMountainField`** takes five column arguments; Standard passes the same column five times. A list of columns would be cleaner.
- **`AssignStartingPlots:FindStart`'s nil-start check** never runs here: the map uses rectangular regions (`self.method = 3`), and `ChooseLocations` then calls `FindStartWithoutRegardToAreaID`.
- **`DISABLE_FRONT_MOUNTAIN_RIDGES`** is a leftover testing switch.
- **`Weevee - v11.0.15.lua`** is an old untracked copy in the folder that still includes the old `DEF*` helper names.
