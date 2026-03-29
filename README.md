# Multiplication Table — Abigail

An adaptive, gamified multiplication table practice app built for a child named Abigail (אביגיל).

**Live app:** https://levnir.github.io/multiplication-table/

---

## Features

- Adaptive question selection — harder or slower questions appear more often
- Visible per-question timer with pause/resume (⏸ השהה / ▶ המשך)
- Score, streak, and correct-answer counters
- Heatmap (📊) showing which questions need the most practice
- Two sound modes: fun (crowd cheers / fail sounds) or synthesized tones
- Hebrew RTL layout using the Rubik font
- Mascot image on welcome screen (`opening-image.png`)
- Firebase Realtime Database for cross-device persistence
- localStorage fallback when offline
- Responsive layout: desktop, tablet (portrait & landscape), and small phones

---

## How the Adaptive Algorithm Works

### Weight matrix

Every multiplication pair `(i, j)` for `i, j ∈ {1..10}` has a weight stored in a 10×10 matrix `weights[i][j]`.

Initial weights are set by multiplying two difficulty vectors:

```
INITIAL_DIFFICULTY = [1, 1.2, 1.5, 1.8, 2.2, 3.0, 4.0, 4.5, 4.5, 2.5]
weights[i][j] = INITIAL_DIFFICULTY[i] * INITIAL_DIFFICULTY[j]
```

So `7×8` starts with weight `4.0 × 4.5 = 18`, while `1×2` starts with `1 × 1.2 = 1.2`.

### Question selection — weighted random sampling

Each question is drawn with probability proportional to its weight:

```
P(i, j) = weights[i][j] / sum of all weights
```

Implementation: sum all 100 weights, pick a uniform random number in `[0, total)`, walk through pairs subtracting each weight until the running value drops below zero.

### Weight updates after each answer

`TARGET_TIME = 10` seconds (considered fast for a child).

**Correct on first try:**
```
multiplier = max(0.3, (t / TARGET_TIME) ^ 0.6)
```
- Fast answer (t < 10s): multiplier < 1 → weight decreases → asked less often
- Slow answer (t > 10s): multiplier > 1 → weight increases → asked more often

**Wrong attempt(s) then correct:**
```
multiplier = min(1.5 + wrongAttempts × 0.5 + (t / TARGET_TIME) × 0.2, 4.0)
```
Weight always increases; more wrong attempts and more time = larger increase.

**Clamping:** weights are bounded to `[0.5, 50]`.

### Heatmap

`renderHeatmap()` reads the current `weights` matrix, normalizes values relative to the current min and max, and colors each cell:

```
color = hsl(120 × (1 − normalized), 65%, 50%)
```

- normalized = 0 (lowest weight) → green (well practiced)
- normalized = 1 (highest weight) → red (needs work)

Colors are always relative to each other, so the heatmap always highlights which questions need the most attention compared to the rest.

---

## Persistence

- **Firebase Realtime Database** — weights, score, streak, and correct count are saved after every answer to path `abigail/` in the `multiplication-table-129e3-default-rtdb` database
- **localStorage** — fallback when Firebase is unavailable (`STORAGE_KEY = 'abigail_multitable_v1'`)
- On load: Firebase is tried first; if it fails or is empty, localStorage is used; if both are empty, weights are initialized from scratch
- Stats (score, streak, correctCount) can be manually edited in the Firebase Console under the `abigail/` path

**Firebase SDK:** version 9.6.1 compat — do not upgrade without verifying CDN availability (v10 was missing from gstatic CDN and crashed the app).

**API key security:** The Firebase API key is in client-side code (standard for Firebase web apps). It is restricted in Google Cloud Console to HTTP referrer `https://levnir.github.io/*` and to the Firebase Realtime Database API only.

**Firebase array note:** Firebase stores JS arrays as numeric-keyed objects `{"0":...,"1":...}`. The `toArray()` function in app.js converts them back before use.

---

## Running Locally

Opening `index.html` directly in Chrome as a `file://` URL causes aggressive CSS caching — edits to CSS files may not appear even after refresh. The reliable solution is a local server:

```
cd "C:\Users\USER\Documents\apps\multiplication-table"
python -m http.server 8000
```

Then open `http://localhost:8000`. Changes to any file are visible immediately on refresh. Firebase still works normally.

---

## Files

| File | Purpose |
|------|---------|
| `index.html` | HTML structure, Firebase SDK scripts |
| `app.js` | All game logic — weights, question selection, timer, sound, Firebase |
| `styles.css` | Styling and responsive layout |
| `sounds.json` | Manifest of sound file paths |
| `sounds/success/` | Correct-answer sound clips |
| `sounds/fail/` | Wrong-answer sound clips |
| `opening-image.png` | Mascot image shown on welcome screen (928×1152px) |

---

## Responsive Breakpoints

| Condition | Behavior |
|-----------|---------|
| Base (desktop/computer) | Full-size layout, max-width 480px |
| `orientation: portrait` + `min-width: 600px` | Tablet portrait |
| `orientation: landscape` + `min-width: 600px` | Tablet landscape — compact spacing |
| `max-width: 480px` | Small phone — reduced fonts and spacing |
| `max-height: 680px` | Short/landscape screen — reduced vertical spacing |

---

## Known Bugs Fixed

| Bug | Fix |
|-----|-----|
| Firebase arrays deserialized as plain objects | `toArray()` converter applied to outer and inner weight arrays |
| Heatmap modal visible on page load | `style="display:none"` inline on modal (body is `display:flex`) |
| Firebase SDK v10 not on CDN — all buttons dead | Downgraded to SDK 9.6.1 compat |
| Question equation displayed as `? = 3 × 5` | `direction: ltr` on `.question-equation` |

---

## Hosting

- GitHub repository: https://github.com/levnir/multiplication-table
- Hosted via GitHub Pages from the `master` branch root
- GitHub Pages CDN can lag a few minutes after a push before serving updated files
