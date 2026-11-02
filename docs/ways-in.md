# Ways in: guaranteed openings through the center barrier

Reference for the rule that makes sure each side of the map can be attacked
through the snow. Built for Standard-Diagonal (2026-11-02, commit `66d8b2d`).
Read this before applying the same idea to another climate.

## What it is for

The teams play the economic part of the game on their own side of the map.
Crossing the center snow becomes possible at a late tech, and then one team
usually attacks the other: it moves out of its own land, across the snow, and
into the defender's land.

So the question is not "is the barrier passable somewhere", but "can the
defending side be entered". That depends on:

- the snow the attacker crosses, including the half next to the attacker,
  since the mirror decides what lies in the center;
- the defender's own snow half, tundra column and front line.

It does **not** depend on:

- the attacker's own front line and tundra: they start behind it and can pick
  where to come out;
- mountains further back on the defender's side, behind the tundra column:
  those are the defender's normal defence and are meant to be there.

A rated map whose only straight passage was 3 rows wide, narrowed by a
mountain in the center, was too hard to push through. Two separate openings
are better than one, even if neither is a clean set of empty rows.

## The rule

After the mirror, each side needs at least `frontOpenings.min` (2) openings
of at least `frontOpenings.width` (4) tiles into it. Config, in
`GetBarrierConfig`, Standard-Diagonal only:

```lua
frontOpenings = {width = 4, min = 2},
```

- A passage can bend, run diagonally, or squeeze between mountains.
- Its **width** is its narrowest point: the number of tiles side by side there.
- An opening counts once per 4 tiles of width, so an 8-wide one counts as 2.
  Without this, a snow with no mountains at all would be "one opening" and
  fail, and no repair could fix that, since removing mountains never splits
  an opening.
- Water inside the strip (ice lakes) blocks like mountains. Hills don't block.

The east side's strip is the exact 180-degree mirror image of the west's, so
only the west is measured, and every change is made in mirror pairs.

## The strip that is measured

Per row `y` (rows 1 to `iH - 2`; the polar edge rows are left out), with
`fold` the barrier's center (`floor(iW / 2)`, or `TiltedFoldMid(y)` on
Diagonal) and `half` half the center snow width (2):

| Part | Columns | Role |
|---|---|---|
| Defender's front line | `fold - half - 2` | source side: the defender's land starts west of it |
| Defender's tundra | `fold - half - 1` | |
| Defender's snow half | `fold - half` .. `fold - 1` | |
| Attacker's snow half | the mirror copies of the defender's snow | its far edge (copy of column `fold - half`) is the sink side |

`FrontBarrierBand()` builds it from the west tiles and their copies via
`getMirroredPlot`, so the strip follows the fold exactly on Diagonal.

## How it is measured: `MeasureFrontOpenings`

The widths come from a **minimum vertex cut**: the smallest set of open tiles
whose removal would cut every path from the front line to the snow's far edge.

1. Each open tile becomes two nodes (in, out) joined by a link of capacity 1.
   Every other link (tile to neighbouring tile, source to the front line, far
   edge to the sink) is unlimited, so only tiles can be cut.
2. Max flow (Edmonds-Karp, breadth-first augmenting paths) from source to
   sink. By Menger's theorem the flow equals the size of the minimum cut.
3. The cut tiles: in-node reachable from the source in the residual graph,
   out-node not.
4. Cut tiles that touch each other form one group; each group is one opening,
   and its size is that opening's width at its narrowest point.

The strip is about 6 tiles by 18 rows, so this is fast enough to run many
times per map.

## How it is repaired: `EnsureFrontOpenings` and `CheapestFrontCorridor`

If a side has fewer than 2 openings of 4, the map is **repaired, not
rerolled**:

1. For every row where a corridor could cross the strip's middle column, find
   the cheapest corridor 4 tiles wide from the front line to the snow's far
   edge (dynamic programming, one column per step, may shift one row per
   step). When it shifts a row it covers both positions in that column (5
   tiles), because a plain 4-tile shift is only 3 wide on a hex grid.
2. Cost per tile: open 0, front line or tundra mountain 1, snow mountain 3,
   water impossible. So corridors follow the existing openings and spare the
   snow, as the user prefers mountains inside the snow.
3. Each candidate corridor is tried on a copy and measured. The cheapest one
   that adds an opening wins; if none does, the cheapest that widens any.
4. The winner's mountains turn into hills, each together with its mirror copy
   (`getMirroredPlot`). That gives exactly the map you would get by changing
   the tile before the mirror and mirroring again. The copy of a front-line or
   tundra tile is on the attacker's line, outside the strip, so copies are
   changed on the map directly, not looked up in the strip.
5. Measure again; repeat until the side has 2.

Two earlier versions failed in the simulation and are worth knowing about:

