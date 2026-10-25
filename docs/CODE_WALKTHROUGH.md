# Code walkthrough

Everything lives in **`index.html`** (about 1,500 lines). There is no build step, no framework, no image files and no sound files: the pixel art is drawn by code and the sound is synthesized. Open the file in a browser and it runs.

This guide goes through the file top to bottom, then traces what happens in a single frame and over a single wave, and ends with recipes for extending the game. Line numbers are approximate and refer to `index.html`.

---

## 1. The page shell (lines 1–146)

```html
<header class="top"> … cash / day / rep chips … </header>
<div class="stage"><canvas id="cv" width="640" height="360"></canvas></div>
<section class="panel" id="panel"></section>
```

The page has three parts:

- **Header chips:** cash, day, reputation, a Notebook button and a sound toggle.
- **The stage:** one canvas, 640×360 real pixels.
- **The panel:** an HTML area under the canvas for menus, sliders, review text and the touch D-pad.

Menus, forms and text are much easier and more accessible in HTML, while the canvas is better at fast pixel art. The game uses each for what it's good at.

**CSS (lines 6–135)**

- The 16-colour **Sweetie-16** palette is defined once as CSS variables (`--k`, `--sun`, `--aqua`…). The same 16 hex values are repeated in JavaScript as `P`, so the HTML panel and the canvas always match.
- Two Google Fonts: **Press Start 2P** for headings, buttons and canvas text, and **VT323** for readable body text.
- `canvas { image-rendering: pixelated }` keeps the art crisp when the browser scales it.
- `.stage` sizes itself from the viewport height so the game and its panel fit on a laptop screen.

---

## 2. Basics (lines 147–176)

The whole script is wrapped in `(() => { 'use strict'; … })();` so nothing leaks into the global scope.

```js
const W = 320, H = 180, TY = 164, HY = 98, RX = 172;
function resetT(){ ctx.setTransform(2,0,0,2,0,0); ctx.imageSmoothingEnabled = false; }
```

The **game world is 320×180**, a classic low resolution. Every frame starts with `resetT()`, which scales the canvas by 2, so all drawing code thinks in 320×180 units and each game pixel becomes a 2×2 block on screen.

Named constants used throughout:

| Constant | Meaning |
|---|---|
| `TY` | Y of the wave trough (bottom of the face) |
| `HY` | Y of the horizon |
| `RX` | Fixed screen X of the surfer (the camera follows them) |

Helpers:

- `rect(x,y,w,h,color)` is the main drawing primitive. It rounds to whole pixels, which keeps the 8-bit look.
- `text(...)` draws pixel-font text with a 1-pixel drop shadow.
- `hash(x,y,f)` is a cheap integer hash that turns coordinates into pseudo-random numbers. It drives the whitewater sparkle and EPS foam beads. Because it's deterministic, foam can "boil" per frame (change `f`) without storing any state.
- `fmtLen(114)` returns `9'6"`. `frac(2.375)` returns `2 3/8`, matching how shapers write dimensions.
- `u32(hex)` packs a colour into one 32-bit integer for direct pixel writes (see `renderSea`).

---

## 3. Sound (lines 177–202)

```js
function beep(f=440, d=0.08, type='square', vol=0.05, slide=0, delay=0) { … }
function noise(d=0.5, vol=0.08) { … }
const sfx = { click, paddle, pop, trick, big, wipe, cash, plane, pump };
```

All sound comes from the Web Audio API.

- **`beep`** makes one oscillator note. The square wave is the classic NES sound, and `slide` bends the pitch for "zip" effects.
- **`noise`** fills a buffer with random values that fade out, which sounds like a wave crashing on you.
- **`sfx`** names every sound effect. Trick sounds are rising three-note arpeggios that get higher for bigger point values.

Browsers only allow audio after the player interacts, so `ac()` creates the audio context lazily the first time something is clicked or pressed.

---

## 4. Game data: board parts and stats (lines 203–280)

This is the heart of the design. Every board part is a plain data object holding **stat modifiers**, a **tip** and an **unlock price**.

