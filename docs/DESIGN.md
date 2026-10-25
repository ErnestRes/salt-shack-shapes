# Design notes

Play: open `index.html` in a browser.
One HTML file (~1,450 lines). No libraries, no image or sound files: every pixel is drawn by code and every sound is synthesized.

## The pitch
You own a tiny surf shop. Customers walk in with a request. You shape the board, then surf it at their break while they watch. The board you build is your character build for the surf session, and surfing history is the progression: you start with ancient Hawaiian wood and 1950s balsa and earn your way to foam, epoxy and the shortboard.

## Core loop
1. **Customer.** Each of 11 customer types has a quote, a home break, a wave size and a budget. New types appear as you unlock gear.
2. **Shape (3 steps).**
   - Step 1: shape (9 outlines, olo to fish) and material (wood, balsa, PU foam, soft-top, EPS/epoxy).
   - Step 2: length, float (width + thickness together), rocker.
   - Step 3: tail (6) and fins (6). Rails come from the outline automatically.
   - The right side updates live: the customer's checklist (✓ ~ ✗), a board-fit %, six stats (Paddle, Stable, Glide, Turn, Hold, Nose), size, volume, weight and blank cost.
3. **Surf three waves** at the customer's break. Board stats drive every part of the physics.
4. **Review.** Stars = 80% board fit + 20% surf score. Pay scales with fit; tip scales with wave scores.
5. **Supply Order.** Spend money to unlock materials, outlines and fins, which bring in new customers. Goal: 4★ from every customer type (Quiver Log).

## Surf session, phase by phase
- **Paddle:** a swell rolls in from the left. Tap A/Space/→ to stroke (arm animation, splash, wake). Paddle speed must pass the NEED line when the swell reaches you. The need line rises with wave size; paddle power comes from volume and length.
- **Pop:** "CAUGHT IT!" The swell lifts you and stands up into a wall (the wave shape morphs and the curl dissolves in). Press ↑ before the timer bar empties; more stable boards give a longer window.
- **Takeoff:** about half a second of pop-up animation: push-up, tuck the feet, rise, stand. The board drops down the face. Then a takeoff check: flat rocker on a steep or hollow wave can pearl; a very tippy board can lose balance.
- **Ride:** about 20 to 26 seconds (15 to 19 at The Bombie).
- **End:** kick out over the back (tap A to claim), or wipe out (board flies, rider tumbles, then bobs up). Then paddle back out for the next wave.

## Moves
| Move | How |
|---|---|
| Bottom turn | Drop to the bottom, press ↑ |
| Top turn / Off the lip | Climb to the top, press ↓ (near the curl = off the lip) |
| Tail slide | Hold ← through a top turn (needs Turn 5+) |
| Floater | Hold ↑ along the lip near the curl, come down before you stall |
| Cutback / Roundhouse | Hold ← on the shoulder; go in fast for a roundhouse |
| Drop-knee turn | Boards 8 ft+: hold A through a bottom turn |
| Noseride, hang five, hang ten | Hold A mid-face; release before the NOSE meter fills |
| Spinner | Double-tap A mid-face (stable/long boards land it) |
| Barrel + grab rail | Hollow waves: stall into the curl mid-face; hold A to go deeper |
| Air | Hit the lip fast going up, press A |
| Claim | Tap A while kicking out |
| Combo | Chain moves within 2.5 s: +12% per move up to ×1.72 |

Pocket bonus ×1.5 near the curl; shoulder ×0.6. Customer style multipliers (a log purist pays double for noseriding, a grom for airs). Wave size multiplier up to ×2.2.

## How board stats change the ride
- **Paddle:** whether you catch the wave at all.
- **Stable:** pop-up window, takeoff balance, spinner landings.
- **Glide:** trim speed, the speed the board settles at without pumping. High glide outruns the curl on its own; low glide needs pumping.
- **Turn:** pump strength, trick points, how fast you can move up and down.
- **Hold:** the speed limit before chatter and spin-out, how deep you can sit in the barrel.
- **Nose:** noseride points and how fast the NOSE strain meter fills.
- **Length and weight:** steering inertia. A 9'6" log takes about 4× longer than a shortboard to change direction; an olo is slower still.

