# orbit-doodle

A drawing toy where the pen orbits your cursor — you steer the center, physics draws the flourishes.

![screenshot](screenshot.png)

*One drawing made with all three pens — drift's wide slow loops, orbit's coils, coil's tight spring — over the control bar: five colours, the pen picker, undo and redo.*

**[Live demo](https://yinggarykairui.github.io/orbit-doodle/)**

## What it does

Press and drag, and a pen circles your pointer, laying down a trail. The orbit center lags behind your hand, so fast strokes stretch into loops and slow ones coil into knots, and the width tapers with speed. Three pens change the physics: `orbit` is the original, `coil` draws a tight fast spring, `drift` swings wide and lazy. Every stroke is kept as a path, not a picture, so undo and redo step through the drawing: the ↶ ↷ buttons, Ctrl/Cmd+Z to undo, Ctrl/Cmd+Shift+Z or Ctrl+Y to redo. Clear is one undoable step, resizing redraws the art instead of stretching it, and five colours plus Save PNG finish the bar; mouse or one finger both work.

## How to run

Open `index.html` in any browser. No build, no install, no network.

## Why it exists

Seeded idea from the factory queue: orbit physics turns clumsy pointer input into flourishes, so anyone can draw something worth keeping. Revisited to add the thing every drawing toy owes you — a way back from a mistake — and, once strokes were recorded instead of painted, two more pens for free.

---

*Day 004 of an autonomous build factory — [factory-hub](https://github.com/yinggarykairui/factory-hub)*
