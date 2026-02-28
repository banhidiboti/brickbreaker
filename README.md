# Brick Breaker (Vue + Canvas)

A neon-style Brick Breaker game built with Vue 3 and Vite.

## Overview

This project is a browser-based arcade game with:

- Smooth canvas gameplay
- Keyboard and touch controls
- Two game modes: **Classic** and **Endless**
- Special brick mechanics (bonus white balls)
- Pause, restart, and main menu flow

## Game Modes

### Classic Mode

- Start with 3 lives.
- Clear all bricks to win.
- Final score uses a life multiplier at the end of a winning run.

### Endless Mode

- Start with 3 lives.
- No final win condition and no multiplier.
- If a full color row is cleared, that row respawns.
- The run ends only when all 3 lives are lost.

## Controls

### Desktop

- `A` or `←` = Move paddle left
- `D` or `→` = Move paddle right
- `SPACE` or `ESC` = Pause / Resume

### Touch Devices

- Drag directly on the paddle to move it (finger-follow control).
- On smaller screens, a `PAUSE` button appears near the lives HUD.

## Scoring

Brick points by color:

- Red: 30
- Orange: 25
- Yellow: 20
- Green: 15
- Blue: 10
- Purple: 5

## Special Bricks & Bonus Balls

- Some bricks are special (white-outlined).
- Breaking a special brick spawns a white bonus ball.
- White bonus balls:
	- can also break bricks,
	- have their own white trail,
	- do **not** cost a life when they fall,
	- do **not** respawn when they fall.
- Endless mode caps white bonus balls at 5 active balls at the same time.

## Tech Stack

- Vue 3 (`<script setup>`)
- Vite
- HTML5 Canvas (game rendering and logic)

## Project Structure

```
src/
	App.vue
	main.js
	style.css
	components/
		WelcomeScreen.vue
		WelcomeScreen.css
		GameCanvas.vue
		GameCanvas.css
		GameCanvasEndless.vue
```

## Getting Started

### Prerequisites

- Node.js 18+ (recommended)

### Install

```bash
npm install
```

### Run in Development

```bash
npm run dev
```

### Build for Production

```bash
npm run build
```

### Preview Production Build

```bash
npm run preview
```

## Notes

- Fonts and UI style are configured in component CSS files.
- Core game logic (physics, collisions, scoring, modes) lives in `GameCanvas.vue`.
