# Digimon World 2003 — bugs in the original game

Bugs and oddities of the original European version (SLES-03936) found by reading its decompiled code
([dw3_decomp](https://github.com/ReGame-Labs/dw3_decomp)). Each entry names the code it comes from. The native PC
port is built from that same code, so it has them too unless noted.

## Confirmed bugs

### 1. Every elemental field boosts Thunder

`boostFieldElement` — `src/fightstg/tech_action.c`

The field's element is taken from the technique's *icon* field (`boostElement = tech->icon`), which is 6 (Thunder)
for all seven field techniques, instead of its element. So Fire Field, Water Field, … Dark Field all boost Thunder
and weaken Dark. The message still names the right element (it uses `tech->element`).

*Fixed by:* the Harmony mod (*Fixed Field Moves*).

### 2. A growth of 5 grows less than a growth of 4 (stats)

`STFGTREP_raiseStats` — `src/stfgtrep/growth.c`

Each level, a stat gains `STFGTREP_statGains[tier][growth + random(0–4)]`. The table has 9 columns (0–8), so a
growth of 5 can ask for column 9, which doesn't exist; the PlayStation reads the first value of the next row instead,
which is 0 or 1. One level-up in five, the stat that should grow the most gains almost nothing:

| Levels | Average gain, growth 4 | Average gain, growth 5 |
|---|---|---|
| 2–4 | 11.4 | 10.0 |
| 5–19 | 8.8 | 7.6 |
| 20–39 | 5.6 | 4.8 |
| 40–59 | 4.6 | 4.0 |
| 60–79 | 3.0 | 2.6 |

From level 80 the last row reads past the whole table, into the resistance gains, whose first value is also 0.

Affected Rookies: **Kumamon** (Strength and Defense), **Renamon** (Spirit), **Monmon** (Speed), **Veemon** (Wisdom and
Charisma). The very stats each was designed to excel at grow about 13% slower than a 4 would.

### 3. A growth of 5 grows less than a growth of 4 (resistances)

`STFGTREP_raiseStats` — `src/stfgtrep/growth.c`

Up to level 40, each resistance gains `STFGTREP_resistGains[growth + random(0–3)]`, a table of 8 values
(`0 0 0 1 1 1 2 2`). A growth of 5 can ask for value 8, past the end; on the disc the next word is 0. A growth of 5
averages **1.25** a level, a growth of 4 **1.5**.

Seven of the eight Rookies have one resistance with growth 5: Kotemon (Water), Monmon (Machine), Agumon (Fire),
Veemon (Ice), Guilmon (Wind), Renamon (Thunder), Patamon (Dark). Kumamon has none.

### 4. Magic without an element reads the target's Speed as a resistance

`FIGHTSTG_rollMagicCritical` — `src/fightstg/rolls.c`

The magic critical chance is `min(64, ElementStrength × 100 / resist[element − 2]) + 4`. For a technique with no
element, `element − 2` is −1, and the value read is the one stored just before the resistances: the target's
**Speed**. So the critical chance of element-less magic depends on the target's Speed:

* Most of these techniques (the status and drain spells) have an element strength of 1: `100 / Speed` is about 0–2,
  for a 3–5% critical chance.
* **Gate of Destiny** (64), **KongouKaimandara** (72) and **Back Storm** (72) have a high element strength but no
  element: against a target with Speed up to about 100 they reach the 53% cap (`64 × 100 / 100 = 64`), and 28% at
  Speed 200.

### 5. The Shock Horn never paralyzes

`FIGHTSTG_applyTechEffect` — `src/fightstg/tech_effects.c`; `WEAPON_DATA` — `src/main/game/items.c`

"Horn that paralyzes the body", but its record has a status chance of **0** and a strength of 0. The plain attack only
tries a weapon's status when that chance isn't 0 (`if (stats[0].statuses[HIT_PARALYSIS].chance) FIGHTSTG_tryParalysis()`),
so the Shock Horn never paralyzes: it is a horn of Strength +90 and nothing else. The other status weapons have
24–64/128.

## Oddities

Behaviour that looks unintended, or at least surprising, but may be by design.

* **Debuffs last according to the target's Spirit, not the user's.** `FIGHTSTG_queueBoostEnd` times a boost by the
  Spirit of the Digimon it is on. For a buff that is the user; for a debuff (Armor Break, AncientGreymon…) it is the
  enemy, so debuffs last longer on high-Spirit enemies.
* **The field elements the code weakens don't match the descriptions.** "Fire Field: strengthen fire, weaken ice",
  but `FIGHTSTG_opposedElements` makes the Fire Field weaken Water (Ice Field weakens Fire, and so on). See
  [FORMULAS.md](FORMULAS.md#elemental-fields).
* **A field never boosts an accessory's element.** `FIGHTSTG_getElementBoost` only looks at the technique's element,
  so a basic attack with Fire Power gets nothing from a Fire Field.
* **A critical-chance item lowers accuracy.** In `FIGHTSTG_rollHit` the player's level difference has the critical
  bonus taken off it, so Search Crest, Saber Fang or Shuriken cost some accuracy.
* **Multi-hit techniques roll one critical for all their hits**: all hits are critical, or none.
* **No minimum damage**: a hit can do 0.
* **Two Digimon share more digivolution EXP than one.** `STFGTREP_getDigimonExp` gives the forms used in a battle
  100% of the DV EXP if one was used, **60% each if two** (120% in all) and an equal split if three or more. Using
  exactly two forms a battle earns the most DV EXP.
* **Running away starts hard**: the first attempt has a base chance of 8/128 (6%), the second 16/128, and so on, plus
  the level and Speed differences (`FIGHTSTG_testRunAway`).

## Fixed in the European version

* **Field gauges** (`src/fieldstg/data/fieldstg.c`): the USA version has no row 8 in `FIELDSTG_gaugeRows` and reads
  past the table (a null pointer) once `GAME.randomGauges` is used up. The European version adds the row.

* **An asleep enemy could run away** (USA). The European `FIGHTSTG_testRunAway` stops it.
* **The Counter Crest countered every hit** (USA). The European version rolls `25% + 50% × damage / max HP`
  (`FIGHTSTG_rollCounter`) before the counterattack.

## Latent

* **Boost slot 3** (`FIGHTSTG_boostStats`): a fourth boost would be worked out from Attack, added to an enemy's
  Spirit, and ignored for partners. No technique uses it.

## Reported elsewhere

* **Picking Claw and Snapping Claw when the enemy is knocked out**: listed as fixed by the Harmony mod; its cause is
  not traced here.