```js
const MATERIALS = {
  wood:  { name:'Solid Wood', density:1.1, cost:20, unlock:0, glide:0.4, hold:0.3, turn:-0.5, paddle:-0.2, … },
  eps:   { name:'EPS + Epoxy', density:0.17, cost:70, unlock:450, paddle:0.6, glide:0.5, hold:-0.8, turn:0.3, … },
};
const TEMPLATES = {
  log: { name:'Longboard', min:108, max:126, vf:0.56, noseA:4.5, tailW:0.72, wp:0.5, noseF:2.5,
         def:{ L:114, W:23, T:3, rocker:2, tail:'square', fins:'single', rails:'soft' }, … },
};
```

What each table holds:

| Table | Contents |
|---|---|
| `MATERIALS` | Density (lb per liter, so weight can be computed), blank cost, unlock price and stat modifiers |
| `TEMPLATES` | Length range in inches, default dimensions, and outline numbers used by the drawing code |
| `TAILS`, `FINS`, `RAILS` | Just stat modifiers and tips |

The template outline numbers are:

- `wp`: where the wide point sits.
- `noseA`: how round the nose is (high means blunt).
- `tailW`: how full the tail is.
- `vf`: a volume factor (how "boxy" the shape is).

### `calcStats(board)` (line 263)

This one function turns any combination of parts into numbers the game can use.

```js
const vol    = b.L * b.W * b.T * 0.016387 * tp.vf;   // cubic inches → liters, scaled by the outline
const weight = vol * m.density + (b.mat==='wood' ? 0 : 1.5);
s.paddle = Math.sqrt(vol)*0.75 + (ft-6)*0.4 - Math.max(0,weight-22)*0.03 + (m.paddle||0);
s.turn   = 8 - (ft-5)*0.8 + (b.rocker-3)*0.6 + tl.turn + f.turn + r.turn - weight*0.03 + (m.turn||0);
…
for (const k in s) s[k] = Math.round(clamp(s[k],0,10)*10)/10;
```

Each stat is a readable linear formula. For example, Turn gets worse with length and weight, better with rocker, and then each part adds or subtracts. `Math.sqrt(vol)` gives diminishing returns, so doubling volume doesn't double paddle power. Everything is clamped to 0–10.

The formulas encode real shaping knowledge:

- Long, flat boards glide.
- Pin tails and narrow outlines hold at speed.
- Wide, soft-railed, single-fin logs noseride.
- Twin fins turn loosely but don't hold.

Because it's all data, rebalancing the game means editing numbers, not logic.

---

## 5. Spots and customers (lines 281–386)

```js
const SPOTS = { cove:{ kind:'mushy', size:[2,3], sky:'day' }, reef:{ kind:'hollow', … }, … };
const C = {
  stat: (k,min,w,good,bad) => ({ w, good, bad, stat:k, target:min,
          f:(b,s) => s[k]>=min ? 1 : clamp(1-(min-s[k])/3.5, 0, 1) }),
  len:  (lo,hi,w,good,bad) => ({ … }),
  is:   (field,vals,w,good,bad) => ({ … f: b => vals.includes(b[field]) ? 1 : 0 }),
};
```

Customers (`ARCH`, the archetypes) are built from **criteria factories** (`C`). Each factory returns an object with:

- **`w`**: a weight (how much the customer cares).
- **`f`**: a scoring function returning a score from 0 to 1.
- **`good` / `bad`**: the review lines to use.

`C.stat` gives partial credit: missing the target by 3.5 points scores zero.

```js
{ id:'charger', title:'Big-Wave Charger', spot:'bombie', price:390,
  ask:'The Bombie is showing eighteen feet…',
  style:{ make:2, tube:1.5, snap:0.8 },
  req: () => true,
  crit:[ C.stat('hold',8,3,'It held its line at insane speed.','It chattered and spun out at speed.'), … ] }
```

The fields:

- **`style`**: multiplies trick points during the surf session. A charger values making the drop; a grom values airs.
- **`req`**: decides whether this customer can appear, based on what you've unlocked. Unlocking the Shortboard is what makes the Comp Grom walk in.

The same `crit` list is reused in three places:

1. The live checklist while you shape.
2. The board-fit percentage.
3. The final review lines.

So the customer always judges you exactly the way the checklist predicted.

---

## 6. Notebook (lines 387–480)

