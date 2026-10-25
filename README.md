# Salt Shack Shapes

An 8-bit surf shop game for the browser. Customers walk in with a request, you shape their board from the blank up, then you surf it at their break while they watch from the sand.

Surfing history is the progression. You start with what ancient Hawaiians and 1950s shapers had (solid wood, balsa, olos, alaias, logs and guns) and earn your way to polyurethane foam, soft-tops, epoxy, eggs, fish, hybrids and shortboards.

**Play:** open `index.html` in any modern browser. Nothing to install, no build step.

## Features

- **Board shaping in 3 steps.**
  1. Pick a shape (9 outlines) and a material (5).
  2. Set length, float and rocker.
  3. Pick a tail (6) and fins (6).

  A live checklist shows what the customer wants, and stats update as you shape: Paddle, Stable, Glide, Turn, Hold, Nose.
- **Physics that respond to the board.**
  - Volume decides whether you catch the wave.
  - Hold decides whether you chatter and spin out.
  - Rocker decides whether you pearl on the drop.
  - Length and weight set how slowly the board turns, so logs feel heavy and shortboards snap.
- **Animated takeoff.** Paddle strokes, the swell standing up into a breaking wall, and a multi-frame pop-up.
- **More than a dozen moves:** bottom turns, off the lips, tail slides, floaters, cutbacks and roundhouses, drop-knee turns, noseriding (hang five, hang ten), spinners, barrels with rail grabs, airs, claims and combos.
- **11 customer types and 4 surf spots:** a mushy beachbreak, a peeling point, a hollow reef and a giant outer reef.
- **Shaper's Notebook:** the history of boards from wood to foam and longboard to shortboard, plus board anatomy and a timeline.
- **Progression:** pay depends on board fit and your surfing. Spend it on new materials, outlines and fins. Progress saves in the browser.
- **No assets:** every pixel is drawn by code and every sound is synthesized with the Web Audio API.

## Controls

| Input | Keyboard | Touch |
|---|---|---|
| Paddle | Space or → (tap fast) | A or ▶ |
| Pop up / climb | ↑ or W | ▲ |
| Drop | ↓ or S | ▼ |
| Pump | → or D | ▶ |
| Stall / cutback | ← or A key | ◀ |
| Noseride (hold), spinner (double tap), air (at the lip), claim | Space | A |

See the in-game **Move List** for every trick.

## Tech

- One file: HTML + CSS + vanilla JavaScript, about 1,500 lines.
- Canvas 2D at 320×180, scaled 2× with pixelated rendering. The wave is rendered pixel by pixel into an `ImageData` buffer every frame.
- Data-driven design: board parts and customers are plain data tables, and one `calcStats()` function turns any board into ride stats.
- Sweetie-16 palette; fonts Press Start 2P and VT323 (Google Fonts).

## Docs

- [Code walkthrough](docs/CODE_WALKTHROUGH.md): the file section by section, how one frame and one wave run, and how to add shapes, customers, tricks and spots.
- [Design notes](docs/DESIGN.md): the game loop, stats, moves, scoring and progression.
