# Weevee map: working notes

Read this before changing the map. Keep it short and correct it when it turns out wrong.

## The situation
The map grew by adding rule on top of rule, with later passes correcting earlier ones, and each climate adds more. A full rewrite is not planned. It gets cleaner slowly: every change follows the principles below and leaves the code it touches a little better.

## Start here (state on 2026-11-02)
11.1.0 was released on 2026-11-02 (bumped and pushed; repo `Weevee-v11.1.0-Map`), with everything below. Work after it starts a new "Next version" changelog in `README.md` when players would notice it; the user edits the changelog, then asks for the bump and push (`VERSION-BUMP.txt`).

**Other player counts (started 2026-11-02):** 3v3 on Small is the tuned case and needs accuracy; 1v1 (Duel) and 4v4 (Standard) only must not clearly break or violate a rule (user). Canvas: 10.5's size table (`BaseGridSize`), with 11's Small-tuned changes scaled by size relative to 44x26 (`ScaleCanvasCols/Rows`): explo trim, Oasis band (`OasisWestDesertColumns`), Diagonal height (multiple of 4, at least 16) and width. Small unchanged; Frosty (WIP) not scaled. Start-region top/bottom margins shrink to leave at least 8 rows (Duel, Diagonal below Small). Luxury tiers (`ResolveLuxTargets`): one per civ per side, per the user: unique 15-24, a 7-13 tier per middle civ, top tier 3 (4v4: 15-24/7-13/7-13/3); one civ per side uses `LUX_DUEL_TIERS` (8-11/3-4/1, read off Duel maps; 10.5 has no numbers). Rolled once per game: rolled per attempt, kept Small Diagonal maps averaged 19.0 unique against 19.9 rolled (72 games), because retries replaced missed high targets. The quota counts, pads and trims the side the mirror keeps (`LuxuryCountedPlot`). Sea resources per side 9-15, Duel 5-8 (`SeaResourceBand`). Last batch (before the once-per-game change): plain 4v4 Standard 3/3 accepted on attempt 1, every tier on target; plain Duel accepted on attempts 2 and 1; Small Diagonal 9 games, none exhausted, unique tier never failed after the mirror fix, misses only in the 2+ tier at targets 11-13, the old front gap check rejected about 20 of 32 attempts. Known: the 3v3 3+ tier always ends at 4 (target 3) because the trim can't remove the 3 regionals or marble; the user is fine with it. Still fixed numbers: plain Duel's region is 1 column wide, min start distance 7, start edge margin 4, mountain budgets. A game without two teams crashes in the stock random-division path (`WV11AssignStartingPlotsW8.lua:4129`): this map is for two teams only.

**Where it runs:** the working folder is now `Assets\Maps\Weevee-v11.08-Map-main`, and the game plays these files directly (`Evaluating Map Path` in `net_message_debug.log`). No copy to keep in sync. The Weevee mod is `Assets\DLC\snowve`, with its DLL source in `CvGameCoreDLL_Expansion2`.

