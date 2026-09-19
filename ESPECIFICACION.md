# Clave Maestra — especificación completa del producto

> Documento de diseño dirigido por especificación (*spec-driven design*).
> Objetivo: que otra IA (o un equipo) pueda reconstruir esta aplicación desde cero y obtener un producto prácticamente idéntico.
> Idioma del producto: **español de España**. Nomenclatura musical **latina** (Do, Re, Mi…).
> Versión del documento: 1.4 · Corresponde a la versión 15 de la app (piano MIDI, dos barras de tempo, modo horizontal y voz).

---

## 1. Resumen ejecutivo

**Clave Maestra** es una aplicación web (una sola página, sin servidor ni cuentas) para aprender a **leer música y tocar el piano** desde cero, en español, con estética y mecánicas de videojuego.

- **Plataforma:** web estática. Pensada para iPhone en vertical (se añade a la pantalla de inicio como app), funcional también en iPad y escritorio.
- **Sin backend:** todo el estado vive en `localStorage`, más un sistema de **contraseñas de progreso** tipo videojuego clásico y una copia en archivo JSON.
- **Dos campañas:** *Solfeo* (nombrar notas) y *Piano* (tocarlas en un teclado en pantalla).
- **Modos de respuesta:** botones de nota, teclado de piano en pantalla y **voz** (reconocedor propio entrenado con la voz del usuario; sin servicios externos).
- **Contenido complementario:** arcade con varios juegos, biblioteca de partituras de dominio público y un grimorio de teoría consultable.

### Principios de producto

1. **Piano primero.** Se enseñan las dos claves (sol y fa) *a la vez*: dominar solo la clave de sol deja coja la mano izquierda.
2. **Notas faro.** No se memoriza todo el pentagrama: se aprenden anclas (Do central, Sol de la clave de sol, Fa de la clave de fa) y se cuenta desde ellas.
3. **Progresión medida.** Cada capítulo añade 2–4 notas y 0–2 figuras rítmicas nuevas, y las ejercita en varios formatos antes del jefe.
4. **El usuario manda.** Cualquier nivel se puede saltar; los saltos se registran en el perfil (no se ocultan ni se castigan).
5. **Diversión explícita.** Rangos de rol, narrador con humor, frases célebres, logros, combos y mensajes burlones al fallar.
6. **Honestidad legal.** Solo música de dominio público. Nada de piezas con derechos vigentes.

---

## 2. Usuario y contexto

- **Usuario primario:** adulto principiante absoluto en lectura musical que quiere aprender piano por su cuenta, en sesiones cortas de móvil.
- **Contexto de uso:** iPhone en la mano, a veces con el piano delante, a veces sin instrumento (en el metro).
- **Requisitos derivados:**
  - Sesiones de 1–3 minutos por nivel.
  - Botones grandes, alcanzables con el pulgar; nada crítico en la zona superior.
  - Debe funcionar sin conexión y sin cuenta.
  - El progreso no se puede perder por limpiar el navegador (de ahí la contraseña).

---

## 3. Modelo musical (núcleo del dominio)

### 3.1 Representación de notas

- Una nota es un **índice diatónico** `d` = `octava_cientifica * 7 + indice_letra`, con letras `C D E F G A B` → `0..6`, más una alteración `acc ∈ {-1, 0, +1}`.
  - Ejemplo: `C4` (Do central) → `d = 4*7 + 0 = 28`.
- MIDI: `midi = 12*(floor(d/7)+1) + [0,2,4,5,7,9,11][d%7] + acc`.
- Frecuencia: `440 * 2^((midi-69)/12)`.
- **Nomenclatura mostrada:** española, con octava = octava científica − 1. El **Do central es `Do3`** (sistema franco-belga). En la teoría se explica la equivalencia con el inglés (C4).

### 3.2 Geometría del pentagrama (renderizado SVG)

- Separación entre líneas `S = 12` unidades de viewBox.
- Primera línea (la de abajo) de cada clave:
  - Clave de sol: `Mi4` científico → `d = 30`.
  - Clave de fa: `Sol2` científico → `d = 18`.
- Posición vertical: `pos = d - primera_linea`; `y = y0 - pos*S/2` (pos par = línea, impar = espacio).
- **Líneas adicionales:** para `pos ≤ -2`, líneas en cada `pos` par descendente; para `pos ≥ 10`, ascendente. Ancho ±11 (±13 para la redonda).
- **Claves tipográficas:** se usa la fuente *Noto Music* subconjunto (glifos `𝄞 𝄢 ♯ ♭ ♮`) incrustada como `@font-face` en base64 (≈1,7 KB) para no depender de CDN.
  - Tamaño: `font-size = 4*S` (1 em = 4 espacios).
  - **Línea base del glifo = primera línea del pentagrama.**
  - **Corrección crítica:** la clave de fa de esa fuente tiene los dos puntos centrados 0,42 espacios por debajo de donde deben abrazar la 4ª línea. Se dibuja en `y0 - 0.42*S`. (Sin esa corrección la clave de fa aparece mal colocada; fue un bug real.)

### 3.3 Figuras rítmicas

Catálogo (`FIG_SPECS`), cada una con nº de notas `n`, cabeza, plica, corchetes, puntillo y barras:

| Clave | Figura | Duración |
|---|---|---|
| `redonda` | redonda | 4 pulsos |
| `blanca` / `blancaP` | blanca / con puntillo | 2 / 3 |
| `negra` / `negraP` | negra / con puntillo | 1 / 1,5 |
| `corchea` / `corcheaP` | corchea / con puntillo | ½ / ¾ |
| `semicorchea` | semicorchea | ¼ |
| `dosCorcheas` | 2 corcheas con barra | 1 |
| `dosSemis` | 2 semicorcheas | ½ |
| `cuatroSemis` | 4 semicorcheas | 1 |
| `corcheaPSemi` | corchea con puntillo + semicorchea | 1 |
| `corcheaDosSemis`, `dosSemisCorchea` | grupos mixtos | 1 |

**Silencios** (`REST_FIGS`), tratados como «figuras» de un capítulo aunque no se dibujen con cabeza ni plica:

| Clave | Silencio | Duración | Colocación |
|---|---|---|---|
| `sNegra` | de negra | 1 pulso | centrado en la 3ª línea |
| `sBlanca` | de blanca | 2 pulsos | apoyado **sobre** la 3ª línea |
| `sRedonda` | de redonda | 4 pulsos | colgando **de** la 4ª línea |

Glifos Unicode `U+1D13B`…`U+1D13E` de la fuente musical embebida. Como el origen tipográfico coincide con la **línea inferior** del pentagrama (líneas en 0, 250, 500, 750 y 1000 unidades de em con `font-size = 4·S`), los desplazamientos verticales son: redonda `y0 − 0,952·S`, blanca `y0 − 1,024·S`, negra/corchea/semicorchea `y0`. Los silencios con puntillo se dibujan con el glifo de la mitad de duración más un círculo.

