# Jardín amarillo — campo nocturno en 3D

Un campo de flores amarillas bajo la luna, en 3D y con cámara libre. Noche cerrada, luciérnagas parpadeando entre el pasto, la luna con su halo de luz, nubes opacas rondando el cielo y tres cordilleras recortadas en el horizonte.

Suena `Zamba Surreal.mp3` en bucle, y **a los 30 segundos de que empieza la música arrancan los fuegos artificiales**.

Lo único sobreimpreso es una línea: *Mi Linda Cinthy…*, que aparece con un fundido al cargar. Nada de botones ni interfaz.

**Cinco especies**, generadas proceduralmente y mezcladas al azar:

| Especie | Cómo se arma |
|---|---|
| Margarita | 9–12 pétalos acopados, corazón ámbar |
| Gerbera | dos coronas (18 + 14) de pétalos finos |
| Lirio | 6 pétalos largos y recurvados + 6 estambres |
| Girasol | 20–26 pétalos y disco de semillas abombado, tallo alto con hojas |
| Flor de nube | ramillete que se bifurca, con decenas de pomos color crema |

Cada flor recibe además una inclinación propia de la cabeza, así que ninguna mira exactamente hacia arriba.

## Stack

**Three.js r128** (UMD, desde cdnjs) y nada más. Sin build, sin bundler, sin `node_modules`: un único `index.html`.

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
```

No se usan `OrbitControls` ni ningún addon — los controles de cámara son propios (unas 40 líneas), lo que evita una dependencia extra y da control fino sobre el amortiguado y los límites.

## Cómo verlo

Three.js se carga por CDN, así que conviene servirlo por HTTP en lugar de abrir el archivo directamente:

```bash
python -m http.server 5173   # Python 3.11
# o
npx serve .
```

Luego <http://localhost:5173>.

## Controles

| Acción | Ratón | Táctil | Teclado |
|---|---|---|---|
| Girar la cámara | arrastrar | arrastrar | ← → ↑ ↓ |
| Acercar / alejar | rueda | pellizcar | — |
| Sembrar otro campo | doble clic | doble toque | — |

Tras unos segundos sin tocar nada, la cámara vuelve a girar sola muy lentamente.

## Cómo está hecho

- **Una sola malla para todo el pasto.** ~18 000 briznas (11 000 en móvil) se fusionan en una única `BufferGeometry` con atributos por vértice (`aSway`, `aAnchor`, `aTint`). Una sola *draw call* para todo el campo.
- **El viento vive en la GPU.** Un `ShaderMaterial` desplaza cada vértice según su altura normalizada al cuadrado y la posición de su mata, con tres senoidales desfasadas — la ráfaga recorre el campo en vez de mover todo a la vez. La CPU solo actualiza un `uniform float`.
- **Las flores, igual**: las ~440 (260 en móvil) se fusionan en una sola malla con color y emisión por vértice. Todas las especies salen de tres primitivas —pétalo de 5 vértices, disco y ramita—; lo que cambia es cuántas, con qué curvatura (`midLift` / `tipLift` hacen que el mismo pétalo se acope, se aplane o se recurve) y sobre qué base inclinada. Se mecen con la misma función de viento pero un 15 % menos, porque el tallo es más rígido.
- **Nada se come el encuadre**: una flor que quede pegada al objetivo se pliega hacia su raíz en el vertex shader (`smoothstep` sobre la distancia de su base a la cámara), en vez de tapar la pantalla.
- **Terreno coherente**: `groundH(x, z)` es una sola función compartida por la malla del suelo, el pasto, las flores y la cámara, así nada flota ni se hunde y la cámara no atraviesa el piso.
- **Iluminación pintada a mano**: no hay luces de Three.js en la escena. Cada shader resuelve su propio N·L contra la dirección de la luna, más niebla exponencial propia. Es más barato y permite afinar exactamente el look nocturno.
- **Luciérnagas**: `Points` con blending aditivo; cada partícula calcula su órbita y su parpadeo en el vertex shader a partir de atributos propios (amplitud, frecuencia, fase). Cero trabajo por frame en CPU.
- **Luna y nubes**: billboards con texturas generadas en un `<canvas>` al vuelo (degradados radiales, más unos mares suaves en la luna). Las nubes orbitan la escena y se orientan a la cámara, así que a veces cruzan por delante de la luna.
- **Mobile first**: menos densidad y `pixelRatio` limitado a 1.6 en pantallas pequeñas, encuadre inicial distinto en vertical, gestos de arrastre y pellizco, `touch-action: none` y respeto por `env(safe-area-inset-*)`.
- **Cordilleras**: tres anillos de silueta a 206, 330 y 470 unidades. El perfil sale de cinco senoidales sumadas sobre el ángulo y elevado a una potencia, que es lo que afila los picos en vez de dejar lomas. La base de cada anillo se funde con el color de la niebla —exactamente el color al que se desvanece el suelo a esa distancia—, así que no hay costura en el horizonte. Ninguna pasa de ~10° de elevación, para que la luna quede siempre por encima.
- **La canción y los fuegos**: los navegadores no dejan sonar audio sin un gesto del usuario, así que la página intenta reproducir al cargar y, si la rechazan, vuelve a intentarlo con el primer toque, clic o tecla. El cronómetro de 30 s arranca cuando la música *realmente* empieza; si nunca la dejan sonar, a los 12 s arranca igual para que el cielo no se quede vacío.
- **Fuegos artificiales**: un único pool plano de 2 600 partículas. Tipo 1 es un cohete subiendo, tipo 2 una chispa; que el cohete explote es simplemente que se le acabe la vida. Salen en el azimut hacia donde mira la cámara ±65°, así que siempre se ven.
- Si WebGL no arranca, la página muestra un mensaje en lugar de un lienzo negro. `prefers-reduced-motion` congela viento, nubes y giro automático.

## Estructura

```
Jardin-amarillo/
├── index.html          # todo: markup, estilos, escena y shaders
├── Zamba Surreal.mp3   # la canción; el <audio> la busca por este nombre exacto
└── README.md
```

Si cambias el nombre del mp3, hay que cambiar también el `src` del `<audio>` en `index.html` (va URL-encodeado: `Zamba%20Surreal.mp3`).