`NOTEBOOK` is pure content: materials (wood → foam), shapes (long → short), anatomy, and a timeline. Each entry also has an "In the shop:" line linking the history to a game stat. Keeping research text out of the logic makes it easy to edit or translate.

---

## 7. Save system and global state (lines 481–498)

```js
function freshSave(){ return { money:150, day:1, unlocked:{ mat:['wood','balsa'], … }, ratings:[], served:{}, rack:[], … }; }
function save(){ try{ localStorage.setItem(KEY, JSON.stringify(S)); } catch(e){} }
```

All persistent progress lives in one object, **`S`**: money, day, unlocks, ratings, the board rack, the current customer and board, and the last result. `save()` serialises it to `localStorage`.

Every storage call is wrapped in `try/catch` because private browsing can block storage, and the game should still run.

`load()` merges the saved data over `freshSave()`, so older saves gain any new fields automatically.

Session (non-saved) state:

- `screen`: which screen is showing.
- `tick`: seconds since start.
- `surf`: the active surf session, or `null`.

Input uses two objects:

```js
const keys = { up:0, down:0, left:0, right:0, a:0 };  // what's held right now
const pressed = {};                                     // what went down THIS frame
function consume(k){ const v=!!pressed[k]; pressed[k]=false; return v; }
```

`keys` answers "is ↑ held?", which steering needs. `pressed` answers "was A just tapped?", which paddling, pumping and double-taps need. `consume()` reads a tap and clears it, so one tap can't trigger two things.

---

## 8. Board drawing (lines 499–572)

### The outline: `halfFrac(b, t)` (line 500)

`t` runs from 0 at the tail to 1 at the nose. The function returns how wide the board is at that point, as a fraction of its maximum width.

```js
if (t >= wp) { const u=(t-wp)/(1-wp); s = Math.sqrt(Math.max(0, 1 - Math.pow(u, tp.noseA))); }  // nose
else         { const u=(wp-t)/wp;     s = 1 - (1-tp.tailW)*u*u; }                              // tail
switch (b.tail) { case 'pin': if (t<0.22) s *= Math.pow(t/0.22, 0.75); break; … }               // tail end
```

- **Nose half:** a superellipse curve. A big `noseA` (4.5 for a log) stays wide almost to the tip, giving a round nose. A small one (1.3 for a shortboard) tapers early to a point.
- **Tail half:** a parabola down to the template's tail width.
- **Tail end:** reshaped by the tail type. Pin tapers to a point, round adds a curve, squash and diamond clip the corners, and swallow is cut later as a notch.

### Painting it: `boardCanvas()` (line 518)

The board is drawn **one pixel at a time** along its length. Each pixel gets a colour depending on the material:

| Material | Pattern |
|---|---|
| Wood | Wavy grain lines (`Math.sin` wobble) |
| Balsa | Glued plank lines |
| PU foam | White with an orange wooden stringer down the middle |
| EPS | Scattered "beads" from `hash()` |
| Soft-top | Blue waffle deck with a white rail band |

Drawing thousands of 1-pixel rectangles every frame would be slow, so each finished board is **cached in a small off-screen canvas** under a key made of its visible properties. Later frames reuse the image with one `drawImage`. The cache is cleared if it grows past 80 entries.

### Side profile: `rockerAt()` and `drawProfile()` (line 553)

`rockerAt(b, t)` returns how far the bottom curves up at each point. Nose rocker is steeper than tail rocker, and both grow with the Rocker slider. Thickness tapers toward both ends (the "foil"). Fins are small raked triangles at real positions for single, 2+1, twin, thruster or quad.

---

## 9. People (lines 573–658)

```js
function drawPerson(x, y, lk, pose='stand', f=0, flip=false) { switch(pose){ … } }
```

Every character is built from a handful of `rect` calls: legs, shorts, shirt, arms, head and hair. `lk` (the "look") holds four colours, so customers get random skin, hair, shirt and shorts from small palettes.

There are about 20 poses: walk, paddle (4 stroke frames), push-up, tuck, rise, surf, tube, grab, noseride, cross-step, drop-knee, spinner, claim, tumble, swim and others. Animation is just `f` (time) picked into frames with `Math.floor(f*8) % 4`. The spinner reuses the surf pose with `flip` toggled for the side views.

