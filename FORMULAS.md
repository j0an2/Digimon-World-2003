# Digimon World 2003 — battle formulas

How the original European version (SLES-03936) computes damage, healing, buffs and fields, read from its
decompiled C code ([dw3_decomp](https://github.com/ReGame-Labs/dw3_decomp)). Each section names the function and
file it comes from, so you can check it yourself.

Integer division rounds down everywhere, as on the PlayStation. Many values are stored in 128ths or 64ths: a value
of 64 in 128ths is 50%.

## What each stat does

Where the battle code reads each stat:

| Stat | What it does |
|---|---|
| **Strength** (Attack) | Physical damage dealt |
| **Defense** | Reduces physical damage taken, and only that |
| **Spirit** | Magic damage dealt **and** taken · the strength of the elemental fields you set · how long your buffs last (and the debuffs put on you) · how long the status ailments you inflict last |
| **Wisdom** | Magic accuracy and evasion · inflicting and resisting status ailments · the HP your healing restores · resisting an enemy's stat raises |
| **Speed** | Physical accuracy and evasion · how often a Digimon acts · running away |
| **Charisma** | Not used in battle: it is a requirement for some digivolutions |

Defense, Spirit and Wisdom are three different defenses: Defense against physical damage, Spirit against magic
damage, Wisdom against being hit by magic and against status ailments.

What Wisdom does, in detail:

| | Yours (attacking) | The target's (defending) |
|---|---|---|
| Magic hit chance | + Wisdom / 8 | − Wisdom / 8 |
| Inflicting a status ailment | + Wisdom / 8 | − Wisdom / 16 |
| Healing | + Wisdom × Power / 8 HP | – |

## Physical damage

`FIGHTSTG_computeDamage`, `FIGHTSTG_adjustDamage` — `src/fightstg/damage.c`

```
base = Power × Attack(user) / Defense(target)
```

### Example: level-5 Kotemon vs the first Kunemon

Kotemon's basic attack (technique 1: power 60, accuracy 110, no element) against the Kunemon of the first areas
(level 1, 120 HP, strength 16/16, from `src/fieldstg/data/fieldstg.c`). Kotemon has no equipment.

**Kotemon's Attack at level 5.** A new game starts it at 48 Strength (`DIGIMON_DATA`, `initNewGameData` in
`src/main/game/party.c`). Each level adds `STFGTREP_statGains[tier][growth + random(0–4)]`
(`STFGTREP_raiseStats`, `src/stfgtrep/growth.c`): with Kotemon's Strength growth 4, +8 to +14 for levels 2–4 and
+6 to +11 for level 5. So at level 5 its Attack is **78 to 101, 91 on average**.

**Kunemon's Defense.** 42 in the enemy table, × strength 16 / 16 = **42**.

```
damage = Power × Attack(user) / Defense(target)
       = 60 × 91 / 42 = 130
```

| Kotemon's Attack | Damage |
|---|---|
| 78 (unlucky level-ups) | 111 |
| 91 (average) | 130 |
| 101 (lucky level-ups) | 144 |

None of the other steps apply: no field, no family, no charge, no element.

**Hit chance**: `110 + 110 × (SpeedDiff / 8 + LevelDiff) / 128` in 128ths (`FIGHTSTG_rollHit`). With Kotemon's
Speed about 63 against 48 and 4 levels more: `110 + 110 × (1 + 4) / 128 = 114` → **89%**.

**Critical**: 4/128 = **3.1%**, for ×1.5 to ×2.5.

Kunemon has 120 HP, so a level-5 Kotemon's basic attack **kills it in one hit 96% of the time** when it lands.

### Example: an elemental physical technique

The same level-5 Kotemon (Attack 91) and Kunemon (Defense 42), now with Kotemon's own technique **Hot Head**
(technique 53: power 75, Fire, element strength 48, accuracy 127, 20 MP). Kunemon's Fire resistance is 90.

```
base   = Power × Attack(user) / Defense(target)
damage = base + base × 2 × ElementStrength / Resistance(target, element)
       = base × (1 + 2 × ElementStrength / Resistance)

base   = 75 × 91 / 42                = 162
damage = 162 × (1 + 2 × 48 / 90)     = 334
```

| | Basic attack | Hot Head |
|---|---|---|
| Base | 130 | 162 |
| Element | – | ×2.07 |
| **Damage** | **130** | **334** |
| Hit chance | 89% (accuracy 110) | 100% (accuracy 127, see [Hit chance](#hit-chance)) |

The element multiplier is `1 + 2 × ElementStrength / Resistance`, so with Hot Head's 48:

| Target's Fire resistance | Multiplier | Damage on this Kunemon |
|---|---|---|
| 50 | ×2.92 | 473 |
| 90 (Kunemon) | ×2.07 | 334 |
| 160 | ×1.60 | 259 |
| 300 | ×1.32 | 213 |

A physical technique's element only ever adds damage; a high resistance just adds less. (Magic is different: a
high resistance can halve it.)

### Example: a Fire Field

The same Kotemon (Attack 91) and Kunemon (Defense 42, Fire and Water resistance 90) with a **Fire Field** set by a
Digimon with 80 Spirit. The field acts **before** the element, on the base value
(`FIGHTSTG_getElementBoost`, see [Elemental fields](#elemental-fields)):

```
FieldBoost = 64 + Spirit(whoever set the field) / 10          (at most 127)

technique of the field's element:  boosted  = base × (1 + FieldBoost / 128)
technique the field weakens:       weakened = base × (1 − FieldBoost / 256)
technique with no element:         unchanged
then the element step:             damage   = boosted × (1 + 2 × ElementStrength / Resistance)
```

```
FieldBoost = 64 + 80 / 10 = 72
base       = 75 × 91 / 42  = 162          (Hot Head's power; the basic attack's base is 130)

Fire attack (Hot Head):   162 × (1 + 72 / 128) = 253  →  253 × (1 + 2 × 48 / 90) = 522
Water attack (weakened):  162 × (1 − 72 / 256) = 117  →  117 × (1 + 2 × 48 / 90) = 241
Basic attack:             no element, so the field does nothing  →  130
```

With more Spirit the field is stronger, but the code caps FieldBoost at 127 (`if (element >= 0x80) element = 0x7F`
in `boostFieldElement`). That is reached at 630 Spirit; above it, up to the maximum of 999, the field does not
improve (64 + 999 / 10 would be 163):

| Spirit of whoever set the field | FieldBoost | Fire attack (Hot Head) | Water attack (weakened) |
|---|---|---|---|
| 80 | 72 | 162 × (1 + 72/128) = 253 → **522** (+56%) | 162 × (1 − 72/256) = 117 → **241** (−28%) |
| 200 | 84 | 162 × (1 + 84/128) = 268 → **553** (+66%) | 162 × (1 − 84/256) = 109 → **225** (−33%) |
| 600 | 124 | 162 × (1 + 124/128) = 318 → **657** (+97%) | 162 × (1 − 124/256) = 84 → **173** (−48%) |

The percentages compare with 334, the damage without a field. The basic attack stays at 130 in all three.

The game rounds each part down on its own (162 − 45 = 117, where 162 × 0.719 would give 116), so a result can
differ by 1 from the one-line formula.

| Attack | No field | Fire Field | |
|---|---|---|---|
| Basic attack (no element) | 130 | 130 | same |
| Fire (Hot Head) | 334 | **522** | +56% |
| Water (a Hot Head made Water, for comparison) | 334 | **241** | −28% |

These are the numbers of a field that works as intended (with Harmony's *Fixed Field Moves*). In the original game
and the PC port the Fire Field boosts Thunder and weakens Dark instead, so against Kunemon all three stay as without
a field.

### Example: a technique against its family

**Bug Buster** (technique 131: power 130, accuracy 110, 24 MP, no element, family Insect; learned by Hookmon and
Armormon) against the same Kunemon, which **is an Insect**. Same Attack 91, to compare.

```
base   = Power × Attack(user) / Defense(target)
family = base × (1 + 1/2)                 when the technique's family is the target's

base   = 130 × 91 / 42  = 281
family = 281 × 1.5      = 421
```

The family match also adds **60/128 to the critical chance** (`FIGHTSTG_rollCritical`), which the game's
description doesn't mention: `(4 + 60) / 128 = 50%` instead of 3.1%.

| | Hot Head | Bug Buster |
|---|---|---|
| Damage without critical | 334 | **421** |
| Critical chance | 3.1% | **50%** |
| Damage with a critical | 501–829 | **631–1045** |
| **Average damage** | ≈ 344 | **≈ 630** |
| Hit chance | 100% | 89% |

Against its family a Buster does +50% and lands a critical every other hit: on average almost twice an elemental
technique that hits a weakness. No technique has both a family and an element: the 13 family techniques have none.

Families, named from the Buster techniques' descriptions: 2 Dinosaur, 3 Evil, 4 Ghost, 5 Machine, 7 Bird,
8 Insect, 10 Aquatic, 11 Dragon.

### Example: elemental accessories

The accessories Fire Power, Water Power, Ice Power, Wind Power, Bolt Power, Metal Power and Dark Power give the
Digimon an element (`FIGHTSTG_computeStats`). Each comes in three strengths:

| Accessory | Element strength |
|---|---|
| … Power. | 8 |
| … Power 2 | 16 |
| … Power S | 24 |

**Only one can be worn**: all 21 are one accessory group, and the equipment screen allows one of a group at a time
(`EQUIP_KIND_GROUP_ACCESSORY`, `src/stitshop/equip.c`). So two Fire, or Fire and Water, is not possible.

The battle uses the accessory's element **for the basic attack and the shared techniques without an element**
(Bug Buster, Big Shot, Hammer Rush…). It does **not** use it for a Digimon's **own techniques** (Vee Head Butt,
Bear Fist, Terra Force…) or **pair techniques**, even when they have no element. A technique with an element
uses its own and ignores the accessory. The elemental
field looks at the **technique's** element only, so it never boosts an accessory's element.

```
damage = base × (1 + 2 × AccessoryStrength / Resistance(target, accessory's element))

basic attack, Fire Power S:  130 × (1 + 2 × 24 / 90) = 199
Bug Buster,  Fire Power S:   421 × (1 + 2 × 24 / 90) = 645      (421 = 281 × 1.5, the family bonus)
```

Same Kotemon (Attack 91) and Kunemon (Defense 42, Fire resistance 90):

| | No accessory | Fire Power. (8) | Fire Power 2 (16) | Fire Power S (24) |
|---|---|---|---|---|
| Basic attack | 130 | 153 | 176 | **199** |
| Hot Head (Fire technique) | 334 | 334 | 334 | 334 — the accessory is ignored |
| Basic attack + Fire Field (boost 72) | 130 | 153 | 176 | 199 — the field does nothing |
| Hot Head + Fire Field | 522 | 522 | 522 | 522 |
| Bug Buster (no element, against an Insect) | 421 | 495 | 570 | **645** |

The accessories are weak next to a technique's own element (Hot Head 48, Burn Slash 64), but they add to the
Buster techniques, which have none: Bug Buster with Fire Power S does +53% on top of its family bonus. Against a
given enemy, the accessory worth wearing is the one of the element it resists least.

## Hit chance

`FIGHTSTG_rollHit` — `src/fightstg/rolls.c`

```
chance (in 128ths) = Accuracy + Accuracy × (SpeedDiff / 8 + LevelDiff) / 128
SpeedDiff = Speed(user) + accessories' accuracy − Speed(target)
LevelDiff = Level(user) − Level(target)
```

The game draws a number from 0 to 127 and the hit lands if it is **below** the chance, so a chance of 128 or more
never misses. Each technique has its own Accuracy: a level-5 Kotemon (Speed 63, 4 levels above) against Kunemon
(Speed 48) gets SpeedDiff = 15, which counts as 15 / 8 = 1, and LevelDiff = 4:

| Technique | Accuracy | Chance | |
|---|---|---|---|
| Basic attack | 110 | 110 + 110 × (1 + 4) / 128 = 110 + 4 = 114 | 89% |
| Hot Head | 127 | 127 + 127 × (1 + 4) / 128 = 127 + 4 = 131 | never misses |

An enemy's hit chance uses the same formula minus the target's evasion, and is never below 32 (25%).

### Evasion: Dance Feather and Dance Wing

The evasion accessories, **Dance Feather (Evasion +10)** and **Dance Wing (Evasion +20, Charisma +1)**, are in the
same group, so only one can be worn. Armor with evasion adds to it the same way. Evasion is taken off the enemy's
SpeedDiff, so a Dance Feather is **10 more Speed, for dodging only**:

```
enemy's chance = Accuracy + Accuracy × ((Speed(enemy) − Speed(you) − Evasion) / 8 + LevelDiff) / 128
```

Because the Speed difference is divided by 8 and everything is then divided by 128 and rounded, the gain is tiny.
An enemy technique with accuracy 100:

| Situation | Nothing | Dance Feather | Dance Wing |
|---|---|---|---|
| Same Speed and level | 78.1% | 78.1% (no change) | 77.3% |
| Enemy 20 Speed slower | 77.3% | 76.6% | 75.8% |
| Enemy 50 Speed faster, 5 levels above | 84.4% | 83.6% | 82.8% |

About **1 point less** with the Feather (sometimes none, from the rounding) and **1.5 points** with the Wing. It does
nothing against:

* **magic**: `FIGHTSTG_rollMagicHit` only looks at Wisdom and level;
* enemies that already hit **more than 100%** of the time, or that are at the **25% floor**;
* your own attacks: the enemy's evasion isn't counted when you attack, and yours only when you are attacked.

Almost decorative: an element accessory, an HP Proxy or, on a Buster, a Search Crest does more.

## Magic damage

`FIGHTSTG_computeMagicDamage` — `src/fightstg/damage.c`

```
power = Power × (Spirit(user) × 50 / Spirit(target) + 50) / 100
```

The multiplier is `0.5 + 0.5 × Spirit(user) / Spirit(target)`: ×1 with equal Spirit, ×1.5 with twice the target's,
×2 (the cap) from three times; ×0.75 with half, ×0.625 with a quarter, never below ×0.5. Physical damage
has no such bound: twice the Attack is twice the damage.

**Wisdom does not reduce magic damage**: the target's **Spirit** plays the part Defense plays for physical attacks.
Wisdom decides whether magic **lands** (`FIGHTSTG_rollMagicHit`):

```
chance (in 128ths) = Accuracy + Accuracy × (WisdomDiff / 8 + LevelDiff) / 128
WisdomDiff = Wisdom(user) − Wisdom(target)
```

and also counts for status chances and healing.

Then: the field boost, and the target's resistance
to the element:

| Resistance | Damage multiplier |
|---|---|
| under 100 | (400 − 3 × res) % — up to ×4 |
| 100 to 299 | (125 − res / 4) % |
| 300 and up | (65 − res / 20) % |

A magic technique's *element strength* does not change its damage: only the element against this table does. It
raises its **critical chance** instead (see below).

Then +50% against the target's family, the critical hit (+50% to +150%), the damage cut, at most 5 × `power`, and
at most 9999 (European version).

### Example: level-5 Renamon's Diamond Storm vs the first Kunemon

**Diamond Storm** (technique 59: magic, power 100, Ice, element strength 32, accuracy 127, 30 MP), Renamon's own
technique from the start, against the same Kunemon (level 1, Spirit 40, Wisdom 50, Ice resistance 90).

**Renamon's Spirit at level 5.** It starts at 61 with Spirit growth 5. The gain table only has columns 0–8, so
`growth + random(0–4)` can read column 9, which on the PlayStation is the first value of the next row: a tiny gain
now and then. At level 5 its Spirit is **64 to 114, 99 on average**.

```
power  = Power × (Spirit(user) × 50 / Spirit(target) + 50) / 100       (kept between Power/2 and Power × 2)
damage = power × (400 − 3 × Resistance) / 100                            (resistance under 100)

power  = 100 × (99 × 50 / 40 + 50) / 100 = 100 × (123 + 50) / 100 = 173
damage = 173 × (400 − 3 × 90) / 100      = 173 × 1.30              = 224
```

| Renamon's Spirit | power | Damage |
|---|---|---|
| 64 (unlucky level-ups) | 130 | 169 |
| 99 (average) | 173 | 224 |
| 114 (lucky level-ups) | 192 | 249 |

**Hit chance**: `127 + 127 × (WisdomDiff / 8 + LevelDiff) / 128`. Renamon's Wisdom is about 92 against 50, so
`127 + 127 × (5 + 4) / 128 = 135`: it never misses.

**Critical.** A magic critical depends on the element (`FIGHTSTG_rollMagicCritical`):

```
value  = ElementStrength × 100 / Resistance(target, element)        (at most 64)
chance = value + 4                                                  (in 128ths)

Diamond Storm (strength 32) vs Kunemon (Ice 90):  value = 32 × 100 / 90 = 35  →  39 / 128 = 30%
Flame Ball    (strength 16) vs Kunemon (Fire 90): value = 16 × 100 / 90 = 17  →  21 / 128 = 16%
```

Both have power 100 and do the same 224 to Kunemon, since a magic technique's element strength doesn't change its
damage; Diamond Storm, with twice the strength, lands almost twice as many criticals. A critical adds +50% to
+150%.

Diamond Storm's critical chance by the target's Ice resistance:

| Ice resistance | Value | Critical chance |
|---|---|---|
| 50 | 64 (the cap) | 68/128 = 53% (the most without a family match) |
| 90 (Kunemon) | 35 | 30% |
| 100 | 32 | 28% |
| 200 | 16 | 16% |
| 300 | 10 | 11% |

So magic of an element the enemy is weak to gets more damage **and** more criticals.

**A target with a status ailment takes more criticals**, physical and magic alike, by the strength of the technique
that inflicted it:

| Target | Extra critical chance (in 128ths) |
|---|---|
| Paralyzed or asleep | + strength / 8 |
| Confused | + strength / 2 |

| Status technique's strength | Paralysis or sleep | Confusion |
|---|---|---|
| 80 (Spinal Tap, Stun Shock, Hypno Gas, Panic Bites, Confuse Gas) | +10 (+8%) | +40 (+31%) |
| 96 (Brain Freeze, Paralyze Shock, Hypno Nebula) | +12 (+9%) | +48 (+38%) |
| 104 (T-Sword) | +13 (+10%) | – |
| 127 (Mugenhadou, Confuse Stab, Confuse Nebula) | +15 (+12%) | +63 (+49%) |

Diamond Storm against a Kunemon confused by Confuse Nebula: `39 + 63 = 102` → **80% criticals**.

Kunemon has 120 HP: Diamond Storm always kills it. For comparison, Renamon's basic attack (Attack about 61) does
`60 × 61 / 42 = 87`.

How the target's resistance changes Diamond Storm (power 173):

| Ice resistance | Multiplier | Damage |
|---|---|---|
| 50 | ×2.50 | 432 |
| 90 (Kunemon) | ×1.30 | 224 |
| 100 | ×1.00 | 173 |
| 160 | ×0.85 | 147 |
| 200 | ×0.75 | 129 |
| 300 | ×0.50 | 86 |

Unlike a physical technique's element, magic can be **halved** by a high resistance, and more than doubled by a low
one.

## Critical hits

`FIGHTSTG_rollCritical` — `src/fightstg/rolls.c`

A physical hit is critical with this chance, in 128ths:

```
chance = 4                                  (3.1%)
       + EffectStrength                     if the technique has the Higher critical effect
       + the equipment's critical bonus     (×2 for special Digimon) if the technique has no effect at all
       + 60 if the technique is strong against the target's family (+16 for a weapon's family instead)
       + more while the target is paralyzed, asleep or confused
```

| Technique | Critical chance |
|---|---|
| Big Shot | 44/128 = 34% |
| Swing Swing, Captain Cannon, Maltcyclone | 64/128 = 50% |
| Pinpoint Shot | 68/128 = 53% |

A critical adds `damage × (32 to 95) / 64`: **+50% to +148%**, about double on average.

A **magic** critical (`FIGHTSTG_rollMagicCritical`) has its own chance, from the technique's element strength
against the target's resistance:

```
chance (in 128ths) = min(64, ElementStrength × 100 / Resistance(target, element)) + 4
```

(+60 more against the technique's family), and adds +50% to +150%.

### Critical equipment: Search Crest, Saber Fang, Shuriken

`FIGHTSTG_computeStats` (`src/fightstg/stats.c`) and `FIGHTSTG_rollCritical`

| Item | Kind | Critical bonus | Who |
|---|---|---|---|
| Search Crest | Accessory | +32 | Every Rookie |
| Shuriken | Weapon (both hands) | +32 | Monmon |
| Saber Fang | Weapon | +64 | Agumon, Veemon, Guilmon |

**When it counts.** Only for the **basic attack and physical techniques with no effect of their own**. A technique with
an effect uses its own and ignores the equipment:

* **Higher critical** techniques (Big Shot, Pinpoint Shot…) keep their own chance: Big Shot is 34% with or without the
  crest. The two never add up.
* **Multi-hit** techniques (Hammer Rush, Impact Rush…) and status techniques get no bonus.
* **Magic** has its own critical roll (above) and ignores it.

**They don't stack.** A critical weapon (or any status or drain weapon) makes the game skip the accessories, so a
Search Crest does nothing next to a Shuriken or a Saber Fang. Two Search Crests count as one.

**Aura Plug doubles it.** The Aura Plug item puts the partner in its *special state* until it is switched out or
knocked out, and the equipment's bonus counts twice: `bonus × 2 + 4`.

**The cost: accuracy.** In `FIGHTSTG_rollHit` the bonus is taken off the level difference, for the same techniques it
helps (see [Bugs](bugs.html)):

```
hit chance = Accuracy + Accuracy × (SpeedDiff / 8 + LevelDiff − CriticalBonus) / 128
```

A +32 costs a technique with accuracy 110 about 27/128 (21 points of hit chance), unless you are so far above the
enemy that it still hits every time.

### Example: the Buster techniques

The Buster techniques (Bug Buster, Wing Buster, Fish Buster, Dino Buster, Dramon Buster, Devil Buster; Hookmon,
Armormon and Cannondramon learn them) have **no effect**, so the crest counts, and against their own family they
already add +60 to the critical chance:

| Bug Buster against an Insect | Critical chance |
|---|---|
| Nothing | 4 + 60 = 64/128 = **50%** |
| Search Crest (or Monmon's Shuriken) | 4 + 32 + 60 = 96/128 = **75%** |
| Search Crest + Aura Plug | 4 + 64 + 60 = 128/128 = **100%: every hit is a critical** |

With the [family example](#example-a-technique-against-its-family)'s numbers (Bug Buster does 421 against Kunemon, about
838 with a critical), the average damage of one use, counting the hits that miss. *Advantage* is
`SpeedDiff / 8 + LevelDiff`, how far above the enemy the user is:

| Advantage | Nothing | Search Crest | Search Crest + Aura Plug |
|---|---|---|---|
| 5 (an even fight) | 89% hits → **561** | 68% → 499 | 68% → 570 |
| 20 | 99% → **625** | 78% → 574 | 78% → 655 |
| 40 | 100% → 630 | 91% → 665 | 91% → **760** |
| 60 or more | 100% → 630 | 100% → 734 | 100% → **839** |

75% or 100% criticals look huge, but in an even fight the misses cost more than the criticals give: the crest only
pays off when you clearly outclass the enemy. Then a Buster with Search Crest and Aura Plug is a guaranteed critical,
about **1.3 times** the damage without them.

## Multi-hit

`FIGHTSTG_rollMultiHit` — `src/fightstg/tech_effects.c`; the total in `src/fightstg/tech_action.c`

The damage is computed **once** (with one critical roll) and multiplied by the hits that land:

```
total = damage × hits landed          (at most 9999, European version)
```

The first hit uses the technique's accuracy; each further hit rolls accuracy again. Blade Twister (power 120,
3 hits) is like a 360-power technique when all three land.

## Healing

`FIGHTSTG_computeHeal` — `src/fightstg/damage.c`

```
HP = Power × 64 + Wisdom × Power / 8        (at most 9999, European version)
```

| Technique | Power | Heals |
|---|---|---|
| Small Heal | 8 | 512 + Wisdom × 1 |
| Mega Heal / Giga Heal | 32 | 2048 + Wisdom × 4 |
| Final Heal | 64 | 4096 + Wisdom × 8 |
| Full Heal | 127 | 8128 + Wisdom × 15.9 |

## Buffs and debuffs

`FIGHTSTG_changeBoost` — `src/fightstg/boosts.c`; the techniques' amounts in `FIGHTSTG_techBoosts` and
`FIGHTSTG_sideBoosts` — `src/fightstg/tech_action.c`

```
boost += stat × EffectStrength / 128        (− for debuffs)
```

The boost is a share of the stat without boosts. It **stacks** with each use, between **−50%** and **+100%** of the
stat. It ends after `random(0–2000) + 2000 + Spirit × 10` battle time units, and using it again resets the timer.
The Spirit is that of the Digimon the boost is on: a debuff on a high-Spirit enemy lasts longer.

**In turns:** battles run on a timeline, not fixed turns. An enemy acts about every 1000 units (707 to 1414,
from both sides' Speed: `1000 × Speed(other) / √(Speed(own) × Speed(other))`), so a buff lasts **about 2–4
turns, plus one turn per 100 Spirit**.

| Technique | Effect |
|---|---|
| Double Power / Mega Strength | Attack +37.5% (the user) / +50% (all allies) |
| Double Guard / Mega Protection | Defense +37.5% (the user) / +50% (all allies) |
| Speed Up / Mega Boost | Speed +25% (the user) / +37.5% (all allies) |
| Lovely Charm, Grisly Wing | Enemy attack −18.75%, −25% |
| Armor Break / Armor Off | Enemy defense −18.75% / −37.5% |
| Slow Down / Mega Break | Enemy speed −12.5% / −25% |

## Charge

`chargeUser` — `src/fightstg/tech_action.c`; used by `FIGHTSTG_adjustDamage` as the damage bonus

Soul Charge (80) and Misshukikou (127) add `EffectStrength / 64` to the **next physical technique's** damage:
**+125%** and **+198%**. The charge is spent by that technique.

## Drains

Drain HP techniques heal `EffectStrength / 128` of the damage dealt: Energy Leech 50%, Touch of Evil 62.5%,
Energy Drain 99%. Vampire Jewel drains 6.25% of the target's max MP.

## Status ailments

`src/fightstg/rolls.c`

```
chance (in 128ths) = EffectChance + Wisdom(user) / 8 − (Resistance(target) + Wisdom(target) / 2) / 8
```

kept between 1 and 127, rolled when the hit lands. The resistance is the target's resistance to that ailment
(PSN, PAR, CNF, SLP, KO on the enemy explorer) **plus one of its element resistances**:

| Ailment | Element resistance added |
|---|---|
| Poison | Water |
| Paralysis | Thunder |
| Confusion | Wind |
| Sleep | Ice |
| Knock-out | Dark |

### What each ailment does

| Ailment | Effect |
|---|---|
| Poison | Takes HP away about once a turn (every 1000 time units, a turn at equal speed) until cured or the battle ends, and can knock out: (technique strength / 2 − (Poison res. + Water res.) / 10)% of the max HP each time, at least 1 and at most half the max HP |
| Paralysis | Each turn it may lose it (strength − (Paralysis res. + Thunder res.) / 8 in 128ths, at least 32/128), and it can't counterattack; it wears off after a while (later with more strength); criticals against it +strength/8 |
| Confusion | A confused enemy loses its turns; a confused partner can only use the plain attack; it wears off after a while; criticals against it +strength/2 |
| Sleep | It can't act (a partner can't attack, use techniques or digivolve) until it wears off or is hit, which wakes it up; criticals against it +strength/8 |
| Knock-out | The target falls at once, whatever its HP |

### Poison techniques

Each poison technique has two numbers: the chance to poison (EffectChance) and the poison's strength
(EffectStrength), which sets how much it hurts (`FIGHTSTG_getDamage`, `src/fightstg/damage.c`):

```
HP lost each tick = (strength / 2 − (Poison res. + Water res.) / 10)% of the max HP     (at least 1, at most 50%)
```

| Technique | Kind | Power | MP | Chance to poison | Strength | vs Yanmamon (Poison 99, Water 91) |
|---|---|---|---|---|---|---|
| Poison Bites | Physical | 75 | 18 | 72/128 (56%) | 80 | 40 − 19 = **21%** a tick |
| Evil Poison | Magic | 60 | 18 | 72/128 (56%) | 80 | 21% |
| Venom Stab | Physical | 200 | 36 | 80/128 (63%) | 96 | 48 − 19 = **29%** |
| Deadly Poison | Magic | 360 | 90 | 112/128 (88%) | 127 | 63 − 19 = **44%** |
| Thorn Whipping (Rosemon only) | Physical | 340 | 130 | 127/128 (99%) | 127 | 44% |
| Venom Fang (weapon, plain attack) | Physical | – | – | 64/128 (50%) | 64 | 32 − 19 = **13%** |

A tick comes about once a turn (every 1000 time units) until the poison is cured or the battle ends; as a share of
the max HP, it is as strong against bosses. **Poisoning a target that is already poisoned replaces its poison**
(`FIGHTSTG_inflictPoison` overwrites the queued damage): a Poison Bites after a Deadly Poison brings it down from 44%
to 21%.

### Example: a poison technique vs Yanmamon

A technique with EffectChance 64, used by a Digimon with Wisdom 100, against Yanmamon (Poison 99, Water 91,
Wisdom 70):

```
64 + 100 / 8 − (99 + 91 + 70 / 2) / 8 = 64 + 12 − 28 = 48   →   48 / 128 = 37.5%
```

Every 8 points of your Wisdom add 1/128 (about 0.8%).

### Status weapons

A weapon with a status (worn in a weapon slot, not as an accessory) gives its chance to the **basic attack and the
physical techniques with no effect of their own**, when the hit lands (never to magic); a technique with its own effect uses the technique's chance instead. The
same formula, with the weapon's chance as EffectChance:

| Weapon | Ailment | Chance |
|---|---|---|
| Venom Fang | Poison | 64/128 |
| Rock Breaker | Confusion | 32/128 |
| Raven Bow | Confusion | 24/128 |
| Needle Glove | Knock-out | 24/128 |
| Soul Chopper | Knock-out | 32/128 |
| Shock Horn | Paralysis | 0/128: **never** (see [Bugs](BUGS.md#5-the-shock-horn-never-paralyzes)) |

### Battles that block them

Each battle can keep you from inflicting each ailment at all (`Encounter.blocks`, copied to `BATTLE_SETUP.blocks`),
whatever the chance. Most story battles block some: of 150, 39 block poison, 48 paralysis, 58 confusion, 53 sleep
and 62 knock-out; of the other 184 battles, only 2 to 10 block each. The first boss, Pharaohmon, cannot be
confused, put to sleep or knocked out. The same list also blocks drains, steals, lowering the enemy's
attack, defense or speed, and running away.

## Elemental fields

`boostFieldElement` — `src/fightstg/tech_action.c`; `FIGHTSTG_getElementBoost` — `src/fightstg/damage.c`

```
boost = 64 + Spirit / 10        (at most 127)
```

* Techniques of the field's element: `+boost / 128` → **+50%**, up to +99%.
* Techniques whose element the field weakens: `−boost / 256` → **−25%**, down to −50%.
* The field lasts `Spirit × 12 + 1000` battle time units: about **1 turn, plus one per 83 Spirit**.

Which element each field weakens, according to the code (`FIGHTSTG_opposedElements`):

| Field | Boosts | Weakens |
|---|---|---|
| Fire Field | Fire | Water |
| Water Field | Water | Wind |
| Ice Field | Ice | Fire |
| Wind Field | Wind | Ice |
| Thunder Field | Thunder | Dark |
| Metal Field | Machine | Thunder |
| Dark Field | Dark | Machine |

The game's own descriptions name other elements ("Strengthen fire, weaken ice"); the table above is what the code
does.

> **Bug in the original game:** `boostFieldElement` takes the field's element from the technique's *icon* field,
> which is 6 (Thunder) for all seven fields. So **every field boosts Thunder** (and weakens Dark), whatever its name
> says. The Harmony mod's *Fixed Field Moves* patch fixes it. The native PC port, built from the original code,
> keeps the bug.

## Learning digivolutions

`STFGTREP_learnDigimon` — `src/stfgtrep/growth.c`; the lists in `STFGTREP_evolutionLists`

After a battle the game walks the partner's list of 44 forms **in order** and gives it the **first** form it doesn't
have whose requirements are all met:

* up to two forms the partner must have, each at a minimum **DV level** (that form's own level), and
* one stat at a minimum: a battle stat, the partner's level, or a resistance.

The form takes the first free DV slot. The requirements of every form for every Rookie are in
[`data/evolutions.csv`](data/evolutions.csv) and on the [evolution chart](evolutions.html).

## Notes (to place)

### Where Attack and Defense come from

`FIGHTSTG_computeStats` — `src/fightstg/stats.c`

* **A partner**: its own Strength or Defense, plus what its weapons and armor add, plus the **digivolved form's**
  stats (each form adds its own, as in `DIGIMON_DATA`), plus active buffs. A stat that ends at 0 or less counts as 1.
* **An enemy**: its battle table stats × the battle's **strength / 16**. The same Digimon gets stronger in later
  areas: the first Kunemon has strength 16 (×1), later ones 35, 38 and 45 (×2.2 to ×2.8).

### Equipment that changes physical damage

| Item | Effect |
|---|---|
| Fire Power, Water Power, Ice Power, Wind Power, Bolt Power, Metal Power, Dark Power (accessories) | Give the Digimon an **element**, used by the basic attack and the shared techniques with no element (not by a Digimon's own techniques or pair techniques), with the accessory's amount as element strength |
| Multi Crest | **Three hits of 40%** each (120% if all land), each rolling accuracy. Ignored while a status or drain weapon, Saber Fang or Shuriken is worn (as are the Search Crest and the counter accessory): `fightstg/stats.c` only reads those accessories when no such weapon was found |
| Search Crest; weapons Saber Fang, Shuriken | Raise the **critical chance** by their amount (doubled for special Digimon); only for techniques with no effect |
| Weapons with a family | +50% damage and +16/128 critical chance against that family |
| Venom Fang, Shock Horn and other status weapons | The plain attack can inflict their status |
| Glasses / Dance Feather, Dance Wing | Accuracy / evasion (see [Evasion](#evasion-dance-feather-and-dance-wing)) |
| HP Proxy / HP Mega Proxy | Take **a fixed amount** off every hit received (the Harmony mod makes it 10%/20%) |

### Counterattacks

`FIGHTSTG_computeCounterDamage` — `src/fightstg/damage.c`

Counter techniques (Maul Attack, Counter Alert, Counter Strike) hit back for
`damage received × EffectStrength / 64` (`/ 32` for special Digimon), then go through the steps above.

### Enemies attacking you

Same formula, with the battle's strength already in the enemy's Attack and in its techniques' power. An enemy hits
**at least 25%** of the time (a chance of 32/128), however fast you are, and its criticals work like yours.

### Rounding and limits

Every step rounds **down** on its own, as on the PlayStation. There is **no minimum damage**: when Defense is much
higher than Attack, or a damage cut absorbs it, a hit can do 0.

### The steps after the base value

An enemy's technique is also multiplied by its strength / 16 (`BATTLE_SETUP.enemies[].strength`).

Then, in this order:

1. **Field boost**: if the technique's element is the field's, `+ base × boost / 128`. If the field weakens it,
   `− base × boost / 256` (see [Elemental fields](#elemental-fields)).
2. **Family**: `+50%` if the technique (or, for techniques with no family, one of the user's weapons) is strong
   against the target's family.
3. **Damage bonus** (a charge from Soul Charge or Misshukikou): `+ damage × bonus / 64`.
4. **Element vs resistance**: `+ damage × ElementStrength × 2 / Resistance(target, element)`. With resistance 100 and
   strength 48 that is ×1.96; with resistance 300, ×1.32. A technique without an
   element uses the user's element from an accessory, if any (not for a Digimon's own techniques or pair
   techniques).
5. **Triple hit** accessory: each hit does `× 4 / 10`.
6. **Critical hit**: `+ damage × (32 to 95) / 64`, i.e. +50% to +148%.
7. **Damage cut** of the target: subtracted, never below 0.
8. **Caps**: at most 5 × the base value, and at most 9999 (European version only).