`figsUpTo(cap)` devuelve las figuras acumuladas **sin** los silencios (se usa para sortear figuras en los niveles de lectura); los silencios solo aparecen en las canciones y en la teoría.

**Tamaño del pentagrama (`viewBox` adaptativo).** Ningún pentagrama tiene medidas fijas: el `viewBox` se calcula para que el dibujo llene el hueco en las dos direcciones.

- *Alto*: se recorta al rango real de notas del nivel o de la pieza. Como las plicas apuntan hacia el centro, basta con el rango más una plica arriba y otra abajo: `top = max(hi, lo+7) + 1,6`, `bot = min(lo, hi-7) − 1,6` (en posiciones de pentagrama, donde 0 es la 1ª línea y 8 la 5ª). En un nivel típico eso baja el alto de 156 a ~90 unidades.
- *Ancho*: `W = alto × (ancho/alto del hueco medido en el DOM)`, con un mínimo. Así el `viewBox` tiene la misma forma que la caja y la escala no desperdicia ni ancho ni alto.
- Las canciones de una sola clave dibujan **un solo pentagrama** en vez del gran pentagrama: ocupa el doble.
- Se vuelve a medir después de `show(...)`, después de ocultar la introducción (el escenario crece) y al girar el móvil, porque un elemento oculto mide cero y el dibujo saldría diminuto.

Efecto medido con el mismo contenido: en móvil horizontal la altura del pentagrama pasa de 48 px a 98–132 px.

Reglas de dibujo:
- Plica arriba si la posición media del grupo está por debajo de la 3ª línea (`pos < 4`), abajo en caso contrario. Longitud 3,3–3,5 espacios.
- Barras: recta entre los extremos de las plicas, pendiente limitada a ±0,8 espacios, desplazada para que ninguna plica quede más corta de 2,7 espacios; grosor 5 unidades; barras secundarias a 8 unidades hacia dentro; medias barras (*stubs*) de 11 unidades cuando el nivel de barra cubre una sola nota.
- Puntillo: círculo r=2 a la derecha de la cabeza; si la nota está en línea, en el espacio superior.
- Alteración: glifo a la izquierda de la cabeza (`y + 6` para ♯, `y + 2,5` para ♭).

### 3.4 Formato de melodías

Cadena de tokens `clave:nota:pulsos` separados por espacios:

```
s:E4:1 s:D4:1 s:C4:1 s:D4:1 s:E4:1.5 s:D4:0.5 f:C3:2
```

`s` = clave de sol (pentagrama superior), `f` = clave de fa (inferior). El tiempo de inicio de cada nota es la suma acumulada de los pulsos anteriores. La figura dibujada se deduce de los pulsos (`{4:redonda, 3:blancaP, 2:blanca, 1.5:negraP, 1:negra, 0.5:corchea, 0.25:semicorchea}`).

El token `r:pulsos` es un **silencio**: hereda la clave de la nota anterior, se dibuja con su glifo, **no se responde** y no cuenta como fallo; el motor avanza el cursor por encima de él (`skipRests()`).

Cada melodía lleva además:
- `beats`: numerador del compás (4 o 3). Se dibuja el signo de compás (`beats/4`) al principio de los dos pentagramas y una **barra de compás** cada `beats` pulsos, más una **doble barra** al final.
- `up` (opcional): pulsos de **anacrusa**, para que las barras caigan donde deben cuando la pieza empieza en parte débil.

**Invariante de tempo:** ninguna canción de jefe baja de la corchea, porque hay que **nombrarla en voz alta a tempo**: a 60–74 ppm una corchea da 400–500 ms, ya en el límite del reconocedor. Las semicorcheas y los grupos mixtos se practican solo en los niveles de lectura, donde no hay reloj.

#### Las dos barras

El pentagrama que corre lleva **dos líneas verticales**, no una:

| Línea | Color | Qué significa |
|---|---|---|
| **A compás** | verde, continua | Donde cae la nota en el pulso. Es el objetivo. |
| **Tarde** | roja, discontinua, por detrás | Si la nota la cruza, se acabó (o se pierde una vida). |

Entre las dos hay una **franja ámbar**: el margen real. Su anchura es `grace = clamp(48/bpm, 0,5, 1) pulsos`, constante durante toda la pieza, y la cabeza de la nota en juego pasa de azul a ámbar al entrar en ella. Con una sola línea el jugador leía «¡YA!» y creía que ya llegaba tarde; el margen existía desde el principio, pero no se veía.

La geometría se deriva del ancho: `PH = max(124, 0,34·W)` para la línea de tempo, `PXB = max(46, (W − PH)/7)` para que se vean unos siete pulsos por delante, y la línea de fallo en `PH − grace·PXB`.

---

## 4. Contenido

### 4.1 Campaña «Solfeo» — 11 capítulos, 58 niveles

Cada capítulo define: `zone` (nombre del lugar en el mapa), `rank` (rango que otorga), `add` (notas nuevas), `figs` (figuras y silencios nuevos), `story` (texto del narrador), `quote` (frase célebre con autor), `song` (canción del jefe: nombre, origen, bpm, compás, notas).

| # | Zona | Rango | Notas nuevas | Figuras nuevas | Jefe | Niveles |
|---|---|---|---|---|---|---|
| 1 | La Aldea del Do Central | Aprendiz | Do3 (sol), Re3 (sol), Mi3 (sol) | negra, blanca, silencio de negra | Mary tenía un corderito · 4/4 · 60 ppm · 26 notas | 4 |
| 2 | El Bosque de la Clave de Fa | Escudero | Do3 (fa), Si2 (fa), La2 (fa) | redonda, silencio de blanca | Bollitos calientes · 4/4 · 62 ppm · 13 notas | 4 |
| 3 | La Torre del Sol | Caballero | Fa3 (sol), Sol3 (sol) | negra con puntillo, corchea | Himno de la alegría · 4/4 · 66 ppm · 30 notas | 4 |
| 4 | Las Mazmorras del Fa | Paladín | Sol2 (fa), Fa2 (fa) | dos corcheas | Al claro de la luna · 4/4 · 66 ppm · 23 notas | 5 |
| 5 | El Valle Grave | Campeón | Mi2 (fa), Re2 (fa), Do2 (fa) | blanca con puntillo | Martinillo (mano izquierda) · 4/4 · 64 ppm · 19 notas | 5 |
| 6 | El Puente de las Dos Manos | Guardián | La3 (sol) | — | El puente de Londres · 4/4 · 66 ppm · 28 notas | 5 |
| 7 | Las Almenas de la Clave de Sol | Maestro de armas | Si3 (sol), Do4 (sol) | — | Canción de cuna · 3/4 · 66 ppm · 36 notas | 6 |
| 8 | Las Cuevas del Bajo | Hechicero | Si1 (fa), La1 (fa), Sol1 (fa) | semicorchea, dos semicorcheas, cuatro semicorcheas | En la granja de mi tío · 4/4 · 70 ppm · 18 notas | 6 |
| 9 | El Pico Nevado | Archimago | Re4 (sol), Mi4 (sol), Fa4 (sol), Sol4 (sol) | corchea con puntillo, corchea con puntillo + semicorchea | Cascabeles · 4/4 · 74 ppm · 37 notas | 6 |
| 10 | El Abismo Grave | Leyenda | Fa1 (fa), Mi1 (fa), Re1 (fa), Do1 (fa) | corchea + dos semicorcheas, dos semicorcheas + corchea | La Escalera del Abismo · 4/4 · 72 ppm · 18 notas | 6 |
| 11 | El Trono de las Dos Claves | Señor del Pentagrama | La4 (sol) | — | La Oda de las dos manos · 4/4 · 72 ppm · 36 notas | 7 |