---

## 10. Screens (lines 659–820)

Each screen has a **canvas draw function** (scenery) and a **panel function** (HTML, section 12).

| Screen | Canvas | What's going on |
|---|---|---|
| Title | `drawTitle` | Sunset bands, striped retro sun, a tiny surfer riding a looping wave |
| Shop | `drawShop` | Wall, window with an animated sea, your last 7 boards on the rack, counter, and the customer walking in from the door |
| Shaping bay | `drawShape` | Side lights (real shaping rooms use them to show the curves), the board top-down at a fixed 1.3 px per inch so a 15-ft olo and a 5-ft fish read at true relative size, rocker profile, ruler, falling foam dust when you change something |
| Notebook | `drawLineup` | Every template standing in a row from longest to shortest; locked ones as grey silhouettes |
| Results | `drawResults` | Beach scene; stars fill in half a star at a time |

`newCustomer()` (line 660) picks a customer type you can serve. It weights types you haven't served yet three times more heavily, never repeats the last type, and always makes the very first customer a beginner so the first board is easy.

---

## 11. The surf engine (lines 821–1223)

This is the most interesting code in the game.

### Setup: `startSession()` (line 831)

It turns the board's stats and the spot into session constants:

```js
cs:   (34 + sz*3.2) * km,            // curl speed: how fast the breaking part chases you
Hmax: clamp(34 + sz*5.5, 40, 128),   // face height in pixels
need: (22 + sz*2.6) * …,             // paddle speed needed to catch it
resp:     clamp((2 + st.turn*0.55) / (1 + Math.max(0,ft-6)*0.22 + r.weight/60), 0.6, 8),
turnRate: (0.75 + st.turn*0.14) / (1 + Math.max(0,ft-7)*0.12),
```

`resp` and `turnRate` are why longboards feel slow:

- **`resp`** is how quickly the board reacts to input.
- **`turnRate`** is the top speed of moving up and down the face.

Both are divided by length and weight. The results:

| Board | Response | Turn rate |
|---|---|---|
| Shortboard | 6.4 | 2.1 |
| Log | 1.4 | 0.75 |
| Olo | 0.6 | 0.38 |

### The state machine: `updateSurf(dt)` (line 885)

Each wave moves through phases. Every frame, the code for the current phase runs.

```
paddle ──caught──▶ pop ──↑ pressed──▶ takeoff ──▶ ride ──time up──▶ rideEnd ─┐
   │                │                                │                       ├─▶ between ─▶ next wave
   └─missed         └─too slow──▶ wipe ◀──fell────────┘                       │   (after 3 waves: endSession)
         └───────────────────────────────────────────────────────────────────┘
```

- **`paddle`:** the swell (`sw` = its x position, `swA` = its height) rolls right. Each tap adds paddle speed `ps`, and `ps *= Math.exp(-0.9*dt)` makes it fade. If `ps ≥ need` while the swell is under you, you catch it.
- **`pop`:** the swell grows and `morph` rises toward 0.7, so the bump starts turning into the wall. Press ↑ before the `popWin` timer ends (longer for stable boards).
- **`takeoff`:** about 0.55 s. `morph` reaches 1, the rider slides down the face, and the pose changes every ~0.14 s (push-up → tuck → rise → surf). At the end, a dice roll checks for a pearl (flat rocker on steep waves) or losing balance (tippy boards).
- **`ride`:** handled by `updateRide()`.
- **`rideEnd` / `wipe`:** short animations, then `finishWave()` records the score.

### The ride physics: `updateRide(dt)` (line 941)

The whole simulation runs on four numbers:

| Var | Meaning |
|---|---|
| `p` | Height on the face. 0 is the trough, 1 is the lip. |
| `vp` | Vertical speed. Positive is climbing. |
| `v` | Speed along the wave, in pixels per second |
| `d = x - cx` | How far you are ahead of the curl. Negative means it has caught you. |

**Steering with inertia:**

```js
const target = keys.up ? sv.turnRate : keys.down ? -sv.turnRate : -0.18;  // no input: drift down
sv.vp += (target - sv.vp) * Math.min(1, dt * sv.resp);                   // ease toward the target
sv.p  += sv.vp * sv.pScale * dt;
```