- **Removing one mountain at a time, keeping the best**: gets stuck. A passage
  usually has several chokes in a row, and removing one mountain alone widens
  nothing.
- **Carving corridors and reserving them so they aren't found again**: runs
  out of rows. Each corridor plus its mirror reserved 8 of 18 rows.

## Why repair instead of reroll

The rule it replaced rerolled any map without 4 straight rows free of
mountains from the front line through the snow. Every snow mountain blocks a
row, so rerolling filtered out exactly the maps the user likes (91 logged
attempts):

| Snow mountains in the attempt | Passed the old check |
|---|---|
| 0-2 | 53% |
| 3-4 | 44% |
| 5-6 | 24% |

Kept maps averaged 2.6 snow mountains against 3.1 rolled, and the snow looked
boring. Repair keeps every map and only changes the few mountains in the way.
The same reroll bias showed up with the luxury targets (fixed by rolling them
once per game), so check any reroll rule for it.

## Numbers

Simulated outside the game (lupa with Lua 5.1, the real functions, a stubbed
hex `Map`):

- The widths matched an independent max-flow (networkx) on 300 random
  barriers; the mirror never broke; the target was reached every time.
- Generation-like layouts (front line 0-3 mountains with a 4-row opening, 1-5
  snow groups of 1-3 tiles, 1-3 contact groups): the old rule passed 46% of
  them, real maps 43%, so they are a fair stand-in. Of those, 73% needed
  repair, losing about 2 mountains per side.
- Requiring 3 openings instead of 2 would have repaired 93% and removed about
  half the barrier's mountains.

In the game (first 8 Standard-Diagonal maps): 3 needed nothing, 5 had one
corridor carved, costing 2-4 mountains in total (1-2 per side). The user rated
every one solid.

## Logs

One line per map in `Lua.log` and `weevee_persist.log`:

```
Front openings after mirror: widths 4 3 2 1 1 1 | carved 1 corridors, 4 mountains to hills (front line and tundra 2, snow 2) | final widths 6 5 2 1 = 2 openings of 4, minimum 2
```

- `widths` before: every opening's width, widest first.
- `mountains to hills` counts both copies, so halve it for one side.
- `final widths ... = N openings of 4`: the counted result.

To rate a batch, match ratings to these lines in order, together with the
`Mountain roll:` lines (written before the repair).

## Applying it to another climate

The functions are written for any climate with a `GetBarrierConfig`; adding
`frontOpenings` to a climate's config switches it on. Check these first:

1. **Where is the front?** The strip assumes one front-line column and one
   tundra column before the snow (Standard's `frontMountains = "line"`).
   Climates on the five-column mountain field (`"field"`: Oasis, Peaky,
   Murky) need the strip widened to cover the field, so the source sits at
   the field's back edge. Make that a config value (for example the number of
   columns before the snow), not a `kind ==` check.
2. **What counts as blocking?** Mountains and water now. A climate with other
   impassable or very slow terrain in the barrier needs a decision on that.
3. **Cost classes.** Front line and tundra 1, snow 3. In another climate the
   question is which mountains carry the look and should be spared.
4. **Straight barriers** need nothing special: `fold` is `floor(iW / 2)`.
5. **Width and count.** 4 wide and 2 per side were judged on Diagonal's
   20-row Small map. Taller maps or wider barriers may want more; judge in the
   game.
6. **Old rules to remove.** If the climate has a reroll or row-based gap
   check for the same purpose, the repair replaces it.
7. **Pass order.** It runs after `MirrorPlotsAfterResourcePlacement`, since
   only the mirrored map shows what the center looks like. Changing a
   mountain to a hill there needs no resource handling: mountains carry no
   resources.

## Rebuilding the simulation

The test scripts lived in a session scratchpad and are gone. To rebuild:

- Pull `PlotDirNoXWrap`, `getMirroredPlot` and the block from
  `FrontBarrierBand` to `EnsureFrontOpenings` out of the map script by regex.
- Stub `Map.GetGridSize`, `Map.GetPlot` and `Map.PlotDirection` with Civ V's
  hex offsets (y grows north; odd rows sit half a tile east):
  even rows `{0,1},{1,0},{0,-1},{-1,-1},{-1,0},{-1,1}`, odd rows
  `{1,1},{1,0},{1,-1},{0,-1},{-1,0},{0,1}` for directions 0-5.
- Stub `GetBarrierConfig` (with `frontOpenings`), `ResolveSnowWrapWidths`
  (`0, 4`) and `WeeveeDbgPersist`.
- Place mountains on the west and mirror them with `(iW-1-x, iH-1-y)`.
- Check the widths' sum against networkx `maximum_flow_value` on the same
  strip, and that every mountain's mirror copy is still a mountain afterwards.