**Orden de conceptos (el porqué de esta secuencia):**

1. Se empieza por el **Do central y sus dos vecinos** en clave de sol: tres notas, una sola mano, canción entera desde el minuto uno.
2. El capítulo 2 salta **enseguida** a la clave de fa por el mismo Do central, que es el puente natural entre las dos claves. Retrasarla haría que la mano izquierda se percibiera como «lo difícil».
3. A partir de ahí se **alternan** las claves (3 sol · 4 fa · 5 fa · 6 ambas · 7 sol · 8 fa · 9 sol · 10 fa · 11 sol), en vez de agotar una y luego la otra.
4. Cada clave se completa primero en su **posición de cinco dedos** (capítulos 3–5) y solo después se extiende hacia los extremos.
5. Las **notas faro** se presentan justo cuando aparece su clave: Sol3 en el capítulo 3, Fa2 en el 4.
6. El capítulo 6 no añade casi notas: su contenido real es el **cambio de pentagrama** (mirar la clave antes de leer), que es el error más común.
7. Las novedades rítmicas van de lo lento a lo rápido y **cada una se estrena en la canción del capítulo**: negra/blanca → redonda → puntillo y corchea → dos corcheas → blanca con puntillo → compás de 3/4. Las figuras rápidas (semicorchea y grupos mixtos) llegan al final y se practican solo leyendo.
8. Los **silencios** entran con la primera y la segunda figura (negra y blanca), porque son parte del compás desde el principio.

**Reglas de contenido (verificadas por un test automático sobre los datos, `audit`):**

- Cada canción usa **solo notas, figuras y silencios ya aprendidos** en ese capítulo o anteriores.
- Cada nota nueva del capítulo suena en su canción **al menos 2 veces** (3 si la canción pasa de 20 notas).
- Cada **figura cantable** nueva (redonda, blanca, blanca con puntillo, negra, negra con puntillo, corchea, dos corcheas) y cada **silencio** nuevo aparecen en la canción de ese capítulo.
- Los **pulsos totales** de cada canción son múltiplo del compás (descontando la anacrusa).
- El **salto máximo** entre notas consecutivas de la misma clave está limitado por capítulo: 2 grados (caps. 1–2), 3 (3–4), 4 (5–7), 6 (8–10), 7 (11).
- Las frases célebres de atribución dudosa se marcan con «atribuida a».
- El narrador es **Maese Clavius, mago del pentagrama**: explica con humor y da un truco memotécnico por capítulo.

**Composición de cada capítulo** (`lectura → entreno → retos → jefe`), fijada en tabla para que la variedad crezca sin repetir el mismo reto seguido:

```
1: lectura → entreno → quiz → jefe
2: lectura → entreno → teclado → jefe
3: lectura → entreno → tempo → jefe
4: lectura → entreno → claves → quiz → jefe
5: lectura → entreno → teclado → crono → jefe
6: lectura → entreno → claves → tempo → jefe
7: lectura → entreno → teclado → quiz → crono → jefe
8: lectura → entreno → claves → tempo → teclado → jefe
9: lectura → entreno → crono → quiz → tempo → jefe
10: lectura → entreno → teclado → claves → crono → jefe
11: lectura → entreno → tempo → quiz → claves → teclado → jefe
```

### 4.2 Campaña «Piano» — 8 capítulos, 36 niveles

Mismo esqueleto, pero la lección enseña **dónde cae la tecla y con qué dedo**, con un diagrama de teclado con números de dedo (1–5) coloreados por mano (naranja derecha, azul izquierda). Se responde siempre **pulsando la tecla exacta** (la octava cuenta).

| # | Zona | Rango | Notas nuevas | Figuras nuevas | Recital | Niveles |
|---|---|---|---|---|---|---|
| 1 | La Posición de Do | Manos nuevas | Do3 (sol), Re3 (sol), Mi3 (sol) | negra, blanca | Mary tenía un corderito · 4/4 · 60 ppm · 13 notas | 4 |
| 2 | La Mano Izquierda | Dos manos | Do3 (fa), Si2 (fa), La2 (fa) | redonda | Bollitos calientes · 4/4 · 60 ppm · 14 notas | 4 |
| 3 | El Puño Completo | Cinco dedos | Fa3 (sol), Sol3 (sol) | negra con puntillo, corchea | Himno de la alegría · 4/4 · 64 ppm · 15 notas | 4 |
| 4 | El Sótano | Bajo firme | Sol2 (fa), Fa2 (fa) | dos corcheas | Al claro de la luna · 4/4 · 64 ppm · 12 notas | 4 |
| 5 | El Diálogo | Manos juntas | La3 (sol), Mi2 (fa), Re2 (fa), Do2 (fa) | — | Martinillo · 4/4 · 52 ppm · 35 notas | 5 |
| 6 | La Octava | Registro amplio | Si3 (sol), Do4 (sol) | blanca con puntillo | Canción de cuna · 3/4 · 64 ppm · 36 notas | 5 |
| 7 | El Bosque Negro | Teclas negras | Fa♯3 (sol) | — | Cumpleaños feliz (en Sol) · 3/4 · 64 ppm · 22 notas | 5 |
| 8 | El Recital | Concertista | Re4 (sol), Mi4 (sol) | semicorchea, dos semicorcheas | La Oda de las dos manos · 4/4 · 62 ppm · 35 notas | 5 |

Orden: posición de Do de la derecha → misma posición con la izquierda → completar los cinco dedos de cada mano → las dos manos dialogando → cambio de posición (pasar del meñique) → primera tecla negra (Fa♯, con una pieza en Sol mayor que la necesita de verdad) → recital a dos manos. Las alteraciones se limitan a **un solo sostenido**: dos conceptos nuevos en el mismo capítulo era demasiado.

```
1: pLectura → pTeclas → pTempo → pJefe
2: pLectura → pTeclas → pTempo → pJefe
3: pLectura → pTeclas → pTempo → pJefe
4: pLectura → pTeclas → pTempo → pJefe
5: pLectura → pTeclas → pQuiz → pTempo → pJefe
6: pLectura → pTeclas → pQuiz → pTempo → pJefe
7: pLectura → pTeclas → pQuiz → pTempo → pJefe
8: pLectura → pTeclas → pQuiz → pTempo → pJefe
```