**Done (Standard only unless a line says otherwise):**
- `WV11*` helper names, so 10.5 and 11 can be installed side by side.
- Front mountains, Standard and Standard-Diagonal: one column per side (10.5's column, `iW/2 - 4`, following the fold on Diagonal), `iH * 20, 25 or 30%` mountains per map, tectonics left in the columns behind it as in 10.5 (`frontMountains = "line"` in `GetBarrierConfig`). 10% per map each: 1-2 mountains on the tundra column, and a 3-4 mountain chunk (`PlaceFrontFringeMountains`). `Mountain roll:` in `Lua.log` and `weevee_persist.log` lists each map's random decisions and where its mountains ended up. The user is happy with Oasis and Peaky as they are (`"field"`: five columns, 15 mountains); Murky is unjudged and also still on `"field"`.
- Tundra column right before the snow on each side (`iW/2 - 3`, as in 10.5; follows the fold on Diagonal). Also Standard-Diagonal. The river barrier includes it, so rivers stop one column earlier.
- Standard-Diagonal: each side has at least 2 ways in of 4+ tiles through the snow after the mirror; short of that, corridors are carved (mountains to hills in mirror pairs) instead of rerolling (`frontOpenings`, `EnsureFrontOpenings`; `Front openings after mirror:` in `Lua.log` and `weevee_persist.log`). The user rated every in-game roll solid and wants to keep it. Design, algorithm, numbers and how to apply it to other climates: `docs/ways-in.md` (read it before doing that).
- Standard-Diagonal's front and snow: see "Standard-Diagonal tuning" below.
- Center snow relief, Standard and Standard-Diagonal (`PaintStandardSnowRelief`): 1-5 mountain groups of 1-3 tiles, uneven hill fingers off them instead of a ring, sometimes lone hills. At most 8 mountains per side (5 on Diagonal), at least half flat before lone hills.
- No forest in the center snow barrier, also on Standard-Diagonal; snow elsewhere keeps forests.
- Rivers ending on land walk on to water (`RepairDeadEndRivers`, ported from Lekmap); stuck: a one-tile lake, or near the front or a start the river is removed. All non-WIP climates (`repairDeadEndRivers` in `GetBarrierConfig`); Oasis gets no lakes, and its walks can cross the west desert band where `OasisStripWestHinterlandRivers` thinned rivers out. `River check after mirror:` in `Lua.log` counts dead ends and loop edges over the whole map; loop edges must be 0.
- On Standard-Diagonal the mirror copies rivers from the west of the fold (`MirrorSourcePlot`), not from `x <= mid`.
- All climates: at most 4 luxuries in a capital, 3 for 15% of them (`CapCapitalLuxuries`; check the `Capital luxury cap:` line in `Lua.log`).
- All climates: capitals are filled to their limit by moving copies in (15% get a limit of 3, the rest 4), and 75% of ring 3 luxuries move to rings 1-2 (`ShapeCapitalLuxuries`; `Capital luxuries:` in `Lua.log` and `weevee_persist.log`, `total(within 2)` per west capital). Target: 10-20% of capitals below 4.
- Standard, Standard-Diagonal, Murky, Peaky: at most 3 flat desert tiles in rings 1-2 of a capital, not counting oasis and flood plains (`capitalDesertMax`). Start selection avoids it, `CheckCapitalDesert` rerolls the map, and the last attempt paints the excess to plains.
- Rivers along the front, Standard and Standard-Diagonal (`riverFrontZone = 6`, `riversFlowWest` in `GetBarrierConfig`): rivers start flowing west as in 10.5, since one starting east can't turn back at the barrier; a wider front zone; no keep-direction discount inside it. `RIVER EDGES` in `weevee_persist.log` measures it with a fixed 4-column width: before this, 45% of 55 Standard rolls had `frontNS` 12 or more, median 10, last 20 rolls averaging 12.7. Rolled since, but `RIVER EDGES` hasn't been compared yet: do that next. 10.5 also has no water bias, so `riverWaterDist` is the next suspect.
- At least 6 land tiles between water and the center snow, tundra column included (`WATER_MIN_FRONT_DIST`, was 4). All straight-barrier climates; Standard-Diagonal already has 7 by its own rule.
- All climates: forced luxury placement follows stock's tile lists as well as the game's rule (`LuxuryLegalAt`, `StockOrdered`), first-choice tile type first, as 10.5 does. Stock's `PlaceSpecificNumberOfResources` never checks `CanHaveResource`, which is how jade lands on hills although the Resources table says flat only.
- Standard-Diagonal: the sea resource limits (9-15 per side) now count the side the mirror keeps. They counted `x <= mid`, so water west of the fold past mid was missed and water on the discarded side counted. The luxury quota (count, padding, trim) had the same flaw; it now uses `LuxuryCountedPlot`.
- Standard-Diagonal's water is Standard's (`ShapeNoWrapBackstrip`, `PlaceStandardEdgeSeas`, which handles the fold), plus a big island (5-15 tiles) in a back-coast bay on 25% of maps (`backBayIslandChance`, `PlaceBackBayIsland`; `Back bay island:` in the logs); such a map grows one edge bay fewer. User, 2026-11-02: inland seas had to go, the amount should match Standard, an occasional big island is fine. Simulated with the real functions: 7.4% water against Standard's 7.2% in the same simulation (which runs about 0.8 below the logs). `PlaceDiagonalBackWater` is now Bramble's only. First 7 games: 8.0% salt water like Standard, but natural sea resources 9.7 per side against Standard's 11.5, since the 46x20 map holds about 37 salt tiles per side against 42; maps with two edge bays got 13, 13, 9, 7, with one 9, 9, 8. So 0-2 small lakes per side of 2-4 tiles were added for fish (`lakes` in `GetBarrierConfig`, the interior-lake block in `GeneratePlotsByRegion`); lake fish count toward 9-15. Then the volume was matched instead of the share: the back coast covers about as many rows as Standard's (`backCoastCut = {35, 10}`: 13 or 18 of 20 rows, Standard 13 or 19 of 26). Simulated: 38.2 salt tiles per side against Standard's 37.7, same spread. Raising the two-bay odds added at most 1 tile per side (a two-bay map cuts more coast); bigger bays would change the look. Not yet rolled.
- Standard: 1-3 non-regional luxuries per side in the two columns before the snow (`frontLuxuries` in `GetBarrierConfig`, `MoveLuxuriesToFront`; check `Front luxuries:` in `Lua.log`). Regional luxuries never go there. Not on Standard-Diagonal yet: the pass uses fixed columns. It moved nothing on the first 5 logged rolls because those columns already held 3-6 luxuries; the user finds that fine and is watching it.
- Oasis: rescue edits for a regional luxury stay out of deep desert (`tryTerraformOnPlot`). `Oasis lone luxury tiles:` in `Lua.log` counts what's left; anything above 0 has another source.
- Peaky: 1-3 small back-ocean islands (1-4 tiles) next to the chunky ones.
- All non-Bramble climates: starts moved by the spacing and edge fixes (`FindNearestStartOffEdge`) keep the back distance. One had gone from x=6 to x=2. They can still move toward the front past the region's edge (11 to 13 seen).
- Dead on load fixed in the start finder: a best fallback scoring 0 left a start with no coordinates. An 18-tile region triggered it.

**Standard-Diagonal tuning (where it stands).** The user's last batch was 6 of 8 good or solid. "Difficult" always means too hard to push through when the two teams fight.
- Contact mountains: 1-3 groups with hill fingers where the tundra column meets the snow (`contactMountains`, `PlaceContactMountains`). Group sizes 50/35/15 for 1, 2, 3 tiles (`groupSizeOdds`), for the snow's groups too.
- Limits per side: snow and contact mountains 10 (`barrierMountainMax`), snow alone 5 (`snowMountainMax`), at least 2 snow groups (`snowGroupsMin`).
- Ripple: mountains on the front line and further back within 4-5 tiles of a shape of 2+ tiles become hills (`shapeClearRadius`, `ClearFrontLineAroundShapes`). Single-tile groups clear nothing, so plain solid maps still happen, which the user wants now and then. Extreme case: a roll cleared the whole line and had 11 mountains on the side; the user called it funky but fine to try.
- Wall limit: contact mountains leave the tundra column once line + tundra reach 7 (`frontWallMax`).
- Ways in: see the `frontOpenings` line under Done. A map whose widest straight run of mountain-free rows was 3, caused by a center mountain, was rated too difficult; that rating led to the first rule.

What the rating batches taught:
- Rate whole batches, good and bad, and match them to `Mountain roll:` lines in order. That beat counting and guessing; small batches gave strong signals.
- The total mountain count doesn't predict difficulty (a total of 29 was good, 23 difficult). Line + tundra did at first; after the ripple thinned the line, a lot of snow mountains (5-6) looked like the better signal, on few rolls.
- The look the user likes is mountains well inside the snow with room for hills. Limits that cut volume hurt it: 7 total looked too open, and 8 with connected 2-4 tile structures looked less exciting.
- Don't limit the snow relief to `MirrorSourcePlot`: on Diagonal that leaves about 20 of its 53 tiles, a thin strip behind the tundra, which was rated dull and too difficult. So some snow mountains still land on tiles the mirror overwrites, and bare snow (rated ugly) stays possible. A proper fix has to simulate the mirror first.
- Murky keeps the five-column front and hasn't been judged. Standard hasn't had these Diagonal tools tried on it.

**Endless load (2026-11-01): the load finishes since the river loop fix.** Before it, the load never finished. The last load (Weevee mod on) finished generation without a Lua error: `CvMapGenerator - GenerateRandomMap(), 0.43` in `stopwatch.log`, then `LoadGameState::GameViewState` in `system_init.log`, then no more logging. So it stalls after the map script, in the engine.
Main suspect: `RepairDeadEndRivers` could join a river back onto its own upstream and close it into a loop with no mouth (109 of 400 simulated maps). Fixed in the walk (`flowReaches`), 0 of 2000 after. Lekmap's `LekRepairDeadEndRivers` has the same flaw. Seen in the game: three logged generations with `loop edges 0`, and the user reports it loads. Not proven: that the engine hangs on a loop.

**Keep an eye on: lake tiles with a luxury and river edges (fixed 2026-11-02, not yet confirmed).** `tryTerraformOnPlot` painted any terrain from `Resource_TerrainBooleans`, including `TERRAIN_HILL`, which the Terrains table marks as water; `CvPlot::setTerrainType` then turned the tile into a lake, and `canHaveResource` accepted the luxury there because the terrain is listed and a water plot is neither hills nor flat. It's the last-resort tier of the regional pass, so it was rare. The list now leaves out water terrains.
When reading logs for anything else, also check `weevee_persist.log` for `Lake with river edge`, or `Illegal water resource removed` with terrain 8, in rolls after this fix (the log's last hit is the one that found the bug; 84 rolls logged since, not all of them on the fix). If it shows up again, the fix missed something: look into it. After about 30 clean rolls, delete this paragraph and `LogLakesWithRiverEdges` without asking.

**Next:**
1. The user edits `README.md`; then commit, bump and push when they say so (`VERSION-BUMP.txt`).
2. Look at Standard-Diagonal's new water in the game; compare `WATER COUNT kind=snow tilted=true` (salt %) and `SEA RES` against Standard (8.0% and 11.5 natural sea resources before the change). Before: 10.2% salt, 5.9-16.1%.
3. Compare `RIVER EDGES` before and after the river changes (see the river line above), and judge the rivers on Diagonal, Murky, Oasis and Peaky in the game.
4. Compute the share of capitals below 4 luxuries and within radius 2 from `Capital luxuries:`, against the target of 10-20% below 4.
5. See whether the jade and amber change looks right, and whether `Oasis lone luxury tiles:` stays at 0.
6. `MoveLuxuriesToFront` rarely does anything because the front columns usually hold luxuries already. The user is watching it; remove it if it stays pointless.
7. Open: why a start region can be as small as 18 tiles. Regions are 6 columns wide (`DEF_BACK`, `DEF_FRONTLINE`, `DEF_TOPBOTTOM` at 6 tiles each), which also keeps regional luxuries away from the front.
8. If an endless load returns, read `River check after mirror: ... loop edges N` in `Lua.log` before the game restarts. Test steps are on the Notion page "Tests" under "Weevee map 11".

**Notion:** this project shares the Lekmod workspace ("New Specs", "Tests", tracker row "Weevee 11 map, Standard: ..."). Specs for this map arrive on "New Specs".

## Principles
1. **Fix the cause, don't add a correction.** Before adding a pass that strips, ensures or caps something, find what produced the wrong result and change that. If a correction is really needed (for example the stock game passes can't be changed), its comment says what it corrects.
2. **Replacing means deleting.** When a new version of something works, remove the old one in the same commit. No `_UNUSED` copies, no switches kept "just in case": git has the history.
3. **Climates differ through data where possible.** Prefer a value in `GetBarrierConfig` over another `kind ==` check spread through shared code. Climate-specific passes return early at their top, in one obvious check.
4. **Name the scope of every change.** Say which climates it affects. `IsStandardClimate()` includes Standard-Diagonal; `cfg.tilted` tells them apart. Don't change other climates by accident.
5. **West side first, mirror last.** Every spec and description from the user means the west side, before the mirror. Generate on the west half only and count only the west half. `MirrorPlotsAfterResourcePlacement` is the only source of the east side, so every new operation goes before it. See "The mirror" under Facts.
6. **Comments explain why the code is the way it is now.** The story of earlier attempts belongs in the commit message. When you touch a comment that retells history, shorten it.
7. **Small commits, focused on why.** One topic per commit. Refactors don't change behaviour and are committed apart from changes that do.
8. **Say what was actually checked.** Syntax check, simulated outside the game, or seen in the game are different things; report which. Map changes count as done only once seen in a generated map.
9. **Ignore WIP climates unless the user names them.** That's Frosty, Bramble and Wasteland, marked `(WIP)` in the climate menu. Don't tune them, test them or plan around them.
10. **Clean as you go, only nearby.** While working on a function, remove dead code and duplicates next to it. Larger cleanups get their own commit, and the backlog below gets updated.

## Changelog
`README.md` holds the player changelog, newest version first. Whenever a change players would notice lands, add a line under "Next version"; a version bump turns that heading into the version number.
- For players who don't know the code: plain words, no function names, no reasons, no explanation.
- One short statement per line, e.g. "Capitals now spawn with a maximum of 4 luxuries".
- Name the climates when it isn't all of them. Skip WIP climates.
- Describe the difference from the last released version, so bugs that were never released and internal cleanup stay out.

## Words we use
- **Capital:** the exact tile the player's settler spawns on. **In the capital:** that tile plus radius 3 around it.
- **Regional target:** 6 of a player's regional luxury in their region, 3 of those 6 in their capital. It's the default structure, not a hard quota: landing below it now and then is fine, but other luxury rules mustn't disturb it much.
- **Capital luxury cap:** at most 4 luxuries in a capital, regionals included (`CapCapitalLuxuries`). It moves the others out first.
- **Endless load:** the loading screen never ends, without a crash. `Lua.log` ends normally and shows no error.
- **Dead on load:** the game dies while loading a new game. Usually no player got a valid start. Look in `Documents\My Games\Sid Meier's Civilization 5\Logs\Lua.log` for `Runtime Error` lines from the map's files (it resets when the game launches); `weevee_dbg.log` next to it shows the last step reached. A Lua error in generation stops the map outright: `GenerateMap`'s retry loop doesn't catch it.

## Facts
- **The mirror** is a 180-degree rotation, `(x, y)` to `(iW-1-x, iH-1-y)`, not a left-right flip. It copies plot type, terrain, feature, resource and amount, improvement, rivers and start positions, and is the last step in `StartPlotSystem` that edits plots. Exceptions: `AddGoodies` and `DetermineContinents` run after it, so goody huts wouldn't be mirrored if they're on (off in every logged test; continents are art only), and on Standard-Diagonal the copy follows the fold (`TiltedSkipMirrorDest`, `TiltedCopyHomePastMidToPair`). Some plot-stage helpers also write the east side (`PlaceMirroredMountain`); the mirror overwrites it anyway.
- `Weevee - vX.Y.Z.lua` is the map script; the `WV11*.lua` files are helpers. `include()` finds files by bare name anywhere under `Assets`, so the helpers must keep the `WV11` prefix. Version bumps follow `VERSION-BUMP.txt`.
- Civ V runs Lua 5.1: at most 200 local variables per scope (the map script has 129 at top level). Check syntax with `lupa.lua51`, not a newer Lua.
- According to an earlier comment in the script, calling a missing global during generation aborts it silently, showing up as "no valid start" rather than a `Lua.log` error.
- `Lua.log` holds every map since the game launched; it resets on launch, not per map. `weevee_persist.log` keeps growing across launches.
- Map generation time doesn't matter: tripling it is fine (user, 2026-11-02), so a reroll is a fine way to enforce a rule.
- `canHaveResource` looks only at terrain and hills/flat, so a water plot with a listed terrain passes. `Resource_TerrainBooleans` lists `TERRAIN_HILL`, which is a water terrain, and `setTerrainType` changes the plot type to match. Don't paint terrains from that table without filtering.
- Stock placement (`PlaceSpecificNumberOfResources`) never checks `CanHaveResource`, so the Resources table isn't what 10.5 actually does.
- Rivers can't flow opposite their starting direction (`DoRiver`).
- `Civ5DebugDatabase.db` in the `cache` folder holds the game's data as loaded; the mod's DLL source is the place to check engine rules. Both beat guessing.
- `River check` counts a river ending at a lake corner as reaching water.
- Scripts for simulating real functions outside the game (with `lupa`) were kept only in the session's scratchpad; rebuild as needed: pull the function out by regex, stub `Map` and plots. They test logic, not looks.
- Files are CRLF. Keep the endings when editing with a script.
- Generation order lives in `StartPlotSystem`; pass order matters, so read it before adding a step.

## Cleanup backlog (measured 2026-11-01)
Take these when working nearby. Check each before deleting; a name could still be used in a way a text search misses.
- **Functions nothing refers to (16):** `OasisInJunglePocket`, `OasisJungleExists`, `TongueNoGoWidth`, `TongueMtnDist`, `TongueGetJungleDepth`, `CopyWestToEast`, `WastelandForceCoastalLuxuryRoles`, `ScrubWaterNearSnow_UNUSED`, `GetWastelandSaltCoastPlotIndices`, `RescueUnplacedLuxuryTypes`, `ForceWastelandCoastalLuxuries`, `AddWetlandRiverDesert`, `TongueEconKind`, `EconBandBleed`, `PaintEconBands`, `PlaceMurkTundraLakeFish`.
- **Helpers no map includes:** both map scripts include `WV11MapGeneratorW8`, so `WV11MapGeneratorW.lua` and `WV11AssignStartingPlotsW.lua` look unused. The two generator files differ only in their include line.
- **`StartPlotSystem` has 81 steps**, 17 of them `Strip*` and 9 `Ensure*`, and some run twice (`EnsureLuxuryQuota`, `StripIllegalWaterResources`, `ClampPlayerStartsOffEdges`, `NudgePlayerStartsMinDist`). These are the main place where corrections pile up on corrections.
- **Climate checks:** about 180 `kind ==` / `kind ~=` checks, plus many `IsOasisClimate`, `IsBramble` and `IsTiltedMirrorAxis` calls.
- **`AssignStartingPlots:FindStart`'s nil-start check** never runs here: the map uses rectangular regions (`self.method = 3`), and `ChooseLocations` then calls `FindStartWithoutRegardToAreaID`.
- **World wrap** is frozen off (`WORLD_WRAP_FROZEN_OPS`), so `IsSnowWrapX` branches are dead code. Removed from the front mountains only.
