# Digimon World 2003: bugs del juego original

Bugs y rarezas de la versión europea original (SLES-03936) encontrados leyendo su código descompilado
([dw3_decomp](https://github.com/ReGame-Labs/dw3_decomp)). Cada entrada dice de qué código sale. El port nativo para PC
está hecho a partir de ese mismo código, así que también los tiene, salvo que se diga lo contrario.

## Bugs confirmados

### 1. Todos los campos elementales potencian el Trueno

`boostFieldElement` — `src/fightstg/tech_action.c`

El elemento del campo se toma del campo *icon* de la técnica (`boostElement = tech->icon`), que es 6 (Trueno) en las
siete técnicas de campo, en lugar de su elemento. Así que Fire Field, Water Field, … Dark Field potencian todos el Trueno
y debilitan la Oscuridad. El mensaje sigue nombrando el elemento correcto (usa `tech->element`).

*Lo arregla:* el mod Harmony (*Fixed Field Moves*).

### 2. Un crecimiento de 5 crece menos que uno de 4 (estadísticas)

`STFGTREP_raiseStats` — `src/stfgtrep/growth.c`

En cada nivel, una estadística sube `STFGTREP_statGains[tramo][crecimiento + azar(0–4)]`. La tabla tiene 9 columnas
(0–8), así que un crecimiento de 5 puede pedir la columna 9, que no existe; la PlayStation lee en su lugar el primer valor
de la fila siguiente, que es 0 o 1. Una subida de nivel de cada cinco, la estadística que más debería crecer no sube casi
nada:

| Niveles | Ganancia media, crecimiento 4 | Ganancia media, crecimiento 5 |
|---|---|---|
| 2–4 | 11,4 | 10,0 |
| 5–19 | 8,8 | 7,6 |
| 20–39 | 5,6 | 4,8 |
| 40–59 | 4,6 | 4,0 |
| 60–79 | 3,0 | 2,6 |

A partir del nivel 80 la última fila lee más allá de toda la tabla, en las ganancias de resistencia, cuyo primer valor
también es 0.

Rookies afectados: **Kumamon** (Fuerza y Defensa), **Renamon** (Espíritu), **Monmon** (Velocidad), **Veemon** (Sabiduría
y Carisma). Justo las estadísticas en las que cada uno debía destacar crecen un 13 % más despacio que con un 4.

### 3. Un crecimiento de 5 crece menos que uno de 4 (resistencias)

`STFGTREP_raiseStats` — `src/stfgtrep/growth.c`

Hasta el nivel 40, cada resistencia sube `STFGTREP_resistGains[crecimiento + azar(0–3)]`, una tabla de 8 valores
(`0 0 0 1 1 1 2 2`). Un crecimiento de 5 puede pedir el valor 8, fuera de la tabla; en el disco la palabra siguiente es 0.
Un crecimiento de 5 da de media **1,25** por nivel, uno de 4 **1,5**.

Siete de los ocho Rookies tienen una resistencia con crecimiento 5: Kotemon (Agua), Monmon (Máquina), Agumon (Fuego),
Veemon (Hielo), Guilmon (Viento), Renamon (Trueno), Patamon (Oscuridad). Kumamon no tiene ninguna.

### 4. La magia sin elemento lee la Velocidad del objetivo como resistencia

`FIGHTSTG_rollMagicCritical` — `src/fightstg/rolls.c`

La probabilidad de crítico mágico es `mín(64, FuerzaElemento × 100 / resist[elemento − 2]) + 4`. En una técnica sin
elemento, `elemento − 2` es −1, y el valor que se lee es el que está guardado justo antes de las resistencias: la
**Velocidad** del objetivo. Así que la probabilidad de crítico de la magia sin elemento depende de la Velocidad del
objetivo:

* La mayoría de estas técnicas (las de estados alterados y absorción) tienen una fuerza de elemento de 1: `100 / Velocidad`
  da más o menos 0–2, un 3–5 % de críticos.
* **Gate of Destiny** (64), **KongouKaimandara** (72) y **Back Storm** (72) tienen una fuerza de elemento alta pero ningún
  elemento: contra un objetivo con una Velocidad de hasta unos 100 llegan al máximo del 53 % (`64 × 100 / 100 = 64`), y al
  28 % con Velocidad 200.

### 5. El Cuerno Impac. (Shock Horn) nunca paraliza

`FIGHTSTG_applyTechEffect` — `src/fightstg/tech_effects.c`; `WEAPON_DATA` — `src/main/game/items.c`

«Cuerno que paraliza el cuerpo», pero su ficha tiene una probabilidad de estado de **0** y una fuerza de 0. El ataque
normal solo prueba el estado de un arma cuando esa probabilidad no es 0
(`if (stats[0].statuses[HIT_PARALYSIS].chance) FIGHTSTG_tryParalysis()`), así que el Cuerno Impac. nunca paraliza: es un
cuerno de Fuerza +90 y nada más. Las otras armas de estado tienen 24–64/128.

## Rarezas

Comportamientos que parecen no intencionados, o al menos sorprendentes, pero que pueden ser a propósito.

* **Las debilitaciones duran según el Espíritu del objetivo, no el de quien las usa.** `FIGHTSTG_queueBoostEnd` calcula la
  duración de una mejora con el Espíritu del Digimon que la lleva. En una mejora es quien la usa; en una debilitación
  (Armor Break, AncientGreymon…) es el enemigo, así que las debilitaciones duran más en enemigos con mucho Espíritu.
* **Los elementos que debilita cada campo no coinciden con las descripciones.** «Fire Field: fortalece el fuego, debilita
  el hielo», pero `FIGHTSTG_opposedElements` hace que el Fire Field debilite el Agua (el Ice Field debilita el Fuego, y así
  con los demás). Ver [Fórmulas](formulas.html#campos-elementales).
* **Un campo nunca potencia el elemento de un accesorio.** `FIGHTSTG_getElementBoost` solo mira el elemento de la técnica,
  así que un ataque básico con Pot. Fuego (Fire Power) no gana nada con un Fire Field.
* **Un objeto de críticos baja la precisión.** En `FIGHTSTG_rollHit` a la diferencia de nivel del jugador se le resta el
  bonus de crítico, así que Buscar Vale (Search Crest), Colmi. Sable (Saber Fang) o Shuriken cuestan algo de precisión.
* **Las técnicas de varios golpes tiran un solo crítico para todos**: o todos los golpes son críticos, o ninguno.
* **No hay daño mínimo**: un golpe puede hacer 0.
* **Dos Digimon reparten más EXP de digievolución que uno.** `STFGTREP_getDigimonExp` da a las formas usadas en un combate
  el 100 % de la DV EXP si se usó una, **el 60 % a cada una si son dos** (un 120 % en total) y un reparto a partes iguales
  si son tres o más. Usar exactamente dos formas por combate es lo que más DV EXP da.
* **Huir cuesta al principio**: el primer intento tiene una probabilidad base de 8/128 (6 %), el segundo 16/128, y así
  sucesivamente, más las diferencias de nivel y de Velocidad (`FIGHTSTG_testRunAway`).

## Arreglados en la versión europea

* **Medidores de campo** (`src/fieldstg/data/fieldstg.c`): la versión de EE. UU. no tiene la fila 8 en `FIELDSTG_gaugeRows`
  y lee fuera de la tabla (un puntero nulo) cuando se acaba `GAME.randomGauges`. La versión europea añade la fila.

* **Un enemigo dormido podía huir** (EE. UU.). El `FIGHTSTG_testRunAway` europeo lo impide.
* **Mostr. Vales (Counter Crest) contraatacaba todos los golpes** (EE. UU.). La versión europea tira
  `25 % + 50 % × daño / PV máximos` (`FIGHTSTG_rollCounter`) antes del contraataque.

## Latentes

* **Casilla de mejora 3** (`FIGHTSTG_boostStats`): una cuarta mejora se calcularía a partir del Ataque, se sumaría al
  Espíritu de un enemigo y no se tendría en cuenta en los compañeros. Ninguna técnica la usa.

## Mencionados en otros sitios

* **Picking Claw y Snapping Claw cuando el enemigo queda KO**: el mod Harmony lo da como arreglado; su causa no se ha
  rastreado aquí.