### 4.3 Tipos de nivel (nodos del mapa)

| Tipo | Campaña | Descripción | Aprobado | XP |
|---|---|---|---|---|
| `lectura` / `pLectura` | ambas | Pergamino con historia, notas/figuras nuevas y enlaces a teoría | leerlo | 20 |
| `entreno` | solfeo | 15 notas al azar con sorteo ponderado (§4.3.1) | ≥80 % | 40 |
| `teclado` / `pTeclas` | ambas | Ves la nota, pulsas la tecla exacta en el piano | ≥80 % | 50 |
| `claves` | solfeo | 16 notas alternando clave (70 % de cambio) | ≥80 % | 50 |
| `crono` | solfeo | 45 s, objetivo `round(9 + cap*1,3)` aciertos; fallo −2 s | objetivo | 50 |
| `tempo` / `pTempo` | ambas | Notas a tempo (46+4·cap ppm), 3 vidas | sobrevivir | 60 |
| `quiz` / `pQuiz` | ambas | 6 preguntas tipo test con explicación | ≥4/6 | 40 |
| `jefe` / `pJefe` | ambas | Canción completa a tempo, sin fallos | perfecto | 150 |

Composición por capítulo (solfeo): `lectura → entreno → [1-4 retos rotando] → jefe`, es decir **4 niveles** en los capítulos iniciales y hasta **7** en el último (58 en total). El criterio: al principio el jugador tiene que llegar pronto a una canción completa (si no, abandona); al final ya aguanta sesiones largas y agradece la variedad. La rotación está fijada en tabla (§4.1) para que ningún reto se repita dos capítulos seguidos.

**Exigencia del reconocedor por tipo de nivel** (`LEVEL_MARGIN`, no es una opción del usuario): entreno/teclado/quiz 1,20 · cambio de clave 1,16 · partituras 1,10 · contrarreloj y arcade 1,08 · a compás y jefes 1,06. Cuanto más alto, más seguro tiene que estar el reconocedor para aceptar la respuesta; se afloja donde el reloj aprieta.

**Estrellas:** 1★ aprobar, 2★ ≥90 %, 3★ ≥95 % y ≤2,5 s por nota (en contrarreloj y a compás, por objetivos/fallos).

### 4.3.1 Sorteo ponderado de notas

En **todos** los niveles de notas sueltas el sorteo no es uniforme:

- Cada nota del capítulo actual entra **3 veces** en la bolsa (las nuevas salen el triple).
- Cada nota suma entre 0 y 3 copias extra según lo que el usuario **esté fallando últimamente**: `peso = min(3, redondeo(fallos_recientes·1,6 + tasa_de_fallo·2))`, donde `fallos_recientes` es una media exponencial por nota (factor 0,82 por respuesta) y `tasa_de_fallo` solo cuenta si la nota se ha visto ≥3 veces.

Así el juego insiste en lo nuevo y en lo que se resiste, sin dejar de repasar el resto.

### 4.4 Arcade

- **Invasores del pentagrama** (modo estrella): las notas avanzan desde la derecha hacia una línea de defensa; se destruyen nombrándolas (o tocándolas en el teclado). 3 vidas, combo que multiplica puntos cada 5 aciertos, oleada más rápida cada 8 destruidas (velocidad ×1,14; intervalo ×0,87, mínimo 0,85 s). Récord guardado.
  - **Selector de rango:** claves (ambas/sol/fa), presets (Aprendido, Pentagrama, Con 1 línea adicional, Todo) y rango a medida con dos desplegables por clave; vista previa en el gran pentagrama y recuento de notas.
  - **Repasar mis notas difíciles:** interruptor que mantiene todas las notas del rango pero multiplica la aparición de las que más fallas (mismo peso de §4.3.1, aplicado ×2). El pie del selector dice en qué notas va a insistir.
- **Más juegos** (se desbloquean al vencer jefes 3, 5, 7 y 9): Contrarreloj libre (60 s), Cambio de clave, Sostenidos y bemoles, Líneas adicionales.

### 4.5 Partituras

Biblioteca de 10 piezas **de dominio público**, con dificultad 1–5, que se abren según el capítulo alcanzado: Estrellita, Cumpleaños feliz, La cucaracha, Himno de la alegría, Cascabeles, Marcha fúnebre (Chopin), Greensleeves, En la cueva del rey de la montaña (Grieg), Para Elisa, Marcha turca (Mozart).

Cada pieza ofrece: **Escuchar** (suena con los nombres), **Practicar** (la línea espera), **Solfear a tempo** y **Tocar en el teclado**. No se pierde: se cuentan fallos (0 → 3★, ≤2 → 2★, ≤5 → 1★) y se guarda el mejor resultado.

> **Restricción legal explícita:** no se incluyen obras con derechos vigentes (por ejemplo *La Pantera Rosa*, 1963, o *Tequila*, 1958). Si el usuario las pide, se explican los derechos y se ofrecen equivalentes de dominio público con carácter parecido.

### 4.6 Teoría («Grimorio»)

21 lecciones con diagramas SVG generados por código (pentagrama anatómico, teclado, gran pentagrama, tablas de figuras y de silencios, y un **pentagrama de ejemplo con signo de compás y barras** generado desde una melodía). Se agrupan en tres estados según el capítulo alcanzado: **Ya deberías saberlo**, **Lo siguiente**, **Más adelante** (plegado), y se ordenan por capítulo. Consultables sin haber jugado. Categorías: Lectura, Figuras y ritmo, Alteraciones.

Orden de las lecciones por capítulo:

| Cap. | Lecciones |
|---|---|
| 1 | El pentagrama · Nombres de las notas y octavas · Negra y blanca · Pulso y tempo · **El compás y las barras** · **Los silencios** · Líneas adicionales y el Do central |
| 2 | La redonda · La clave de fa en 4ª · El gran pentagrama |
| 3 | La clave de sol · El puntillo · La corchea |
| 4 | Leer con notas faro · Barras: corcheas unidas · **Segundas y terceras: pasos y saltos** |
| 7 | Sostenidos, bemoles y becuadros · **El vals: compás de 3/4** |
| 8 | La semicorchea |
| 9 | Ritmos combinados |
| 10 | Dos líneas adicionales |

(En negrita, las cuatro lecciones añadidas en la revisión pedagógica.)

Banco de preguntas: **56 para solfeo** + 12 para piano, cada una con 1 respuesta correcta, 3 distractores y una explicación. Cada pregunta está etiquetada con el capítulo a partir del cual es justa; el examen saca 4 preguntas de los dos últimos capítulos y 2 de cualquier capítulo anterior.

---

## 5. Sistemas de juego

### 5.1 Mapa

