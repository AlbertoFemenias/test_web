# Clave Maestra — especificación completa del producto

> Documento de diseño dirigido por especificación (*spec-driven design*).
> Objetivo: que otra IA (o un equipo) pueda reconstruir esta aplicación desde cero y obtener un producto prácticamente idéntico.
> Idioma del producto: **español de España**. Nomenclatura musical **latina** (Do, Re, Mi…).
> Versión del documento: 1.0 · Corresponde a la versión 10 de la app.

---

## 1. Resumen ejecutivo

**Clave Maestra** es una aplicación web (una sola página, sin servidor ni cuentas) para aprender a **leer música y tocar el piano** desde cero, en español, con estética y mecánicas de videojuego.

- **Plataforma:** web estática. Pensada para iPhone en vertical (se añade a la pantalla de inicio como app), funcional también en iPad y escritorio.
- **Sin backend:** todo el estado vive en `localStorage`, más un sistema de **contraseñas de progreso** tipo videojuego clásico y una copia en archivo JSON.
- **Dos campañas:** *Solfeo* (nombrar notas) y *Piano* (tocarlas en un teclado en pantalla).
- **Modos de respuesta:** botones de nota, teclado de piano en pantalla y **voz** (reconocedor propio entrenado con la voz del usuario).
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

`s` = clave de sol (pentagrama superior), `f` = clave de fa (inferior). El tiempo de inicio de cada nota es la suma acumulada de los pulsos anteriores. La figura dibujada se deduce de los pulsos (`{4:redonda, 3:blancaP, 2:blanca, 1.5:negraP, 1:negra, 0.5:corchea}`).

---

## 4. Contenido

### 4.1 Campaña «Solfeo» — 11 capítulos

Cada capítulo define: `zone` (nombre del lugar en el mapa), `rank` (rango que otorga), `add` (notas nuevas), `figs` (figuras nuevas), `story` (texto del narrador), `quote` (frase célebre con autor), `song` (canción del jefe: nombre, origen, bpm, notas).

| # | Zona | Rango | Notas nuevas | Figuras nuevas | Jefe |
|---|---|---|---|---|---|
| 1 | La Aldea del Do Central | Aprendiz | Do3 Re3 Mi3 (sol) | negra, blanca | Mary tenía un corderito |
| 2 | El Bosque de la Clave de Fa | Escudero | Do3 Si2 La2 (fa) | redonda | Bollitos calientes |
| 3 | La Torre del Sol | Caballero | Fa3 Sol3 (sol) | negra con puntillo, corchea | Himno de la alegría (Beethoven) |
| 4 | Las Mazmorras del Fa | Paladín | Sol2 Fa2 (fa) | 2 corcheas | Al claro de la luna |
| 5 | El Puente de las Dos Manos | Campeón | La3 (sol), Mi2 Re2 Do2 (fa) | semicorchea | Martinillo |
| 6 | Las Almenas de la Clave de Sol | Guardián | Si3 Do4 (sol) | blanca con puntillo | Canción de cuna (Brahms) |
| 7 | Las Cuevas del Bajo | Maestro de armas | Si1 La1 Sol1 (fa) | 2 semicorcheas | En la granja de mi tío |
| 8 | El Pico Nevado | Hechicero | Re4 Mi4 Fa4 Sol4 (sol) | 4 semicorcheas | Cascabeles |
| 9 | El Abismo Grave | Archimago | Fa1 Mi1 Re1 Do1 (fa) | corchea con puntillo (+semi) | Canon de Pachelbel (bajo) |
| 10 | Las Nubes del Agudo | Leyenda | La4 Si4 Do5 (sol) | corchea + 2 semis | Canción de cuna en las nubes |
| 11 | El Trono de las Dos Claves | Señor del Pentagrama | — | — | La Oda de las dos manos |

Reglas de contenido:
- Cada canción usa **solo notas y figuras ya aprendidas** en ese capítulo o anteriores (verificable con un test automático).
- Las frases célebres de atribución dudosa se marcan con «atribuida a».
- El narrador es **Maese Clavius, mago del pentagrama**: explica con humor y da un truco memotécnico por capítulo.

### 4.2 Campaña «Piano» — 8 capítulos

Mismo esqueleto, pero la lección enseña **dónde cae la tecla y con qué dedo**, con un diagrama de teclado con números de dedo (1–5) coloreados por mano (naranja derecha, azul izquierda).

| # | Zona | Rango | Contenido |
|---|---|---|---|
| 1 | La Posición de Do | Manos nuevas | Do-Re-Mi derecha, dedos 1-2-3 |
| 2 | La Mano Izquierda | Dos manos | Do-Si-La izquierda, pentagrama inferior |
| 3 | El Puño Completo | Cinco dedos | Fa-Sol derecha (posición de cinco dedos) |
| 4 | El Sótano | Bajo firme | Sol-Fa izquierda; el meñique toca lo más grave |
| 5 | El Diálogo | Manos juntas | La derecha + Mi-Re-Do izquierda |
| 6 | La Octava | Registro amplio | Si-Do agudos; cambio de posición |
| 7 | El Bosque Negro | Teclas negras | Fa♯ y La♯/Si♭ |
| 8 | El Recital | Concertista | Re-Mi agudos; pieza a dos manos |

