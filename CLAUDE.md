# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Tetris implementado en JavaScript vanilla con HTML5 Canvas. Sin dependencias, sin build, sin `package.json`. Es un proyecto de práctica/curso.

## Running the game

No hay build ni tests. Para probar cambios, abrir `index.html` directamente o servirlo con cualquier servidor estático:

```bash
open index.html                 # macOS, abre el archivo directamente
python3 -m http.server 8000     # o npx serve .
```

No existen linters ni test runners configurados en este repo.

## Architecture

Tres archivos, sin módulos ni bundler — `game.js` se carga como `<script>` clásico y todo vive en el scope global.

- **`index.html`**: dos `<canvas>` — `#board` (300×600, tablero 10×20 a 30px/celda) y `#next-canvas` (preview de la siguiente pieza) — más el panel de HUD (score/lines/level) y el overlay de pausa/game over.
- **`style.css`**: tema dark/retro, sin variables de configuración relevantes para lógica.
- **`game.js`** (~300 líneas): toda la lógica del juego, organizada como funciones puras/casi-puras sobre estado global (`board`, `current`, `next`, `score`, `lines`, `level`, `paused`, `gameOver`, `dropInterval`, etc. — declaradas en la línea 43).

### Flujo del juego

```
init() → createBoard() + next=randomPiece() + spawn() → requestAnimationFrame(loop)

loop(ts)  // basado en rAF, acumula dt contra dropInterval
  ├─ si dropAccum ≥ dropInterval → baja la pieza o lockPiece()
  └─ draw()

keydown → mover / tryRotate() / softDrop() / hardDrop() / togglePause()
```

`spawn()` mueve `next` a `current` y genera una nueva `next`; si la pieza recién generada colisiona de inmediato, se llama a `endGame()`.

### Puntos clave de la lógica (game.js)

- **Piezas**: las 7 piezas estándar están definidas como matrices cuadradas en `PIECES` (línea 18); el índice en la matriz es también el índice de color en `COLORS`.
- **Rotación** (`rotateCW`): transposición + reverso de filas — no usa SRS/kick tables completas, solo wall kicks básicos en `tryRotate()` (desplaza ±1 y ±2 columnas antes de descartar el giro).
- **Colisión** (`collide`): única función que valida límites del tablero y solapamiento; toda la lógica de movimiento/rotación pasa por ella.
- **Líneas** (`clearLines`): recorre el tablero de abajo hacia arriba, usa `splice`/`unshift`; al reincidir en el mismo índice `r` tras eliminar una fila (`r++` dentro del loop) para no saltarse la fila que baja a ocupar su lugar.
- **Puntuación/nivel**: `LINE_SCORES` (línea 29) multiplicado por `level`; el nivel sube cada 10 líneas y `dropInterval = max(100, 1000 - (level-1)*90)`.
- **Ghost piece**: `ghostY()` proyecta la caída hacia abajo reutilizando `collide`; se dibuja con `globalAlpha = 0.2` en `draw()`.

### Al modificar tamaños del tablero

Si se cambia `COLS`, `ROWS` o `BLOCK` en `game.js`, hay que ajustar manualmente `width`/`height` del `<canvas id="board">` en `index.html` (deben ser `COLS × BLOCK` y `ROWS × BLOCK`) — no se calculan dinámicamente.