- **Mapa de fantasía** en pergamino, desplazamiento vertical continuo, con cartela de título, brújula, y una **región por capítulo**.
- Cada región tiene: color de tierra propio, decoración procedural (árboles, pinos, casas, rocas, montañas, nubes…) con semilla fija por capítulo, un **lugar emblemático** dibujado (aldea, torre, mazmorra, río, castillo, cueva, pico nevado, abismo, islas flotantes, trono) y un **borde de bioma** (costa, acantilado, bosque, sima, nieve, nubes, muralla).
- Un **camino** (curva suave que pasa por los nodos, con guiones) enlaza los niveles; los nodos son medallones con icono y color por tipo, estrellas obtenidas y etiqueta.
- Estados de nodo: `done`, `skipped` (borde discontinuo + etiqueta), `open` (medallón con «Estás aquí» y pulso), `locked` (candado).
- Las regiones no alcanzadas se cubren con **niebla de guerra** («Tierra inexplorada»).
- Selector de campaña (Solfeo / Piano) encima del mapa.

### 5.2 Progresión, saltos y rangos

- Un nodo se desbloquea cuando el anterior de su campaña está superado **o saltado**.
- **Saltar:** cualquier nodo abierto se puede saltar; un nodo bloqueado ofrece «Saltar hasta aquí» (marca como saltados todos los anteriores pendientes).
- Los saltos se listan en el perfil con botón para jugarlos.
- **Rango:** el de la última campaña de solfeo con **jefe vencido de verdad** (saltarlo no da rango).
- **Nivel por XP:** `nivel n` requiere `150 + 75·(n-1)` XP acumulados en ese nivel.

### 5.3 Perfil y logros

Ficha (nivel, rango, barra de XP), estadísticas (notas leídas, precisión, tiempo medio por nota, mejor racha, jefes, niveles, estrellas, récord de arcade, días jugados), **notas más difíciles** (porcentaje de fallo por nota vista ≥3 veces), progreso por campaña y partituras, **18 logros** y lista de saltados.

### 5.4 Guardado

1. **`localStorage`** con la clave `clave-maestra`/`solfeo-bolsillo-vN` (migración desde versiones anteriores).
2. **Contraseña de progreso** (tipo videojuego clásico): 32 caracteres en grupos de 5, alfabeto de 32 símbolos sin caracteres ambiguos (`ABCDEFGHJKLMNPQRSTUVWXYZ23456789`).
   - Campos: versión (3 bits) · nodos alcanzados solfeo (7) · superados solfeo (63 bits de máscara) · alcanzados piano (6) · superados piano (40) · XP/10 (16) · récord arcade/10 (12) · suma de control (9).
   - La suma de control rechaza contraseñas mal copiadas.
3. **Copia en archivo JSON** (incluye estrellas, estadísticas, ajustes y la voz entrenada). Se oculta cuando la app corre embebida (donde el navegador bloquea las descargas).

Al terminar un capítulo se muestra una pantalla de celebración con el rango obtenido y **la contraseña nueva, con botón de copiar**.

---

## 6. Entradas del usuario

### 6.1 Botones de nota

Siete botones en **dos filas encadenadas de izquierda a derecha**, imitando las teclas blancas del piano: arriba Do · Mi · Sol · Si y abajo, desplazados medio botón, Re · Fa · La. Rejilla de 8 columnas donde cada botón ocupa 2 y la fila inferior empieza en la columna 2. Tamaño de fuente fluido, estilo tecla gruesa. En niveles con alteraciones, una fila superior con ♭ / ♮ / ♯.

### 6.2 Teclado de piano en pantalla

- Teclas blancas en rejilla; negras superpuestas al 62 % de altura, posicionadas entre blancas (no hay negra tras Mi ni tras Si).
- El rango se calcula a partir de las notas del nivel, redondeando a Do abajo y Si arriba, mínimo una octava y media, máximo cuatro octavas (28 teclas blancas en el último capítulo).
- La respuesta correcta exige **la tecla exacta** (altura y octava), no solo el nombre: es lo que enseña la relación partitura↔teclado.
- Nombres en las teclas configurables (todas / solo los Do).
- Retroalimentación: la tecla pulsada se ilumina verde o roja y, si fallas, se marca en dorado la correcta.

#### Ancho de tecla y orientación

El teclado **no tiene un ancho de tecla fijo**: al montarse mide el espacio real disponible y reparte `(ancho − 20) / nº de teclas`, acotado entre **26 px** (mínimo tocable con el pulgar) y **62 px**. Se refija en `resize` y `orientationchange`.

- Si el reparto sale ≥ 26 px, el teclado **entra entero** y se centra, sin desplazamiento horizontal.
- Si sale por debajo, el teclado pasa a 26 px por tecla con scroll horizontal, que es el peor caso: el jugador tiene que buscar la tecla mientras corre el reloj.

Cuando hay que desplazarlo, el teclado **se centra en las notas que el ejercicio usa de verdad** (el punto medio entre su nota más grave y la más aguda), no en el rango redondeado a octavas completas. Se recentra al girar y al volver a medir la pantalla.

Para evitar ese peor caso en el móvil, antes de empezar cualquier nivel que se responde con el teclado (niveles `teclado`/`pTeclas`, recitales de piano, partituras «Tocar en el teclado» y arcade con entrada de teclado) se muestra la pantalla **«Gira el móvil»**, con el número de teclas del nivel. Se muestra solo si se cumplen las cuatro condiciones:

1. el teclado no cabe entero,
2. la pantalla está en vertical (en horizontal no hay nada que girar),
3. falta ancho de verdad (`ancho < 0,72 × ancho necesario`; si falta poco no merece interrumpir — por eso el iPad en vertical nunca lo pide),
4. girar lo arregla (`alto ≥ 0,9 × ancho necesario`).

Al aparecer intenta `screen.orientation.lock("landscape")`, que funciona en Chrome/Android y falla silenciosamente en iOS. **No se puede forzar la orientación en la web de iPhone**, así que el bloqueo es lo más parecido a obligatorio que permite la plataforma: en pantallas pequeñas (lado mayor < 950 px) el botón «Seguir en vertical» **no aparece hasta pasados 5 segundos**; hasta entonces solo se ve el aviso de que la salida llegará si de verdad no se puede girar. En pantallas grandes la salida está desde el principio. Existe siempre porque, con el bloqueo de rotación del sistema activado, sin ella el nivel sería injugable. Al girar, el aviso se retira solo. La decisión de «seguir en vertical» dura la sesión (se olvida al recargar), porque es un empujón útil, no una preferencia.

Si hay un piano MIDI conectado (§6.5) el aviso no aparece: no se toca la pantalla.

Cifras reales medidas (rango máximo, 28 teclas blancas → 748 px necesarios):

| Pantalla | Ancho de tecla | ¿Cabe? | ¿Pide girar? |
|---|---|---|---|
| Móvil vertical 390×844 | 26 px | no (14 de 28 visibles) | **sí** |
| Móvil horizontal 844×390 | 28,8 px | sí | no |
| iPad vertical 820×1180 | 26,7 px | sí | no |

