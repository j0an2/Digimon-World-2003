# Digimon World 2003: fórmulas de combate

Cómo calcula la versión europea original (SLES-03936) el daño, la curación, las mejoras y los campos, leído de su
código C descompilado ([dw3_decomp](https://github.com/ReGame-Labs/dw3_decomp)). Cada sección dice la función y el
archivo de los que sale, para que lo puedas comprobar tú.

La división entera redondea siempre hacia abajo, como en la PlayStation. Muchos valores se guardan en 128avos o
64avos: un valor de 64 en 128avos es un 50 %.

## Qué hace cada estadística

Dónde lee el código de combate cada estadística:

| Estadística | Qué hace |
|---|---|
| **Fuerza** (Ataque) | El daño físico que haces |
| **Defensa** | Reduce el daño físico que recibes, y solo eso |
| **Espíritu** | El daño mágico que haces **y** que recibes · la fuerza de los campos elementales que pones · lo que duran tus mejoras (y las debilitaciones que te ponen) · lo que duran los estados alterados que provocas |
| **Sabiduría** | Acierto y esquiva de la magia · provocar estados alterados y resistirlos · los PV que cura tu curación · resistir que el enemigo se suba estadísticas |
| **Velocidad** | Acierto y esquiva físicos · cada cuánto actúa un Digimon · huir |
| **Carisma** | No se usa en combate: es un requisito de algunas digievoluciones |

Defensa, Espíritu y Sabiduría son tres defensas distintas: la Defensa contra el daño físico, el Espíritu contra el
daño mágico, y la Sabiduría contra que te acierte la magia y contra los estados alterados.

Qué hace la Sabiduría, en detalle:

| | La tuya (al atacar) | La del objetivo (al defender) |
|---|---|---|
| Acierto de la magia | + Sabiduría / 8 | − Sabiduría / 8 |
| Provocar un estado alterado | + Sabiduría / 8 | − Sabiduría / 16 |
| Curación | + Sabiduría × Potencia / 8 PV | – |

## Daño físico

`FIGHTSTG_computeDamage`, `FIGHTSTG_adjustDamage` — `src/fightstg/damage.c`

```
base = Potencia × Ataque(usuario) / Defensa(objetivo)
```

### Ejemplo: Kotemon de nivel 5 contra el primer Kunemon

El ataque básico de Kotemon (técnica 1: potencia 60, precisión 110, sin elemento) contra el Kunemon de las primeras
zonas (nivel 1, 120 PV, fuerza 16/16, de `src/fieldstg/data/fieldstg.c`). Kotemon no lleva equipo.

**El Ataque de Kotemon a nivel 5.** Una partida nueva lo empieza con 48 de Fuerza (`DIGIMON_DATA`, `initNewGameData` en
`src/main/game/party.c`). Cada nivel suma `STFGTREP_statGains[tramo][crecimiento + azar(0–4)]`
(`STFGTREP_raiseStats`, `src/stfgtrep/growth.c`): con el crecimiento de Fuerza 4 de Kotemon, de +8 a +14 en los niveles
2–4 y de +6 a +11 en el nivel 5. Así que a nivel 5 su Ataque es **de 78 a 101, 91 de media**.

**La Defensa de Kunemon.** 42 en la tabla de enemigos, × fuerza 16 / 16 = **42**.

```
daño = Potencia × Ataque(usuario) / Defensa(objetivo)
     = 60 × 91 / 42 = 130
```

| Ataque de Kotemon | Daño |
|---|---|
| 78 (subidas de nivel con mala suerte) | 111 |
| 91 (media) | 130 |
| 101 (subidas de nivel con buena suerte) | 144 |

No se aplica ninguno de los otros pasos: ni campo, ni familia, ni carga, ni elemento.

**Probabilidad de acierto**: `110 + 110 × (DifVelocidad / 8 + DifNivel) / 128` en 128avos (`FIGHTSTG_rollHit`). Con la
Velocidad de Kotemon en torno a 63 contra 48 y 4 niveles más: `110 + 110 × (1 + 4) / 128 = 114` → **89 %**.

**Crítico**: 4/128 = **3,1 %**, de ×1,5 a ×2,5.

Kunemon tiene 120 PV, así que el ataque básico de un Kotemon de nivel 5 **lo mata de un golpe el 96 % de las veces**
que acierta.

### Ejemplo: una técnica física con elemento

El mismo Kotemon de nivel 5 (Ataque 91) y Kunemon (Defensa 42), ahora con la técnica propia de Kotemon **Hot Head**
(técnica 53: potencia 75, Fuego, fuerza de elemento 48, precisión 127, 20 PM). La resistencia al Fuego de Kunemon es 90.

```
base = Potencia × Ataque(usuario) / Defensa(objetivo)
daño = base + base × 2 × FuerzaElemento / Resistencia(objetivo, elemento)
     = base × (1 + 2 × FuerzaElemento / Resistencia)

base = 75 × 91 / 42                = 162
daño = 162 × (1 + 2 × 48 / 90)     = 334
```

| | Ataque básico | Hot Head |
|---|---|---|
| Base | 130 | 162 |
| Elemento | – | ×2,07 |
| **Daño** | **130** | **334** |
| Probabilidad de acierto | 89 % (precisión 110) | 100 % (precisión 127, ver [Probabilidad de acierto](#probabilidad-de-acierto)) |

El multiplicador del elemento es `1 + 2 × FuerzaElemento / Resistencia`, así que con los 48 de Hot Head:

| Resistencia al Fuego del objetivo | Multiplicador | Daño a este Kunemon |
|---|---|---|
| 50 | ×2,92 | 473 |
| 90 (Kunemon) | ×2,07 | 334 |
| 160 | ×1,60 | 259 |
| 300 | ×1,32 | 213 |

El elemento de una técnica física solo suma daño; una resistencia alta simplemente suma menos. (La magia es distinta:
una resistencia alta la puede dejar en la mitad).

### Ejemplo: un Fire Field

El mismo Kotemon (Ataque 91) y Kunemon (Defensa 42, resistencia al Fuego y al Agua 90) con un **Fire Field** puesto por
un Digimon con 80 de Espíritu. El campo actúa **antes** que el elemento, sobre el valor base
(`FIGHTSTG_getElementBoost`, ver [Campos elementales](#campos-elementales)):

```
MejoraCampo = 64 + Espíritu(de quien puso el campo) / 10          (como máximo 127)

técnica del elemento del campo:   mejorado  = base × (1 + MejoraCampo / 128)
técnica que el campo debilita:    debilitado = base × (1 − MejoraCampo / 256)
técnica sin elemento:             sin cambios
después el paso del elemento:     daño      = mejorado × (1 + 2 × FuerzaElemento / Resistencia)
```

```
MejoraCampo = 64 + 80 / 10 = 72
base        = 75 × 91 / 42  = 162          (la potencia de Hot Head; la base del ataque básico es 130)

Ataque de Fuego (Hot Head):   162 × (1 + 72 / 128) = 253  →  253 × (1 + 2 × 48 / 90) = 522
Ataque de Agua (debilitado):  162 × (1 − 72 / 256) = 117  →  117 × (1 + 2 × 48 / 90) = 241
Ataque básico:                sin elemento, así que el campo no hace nada  →  130
```

Con más Espíritu el campo es más fuerte, pero el código limita MejoraCampo a 127 (`if (element >= 0x80) element = 0x7F`
en `boostFieldElement`). Se llega con 630 de Espíritu; por encima, hasta el máximo de 999, el campo ya no mejora
(64 + 999 / 10 serían 163):

| Espíritu de quien puso el campo | MejoraCampo | Ataque de Fuego (Hot Head) | Ataque de Agua (debilitado) |
|---|---|---|---|
| 80 | 72 | 162 × (1 + 72/128) = 253 → **522** (+56 %) | 162 × (1 − 72/256) = 117 → **241** (−28 %) |
| 200 | 84 | 162 × (1 + 84/128) = 268 → **553** (+66 %) | 162 × (1 − 84/256) = 109 → **225** (−33 %) |
| 600 | 124 | 162 × (1 + 124/128) = 318 → **657** (+97 %) | 162 × (1 − 124/256) = 84 → **173** (−48 %) |

Los porcentajes se comparan con 334, el daño sin campo. El ataque básico se queda en 130 en los tres casos.

El juego redondea cada parte hacia abajo por separado (162 − 45 = 117, cuando 162 × 0,719 daría 116), así que un
resultado puede diferir en 1 del de la fórmula en una línea.

| Ataque | Sin campo | Fire Field | |
|---|---|---|---|
| Ataque básico (sin elemento) | 130 | 130 | igual |
| Fuego (Hot Head) | 334 | **522** | +56 % |
| Agua (un Hot Head de Agua, para comparar) | 334 | **241** | −28 % |

Estos son los números de un campo que funciona como debería (con el *Fixed Field Moves* de Harmony). En el juego
original y en el port de PC el Fire Field potencia el Trueno y debilita la Oscuridad, así que contra Kunemon los tres se
quedan como sin campo.

### Ejemplo: una técnica contra su familia

**Bug Buster** (técnica 131: potencia 130, precisión 110, 24 PM, sin elemento, familia Insecto; la aprenden Hookmon y
Armormon) contra el mismo Kunemon, que **es un Insecto**. El mismo Ataque 91, para comparar.

```
base    = Potencia × Ataque(usuario) / Defensa(objetivo)
familia = base × (1 + 1/2)                 cuando la familia de la técnica es la del objetivo

base    = 130 × 91 / 42  = 281
familia = 281 × 1,5      = 421
```

Coincidir con la familia también suma **60/128 a la probabilidad de crítico** (`FIGHTSTG_rollCritical`), cosa que la
descripción del juego no dice: `(4 + 60) / 128 = 50 %` en lugar del 3,1 %.

| | Hot Head | Bug Buster |
|---|---|---|
| Daño sin crítico | 334 | **421** |
| Probabilidad de crítico | 3,1 % | **50 %** |
| Daño con crítico | 501–829 | **631–1045** |
| **Daño medio** | ≈ 344 | **≈ 630** |
| Probabilidad de acierto | 100 % | 89 % |

Contra su familia, un Buster hace +50 % y saca un crítico uno de cada dos golpes: de media, casi el doble que una
técnica con elemento que da en una debilidad. Ninguna técnica tiene a la vez familia y elemento: las 13 técnicas de
familia no tienen elemento.

Las familias, con los nombres de las descripciones de las técnicas Buster: 2 Dinosaurio, 3 Maligno, 4 Fantasma,
5 Máquina, 7 Ave, 8 Insecto, 10 Acuático, 11 Dragón.

### Ejemplo: accesorios de elemento

Los accesorios Pot. Fuego (Fire Power), Pot. Agua (Water Power), Pot. Hielo (Ice Power), Pot. Viento (Wind Power),
Pot. Rayo (Bolt Power), Pot. Metal (Metal Power) y Pot. Oscura (Dark Power) dan un elemento al Digimon
(`FIGHTSTG_computeStats`). Cada uno tiene tres fuerzas:

| Accesorio | Fuerza de elemento |
|---|---|
| Pot. … (… Power.) | 8 |
| Pot. 2 … (… Power 2) | 16 |
| Pot. S … (… Power S) | 24 |

**Solo se puede llevar uno**: los 21 son un mismo grupo de accesorios, y la pantalla de equipo solo deja llevar uno del
grupo a la vez (`EQUIP_KIND_GROUP_ACCESSORY`, `src/stitshop/equip.c`). Así que no se pueden llevar dos de Fuego, ni uno
de Fuego y otro de Agua.

El combate usa el elemento del accesorio **para el ataque básico y las técnicas compartidas sin elemento** (Bug Buster,
Big Shot, Hammer Rush…). **No** lo usa para las **técnicas propias** de un Digimon (Vee Head Butt, Bear Fist, Terra
Force…) ni para las **técnicas en pareja**, aunque no tengan elemento. Una técnica con elemento usa el suyo y no hace
caso del accesorio. El campo elemental solo mira el elemento **de la técnica**, así que nunca potencia el de un accesorio.

```
daño = base × (1 + 2 × FuerzaAccesorio / Resistencia(objetivo, elemento del accesorio))

ataque básico, Pot. S Fuego:  130 × (1 + 2 × 24 / 90) = 199
Bug Buster,    Pot. S Fuego:  421 × (1 + 2 × 24 / 90) = 645      (421 = 281 × 1,5, el bonus de familia)
```

El mismo Kotemon (Ataque 91) y Kunemon (Defensa 42, resistencia al Fuego 90):

| | Sin accesorio | Pot. Fuego (8) | Pot. 2 Fuego (16) | Pot. S Fuego (24) |
|---|---|---|---|---|
| Ataque básico | 130 | 153 | 176 | **199** |
| Hot Head (técnica de Fuego) | 334 | 334 | 334 | 334 — no hace caso del accesorio |
| Ataque básico + Fire Field (mejora 72) | 130 | 153 | 176 | 199 — el campo no hace nada |
| Hot Head + Fire Field | 522 | 522 | 522 | 522 |
| Bug Buster (sin elemento, contra un Insecto) | 421 | 495 | 570 | **645** |

Los accesorios son flojos al lado del elemento propio de una técnica (Hot Head 48, Burn Slash 64), pero se suman a las
técnicas Buster, que no tienen elemento: Bug Buster con Pot. S Fuego hace +53 % además de su bonus de familia. Contra
un enemigo concreto, el accesorio que conviene llevar es el del elemento que menos resiste.

## Probabilidad de acierto

`FIGHTSTG_rollHit` — `src/fightstg/rolls.c`

```
probabilidad (en 128avos) = Precisión + Precisión × (DifVelocidad / 8 + DifNivel) / 128
DifVelocidad = Velocidad(usuario) + precisión de los accesorios − Velocidad(objetivo)
DifNivel     = Nivel(usuario) − Nivel(objetivo)
```

El juego saca un número del 0 al 127 y el golpe acierta si sale **por debajo** de la probabilidad, así que con 128 o más
nunca falla. Cada técnica tiene su propia Precisión: un Kotemon de nivel 5 (Velocidad 63, 4 niveles por encima) contra
Kunemon (Velocidad 48) tiene DifVelocidad = 15, que cuenta como 15 / 8 = 1, y DifNivel = 4:

| Técnica | Precisión | Probabilidad | |
|---|---|---|---|
| Ataque básico | 110 | 110 + 110 × (1 + 4) / 128 = 110 + 4 = 114 | 89 % |
| Hot Head | 127 | 127 + 127 × (1 + 4) / 128 = 127 + 4 = 131 | nunca falla |

La probabilidad de acierto de un enemigo usa la misma fórmula menos la esquiva del objetivo, y nunca baja de 32 (25 %).

## Daño mágico

`FIGHTSTG_computeMagicDamage` — `src/fightstg/damage.c`

```
poder = Potencia × (Espíritu(usuario) × 50 / Espíritu(objetivo) + 50) / 100
```

El multiplicador es `0,5 + 0,5 × Espíritu(usuario) / Espíritu(objetivo)`: ×1 con el mismo Espíritu, ×1,5 con el doble del
objetivo, ×2 (el máximo) a partir del triple; ×0,75 con la mitad, ×0,625 con una cuarta parte, nunca por debajo de ×0,5.
El daño físico no tiene ese límite: el doble de Ataque es el doble de daño.

**La Sabiduría no reduce el daño mágico**: el **Espíritu** del objetivo hace el papel que la Defensa hace en los ataques
físicos. La Sabiduría decide si la magia **acierta** (`FIGHTSTG_rollMagicHit`):

```
probabilidad (en 128avos) = Precisión + Precisión × (DifSabiduría / 8 + DifNivel) / 128
DifSabiduría = Sabiduría(usuario) − Sabiduría(objetivo)
```

y también cuenta para los estados alterados y la curación.

Después: la mejora del campo, y la resistencia del objetivo al elemento:

| Resistencia | Multiplicador de daño |
|---|---|
| menos de 100 | (400 − 3 × res) % — hasta ×4 |
| de 100 a 299 | (125 − res / 4) % |
| 300 o más | (65 − res / 20) % |

La *fuerza de elemento* de una técnica mágica no cambia su daño: solo lo cambia el elemento contra esta tabla. Lo que
sube es su **probabilidad de crítico** (ver más abajo).

Después, +50 % contra la familia del objetivo, el crítico (de +50 % a +150 %), la reducción de daño, como máximo
5 × `poder` y como máximo 9999 (versión europea).

### Ejemplo: Diamond Storm de un Renamon de nivel 5 contra el primer Kunemon

**Diamond Storm** (técnica 59: mágica, potencia 100, Hielo, fuerza de elemento 32, precisión 127, 30 PM), la técnica
propia de Renamon desde el principio, contra el mismo Kunemon (nivel 1, Espíritu 40, Sabiduría 50, resistencia al Hielo 90).

**El Espíritu de Renamon a nivel 5.** Empieza en 61 con un crecimiento de Espíritu 5. La tabla de ganancias solo tiene las
columnas 0–8, así que `crecimiento + azar(0–4)` puede leer la columna 9, que en la PlayStation es el primer valor de la
fila siguiente: de vez en cuando, una ganancia mínima. A nivel 5 su Espíritu es **de 64 a 114, 99 de media**.

```
poder = Potencia × (Espíritu(usuario) × 50 / Espíritu(objetivo) + 50) / 100       (entre Potencia/2 y Potencia × 2)
daño  = poder × (400 − 3 × Resistencia) / 100                                     (resistencia de menos de 100)

poder = 100 × (99 × 50 / 40 + 50) / 100 = 100 × (123 + 50) / 100 = 173
daño  = 173 × (400 − 3 × 90) / 100      = 173 × 1,30              = 224
```

| Espíritu de Renamon | poder | Daño |
|---|---|---|
| 64 (subidas de nivel con mala suerte) | 130 | 169 |
| 99 (media) | 173 | 224 |
| 114 (subidas de nivel con buena suerte) | 192 | 249 |

**Probabilidad de acierto**: `127 + 127 × (DifSabiduría / 8 + DifNivel) / 128`. La Sabiduría de Renamon ronda 92 contra 50,
así que `127 + 127 × (5 + 4) / 128 = 135`: nunca falla.

**Crítico.** Un crítico mágico depende del elemento (`FIGHTSTG_rollMagicCritical`):

```
valor        = FuerzaElemento × 100 / Resistencia(objetivo, elemento)      (como máximo 64)
probabilidad = valor + 4                                                    (en 128avos)

Diamond Storm (fuerza 32) contra Kunemon (Hielo 90): valor = 32 × 100 / 90 = 35  →  39 / 128 = 30 %
Flame Ball    (fuerza 16) contra Kunemon (Fuego 90): valor = 16 × 100 / 90 = 17  →  21 / 128 = 16 %
```

Las dos tienen potencia 100 y hacen los mismos 224 a Kunemon, porque la fuerza de elemento de una técnica mágica no
cambia su daño; Diamond Storm, con el doble de fuerza, saca casi el doble de críticos. Un crítico suma de +50 % a +150 %.

La probabilidad de crítico de Diamond Storm según la resistencia al Hielo del objetivo:

| Resistencia al Hielo | Valor | Probabilidad de crítico |
|---|---|---|
| 50 | 64 (el máximo) | 68/128 = 53 % (lo máximo sin coincidir en familia) |
| 90 (Kunemon) | 35 | 30 % |
| 100 | 32 | 28 % |
| 200 | 16 | 16 % |
| 300 | 10 | 11 % |

Así que la magia de un elemento al que el enemigo es débil hace más daño **y** más críticos.

**Un objetivo con un estado alterado recibe más críticos**, tanto físicos como mágicos, según la fuerza de la técnica que
se lo provocó:

| Objetivo | Probabilidad de crítico extra (en 128avos) |
|---|---|
| Paralizado o dormido | + fuerza / 8 |
| Confundido | + fuerza / 2 |

| Fuerza de la técnica de estado | Parálisis o sueño | Confusión |
|---|---|---|
| 80 (Spinal Tap, Stun Shock, Hypno Gas, Panic Bites, Confuse Gas) | +10 (+8 %) | +40 (+31 %) |
| 96 (Brain Freeze, Paralyze Shock, Hypno Nebula) | +12 (+9 %) | +48 (+38 %) |
| 104 (T-Sword) | +13 (+10 %) | – |
| 127 (Mugenhadou, Confuse Stab, Confuse Nebula) | +15 (+12 %) | +63 (+49 %) |

Diamond Storm contra un Kunemon confundido con Confuse Nebula: `39 + 63 = 102` → **80 % de críticos**.

Kunemon tiene 120 PV: Diamond Storm siempre lo mata. Para comparar, el ataque básico de Renamon (Ataque en torno a 61) hace
`60 × 61 / 42 = 87`.

Cómo cambia Diamond Storm (poder 173) según la resistencia del objetivo:

| Resistencia al Hielo | Multiplicador | Daño |
|---|---|---|
| 50 | ×2,50 | 432 |
| 90 (Kunemon) | ×1,30 | 224 |
| 100 | ×1,00 | 173 |
| 160 | ×0,85 | 147 |
| 200 | ×0,75 | 129 |
| 300 | ×0,50 | 86 |

Al contrario que el elemento de una técnica física, a la magia una resistencia alta la puede dejar **a la mitad**, y una
baja la puede más que doblar.

## Críticos

`FIGHTSTG_rollCritical` — `src/fightstg/rolls.c`

Un golpe físico es crítico con esta probabilidad, en 128avos:

```
probabilidad = 4                                    (3,1 %)
             + FuerzaEfecto                         si la técnica tiene el efecto Más críticos
             + el bonus de crítico del equipo       (×2 para los Digimon especiales) si la técnica no tiene ningún efecto
             + 60 si la técnica es fuerte contra la familia del objetivo (+16 por la familia de un arma, en su lugar)
             + más mientras el objetivo está paralizado, dormido o confundido
```

| Técnica | Probabilidad de crítico |
|---|---|
| Big Shot | 44/128 = 34 % |
| Swing Swing, Captain Cannon, Maltcyclone | 64/128 = 50 % |
| Pinpoint Shot | 68/128 = 53 % |

Un crítico suma `daño × (32 a 95) / 64`: **de +50 % a +148 %**, más o menos el doble de media.

Un crítico **mágico** (`FIGHTSTG_rollMagicCritical`) tiene su propia probabilidad, según la fuerza de elemento de la
técnica contra la resistencia del objetivo:

```
probabilidad (en 128avos) = mín(64, FuerzaElemento × 100 / Resistencia(objetivo, elemento)) + 4
```

(+60 más contra la familia de la técnica), y suma de +50 % a +150 %.

## Varios golpes

`FIGHTSTG_rollMultiHit` — `src/fightstg/tech_effects.c`; el total en `src/fightstg/tech_action.c`

El daño se calcula **una vez** (con una sola tirada de crítico) y se multiplica por los golpes que aciertan:

```
total = daño × golpes acertados          (como máximo 9999, versión europea)
```

El primer golpe usa la precisión de la técnica; cada golpe más vuelve a tirar el acierto. Blade Twister (potencia 120,
3 golpes) es como una técnica de 360 de potencia cuando aciertan los tres.

## Curación

`FIGHTSTG_computeHeal` — `src/fightstg/damage.c`

```
PV = Potencia × 64 + Sabiduría × Potencia / 8        (como máximo 9999, versión europea)
```

| Técnica | Potencia | Cura |
|---|---|---|
| Small Heal | 8 | 512 + Sabiduría × 1 |
| Mega Heal / Giga Heal | 32 | 2048 + Sabiduría × 4 |
| Final Heal | 64 | 4096 + Sabiduría × 8 |
| Full Heal | 127 | 8128 + Sabiduría × 15,9 |

## Mejoras y debilitaciones

`FIGHTSTG_changeBoost` — `src/fightstg/boosts.c`; los valores de las técnicas en `FIGHTSTG_techBoosts` y
`FIGHTSTG_sideBoosts` — `src/fightstg/tech_action.c`

```
mejora += estadística × FuerzaEfecto / 128        (− en las debilitaciones)
```

La mejora es una parte de la estadística sin mejoras. **Se acumula** con cada uso, entre **−50 %** y **+100 %** de la
estadística. Termina al cabo de `azar(0–2000) + 2000 + Espíritu × 10` unidades de tiempo de combate, y volver a usarla
reinicia el contador. El Espíritu es el del Digimon que la lleva: una debilitación sobre un enemigo con mucho Espíritu
dura más.

**En turnos:** los combates van por una línea de tiempo, no por turnos fijos. Un enemigo actúa más o menos cada 1000
unidades (de 707 a 1414, según la Velocidad de los dos: `1000 × Velocidad(otro) / √(Velocidad(propia) × Velocidad(otro))`),
así que una mejora dura **unos 2–4 turnos, más un turno por cada 100 de Espíritu**.

| Técnica | Efecto |
|---|---|
| Double Power / Mega Strength | Ataque +37,5 % (quien la usa) / +50 % (todos los aliados) |
| Double Guard / Mega Protection | Defensa +37,5 % (quien la usa) / +50 % (todos los aliados) |
| Speed Up / Mega Boost | Velocidad +25 % (quien la usa) / +37,5 % (todos los aliados) |
| Lovely Charm, Grisly Wing | Ataque del enemigo −18,75 %, −25 % |
| Armor Break / Armor Off | Defensa del enemigo −18,75 % / −37,5 % |
| Slow Down / Mega Break | Velocidad del enemigo −12,5 % / −25 % |

## Carga

`chargeUser` — `src/fightstg/tech_action.c`; la usa `FIGHTSTG_adjustDamage` como bonus de daño

Soul Charge (80) y Misshukikou (127) suman `FuerzaEfecto / 64` al daño de la **siguiente técnica física**: **+125 %** y
**+198 %**. Esa técnica gasta la carga.

## Absorción

Las técnicas de absorber PV curan `FuerzaEfecto / 128` del daño hecho: Energy Leech 50 %, Touch of Evil 62,5 %,
Energy Drain 99 %. Vampire Jewel absorbe el 6,25 % de los PM máximos del objetivo.

## Estados alterados

`src/fightstg/rolls.c`

```
probabilidad (en 128avos) = ProbEfecto + Sabiduría(usuario) / 8 − (Resistencia(objetivo) + Sabiduría(objetivo) / 2) / 8
```

entre 1 y 127, y se tira cuando el golpe acierta. La resistencia es la del objetivo a ese estado (PSN, PAR, CNF, SLP, KO
en el explorador de enemigos) **más una de sus resistencias elementales**:

| Estado | Resistencia elemental que se suma |
|---|---|
| Veneno | Agua |
| Parálisis | Trueno |
| Confusión | Viento |
| Sueño | Hielo |
| KO | Oscuridad |

### Qué hace cada estado

| Estado | Efecto |
|---|---|
| Veneno | Quita PV más o menos una vez por turno (cada 1000 unidades de tiempo, un turno a igual velocidad) hasta que se cura o acaba el combate, y puede dejar KO: (fuerza de la técnica / 2 − (res. Veneno + res. Agua) / 10) % de los PV máximos cada vez, como mínimo 1 y como máximo la mitad de los PV máximos |
| Parálisis | Cada turno puede perderlo (fuerza − (res. Parálisis + res. Trueno) / 8 en 128avos, como mínimo 32/128) y no puede contraatacar; se pasa al cabo de un rato (más tarde con más fuerza); críticos contra él +fuerza/8 |
| Confusión | Un enemigo confundido pierde sus turnos; un compañero confundido solo puede usar el ataque básico; se pasa al cabo de un rato; críticos contra él +fuerza/2 |
| Sueño | No puede actuar (un compañero no puede atacar, usar técnicas ni digievolucionar) hasta que se le pasa o recibe un golpe, que lo despierta; críticos contra él +fuerza/8 |
| KO | El objetivo cae al momento, tenga los PV que tenga |

### Técnicas de veneno

Cada técnica de veneno tiene dos números: la probabilidad de envenenar (ProbEfecto) y la fuerza del veneno
(FuerzaEfecto), que decide cuánto daño hace (`FIGHTSTG_getDamage`, `src/fightstg/damage.c`):

```
PV que quita cada vez = (fuerza / 2 − (res. Veneno + res. Agua) / 10) % de los PV máximos     (mínimo 1, máximo 50 %)
```

| Técnica | Tipo | Potencia | PM | Prob. de envenenar | Fuerza | Contra Yanmamon (Veneno 99, Agua 91) |
|---|---|---|---|---|---|---|
| Poison Bites | Físico | 75 | 18 | 72/128 (56 %) | 80 | 40 − 19 = **21 %** cada vez |
| Evil Poison | Mágico | 60 | 18 | 72/128 (56 %) | 80 | 21 % |
| Venom Stab | Físico | 200 | 36 | 80/128 (63 %) | 96 | 48 − 19 = **29 %** |
| Deadly Poison | Mágico | 360 | 90 | 112/128 (88 %) | 127 | 63 − 19 = **44 %** |
| Thorn Whipping (solo Rosemon) | Físico | 340 | 130 | 127/128 (99 %) | 127 | 44 % |
| Colmi. Veneno (Venom Fang) (arma, ataque básico) | Físico | – | – | 64/128 (50 %) | 64 | 32 − 19 = **13 %** |

Quita PV más o menos una vez por turno (cada 1000 unidades de tiempo) hasta que se cura el veneno o acaba el combate; como
es una parte de los PV máximos, es igual de fuerte contra los jefes. **Envenenar a un objetivo que ya está envenenado
sustituye su veneno** (`FIGHTSTG_inflictPoison` sobrescribe el daño en cola): un Poison Bites después de un Deadly Poison lo
baja del 44 % al 21 %.

### Ejemplo: una técnica de veneno contra Yanmamon

Una técnica con ProbEfecto 64, usada por un Digimon con 100 de Sabiduría, contra Yanmamon (Veneno 99, Agua 91,
Sabiduría 70):

```
64 + 100 / 8 − (99 + 91 + 70 / 2) / 8 = 64 + 12 − 28 = 48   →   48 / 128 = 37,5 %
```

Cada 8 puntos de tu Sabiduría suman 1/128 (más o menos 0,8 %).

### Armas de estado

Un arma con un estado (en una ranura de arma, no como accesorio) da su probabilidad al **ataque básico y a las técnicas
físicas sin efecto propio**, cuando el golpe acierta (nunca a la magia); una técnica con efecto propio usa la probabilidad
de la técnica. La misma fórmula, con la probabilidad del arma como ProbEfecto:

| Arma | Estado | Probabilidad |
|---|---|---|
| Colmi. Veneno (Venom Fang) | Veneno | 64/128 |
| Rompepiedras (Rock Breaker) | Confusión | 32/128 |
| Arco Cuervo (Raven Bow) | Confusión | 24/128 |
| Guante Agujas (Needle Glove) | KO | 24/128 |
| Cort. Alma (Soul Chopper) | KO | 32/128 |
| Cuerno Impac. (Shock Horn) | Parálisis | 0/128 (solo cuenta tu Sabiduría) |

### Combates que los bloquean

Cada combate puede impedir del todo que provoques cada estado (`Encounter.blocks`, que se copia a `BATTLE_SETUP.blocks`),
sea cual sea la probabilidad. La mayoría de los combates de la historia bloquean alguno: de 150, 39 bloquean el veneno,
48 la parálisis, 58 la confusión, 53 el sueño y 62 el KO; de los otros 184 combates, solo entre 2 y 10 bloquean cada uno.
Al primer jefe, Pharaohmon, no se le puede confundir, dormir ni dejar KO. La misma lista bloquea también las absorciones,
los robos, bajar el ataque, la defensa o la velocidad del enemigo, y huir.

## Campos elementales

`boostFieldElement` — `src/fightstg/tech_action.c`; `FIGHTSTG_getElementBoost` — `src/fightstg/damage.c`

```
mejora = 64 + Espíritu / 10        (como máximo 127)
```

* Técnicas del elemento del campo: `+mejora / 128` → **+50 %**, hasta +99 %.
* Técnicas cuyo elemento debilita el campo: `−mejora / 256` → **−25 %**, hasta −50 %.
* El campo dura `Espíritu × 12 + 1000` unidades de tiempo de combate: más o menos **1 turno, más uno por cada 83 de Espíritu**.

Qué elemento debilita cada campo, según el código (`FIGHTSTG_opposedElements`):

| Campo | Potencia | Debilita |
|---|---|---|
| Fire Field | Fuego | Agua |
| Water Field | Agua | Viento |
| Ice Field | Hielo | Fuego |
| Wind Field | Viento | Hielo |
| Thunder Field | Trueno | Oscuridad |
| Metal Field | Máquina | Trueno |
| Dark Field | Oscuridad | Máquina |

Las descripciones del propio juego nombran otros elementos («Fortalece el fuego, debilita el hielo»); la tabla de arriba es
lo que hace el código.

> **Bug del juego original:** `boostFieldElement` toma el elemento del campo del campo *icon* de la técnica, que es 6
> (Trueno) en los siete campos. Así que **todos los campos potencian el Trueno** (y debilitan la Oscuridad), diga lo que
> diga su nombre. El parche *Fixed Field Moves* del mod Harmony lo arregla. El port nativo para PC, hecho a partir del
> código original, mantiene el bug.

## Aprender digievoluciones

`STFGTREP_learnDigimon` — `src/stfgtrep/growth.c`; las listas en `STFGTREP_evolutionLists`

Después de un combate, el juego recorre **en orden** la lista de 44 formas del compañero y le da la **primera** forma que no
tiene y cuyos requisitos se cumplen todos:

* hasta dos formas que el compañero debe tener, cada una con un **nivel DV** mínimo (el nivel de esa forma), y
* una estadística con un mínimo: una estadística de combate, el nivel del compañero o una resistencia.

La forma ocupa la primera casilla DV libre. Los requisitos de cada forma para cada Rookie están en
[`data/evolutions.csv`](../data/evolutions.csv) y en la [tabla de digievoluciones](evolutions.html).

## Notas (por ordenar)

### De dónde salen el Ataque y la Defensa

`FIGHTSTG_computeStats` — `src/fightstg/stats.c`

* **Un compañero**: su Fuerza o Defensa, más lo que suman sus armas y armaduras, más las estadísticas de la **forma
  digievolucionada** (cada forma suma las suyas, como en `DIGIMON_DATA`), más las mejoras activas. Una estadística que
  queda en 0 o menos cuenta como 1.
* **Un enemigo**: las estadísticas de su tabla de combate × la **fuerza / 16** del combate. El mismo Digimon se hace más
  fuerte en zonas posteriores: el primer Kunemon tiene fuerza 16 (×1), los siguientes 35, 38 y 45 (de ×2,2 a ×2,8).

### Equipo que cambia el daño físico

| Objeto | Efecto |
|---|---|
| Pot. Fuego (Fire Power), Pot. Agua (Water Power), Pot. Hielo (Ice Power), Pot. Viento (Wind Power), Pot. Rayo (Bolt Power), Pot. Metal (Metal Power), Pot. Oscura (Dark Power) (accesorios) | Dan al Digimon un **elemento**, que usan el ataque básico y las técnicas compartidas sin elemento (no las técnicas propias de un Digimon ni las técnicas en pareja), con el valor del accesorio como fuerza de elemento |
| Multivale (Multi Crest) | **Tres golpes del 40 %** cada uno (120 % si aciertan todos), cada uno tira su acierto. No funciona si se lleva un arma de estado o de absorción, el Colmi. Sable o el Shuriken (igual que el Buscar Vale y el accesorio de contraataque): `fightstg/stats.c` solo lee esos accesorios si no encuentra ninguna de esas armas |
| Buscar Vale (Search Crest); armas Colmi. Sable (Saber Fang), Shuriken | Suben la **probabilidad de crítico** en su valor (el doble en los Digimon especiales); solo en técnicas sin efecto |
| Armas con familia | +50 % de daño y +16/128 de probabilidad de crítico contra esa familia |
| Colmi. Veneno (Venom Fang), Cuerno Impac. (Shock Horn) y las otras armas de estado | El ataque básico puede provocar su estado |
| Gafas (Glasses) / Danza Pluma (Dance Feather) | Precisión / esquiva |
| Proxy PV (HP Proxy) / Mega Proxy PV (HP Mega Proxy) | Quitan **una cantidad fija** de cada golpe recibido (el mod Harmony lo cambia a un 10 %/20 %) |

### Contraataques

`FIGHTSTG_computeCounterDamage` — `src/fightstg/damage.c`

Las técnicas de contraataque (Maul Attack, Counter Alert, Counter Strike) devuelven
`daño recibido × FuerzaEfecto / 64` (`/ 32` en los Digimon especiales) y después pasan por los pasos de arriba.

### Los enemigos atacándote

La misma fórmula, con la fuerza del combate ya incluida en el Ataque del enemigo y en la potencia de sus técnicas. Un
enemigo acierta **como mínimo el 25 %** de las veces (una probabilidad de 32/128), por rápido que seas, y sus críticos
funcionan como los tuyos.

### Redondeo y límites

Cada paso redondea **hacia abajo** por separado, como en la PlayStation. **No hay daño mínimo**: cuando la Defensa es
mucho mayor que el Ataque, o una reducción de daño lo absorbe, un golpe puede hacer 0.

### Los pasos después del valor base

La técnica de un enemigo también se multiplica por su fuerza / 16 (`BATTLE_SETUP.enemies[].strength`).

Después, en este orden:

1. **Mejora del campo**: si el elemento de la técnica es el del campo, `+ base × mejora / 128`. Si el campo lo debilita,
   `− base × mejora / 256` (ver [Campos elementales](#campos-elementales)).
2. **Familia**: `+50 %` si la técnica (o, en las técnicas sin familia, una de las armas del usuario) es fuerte contra la
   familia del objetivo.
3. **Bonus de daño** (una carga de Soul Charge o Misshukikou): `+ daño × bonus / 64`.
4. **Elemento contra resistencia**: `+ daño × FuerzaElemento × 2 / Resistencia(objetivo, elemento)`. Con resistencia 100 y
   fuerza 48 eso es ×1,96; con resistencia 300, ×1,32. Una técnica sin elemento usa el elemento del usuario que dé un
   accesorio, si lo hay (no en las técnicas propias de un Digimon ni en las técnicas en pareja).
5. Accesorio de **triple golpe**: cada golpe hace `× 4 / 10`.
6. **Crítico**: `+ daño × (32 a 95) / 64`, es decir, de +50 % a +148 %.
7. **Reducción de daño** del objetivo: se resta, nunca por debajo de 0.
8. **Límites**: como máximo 5 × el valor base, y como máximo 9999 (solo la versión europea).