`vp` eases toward where you're steering, so a low `resp` (longboards) means it eases slowly. This one line creates the heavy feel.

**Gravity, trim and drag:**

```js
sv.v += -sv.vp * sv.pScale * Hc * gK * dt;                    // going down → faster, going up → slower
let trim = sv.cs * (0.72 + st.glide*0.05) * pf * Math.sqrt(sv.power);
sv.v += (trim - sv.v) * dt * 0.6;                             // drag pulls speed toward "trim"
```

- **Gravity:** dropping converts height into speed, like a skateboarder in a halfpipe.
- **Trim:** the speed the board naturally settles at. It comes from Glide and `pf`, which is highest in the pocket and falls off on the shoulder.
- **The key contrast:** a high-glide log's trim is faster than the curl, so it outruns it on its own. A shortboard's trim is slower, so you must pump (→ adds a burst of speed) and use drops.

**Chatter (spin-outs):**

```js
const vmax = trim + sv.Hmax * gK * (0.4 + st.hold*0.07);
if (sv.v > vmax) sv.chatter += (sv.v - vmax)/25 * dt * 1.3; else sv.chatter = Math.max(0, sv.chatter - dt*0.7);
if (sv.chatter > 1) wipe('Spun out…');
```

Hold sets your speed limit. Go over it and a meter fills; if it fills, you spin out. It's a meter rather than an instant failure so you can see it coming and react by climbing or stalling.

**Getting caught:** if `d` drops below zero you're behind the curl, and you wipe out. On hollow waves, a slice behind the curl (`-depth < d < 8`, mid-face) counts as **the barrel** and scores points per second instead.

### How tricks are detected

Most tricks come from **when your vertical direction changes**:

```js
const ns = sv.vp > 0.3 ? 1 : sv.vp < -0.3 ? -1 : 0;   // dead zone stops jitter
if (ns !== 0 && ns !== sv.dir) {
  if (sv.dir===1 && ns===-1 && sv.p>0.68 && sv.p-sv.ext>0.4) → TOP TURN / OFF THE LIP (or TAIL SLIDE if ← held)
  if (sv.dir===-1 && ns===1 && sv.p<0.32 && sv.ext-sv.p>0.4) → BOTTOM TURN (or DROP-KNEE if A held on a long board)
  sv.ext = sv.p; sv.dir = ns;
}
```

`ext` remembers where the last turn happened. Requiring at least 40% of the face between turns stops players from earning points by wiggling.

The other moves:

| Move | Detection |
|---|---|
| Floater | Holding ↑ at `p = 1` near the curl; scored when you come back down, wipeout if you hang too long |
| Air | Hitting the lip going up, faster than the curl, with A pressed in the last 0.3 s; landing chance from Turn, length and stability |
| Noseride | A held over 0.18 s while trimming mid-face; points per second, and the NOSE strain meter fills faster on boards with a low Nose stat |
| Spinner | Two A taps under 0.2 s each, within 0.4 s; 1.2 s cooldown and diminishing points so it can't be spammed |
| Cutback / Roundhouse | ← held on the shoulder (`d > 115`) while still fast |
| Grab rail | A held in the barrel: you can sit 8 px deeper and score ×1.35 |

### Scoring: `award()` (line 863)

```js
let m = pocketMult(d) * (sv.style[kind]||1) * sv.sizeMult;     // pocket ×1.5, customer taste, big waves
if (kind!=='make' && kind!=='claim') { m *= 1 + Math.min(sv.combo,6)*0.12; sv.combo++; sv.comboT = 2.6; }
```

One function handles every trick, so all multipliers apply consistently:

- **Pocket:** ×1.5 near the curl.
- **Customer style:** what this customer values.
- **Wave size:** up to ×2.2.
- **Combo:** grows if you chain moves within 2.6 s.

The wave's judge score is `10 × (1 − e^(−points/1900))`. Early points count most and the score levels off near 10, so one huge trick can't max it out.

### Drawing the wave: `renderSea()` (line 1065)