### 6.3 Piano MIDI («tocar con mi piano»)

En el modo piano se puede responder **tocando el instrumento de verdad**, con la **Web MIDI API**:

- `navigator.requestMIDIAccess({sysex:false})` y escucha de `noteOn` (`status & 0xF0 === 0x90`, velocidad > 0) en todas las entradas; `onstatechange` vuelve a enlazar cuando se conecta o desconecta un teclado.
- El interruptor aparece en el mismo hueco que usa el panel de voz, en todos los sitios donde se responde con teclado (nivel, recital, partitura y arcade); ambos paneles comparten el hueco y el que no está en uso se aparca sin destruirse.
- Con MIDI activo el teclado de la pantalla sigue funcionando y se ilumina la tecla que llega del piano, así se ve la correspondencia.
- Safari en iPhone no admite Web MIDI: el interruptor lo dice y queda desactivado, sin prometer nada que no pueda cumplir.

### 6.4 Voz (reconocedor propio)

Motor de reconocimiento **local**, sin servicios externos, específico para 7 palabras.

1. **Captura:** `getUserMedia` con `echoCancellation`, `noiseSuppression` y `autoGainControl`; filtros paso alto 110 Hz y paso bajo 5,5 kHz; `ScriptProcessor` de 2048 muestras.
2. **Análisis:** ventana de 1024 muestras (21 ms a 48 kHz) con salto de 512; preénfasis 0,97; ventana de Hamming; FFT propia; banco de 24 filtros mel entre 100 y 5000 Hz; 12 coeficientes cepstrales con liftering + energía normalizada como 13ª dimensión.
3. **Detector de voz (VAD):** suelo de ruido adaptativo; inicio cuando la energía supera `ruido + 13 dB` durante 24 ms; fin tras 70 ms por debajo de `ruido + 6 dB`; se descartan segmentos <110 ms o >900 ms (y los largos suben el suelo de ruido).
4. **Clasificación:** DTW (alineamiento temporal dinámico) con banda de Sakoe-Chiba contra las plantillas grabadas por el usuario; puntuación por clase = 0,65·mejor + 0,35·segunda; se acepta si la distancia ≤ límite y la razón con la segunda clase ≥ **margen de exigencia**. Si no, pide repetir (**nunca cuenta como fallo**).
   - **El margen no es un ajuste del usuario: lo fija cada tipo de nivel.** Donde hay tiempo para pensar se exige más seguridad; donde corre el reloj se acepta antes:

   | Tipo de nivel | Margen |
   |---|---|
   | Entrenamiento, nota a tecla, examen, pergamino | 1,12 |
   | Cambio de clave | 1,10 |
   | Contrarreloj, arcade, partitura | 1,06 |
   | Al compás, jefe | 1,05 |

   La tabla se aflojó en la versión 15. La anterior empezaba en 1,20, más estricta que el antiguo ajuste «normal» (1,14) que sustituyó, y el resultado práctico era que el reconocedor pedía repetir más que antes aunque hubiera entendido bien. Un margen alto no acierta más: solo se calla más.
5. **Entrenamiento:** **4 rondas × 7 notas** (28 muestras, algo menos de un minuto) con medidor de nivel, deshacer y validación cruzada *leave-one-out*.
   - **Repaso automático nota a nota.** Al acabar, `selfTestByNote` prueba cada muestra contra un modelo construido sin ella y devuelve, para cada una de las siete notas, cuántas acierta, cuántas duda, cuántas confunde y **con cuál se confunde**. Una nota está floja si falla alguna vez o duda más de una; entonces el micrófono sigue abierto y se piden **dos muestras más solo de esas notas**, hasta dos repasos. Es la diferencia entre «he grabado 28 veces» y «sé que distingo las siete».
   - El resultado se muestra como siete fichas (Do ✓ · Fa ~ *la* · Sol ✕ *do*), de modo que el usuario ve qué nota conviene decir separando mejor la vocal en vez de un porcentaje global.
   - **Puerta de silencio:** tras grabar una nota, el detector se «desarma» y no vuelve a escuchar hasta detectar **260 ms de silencio** continuo (`waitSilence()`), y la pantalla muestra «Espera al silencio…». Así una sola palabra —o su eco— no rellena dos huecos seguidos. Medido con audio sintético: con huecos de 150 ms se captura 1 segmento en vez de 3; con huecos ≥ 300 ms se capturan todos.
6. **Respuesta casi instantánea:** además del segmento final, el detector emite **hipótesis parciales** cada 30 ms a partir de 150 ms de voz; si una hipótesis parcial supera umbrales más estrictos (distancia ≤ 0,75·límite y margen +0,25) se responde **sin esperar al silencio** y se ignora el resto de la locución.
   - Medición con voz sintética: 28/28 aciertos; la mitad de las respuestas se deciden ~330 ms antes del final de la palabra.
7. **Sin dictado del sistema:** se descartó Siri / Web Speech API (lento, oye cualquier palabra y depende de conexión). El motor propio es la única vía de voz.
8. **Permisos:** si la app corre embebida en un iframe sin permiso de micrófono, se detecta y se ofrece «Abrir en pestaña nueva».

---

## 7. Diseño visual

### 7.1 Identidad

Videojuego de fantasía amable: fondo de **noche azul** con pentagrama tenue de fondo, tarjetas de **pergamino** donde se lee la música, botones gruesos con sombra inferior que se hunden al pulsar, tipografía de titular redondeada.

### 7.2 Tokens

```
--night #131B33   --night-2 #1C2747   --night-3 #27345C
--on-night #F4F0FF  --on-night-2 #A8B0CF
--parch #FFF6E4  --parch-2 #F4E7CA  --parch-shadow #D5BD86
--ink #2A2140  --ink-2 #6C6284
--gold #FFC23C / #C98A06     (progreso, recompensa)
--coral #FF5D5D / #C73A45    (jefes, error)
--mint #2FD69B / #149468     (acierto, entrenamiento)
--sky #45B8FF / #1B7FCB      (teclado, claves)
--violet #9277FF / #6547E0   (ritmo, teoría)
--orange #FF9142 / #CF621A   (contrarreloj, saltado)
Tipografías: "Lilita One" (titulares) + "Nunito" (texto) + Noto Music (glifos)
```

**El color codifica el tipo de nivel** y se mantiene en mapa, ficha, cabecera del juego y arcade.

### 7.3 Componentes

Botón grueso (`.btn` con `box-shadow: 0 5px 0 var(--sombra)` y hundimiento al pulsar), tarjeta de pergamino, medallón de nodo, cinta de capítulo, hoja de mapa con viñeta, sello de capítulo superado, píldora de contador, barras de progreso, tostadas de logro, hojas inferiores (*bottom sheets*) para ficha de nivel y ajustes, navegación inferior de 5 pestañas.