### 4.3 Tipos de nivel (nodos del mapa)

| Tipo | Campaña | Descripción | Aprobado | XP |
|---|---|---|---|---|
| `lectura` / `pLectura` | ambas | Pergamino con historia, notas/figuras nuevas y enlaces a teoría | leerlo | 20 |
| `entreno` | solfeo | 15 notas al azar (las nuevas salen ×3) | ≥80 % | 40 |
| `teclado` / `pTeclas` | ambas | Ves la nota, pulsas la tecla exacta en el piano | ≥80 % | 50 |
| `claves` | solfeo | 16 notas alternando clave (70 % de cambio) | ≥80 % | 50 |
| `crono` | solfeo | 45 s, objetivo `round(9 + cap*1,3)` aciertos; fallo −2 s | objetivo | 50 |
| `tempo` / `pTempo` | ambas | Notas a tempo (46+4·cap ppm), 3 vidas | sobrevivir | 60 |
| `quiz` / `pQuiz` | ambas | 6 preguntas tipo test con explicación | ≥4/6 | 40 |
| `jefe` / `pJefe` | ambas | Canción completa a tempo, sin fallos | perfecto | 150 |

Composición por capítulo (solfeo): `lectura → entreno → [2-5 retos rotando] → jefe`. La rotación está fijada en una tabla por capítulo para que la dificultad suba y no se repita el mismo reto seguido.

**Estrellas:** 1★ aprobar, 2★ ≥90 %, 3★ ≥95 % y ≤2,5 s por nota (en contrarreloj y a compás, por objetivos/fallos).

### 4.4 Arcade

- **Invasores del pentagrama** (modo estrella): las notas avanzan desde la derecha hacia una línea de defensa; se destruyen nombrándolas (o tocándolas en el teclado). 3 vidas, combo que multiplica puntos cada 5 aciertos, oleada más rápida cada 8 destruidas (velocidad ×1,14; intervalo ×0,87, mínimo 0,85 s). Récord guardado.
  - **Selector de rango:** claves (ambas/sol/fa), presets (Aprendido, Pentagrama, Con 1 línea adicional, Todo) y rango a medida con dos desplegables por clave; vista previa en el gran pentagrama y recuento de notas.
- **Más juegos** (se desbloquean al vencer jefes 3, 5, 7 y 9): Contrarreloj libre (60 s), Cambio de clave, Sostenidos y bemoles, Líneas adicionales.

### 4.5 Partituras

Biblioteca de 10 piezas **de dominio público**, con dificultad 1–5, que se abren según el capítulo alcanzado: Estrellita, Cumpleaños feliz, La cucaracha, Himno de la alegría, Cascabeles, Marcha fúnebre (Chopin), Greensleeves, En la cueva del rey de la montaña (Grieg), Para Elisa, Marcha turca (Mozart).

Cada pieza ofrece: **Escuchar** (suena con los nombres), **Practicar** (la línea espera), **Solfear a tempo** y **Tocar en el teclado**. No se pierde: se cuentan fallos (0 → 3★, ≤2 → 2★, ≤5 → 1★) y se guarda el mejor resultado.

> **Restricción legal explícita:** no se incluyen obras con derechos vigentes (por ejemplo *La Pantera Rosa*, 1963, o *Tequila*, 1958). Si el usuario las pide, se explican los derechos y se ofrecen equivalentes de dominio público con carácter parecido.

### 4.6 Teoría («Grimorio»)

17 lecciones con diagramas SVG generados por código (pentagrama anatómico, teclado, gran pentagrama, tablas de figuras), agrupadas en tres estados según el capítulo alcanzado: **Ya deberías saberlo**, **Lo siguiente**, **Más adelante** (plegado). Consultables sin haber jugado. Categorías: Lectura, Figuras y ritmo, Alteraciones.

Banco de preguntas: 47 para solfeo + 12 para piano, cada una con 1 respuesta correcta, 3 distractores y una explicación. Cada pregunta está etiquetada con el capítulo a partir del cual es justa.

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
- El rango se calcula a partir de las notas del nivel, redondeando a Do abajo y Si arriba, mínimo una octava y media, máximo cuatro octavas; con desplazamiento horizontal si no cabe.
- La respuesta correcta exige **la tecla exacta** (altura y octava), no solo el nombre: es lo que enseña la relación partitura↔teclado.
- Nombres en las teclas configurables (todas / solo los Do).
- Retroalimentación: la tecla pulsada se ilumina verde o roja y, si fallas, se marca en dorado la correcta.