Instead of many shape-drawing calls, this function **writes every pixel itself** into a 320×180 `ImageData` buffer (`pix`, a `Uint32Array`), then copies it to the screen once per frame. That's 57,600 pixels per frame, which is fast in plain JavaScript.

For each column `x` it works out one of three regions:

1. **Wave face** (ahead of the curl): the lip height comes from `faceH()`, which tapers onto the shoulder. Pixels below the lip get banded colours by depth (white lip → aqua → sky → blue → navy), with **checkerboard dithering** between bands. That dithering is the classic 8-bit way to fake gradients with few colours. Diagonal streaks offset by the camera make the water look like it's moving.
2. **Curl** (just behind the curl): a quarter-ellipse arc is the pitching lip. Under it is a dark barrel (hollow waves) or churning foam.
3. **Whitewater** (further behind): randomly boiling white and mist pixels from `hash()`.

Above the water: sky rows (precomputed per spot), a striped sunset sun, a parallax headland and sea glints.

**The takeoff morph.** While the swell stands up, each column's top edge is a blend:

```js
lip = bump * (1 - m) + wall * m;
const curlOn = wMode !== 0 && (m >= 1 || (hash(x,y,3) % 100) < dis);  // curl pixels dissolve in
```

So the rolling bump smoothly reshapes into the breaking wall, and the curl appears through a pixel dissolve rather than snapping in.

### Sprites and HUD: `drawSurf()` (line 1114)

Draws clouds, the board (a short line of pixels rotated to `sv.ang`, which eases toward the riding angle, slower on long boards), and the rider in the right pose for the moment. On top of that:

- **Spray particles:** stored in world coordinates so they stay put as the camera moves.
- **Floating trick text.**
- **The HUD:** wave count, points, speed bar, pocket, combo, chatter, nose and barrel meters, plus time left in the wave.
- **One-time hints** and the end-of-wave boxes.

### Results: `endSession()` (line 1199)

It scores the board against the customer's `crit` list, then:

- **Stars** = (fit × 0.8 + average wave score / 10 × 0.2) × 5, rounded to half stars.
- **Pay** = price × (0.2 + 0.8 × fit).
- **Tip** = total judge scores × 3 × stars/5.
- **Review:** picks the two worst criteria's `bad` lines and the two best `good` lines.
- **Saving:** updates money, ratings, the rack and the Quiver Log, then saves.

---

## 12. HTML panels (lines 1224–1428)

- `setScreen(s)` switches screens: sets `screen`, spawns a customer if entering the shop, re-renders the panel and chips, and clears held keys.
- `renderPanel()` rebuilds the panel's `innerHTML` from a template-string function per screen (`panelTitle`, `panelShop`, `panelShape`, `panelSurf`, `panelResults`, `panelUpgrades`, `panelNotebook`). Afterwards it puts keyboard focus back on the element that had it, so keyboard users don't lose their place.
- Every clickable element has `data-act="…"`, which is how events find it (section 13).

**Shaping panel.**