### 7.4 Respuesta a pantalla (responsive)

- Base: móvil vertical, ancho máximo 520 px, márgenes laterales 16 px, área segura respetada.
- **Móvil en horizontal: el modo de juego principal.** Es la orientación con la que la app está pensada para jugar, no un apaño: el pentagrama tiene sitio para crecer, el teclado de cuatro octavas entra entero y las **siete notas caben en una sola fila** de botones grandes, en vez de las dos filas encadenadas que hacen falta en vertical.
- **iPad / tablets (≥700 px):** ancho máximo 860 px, tipografía 17 px, mapa centrado a 560 px, rejillas de estadísticas a 4 columnas y logros a 3, pentagramas y teclado más altos, contenido de lectura limitado a 680 px.
- **Modo compacto (`orientation:landscape` y alto ≤ 600 px):** barra superior a 36 px, sin sobretítulo del nivel, insignia y barra de progreso más pequeñas, teclado a 98 px de alto, **botones de nota en una fila de siete** a 52 px, overlays con los botones en horizontal, tarjetas y botones con menos relleno, y respeto de `safe-area-inset` a izquierda, derecha y abajo (la muesca queda en el lateral al girar). Verificado sin recorte ni scroll de página en 844×390 en los tres sitios donde aparece el teclado.
- Accesibilidad: foco visible dorado, `prefers-reduced-motion` desactiva todas las animaciones, roles y etiquetas ARIA en controles.

---

## 8. Arquitectura técnica

- **Un único archivo HTML** autocontenido (~230 KB) con CSS y JS en línea; única dependencia externa: Google Fonts (con alternativas del sistema). La fuente musical va incrustada en base64.
- Sin framework. Todo el estado en un objeto `store` serializado a `localStorage`. Render por plantillas de cadena e inserción en `innerHTML`.
- **Pantallas** (secciones que se muestran/ocultan): `welcome`, `home` (con 5 paneles), `chapter`, `lesson`, `quiz`, `play`, `train`, `song`, `arcade`, `result`, `chapterend`.
- **Motores:**
  - *play*: rondas de notas sueltas o grupos (figuras con barras), con contador, racha, penalizaciones y resultado.
  - *song*: motor a tempo compartido por jefes, retos de ritmo y partituras; desplaza el gran pentagrama contra una línea fija, con cuenta atrás de 4 clics.
  - *arcade*: bucle de animación con enemigos, colisiones con la línea de defensa, combos y oleadas.
  - *voz*: descrito en §6.3.
- **Entrada unificada:** un objeto `ACTIVE` expone `answer / running / busy / readyAt`, de modo que botones, teclado, teclas del ordenador y voz alimentan el mismo punto.
- **PWA:** `manifest.webmanifest`, iconos generados, `sw.js` con caché *stale-while-revalidate* para funcionar sin conexión; `.nojekyll` para GitHub Pages.

---

## 9. Criterios de aceptación (pruebas)

1. Las claves se dibujan en su sitio: la espiral de la clave de sol rodea la 2ª línea; los puntos de la clave de fa abrazan la 4ª.
2. **Auditoría de contenido** (`audit`, test automático sobre los datos, sin navegador): para los 19 capítulos de las dos campañas comprueba que la canción del jefe (a) usa solo notas, figuras y silencios ya introducidos, (b) repite cada nota nueva ≥2 veces (≥3 si pasa de 20 notas), (c) estrena en ella cada figura cantable y cada silencio del capítulo, (d) suma un número de pulsos múltiplo del compás y (e) no supera el salto máximo permitido en ese punto de la progresión.
2 bis. Los **silencios** de una canción se saltan solos: no se responden, no cuentan como fallo y no bloquean el avance del cursor (comprobado jugando los 19 jefes en modo práctica).
3. El reconocedor de voz entrenado con 21 muestras sintéticas acierta 28/28 en muestras no vistas y **rechaza** palabras ajenas («hola», «casa», «mesa»…).
4. Una contraseña alterada en un carácter es rechazada por la suma de control; una válida restaura nodos, saltos, XP y récord.
5. Un capítulo completo (pergamino → entreno → retos → jefe) desemboca en la pantalla de celebración con contraseña nueva.
6. En el nivel de teclado, pulsar la tecla correcta de otra octava se considera error (se exige altura exacta).
7. La app se usa con una sola mano en 390×844 y aprovecha el ancho en 820×1180 sin scroll horizontal.
8. Sin conexión, la app abre desde la caché del service worker.
9. Los **94 nodos** de las dos campañas abren su pantalla sin un solo error de JavaScript (test de navegador que los recorre todos).
11. **Sonido:** el botón «Probar» de ajustes reproduce un Do y dice en qué estado está el contexto de audio; el audio se desbloquea al primer toque de la pantalla.
12. **Dos barras:** en los niveles a compás se ven la línea verde de tempo, la roja de fallo y la franja de margen entre ellas, y la nota en juego cambia a ámbar al entrar en el margen.
10. **Teclado y orientación:** en 390×844 los niveles de teclado piden girar el móvil; en 844×390 y en 820×1180 el teclado del rango más ancho (28 teclas) entra entero, sin scroll horizontal ni recorte por abajo, en los tres contextos que lo usan (nivel de lectura, recital y arcade).

---

## 10. Registro de decisiones (qué pidió el usuario y qué se infirió)

**Pedido explícitamente por el usuario:**

1. App para iPhone que enseñe las notas en clave de sol y de fa, con niveles por dificultad, notas al azar y **nomenclatura española**.
2. Responder con **botones Do-Re-Mi** (en vez de teclado) — decisión inicial, luego ampliada con teclado y voz.
3. **Reconocimiento de voz**, después «casi instantáneo» y que «solo escuche las 7 notas» reduciendo ruido.
4. Sustituir la redonda fija por **figuras variadas** y, más tarde, que **las introduzca el juego poco a poco** según el progreso.
5. Fusionar clave de sol y de fa en un **modo historia** único (razón del usuario: para piano no sirve dominar solo una clave).
6. **Canciones sin derechos** al final de cada capítulo, que haya que solfear **a tempo** y sin fallos para pasar.
7. Nombres «molones», explicaciones divertidas, referencias a videojuegos/rangos, frases célebres y mensajes al fallar.
8. **Modo arcade** tipo invasores; después, fusionar los «retos» dentro del arcade.
9. Apartado de **teoría** consultable, ordenada por lo que ya deberías saber.
10. Guardado pensado para web estática en **GitHub Pages**, con **contraseña** de progreso.
11. **Rediseño gráfico** de videojuego; mapa que parezca un **mapa de fantasía**, no una lista; regiones diferenciadas.
12. Progresión con **retos intermedios** antes del jefe, posibilidad de **saltar niveles** marcándolo en el perfil, **perfil** con rango, XP, estadísticas y logros, y **niveles de teoría**.
13. Corregir la posición de la clave de fa; **icono de ajustes** que no parezca un sol; elegir **rango de notas en el arcade**.
15. **Revisión pedagógica**: que los conceptos entren en el orden adecuado, que el número de niveles sea el justo y que **las canciones de los jefes se correspondan de verdad** con las notas y la dificultad del capítulo.
14. **Pantalla de bienvenida** con carga de código; recordatorio de guardar al terminar capítulo; botones de nota **ordenados de izquierda a derecha**; que se vea bien en **iPad**; **título más molón**; **partituras** jugables; **modo historia de piano** con teoría, entrenamiento, niveles divertidos y jefes; un **nivel de teclado por capítulo**.