### 6.3 Voz (reconocedor propio)

Motor de reconocimiento **local**, sin servicios externos, específico para 7 palabras.

1. **Captura:** `getUserMedia` con `echoCancellation`, `noiseSuppression` y `autoGainControl`; filtros paso alto 110 Hz y paso bajo 5,5 kHz; `ScriptProcessor` de 2048 muestras.
2. **Análisis:** ventana de 1024 muestras (21 ms a 48 kHz) con salto de 512; preénfasis 0,97; ventana de Hamming; FFT propia; banco de 24 filtros mel entre 100 y 5000 Hz; 12 coeficientes cepstrales con liftering + energía normalizada como 13ª dimensión.
3. **Detector de voz (VAD):** suelo de ruido adaptativo; inicio cuando la energía supera `ruido + 13 dB` durante 24 ms; fin tras 70 ms por debajo de `ruido + 6 dB`; se descartan segmentos <110 ms o >900 ms (y los largos suben el suelo de ruido).
4. **Clasificación:** DTW (alineamiento temporal dinámico) con banda de Sakoe-Chiba contra las plantillas grabadas por el usuario; puntuación por clase = 0,65·mejor + 0,35·segunda; se acepta si la distancia ≤ límite y la razón con la segunda clase ≥ margen (1,12 normal / 1,25 exigente). Si no, pide repetir (**nunca cuenta como fallo**).
5. **Entrenamiento:** 3 rondas × 7 notas (~40 s) con medidor de nivel, deshacer y validación cruzada *leave-one-out* que estima la precisión.
6. **Respuesta casi instantánea:** además del segmento final, el detector emite **hipótesis parciales** cada 30 ms a partir de 150 ms de voz; si una hipótesis parcial supera umbrales más estrictos (distancia ≤ 0,75·límite y margen +0,25) se responde **sin esperar al silencio** y se ignora el resto de la locución.
   - Medición con voz sintética: 28/28 aciertos; la mitad de las respuestas se deciden ~330 ms antes del final de la palabra.
7. **Alternativa:** dictado del sistema (Siri / Web Speech API) con diccionario de variantes, por si el usuario no quiere entrenar.
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
- **iPad / tablets (≥700 px):** ancho máximo 860 px, tipografía 17 px, mapa centrado a 560 px, rejillas de estadísticas a 4 columnas y logros a 3, pentagramas y teclado más altos, contenido de lectura limitado a 680 px.
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
2. Toda canción de jefe usa solo notas y figuras ya introducidas (test automático sobre los datos).
3. El reconocedor de voz entrenado con 21 muestras sintéticas acierta 28/28 en muestras no vistas y **rechaza** palabras ajenas («hola», «casa», «mesa»…).
4. Una contraseña alterada en un carácter es rechazada por la suma de control; una válida restaura nodos, saltos, XP y récord.
5. Un capítulo completo (pergamino → entreno → retos → jefe) desemboca en la pantalla de celebración con contraseña nueva.
6. En el nivel de teclado, pulsar la tecla correcta de otra octava se considera error (se exige altura exacta).
7. La app se usa con una sola mano en 390×844 y aprovecha el ancho en 820×1180 sin scroll horizontal.
8. Sin conexión, la app abre desde la caché del service worker.

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
14. **Pantalla de bienvenida** con carga de código; recordatorio de guardar al terminar capítulo; botones de nota **ordenados de izquierda a derecha**; que se vea bien en **iPad**; **título más molón**; **partituras** jugables; **modo historia de piano** con teoría, entrenamiento, niveles divertidos y jefes; un **nivel de teclado por capítulo**.

**Inferido o decidido por el diseñador (con su razón):**

- **Enfoque de notas faro** y progresión alternando claves (método estándar de piano; evita memorizar el pentagrama entero).
- **Do3 como Do central** (convención franco-belga usada en España) y explicación de la equivalencia inglesa.
- **Reconocedor propio por DTW** en lugar de depender del dictado del sistema: más rápido, funciona sin conexión y solo reconoce las 7 notas.
- **Hipótesis parciales** para responder antes de que termine la palabra.
- **Contraseña con suma de control** en lugar de un código largo en base64: más corta y a prueba de erratas.
- **Rangos por jefes vencidos, no por XP**, para que saltar no infle el rango.
- **Paleta por tipo de nivel** y botones gruesos: legibilidad instantánea del mapa y sensación de juego.
- **Mapa procedural con semilla fija**: variedad visual sin imágenes (peso y offline).
- **Teclado que exige la octava exacta**: es la habilidad que de verdad traslada la lectura al instrumento.
- **Piezas de dominio público verificadas** y negativa explícita a incluir obras con derechos vigentes.
- **Arreglos a una sola voz** en las partituras para poder solfearlas y tocarlas con una mano.

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
