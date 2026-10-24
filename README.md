# Digimon World 2003 data

Evolution requirements, techniques and battle formulas of **Digimon World 2003** (PlayStation, Europe,
SLES-03936), taken directly from the game's code instead of from play-testing.

* **[Evolution chart](evolutions.html)** — every form each of the 8 Rookies can learn, what it needs, its full route,
  techniques, stats and resistances, with the game's own models.
* **[Evolution trees](trees.html)** — the same data as diagrams, one per Rookie: hover or click a form to see
  its whole route.
* **[Technique explorer](techniques.html)** — all 443 techniques: sort by any column, filter by type, element and
  effect, and see each effect's real numbers.
* **[Battle formulas](FORMULAS.md)** ([web version](formulas.html)) — damage, healing, buffs, status chances, elemental fields and how
  digivolutions are learned.
* **[Bugs](BUGS.md)** ([web version](bugs.html)) — bugs and oddities of the original game found in its code, such as
  the elemental fields that all boost Thunder.
* **[Enemies](enemies.html)** — every enemy: base stats, resistances, techniques, drop and chance, and each
  battle it is in with its level, HP and stats there. Search by name, place, drop or technique.
* **Spreadsheets** — [`data/techniques.csv`](data/techniques.csv), [`data/evolutions.csv`](data/evolutions.csv),
  [`data/enemies.csv`](data/enemies.csv) and [`data/battles.csv`](data/battles.csv).

<!-- The pages are plain HTML: once GitHub Pages is on, they live at https://USER.github.io/REPO/ -->

## Where the data comes from

The [dw3_decomp](https://github.com/ReGame-Labs/dw3_decomp) project has rewritten the game's executable as C that
compiles back to the exact same bytes. Its tables (`TECHS`, `DIGIMON_DATA`, `STFGTREP_evolutionLists`) and battle
code are the source of the numbers here. Names and descriptions come from the disc's own English text files, and the
Digimon pictures are renders of the disc's battle models made with
[dw3-model-viewer](https://github.com/Feerdus95/dw3-model-viewer).

This covers the **original** game, which is also what the
[native PC port](https://github.com/Feerdus95/dw3-native-port) runs. Mods like
[Harmony](https://github.com/rsigristc/dw2003-qol-patch) change evolutions and balance, so their numbers differ.

## Things to know

* **Stages** (Champion, Ultimate, Mega, Special Megas) follow the order of the game's internal Digimon table; the
  game itself doesn't label them.
* **Families** are named from the Buster techniques' descriptions (Insect, Dragon, Evil…); families 6 and 9
  have no technique that names them.
* A few effects of enemy-only techniques are still unidentified and show as *Other (n)*.
* **Elemental fields are bugged in the original game**: every field boosts Thunder. See
  [FORMULAS.md](FORMULAS.md#elemental-fields).

Corrections are welcome: open an issue with what you saw in the game.

## Credits

* [ReGame-Labs / dw3_decomp](https://github.com/ReGame-Labs/dw3_decomp) (MIT) — the decompilation all of this reads.
* [Feerdus95 / dw3-model-viewer](https://github.com/Feerdus95/dw3-model-viewer) — used to render the Digimon models.
* [Feerdus95 / dw3-native-port](https://github.com/Feerdus95/dw3-native-port) — the PC port.

## Legal

A non-commercial fan project with no affiliation with Bandai Namco. Digimon and Digimon World 2003 are © Bandai /
Bandai Namco, Akiyoshi Hongo and Toei Animation; the Digimon images are renders of the game's models and belong to
them. This repository contains **no game image, BIOS, or files extracted from the disc**: only facts about the game,
documentation and renders.
