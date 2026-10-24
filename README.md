# Digimon World 2003 data

Evolution requirements, techniques and battle formulas of **Digimon World 2003** (PlayStation, Europe,
SLES-03936), taken directly from the game's code instead of from play-testing.

* **[Evolution chart](evolutions.html)** — Every form each Rookie can reach through training.
* **[Evolution trees](trees.html)** — One diagram per Rookie showing how its evolutions connect. Hover over or click a form to see the full path to it.
* **[Technique explorer](techniques.html)** — all 443 techniques: sort by any column, filter by type, element and
  effect, and see each effect's real numbers.
* **[Battle formulas](FORMULAS.md)** ([web version](formulas.html)) — damage, healing, buffs, status chances, elemental fields and how
  digivolutions are learned. WIP.
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
code are the source of the numbers here.

## Things to know

* **Stages** (Champion, Ultimate, Mega, Special Megas) follow the order of the game's internal Digimon table; the
  game itself doesn't label them.
* **Families** are named from the Buster techniques' descriptions (Insect, Dragon, Evil…); families 6 and 9
  have no technique that names them; from their Digimon they are **Beast**(Apemon, Mammothmon, Persiamon...) and Plant (Woodmon, Vegiemon, Cherrymon...)
* **Elemental fields are bugged in the original game**: every field boosts Thunder. See
  [FORMULAS.md](FORMULAS.md#elemental-fields).

## Credits

* [ReGame-Labs / dw3_decomp](https://github.com/ReGame-Labs/dw3_decomp) (MIT) — the decompiled code all the formulas and data come from.
* [Feerdus95 / dw3-model-viewer](https://github.com/Feerdus95/dw3-model-viewer) — the Digimon renders.
* [Feerdus95 / dw3-native-port](https://github.com/Feerdus95/dw3-native-port) — the native PC port of the same game.
* Pages and scripts written with the help of Claude (Anthropic).


## Legal

A non-commercial fan project with no affiliation with Bandai Namco. Digimon and Digimon World 2003 are © Bandai /
Bandai Namco, Akiyoshi Hongo and Toei Animation; the Digimon images are renders of the game's models and belong to
them.