## Code map (top to bottom of the file)
1. **HTML/CSS shell:** header chips (cash, day, rep), a 640×360 canvas that draws a 320×180 pixel world at 2× with `image-rendering: pixelated`, and an HTML panel underneath for menus and controls. Sweetie-16 palette as CSS variables.
2. **Basics:** palette, `rect()` and `text()` helpers, a hash function for noise, inch/fraction formatting.
3. **Sound:** Web Audio square/triangle bleeps and a noise burst for wipeouts.
4. **Data tables:** `MATERIALS`, `TEMPLATES`, `TAILS`, `FINS`, `RAILS`. Each entry carries stat modifiers, a tip and an unlock cost. `calcStats(board)` turns a board into volume, weight and the six 0–10 stats.
5. **Customers:** `SPOTS` and `ARCH`. Each archetype has a quote, spot, price, style multipliers, an unlock requirement and criteria built from `C.stat`, `C.len` and `C.is`, each with a weight and good/bad review lines. The same criteria drive the shaping checklist and the final review.
6. **Notebook data:** the history text (Wood → Foam, Long → Short, Anatomy, Timeline).
7. **Save:** one object `S` (cash, day, unlocks, ratings, rack, current customer and board) stored in localStorage.
8. **Board drawing:** `halfFrac()` gives the outline's half-width at each point from tail to nose. It uses the template's wide point, nose roundness and tail fullness, then reshapes the tail end for pin, round, squash, diamond or swallow. Boards are drawn pixel by pixel in each material's pattern (wood grain, balsa planks, PU stringer, EPS beads, soft-top waffle) and cached as small canvases. A side profile shows rocker, thickness foil and fins.
9. **People:** `drawPerson()` builds sprites from rectangles in about 20 poses (walk, paddle frames, push-up, tuck, surf, noseride, cross-step, drop-knee, spinner, tube, grab, claim, tumble, swim).
10. **Screens:** title, shop, shaping bay, notebook lineup, results, supply order. Each has a canvas draw function plus an HTML panel function. `setScreen()` switches both.
11. **Surf engine:**
    - **State machine:** paddle → pop → takeoff → ride → rideEnd/wipe → between, repeated for three waves, then `endSession()`.
    - **State:** `p` is height on the face (0 trough, 1 lip), `vp` is vertical velocity, `v` is speed, and `d = x − cx` is the distance ahead of the curl.
    - **Gravity and drag:** dropping turns face height into speed, and drag pulls speed toward trim speed (glide × how close you are to the pocket).
    - **Speed inputs:** pumps add bursts of speed, and stalling bleeds it off.
    - **Chatter:** going over the hold limit builds chatter, which ends in a spin-out.
    - **Curl:** falling behind the curl means a wipeout, except in the barrel zone on hollow waves.
    - **Tricks:** detected from direction changes, lip contact and button timing (hold vs. double tap).
12. **Wave rendering:** `renderSea()` writes every pixel of a 320×180 `ImageData` each frame: banded, dithered sky with a striped sunset sun; parallax headland and sea glints; face bands; the curl arc with lip and barrel; churning whitewater noise. During takeoff, `morph` blends the swell's outline into the wall's, and curl pixels dissolve in.
13. **Main loop and input:** `requestAnimationFrame` with capped `dt`. Keyboard and on-screen pad feed `keys` (held) and `pressed` (just pressed this frame).
14. **Scoring:**
    - Judge score = 10 × (1 − e^(−points/1900)).
    - Stars = (fit × 0.8 + avg/10 × 0.2) × 5, in half steps.
    - Pay = price × (0.2 + 0.8 × fit).
    - Tip = sum of judge scores × 3 × stars/5.

## Progression
- **Start:** Solid Wood and Balsa; Olo, Alaia, Longboard and Gun outlines; Finless and Single fins.
- **Unlocks:**

| Cost | Unlocks |
|---|---|
| $120 | Mini-Mal, 2+1 |
| $150 | Egg, Twin |
| $180 | Soft-Top |
| $200 | Fish |
| $220 | Quad, Hybrid |
| $250 | PU, Thruster |
| $300 | Shortboard |
| $450 | EPS |

## Ideas for next versions
- Deck art and resin tints, custom shop name.
- Chiptune music loop, more spots, tide and wind.
- Shared high-score board.
- A human playtest pass to tune difficulty.