- `optGroup()` builds button groups. Locked items appear dashed with their price.
- `panelShape()` lays out the three steps.
- `readoutHTML()` is the right column: the checklist (✓ ~ ✗ from each criterion's score), the board-fit bar, stat bars with gold outlines marking the customer's targets, and the specs.

While a slider moves, only the readout is re-rendered, not the whole panel, so the slider you're dragging isn't destroyed mid-drag.

`setFloat()` turns one "Float" slider value (−2 to +2) into width +0.5" and thickness +0.25" per step from the template's default. This is the simplification that replaced separate width and thickness sliders.

---

## 13. Events and input (lines 1429–1493)

```js
panel.addEventListener('click', e => { const el = e.target.closest('[data-act]'); … handleAct(el.dataset.act, el); });
```

**Event delegation:** one listener on the panel handles every button, even those created later by `innerHTML`. `handleAct()` is a `switch` over actions: `start`, `toshape`, `set` (pick a part), `glass` (pay and start surfing), `buy`, `endsurf`, notebook tabs and so on.

- **Sliders:** an `input` listener reads `data-r`, updates `S.board` and refreshes the readout.
- **Touch D-pad:** `pointerdown` calls `press(key)`, and `pointerup` / `pointercancel` / `lostpointercapture` call `release(key)`. `setPointerCapture` keeps a held button held even if your thumb slides off it.
- **Keyboard:** `KEYMAP` maps arrows/WASD to directions and Space/Z/X/J/K to A. `preventDefault()` only runs during surfing, so arrow keys still move sliders in the shaping screen.
- **Window blur:** clears all keys, so switching tabs can't leave ↑ "stuck".

---

## 14. The main loop (lines 1494–1516)

```js
function frame(now){
  const dt = Math.min(0.05, (now - lastT)/1000);   // seconds since last frame, capped
  resetT();
  switch(screen){ case 'surf': updateSurf(dt); drawSurf(tick); break; … }
  for (const k in pressed) pressed[k] = false;     // taps only last one frame
  requestAnimationFrame(frame);
}
```

- `requestAnimationFrame` runs about 60 times a second, in sync with the display.
- Everything moves by `dt` (time elapsed), so the game runs at the same speed on 60 Hz and 120 Hz screens.
- `dt` is capped at 0.05 s, so returning to a background tab doesn't teleport the surfer.
- Drawing is wrapped in `try/catch` so one bad frame logs an error instead of freezing the game.

---

## Trace: one frame of surfing

1. The browser calls `frame(now)`, and `dt` ≈ 0.0167 s.
2. `updateSurf(dt)` ages particles and pop-up texts, then dispatches on `phase` and calls `updateRide(dt)`.
3. `updateRide` reads `keys` and `pressed`:
   - Updates the A-button timers.
   - Eases `vp`, then moves `p`.
   - Checks the lip (air, floater) and direction changes (turns).
   - Applies gravity, trim, pumps and stall to `v`.
   - Updates chatter.
   - Moves `x` and the curl `cx`.
   - Updates the noseride, barrel and whitewater checks.
4. `drawSurf()` calls `renderSea()` (57,600 pixel writes, one `drawImage`), then draws the board, rider, particles and HUD on top.
5. `pressed` is cleared and the next frame is requested.

## Trace: one wave

1. `beginWave()` resets the per-wave state.
2. You tap A; `ps` climbs; the swell arrives with `ps ≥ need`, and `phase = 'pop'`.
3. You press ↑ and `phase = 'takeoff'`. The pop-up animation plays, the pearl and balance check passes, and `phase = 'ride'`.
4. You surf for about 22 s. Each trick goes through `award()`.
5. `waveT > waveDur`, so `phase = 'rideEnd'`; you hop over the back and tap A to claim.
6. `finishWave()` pushes `{pts, judge}` into `scores`. After the third wave, `endSession()` builds the review.

---

## Recipes: extending the game

**Add a new board shape (for example a "Bonzer")**

1. Add an entry to `TEMPLATES`: name, `min`/`max` length in inches, `vf`, outline numbers (`noseA`, `tailW`, `wp`), `noseF`, an `unlock` price, `def` dimensions and a `tip`.
2. Add its key to `TPL_ORDER` (lineup order) and `HIST_MAT` (material used in the lineup).
3. Optionally add a notebook entry under `NOTEBOOK.shapes` with `tpl:'bonzer'`.

The shaping buttons, Supply Order card, lineup and stats all pick it up automatically.

**Add a new customer**

Push an object into `ARCH` with: `id`, `title`, `spot`, `price`, `names`, `ask`, `style`, `req`, and a `crit` list built with `C.stat`, `C.len` and `C.is`. The checklist, review lines and Quiver Log all come from that object.

**Add a new trick**

Inside `updateRide`, detect the input or situation and call `award('mykind', basePoints, 'MY TRICK', P.lime)`. Give it a `pose` (`sv.pose='…'; sv.poseT=0.4`) and add a line to `MOVES` so it appears in the move list. Customers can value it through `style:{ mykind: 2 }`.

**Add a new surf spot**

Add an entry to `SPOTS` (`kind`: mushy, clean or hollow; a size range; a sky key) and, if you like, a new palette in `SKIES`.

**Tune difficulty**

Most of the game feel comes from a few numbers in `startSession` (`cs`, `Hmax`, `need`, `resp`, `turnRate`) and `updateRide` (the trim factor, the `vmax` hold formula, nose strain rates).