**Inferido o decidido por el diseñador (con su razón):**

- **Enfoque de notas faro** y progresión alternando claves (método estándar de piano; evita memorizar el pentagrama entero).
- **Do3 como Do central** (convención franco-belga usada en España) y explicación de la equivalencia inglesa.
- **Reconocedor propio por DTW** en lugar de depender del dictado del sistema: más rápido, funciona sin conexión y solo reconoce las 7 notas. El dictado del sistema se retiró del producto.
- **Exigencia por nivel y no por ajuste del usuario:** la persona no tiene por qué saber qué significa; el nivel sabe qué precisión debe esperar de ella.
- **Puerta de silencio en el entrenamiento** tras detectar un problema real: una sola palabra llenaba varios huecos seguidos.
- **Hipótesis parciales** para responder antes de que termine la palabra.
- **Contraseña con suma de control** en lugar de un código largo en base64: más corta y a prueba de erratas.
- **Rangos por jefes vencidos, no por XP**, para que saltar no infle el rango.
- **Paleta por tipo de nivel** y botones gruesos: legibilidad instantánea del mapa y sensación de juego.
- **Mapa procedural con semilla fija**: variedad visual sin imágenes (peso y offline).
- **Teclado que exige la octava exacta**: es la habilidad que de verdad traslada la lectura al instrumento.
- **Piezas de dominio público verificadas** y negativa explícita a incluir obras con derechos vigentes.
- **Arreglos a una sola voz** en las partituras para poder solfearlas y tocarlas con una mano.

**Decisiones de la revisión pedagógica (versión 13):**

- **Regla dura «la canción demuestra el capítulo»:** una nota que se estrena tiene que sonar al menos dos veces en la canción del jefe (tres si es larga). Antes había capítulos que estrenaban cuatro notas y la canción usaba una: el jefe no medía lo que el capítulo enseñaba. Ahora lo verifica una auditoría automática sobre los datos, no el ojo.
- **Ritmo antes que nota rápida:** el compás y los silencios se enseñan en el capítulo 1, no al final. Sin compás, «solfear a tempo» es obedecer una animación; con compás, es leer.
- **Techo de velocidad en los jefes:** nunca por debajo de la corchea, porque el jugador tiene que decir el nombre en voz alta. Las figuras rápidas se practican leyendo, sin reloj. Esto obligó a reordenar las figuras: las cantables al principio, las de lectura al final.
- **Un concepto grande por capítulo:** el capítulo del cambio de manos añade una sola nota; el de las teclas negras, un solo sostenido. Antes, ambos mezclaban dos novedades y la canción no llegaba a fijar ninguna.
- **Intervalos como atajo de lectura** (segunda = línea-espacio, tercera = línea-línea): es la técnica que separa leer nota a nota de leer de verdad, y no estaba explicada en ninguna parte.
- **Curva de niveles 4 → 7:** cuatro niveles hasta la primera canción completa (enganche), siete en el último capítulo (variedad para quien ya aguanta). 58 niveles de solfeo y 36 de piano: suficiente para meses de práctica sin que la lista parezca una tarea.
**Decisiones de la versión 15:**

- **El sonido no fallaba: estaba silenciado.** En iOS el audio web obedece al interruptor lateral salvo que la página declare `navigator.audioSession.type = "playback"`, y el contexto no arranca fuera de un gesto. Se declara la sesión, se desbloquea con un búfer mudo en el primer toque y se añade una prueba de sonido en ajustes que dice qué está pasando en vez de dejar al usuario adivinando.
- **Tocar con el piano de verdad.** Es lo que se está aprendiendo; responder en una tecla dibujada es un sucedáneo. Web MIDI lo permite en Chrome y Edge, y donde no existe se dice claramente en vez de esconder la opción.
- **Dos barras en vez de una.** La queja («cuando dice ¡YA! ya llegas tarde») no era de tiempos: el margen existía pero era invisible. Dibujarlo como una franja entre la línea de tempo y la de fallo convierte una sensación de injusticia en información.
- **El pentagrama se mide, no se supone.** Un `viewBox` fijo deja el dibujo pequeño en el centro de una caja apaisada. Calcularlo a partir del rango real de notas y de la forma del hueco dobla el tamaño del pentagrama sin cambiar una sola coordenada del dibujo.
- **Margen de voz más bajo, entrenamiento más largo.** Ante «antes funcionaba mejor», la respuesta no es subir la exigencia sino bajarla y mejorar el modelo: más muestras y un repaso dirigido a las notas que el propio test cruzado señala como flojas.
- **Pedir girar el móvil en vez de fingir que cabe:** un teclado de cuatro octavas con scroll horizontal obliga a buscar la tecla mientras corre el reloj, que es justo lo que el nivel no está midiendo. Como la web de iPhone no puede forzar la orientación, el aviso se pide, se intenta bloquear donde el navegador lo permite y se deja siempre salida.
- **Longitud de las canciones fijada por el repertorio real, no por relleno:** cuando una melodía necesitaba repetir su primera frase para que la nota nueva sonara tres veces, se repite porque la pieza original lo hace (Brahms, Cascabeles), no se alarga artificialmente.

---

## 11. Guía de reconstrucción (orden recomendado)

1. Modelo de notas, dibujo del pentagrama y figuras (§3) con pruebas visuales.
2. Motor de rondas (`play`) con botones y resultado.
3. Datos de campaña de solfeo (§4.1) + mapa simple y desbloqueos.
4. Motor a tempo (`song`) para jefes; después retos de ritmo y partituras.
5. Teoría y exámenes.
6. Perfil, logros, estadísticas y contraseña.
7. Voz (análisis → VAD → DTW → entrenamiento → parciales).
8. Teclado de piano y campaña de piano.
9. Arcade y selector de rango.
10. Capa visual completa (§7), mapa de fantasía y responsive.
11. PWA, bienvenida y celebración de capítulo.

---

## 12. Ideas pendientes (no implementadas)

- Entrada MIDI real (Web MIDI) para tocar con un piano físico.
- Detección de la nota tocada por micrófono (afinador) para practicar con piano acústico.
- Editor de partituras del usuario y importación de MusicXML.
- Ejercicios a dos manos simultáneas (acordes).
- Estadísticas por sesión y repaso espaciado de las notas falladas.
