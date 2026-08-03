# orbit-doodle

A drawing toy where the pen orbits your cursor — you steer the center, physics draws the flourishes.

![One drawing made with all three pens: drift's wide slow loops in gold, orbit's coils in warm white, coil's tight spring in sky blue, each stroke ending in the curl the pen leaves as it settles. Below it the control bar — five colours, the pen picker, undo and redo, Clear and Save PNG.](screenshot.png)

**[Live demo](https://yinggarykairui.github.io/orbit-doodle/)**

## What it does

Open it and the toy draws one stroke of its own — a faint opening flourish, so you see what the pen does before touching anything. It is a demonstration, not your drawing: undo, redo, Clear and Save PNG all stay dim while it is up, and it never reaches an exported PNG. It goes for good the first time you press to draw, or pick a different colour or pen. Press and drag, and a pen circles your pointer: the orbit center lags your hand, so fast strokes stretch into loops and slow ones coil into knots. Three pens change the physics, five colours change the ink, and every stroke is kept as a path, so undo and redo step through the drawing.

## How to run

Open `index.html` in any browser. No build, no install, no network.

## Why it exists

Orbit physics turns clumsy pointer input into flourishes, so anyone can draw something worth keeping. It has been back twice since: once for undo and redo, which paid for two more pens; and once for the opening flourish, because the demo link used to open on a near-black canvas, the full control bar and one line of text asking for a gesture — and nothing the toy had drawn.

---

*Day 009 of an autonomous build factory — [factory-hub](https://github.com/yinggarykairui/factory-hub)*
