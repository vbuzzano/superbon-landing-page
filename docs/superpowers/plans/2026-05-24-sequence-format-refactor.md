# SEQUENCE Format Refactor Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Refactor the `SEQUENCE` array entries to use a normalized, fully-documented per-entry format with smart defaults, add a `forceImpact` flag decoupled from `forceLevel`, then uncomment and lightly enhance the full sequence.

**Architecture:** Introduce a `DEFAULTS` constant block and a single `normalizeEntry(raw)` helper that returns a fully-populated entry. Refactor `fall()`, `tick()`, `triggerShock()` and `showImpactText()` to consume normalized entries instead of reading hardcoded constants. Keep 100 % backward compatibility — every existing entry continues to work unchanged.

**Tech Stack:** Vanilla JS in a single `index.html` (no build, no bundler, no test runner). Verification is manual: open the file in a browser and observe behavior. PowerShell on Windows is the shell.

---

## File Structure

Single file modified throughout:

- **Modify:** `index.html` — all changes live in the `<script>` block (lines 649-1406)

The single-file constraint is structural to this project. We do **not** split it.

Logical zones inside the script (after refactor):

| Lines (current) | Zone | Responsibility |
|---|---|---|
| 650-746 | `SEQUENCE` array | data |
| ~747-790 *(new)* | `DEFAULTS` + `normalizeEntry()` | normalization layer |
| 766-824 | DOM helpers, jaggedLine, makePoly | rendering primitives |
| 826-939 | Guest/impact display | overlays |
| 942-1196 | Lightning + glass crack | SVG effects |
| 1199-1279 | Shock helpers (shake/pump/flash/vignette) | DOM effects |
| 1287-1364 | `triggerShock()` | dispatcher (refactored) |
| 1367-1403 | `fall()`, `tick()` | typing loop (refactored) |

---

## Verification Strategy

No test framework exists. After each task that changes behavior:

1. Open `index.html` in the default browser:
   ```powershell
   Start-Process .\index.html
   ```
2. Open DevTools (F12) → Console tab. **The console must stay clean** — any red error means a regression.
3. Watch the typing loop for the specific behavior the task implements.
4. Close the browser tab when done.

For tasks that need to observe a specific entry quickly, temporarily edit `mi = 0` at line 761 to `mi = <index>` to jump to it, then revert.

---

## Pre-flight: baseline commit

- [ ] **Step 1: Create minimal `.gitignore`**

Create `.gitignore` at project root with:

```
.claude/
old/
*.wav
```

- [ ] **Step 2: Commit baseline**

```powershell
git add .gitignore index.html CLAUDE_CODE_HANDOFF.md sblgs.png sblogo.png sbspaceship.png sbssfull.png sbbgship.png sbdg1.png sbdg2.png sbdg3.png sbbgec1.jpg sbbgec2.jpg sbbgec3.jpg docs/
git commit -m "chore: baseline before SEQUENCE format refactor"
```

Expected: commit succeeds with ~15 files. Run `git log --oneline` and verify one commit exists.

---

## Task 1: Add DEFAULTS constants and `normalizeEntry()` helper

**Files:**
- Modify: `index.html` — insert new block after line 746 (closing `];` of `SEQUENCE`)

This task is **pure addition** — no consumer reads `normalizeEntry` yet, so behavior is unchanged.

- [ ] **Step 1: Read the area around the insertion point**

Read `index.html` lines 740-770 to confirm where to insert. The insertion goes **right after** the line `];` that closes the `SEQUENCE` array (currently line 746), before `const SVG_NS = "http://www.w3.org/2000/svg";`.

- [ ] **Step 2: Insert the DEFAULTS + normalizeEntry block**

Insert this block immediately after the `];` that closes `SEQUENCE`:

```js
            // ── Per-entry defaults ────────────────────────────
            //  Niveau 0 ({ word: "x" }) → toutes ces valeurs s'appliquent.
            //  Toute clé fournie par l'entrée override le défaut correspondant.
            const DEFAULTS = {
                delay:          [700, 600],   // ms : base + random*range entre 2 entrées
                typeSpeed:      [75, 50],     // ms par char tapé
                holdBeforeFall: [700, 400],   // ms : mot complet → fall
                impactChance:   0.33,         // proba impact text si level ≥ medium sans forceLevel
                guestChance:    0.45,         // proba ship/dog si level ≥ medium sans forceLevel
            };

            // Background image pool (overridable per-entry via guest.dogImage)
            const BG_IMAGES_DEFAULT = ["sbdg3.png"];

            function normalizeEntry(raw) {
                const g = raw.guest || {};
                return {
                    word:           raw.word,
                    delay:          raw.delay          || DEFAULTS.delay,
                    forceLevel:     raw.forceLevel,                // undefined = aléatoire pondéré
                    forceImpact:    raw.forceImpact,               // undefined = hérite de forceLevel
                    impact:         raw.impact         || null,    // null = pool global IMPACT_*
                    impactChance:   raw.impactChance != null ? raw.impactChance : DEFAULTS.impactChance,
                    typeSpeed:      raw.typeSpeed      || DEFAULTS.typeSpeed,
                    holdBeforeFall: raw.holdBeforeFall || DEFAULTS.holdBeforeFall,
                    side:           raw.side,                      // "left"|"right"|undefined
                    guest: {
                        force:          g.force != null ? g.force : null, // "ship"|"dog"|"none"|null
                        chance:         g.chance != null ? g.chance : DEFAULTS.guestChance,
                        ignoreCooldown: !!g.ignoreCooldown,
                        dogImage:       g.dogImage || null,
                    },
                    pressureReset:  !!raw.pressureReset,
                    pressureBoost:  raw.pressureBoost  || 0,
                    strikeBg:       raw.strikeBg,                  // bool|undefined ; undefined = heavy only
                };
            }

```

- [ ] **Step 3: Verify the file still parses**

```powershell
Start-Process .\index.html
```

Open DevTools Console. Expected:
- **No red errors.** The page should look and behave exactly as before (the new function isn't called yet).
- Type in the console: `normalizeEntry({ word: "test" })` and press Enter. Expected: an object with all the documented fields, `delay: [700, 600]`, `guest: { force: null, chance: 0.45, ... }` etc.

Close the browser.

- [ ] **Step 4: Commit**

```powershell
git add index.html
git commit -m "feat(sequence): add DEFAULTS + normalizeEntry helper"
```

---

## Task 2: Route `fall()` through `normalizeEntry`

**Files:**
- Modify: `index.html` — `fall()` function at lines 1367-1380

Behavior must remain identical. We are switching from `entry.delay || [2200, 1800]` to `normalized.delay` (which already falls back to `[700, 600]` via `DEFAULTS`).

⚠️ **Behavior change:** the default-when-`delay`-absent shifts from `[2200, 1800]` (old hardcoded fallback) to `[700, 600]` (new `DEFAULTS.delay`). This is *intended* — see user requirement "Niveau 0: delay random, pas trop long". No currently active entry omits `delay`, so no regression visible until Task 9.

- [ ] **Step 1: Read the current `fall()`**

Read `index.html` lines 1367-1380 to confirm current shape.

- [ ] **Step 2: Replace `fall()`**

Find the exact current block:

```js
            function fall() {
                const entry = SEQUENCE[mi];
                el.style.animation = "fall 0.55s ease-in forwards";
                triggerShock(entry.forceLevel);
                setTimeout(() => {
                    el.style.animation = "";
                    el.textContent = "";
                    ci = 0;
                    mi = (mi + 1) % SEQUENCE.length;
                    doNeonSurge();
                    const [base, range] = entry.delay || [2200, 1800];
                    setTimeout(tick, base + Math.random() * range);
                }, 550);
            }
```

Replace with:

```js
            function fall() {
                const entry = normalizeEntry(SEQUENCE[mi]);
                el.style.animation = "fall 0.55s ease-in forwards";
                triggerShock(entry);
                setTimeout(() => {
                    el.style.animation = "";
                    el.textContent = "";
                    ci = 0;
                    mi = (mi + 1) % SEQUENCE.length;
                    doNeonSurge();
                    const [base, range] = entry.delay;
                    setTimeout(tick, base + Math.random() * range);
                }, 550);
            }
```

Note: `triggerShock(entry)` now passes the **full normalized entry** instead of the raw `forceLevel` string. This will break `triggerShock` until Task 4. Expect a runtime error in the next step — that's expected and Task 4 fixes it. **Do not browser-test between this task and Task 4.**

- [ ] **Step 3: Commit**

```powershell
git add index.html
git commit -m "refactor(sequence): fall() consumes normalized entry"
```

---

## Task 3: Route `tick()` through `normalizeEntry`

**Files:**
- Modify: `index.html` — `tick()` function at lines 1382-1403

- [ ] **Step 1: Read the current `tick()`**

Read `index.html` lines 1382-1403.

- [ ] **Step 2: Replace `tick()`**

Find the exact current block:

```js
            function tick() {
                const word = SEQUENCE[mi].word;
                if (ci === 0) {
                    pressure++;
                    const maxTilt = Math.min(2 + pressure * 0.7, 10);
                    currentTilt = (Math.random() > 0.5 ? 1 : -1) * (Math.random() * maxTilt);
                    const shrink = Math.max(0.42, 0.88 - (pressure - 1) * 0.038);
                    const dim = Math.max(0.10, 0.35 - (pressure - 1) * 0.022);
                    logo.style.opacity = String(dim.toFixed(3));
                    logo.style.filter = "brightness(0.4) saturate(0.5)";
                    logoWrap.style.transition = "transform 0.18s ease";
                    logoWrap.style.transform = `scale(${shrink.toFixed(3)}) rotate(${currentTilt.toFixed(1)}deg)`;
                }
                ci++;
                el.textContent = word.slice(0, ci);
                if (ci === word.length) {
                    // Mot complet — court moment lisible, puis tombe
                    setTimeout(fall, 700 + Math.random() * 400);
                    return;
                }
                setTimeout(tick, 75 + Math.random() * 50); // vitesse f05
            }
```

Replace with:

```js
            function tick() {
                const entry = normalizeEntry(SEQUENCE[mi]);
                const word = entry.word;
                if (ci === 0) {
                    if (entry.pressureReset) pressure = 0;
                    pressure++;
                    const maxTilt = Math.min(2 + pressure * 0.7, 10);
                    currentTilt = (Math.random() > 0.5 ? 1 : -1) * (Math.random() * maxTilt);
                    const shrink = Math.max(0.42, 0.88 - (pressure - 1) * 0.038);
                    const dim = Math.max(0.10, 0.35 - (pressure - 1) * 0.022);
                    logo.style.opacity = String(dim.toFixed(3));
                    logo.style.filter = "brightness(0.4) saturate(0.5)";
                    logoWrap.style.transition = "transform 0.18s ease";
                    logoWrap.style.transform = `scale(${shrink.toFixed(3)}) rotate(${currentTilt.toFixed(1)}deg)`;
                }
                ci++;
                el.textContent = word.slice(0, ci);
                if (ci === word.length) {
                    // Mot complet — court moment lisible, puis tombe
                    const [hBase, hRange] = entry.holdBeforeFall;
                    setTimeout(fall, hBase + Math.random() * hRange);
                    return;
                }
                const [tBase, tRange] = entry.typeSpeed;
                setTimeout(tick, tBase + Math.random() * tRange);
            }
```

- [ ] **Step 3: Commit**

```powershell
git add index.html
git commit -m "refactor(sequence): tick() consumes normalized entry"
```

---

## Task 4: Refactor `triggerShock()` signature — accept normalized entry

**Files:**
- Modify: `index.html` — `triggerShock()` function at lines 1287-1364

This task changes the function signature from `triggerShock(forceLevel)` to `triggerShock(entry)`. After this task, the cycle should work end-to-end again (Task 2 already calls it with `entry`).

The only behavioral change in this task: use `entry.side` if set, else random. Everything else preserves current semantics by reading `entry.forceLevel`.

- [ ] **Step 1: Read the current `triggerShock()`**

Read `index.html` lines 1287-1364.

- [ ] **Step 2: Replace the signature and the color/level resolution at the top**

Find the exact opening of `triggerShock`:

```js
            function triggerShock(forceLevel) {
                const isLeft = Math.random() > 0.5;
                const [cr, cg, cb] = isLeft ? [90, 225, 255] : [255, 40, 195];

                let level;
                if (forceLevel) {
                    level = forceLevel;
                    pressure = 0;
                } else {
                    const pressureBoost = Math.min(pressure * 0.038, 0.38);
                    const r = Math.random() - pressureBoost;
                    pressure = 0;
                    if      (r < 0.05) level = "none";
                    else if (r < 0.13) level = "whisper";
                    else if (r < 0.28) level = "light";
                    else if (r < 0.66) level = "medium";
                    else               level = "heavy";
                }
```

Replace with:

```js
            function triggerShock(entry) {
                const forceLevel = entry.forceLevel;
                const isLeft = entry.side === "left"  ? true
                             : entry.side === "right" ? false
                             : Math.random() > 0.5;
                const [cr, cg, cb] = isLeft ? [90, 225, 255] : [255, 40, 195];

                let level;
                if (forceLevel) {
                    level = forceLevel;
                    pressure = 0;
                } else {
                    const effectivePressure = pressure + entry.pressureBoost;
                    const pressureBoost = Math.min(effectivePressure * 0.038, 0.38);
                    const r = Math.random() - pressureBoost;
                    pressure = 0;
                    if      (r < 0.05) level = "none";
                    else if (r < 0.13) level = "whisper";
                    else if (r < 0.28) level = "light";
                    else if (r < 0.66) level = "medium";
                    else               level = "heavy";
                }
```

The remaining body of `triggerShock` (the `if (level === "none")` branch through the heavy branch) is unchanged for now — Tasks 5-8 will refactor specific sub-behaviors.

- [ ] **Step 3: Verify end-to-end in browser**

```powershell
Start-Process .\index.html
```

Watch the typing loop for ~30 seconds. Expected:
- Words type and fall normally.
- Shocks fire (shakes, lightning, occasional impact text).
- **No console errors.**
- The two active entries (`musique`, `electro`) loop. `electro` has `forceLevel: "heavy"` so it should consistently fire a heavy shock with impact text "ELECTRO !!!".

- [ ] **Step 4: Commit**

```powershell
git add index.html
git commit -m "refactor(sequence): triggerShock accepts normalized entry, side override"
```

---

## Task 5: Implement `forceImpact` and `impactChance` overrides

**Files:**
- Modify: `index.html` — `showImpactText()` at lines 912-939, and `triggerShock()` impact branches at lines 1333-1342 and 1354-1363.

After this task:
- `forceImpact: true` → impact text always shown, regardless of level (even on `light`/`whisper`/`none`).
- `forceImpact: false` → impact text never shown.
- `forceImpact: undefined` → inherits from `forceLevel` (defined ⇒ shown), else 33 % tirage (or `entry.impactChance` if set).
- For levels that have no CSS animation (`light`/`whisper`/`none`), animation defaults to `hit-medium`.

- [ ] **Step 1: Patch `showImpactText` to handle missing animation classes**

Read `index.html` lines 912-939.

Find:

```js
                        function showImpactText(level, r, g, b) {
                const entry = SEQUENCE[mi];
                const pool = (entry && entry.impact)
                    ? (level === "heavy" ? entry.impact.h : entry.impact.m)
                    : (level === "heavy" ? IMPACT_HEAVY : IMPACT_MEDIUM);
                impactEl.textContent =
                    pool[Math.floor(Math.random() * pool.length)];
                impactEl.style.color = `rgb(${r},${g},${b})`;
                impactEl.style.textShadow = `0 0 18px rgba(${r},${g},${b},1), 0 0 55px rgba(${r},${g},${b},0.7), 0 0 110px rgba(${r},${g},${b},0.35)`;
                impactEl.className = "";
                void impactEl.offsetWidth;
                impactEl.classList.add("hit-" + level);
                impactEl.addEventListener(
```

Replace with:

```js
                        function showImpactText(level, r, g, b) {
                const entry = normalizeEntry(SEQUENCE[mi]);
                const pool = entry.impact
                    ? (level === "heavy" ? entry.impact.h : entry.impact.m)
                    : (level === "heavy" ? IMPACT_HEAVY : IMPACT_MEDIUM);
                impactEl.textContent =
                    pool[Math.floor(Math.random() * pool.length)];
                impactEl.style.color = `rgb(${r},${g},${b})`;
                impactEl.style.textShadow = `0 0 18px rgba(${r},${g},${b},1), 0 0 55px rgba(${r},${g},${b},0.7), 0 0 110px rgba(${r},${g},${b},0.35)`;
                impactEl.className = "";
                void impactEl.offsetWidth;
                // CSS n'a que hit-medium et hit-heavy — fallback medium pour les niveaux bas
                const animClass = (level === "heavy") ? "hit-heavy" : "hit-medium";
                impactEl.classList.add(animClass);
                impactEl.addEventListener(
```

- [ ] **Step 2: Add a `shouldShowImpact()` helper just above `triggerShock`**

Read `index.html` lines 1282-1290 to confirm where `triggerShock` begins.

Insert this helper **immediately before** `function triggerShock(entry) {`:

```js
            // forceImpact decoupling — explicit override wins, otherwise hérite de forceLevel
            function shouldShowImpact(entry) {
                if (entry.forceImpact === true)  return true;
                if (entry.forceImpact === false) return false;
                if (entry.forceLevel) return true;       // legacy: forceLevel ⇒ impact
                return Math.random() < entry.impactChance;
            }

```

- [ ] **Step 3: Rewrite the medium branch of `triggerShock`**

Find this block in `triggerShock`:

```js
                if (level === "medium") {
                    doShake("medium");
                    doPump("medium");
                    doFlash();
                    doBgFlash("medium");
                    showLightning("medium");
                    // Impact text 1/3 du temps — sinon ship ou chien
                    if (forceLevel) {
                        showImpactText("medium", cr, cg, cb);
                    } else if (Math.random() < 0.33) {
                        showImpactText("medium", cr, cg, cb);
                    } else if (mi - lastGuestAt >= 2 && Math.random() < 0.45) {
                        if (nextGuest === 'ship') { showSpaceship(); } else { showBgImage(); }
                        lastGuestAt = mi;
                        nextGuest = (nextGuest === 'ship') ? 'dog' : 'ship';
                    }
                    return;
                }
```

Replace with:

```js
                if (level === "medium") {
                    doShake("medium");
                    doPump("medium");
                    doFlash();
                    doBgFlash("medium");
                    showLightning("medium");
                    const wantImpact = shouldShowImpact(entry);
                    if (wantImpact) {
                        showImpactText("medium", cr, cg, cb);
                    } else if (mi - lastGuestAt >= 2 && Math.random() < 0.45) {
                        if (nextGuest === 'ship') { showSpaceship(); } else { showBgImage(); }
                        lastGuestAt = mi;
                        nextGuest = (nextGuest === 'ship') ? 'dog' : 'ship';
                    }
                    return;
                }
```

- [ ] **Step 4: Rewrite the heavy branch of `triggerShock`**

Find this block (the end of `triggerShock`):

```js
                // ── HEAVY
                doShake("heavy");
                doPump("heavy");
                doFlash();
                doBgFlash("heavy");
                doVignette();
                showLightning("heavy");
                showStrikeBg();
                // Impact text 1/3 du temps — sinon ship ou chien
                if (forceLevel) {
                    showImpactText("heavy", cr, cg, cb);
                } else if (Math.random() < 0.33) {
                    showImpactText("heavy", cr, cg, cb);
                } else if (mi - lastGuestAt >= 2 && Math.random() < 0.45) {
                    if (nextGuest === 'ship') { showSpaceship(); } else { showBgImage(); }
                    lastGuestAt = mi;
                    nextGuest = (nextGuest === 'ship') ? 'dog' : 'ship';
                }
            }
```

Replace with:

```js
                // ── HEAVY
                doShake("heavy");
                doPump("heavy");
                doFlash();
                doBgFlash("heavy");
                doVignette();
                showLightning("heavy");
                showStrikeBg();
                const wantImpactHeavy = shouldShowImpact(entry);
                if (wantImpactHeavy) {
                    showImpactText("heavy", cr, cg, cb);
                } else if (mi - lastGuestAt >= 2 && Math.random() < 0.45) {
                    if (nextGuest === 'ship') { showSpaceship(); } else { showBgImage(); }
                    lastGuestAt = mi;
                    nextGuest = (nextGuest === 'ship') ? 'dog' : 'ship';
                }
            }
```

- [ ] **Step 5: Also handle forceImpact on the lower-than-medium branches**

These branches currently never call `showImpactText`. With `forceImpact: true` on a `light`/`whisper`/`none` level, we want the impact text to still appear (falling back to `hit-medium` animation per Step 1).

Find this block:

```js
                if (level === "none") {
                    logoRestore(true);
                    if (Math.random() > 0.5) showLightning("light"); // mini bolt 50%
                    return;
                }
```

Replace with:

```js
                if (level === "none") {
                    logoRestore(true);
                    if (Math.random() > 0.5) showLightning("light"); // mini bolt 50%
                    if (entry.forceImpact === true) showImpactText("medium", cr, cg, cb);
                    return;
                }
```

Find:

```js
                if (level === "whisper") {
                    doShake("light");
                    showLightning("light");
                    return;
                }
```

Replace with:

```js
                if (level === "whisper") {
                    doShake("light");
                    showLightning("light");
                    if (entry.forceImpact === true) showImpactText("medium", cr, cg, cb);
                    return;
                }
```

Find:

```js
                if (level === "light") {
                    doShake("light");
                    showLightning("light");
                    if (Math.random() > 0.5) setTimeout(() => showLightning("light"), 80);
                    return;
                }
```

Replace with:

```js
                if (level === "light") {
                    doShake("light");
                    showLightning("light");
                    if (Math.random() > 0.5) setTimeout(() => showLightning("light"), 80);
                    if (entry.forceImpact === true) showImpactText("medium", cr, cg, cb);
                    return;
                }
```

- [ ] **Step 6: Verify in browser**

```powershell
Start-Process .\index.html
```

Watch for ~30 seconds. Expected:
- `electro` (forceLevel heavy) still always shows "ELECTRO !!!" impact text.
- `musique` (forceLevel "none") behaves as before — no impact (forceImpact absent).
- No console errors.

- [ ] **Step 7: Console-test `forceImpact: true` on a low level**

In the browser DevTools console, paste:

```js
const fake = normalizeEntry({ word: "test", forceLevel: "light", forceImpact: true });
triggerShock(fake);
```

Expected: a small shake + lightning + an impact text overlay (drawn from the global `IMPACT_MEDIUM` pool since no `impact` was set). No console errors.

- [ ] **Step 8: Commit**

```powershell
git add index.html
git commit -m "feat(sequence): forceImpact flag decoupled from forceLevel"
```

---

## Task 6: Implement `guest` overrides (force / chance / ignoreCooldown / dogImage)

**Files:**
- Modify: `index.html` — `showBgImage()` at lines 886-902 (accept optional image), `triggerShock` guest dispatch at the `else if` lines in both medium and heavy branches.

After this task:
- `guest.force: "ship"` → guaranteed spaceship (bypasses cooldown if `ignoreCooldown:true`).
- `guest.force: "dog"` → guaranteed dog (uses `guest.dogImage` if set, else random from `BG_IMAGES`).
- `guest.force: "none"` → no guest, ever.
- `guest.force: null` → current random behavior, but using `guest.chance` instead of hardcoded 0.45.
- `guest.ignoreCooldown: true` → skip the "≥2 entries since last guest" check.

- [ ] **Step 1: Make `showBgImage()` accept an optional override image**

Read `index.html` lines 886-902.

Find:

```js
            function showBgImage() {
                if (dogBusy) return;
                dogBusy = true;
                bgImgTag.src =
                    BG_IMAGES[Math.floor(Math.random() * BG_IMAGES.length)];
                bgImgEl.classList.remove("appear");
                void bgImgEl.offsetWidth;
                bgImgEl.classList.add("appear");
                bgImgEl.addEventListener(
                    "animationend",
                    () => {
                        bgImgEl.classList.remove("appear");
                        dogBusy = false;
                    },
                    { once: true },
                );
            }
```

Replace with:

```js
            function showBgImage(overrideSrc) {
                if (dogBusy) return;
                dogBusy = true;
                bgImgTag.src = overrideSrc
                    || BG_IMAGES[Math.floor(Math.random() * BG_IMAGES.length)];
                bgImgEl.classList.remove("appear");
                void bgImgEl.offsetWidth;
                bgImgEl.classList.add("appear");
                bgImgEl.addEventListener(
                    "animationend",
                    () => {
                        bgImgEl.classList.remove("appear");
                        dogBusy = false;
                    },
                    { once: true },
                );
            }
```

- [ ] **Step 2: Add a `dispatchGuest()` helper just above `triggerShock`**

Insert **immediately before** `function shouldShowImpact(entry) {` (added in Task 5):

```js
            // Guest dispatcher — encodes the cooldown + force + chance logic in one place
            function dispatchGuest(entry) {
                const g = entry.guest;
                if (g.force === "none") return;

                const cooldownOk = g.ignoreCooldown || (mi - lastGuestAt >= 2);
                if (!cooldownOk) return;

                let pick = g.force; // "ship" | "dog" | null
                if (!pick) {
                    if (Math.random() >= g.chance) return;
                    pick = nextGuest;
                    nextGuest = (nextGuest === 'ship') ? 'dog' : 'ship';
                }

                if (pick === 'ship') {
                    showSpaceship();
                } else {
                    showBgImage(g.dogImage);
                }
                lastGuestAt = mi;
            }

```

- [ ] **Step 3: Replace the medium-branch guest call**

Find in the medium branch of `triggerShock` (recently rewritten in Task 5):

```js
                    const wantImpact = shouldShowImpact(entry);
                    if (wantImpact) {
                        showImpactText("medium", cr, cg, cb);
                    } else if (mi - lastGuestAt >= 2 && Math.random() < 0.45) {
                        if (nextGuest === 'ship') { showSpaceship(); } else { showBgImage(); }
                        lastGuestAt = mi;
                        nextGuest = (nextGuest === 'ship') ? 'dog' : 'ship';
                    }
                    return;
```

Replace with:

```js
                    const wantImpact = shouldShowImpact(entry);
                    if (wantImpact) {
                        showImpactText("medium", cr, cg, cb);
                    } else {
                        dispatchGuest(entry);
                    }
                    return;
```

- [ ] **Step 4: Replace the heavy-branch guest call**

Find in the heavy branch:

```js
                const wantImpactHeavy = shouldShowImpact(entry);
                if (wantImpactHeavy) {
                    showImpactText("heavy", cr, cg, cb);
                } else if (mi - lastGuestAt >= 2 && Math.random() < 0.45) {
                    if (nextGuest === 'ship') { showSpaceship(); } else { showBgImage(); }
                    lastGuestAt = mi;
                    nextGuest = (nextGuest === 'ship') ? 'dog' : 'ship';
                }
```

Replace with:

```js
                const wantImpactHeavy = shouldShowImpact(entry);
                if (wantImpactHeavy) {
                    showImpactText("heavy", cr, cg, cb);
                } else {
                    dispatchGuest(entry);
                }
```

- [ ] **Step 5: Verify in browser**

```powershell
Start-Process .\index.html
```

Watch for ~60 seconds. Expected:
- Same global behavior as before (guests still appear stochastically on the active entries since their `guest.force` is `null`).
- No console errors.

- [ ] **Step 6: Console-test forced ship**

In DevTools console:

```js
const fake = normalizeEntry({ word: "test", guest: { force: "ship", ignoreCooldown: true } });
triggerShock({ ...fake, forceLevel: "medium" });
```

Expected: a medium shock fires AND the spaceship overlay flies across the screen (because `wantImpact` will be true via the forceLevel, but we should also force the guest path… see Step 7).

⚠️ **Edge case:** when `wantImpact` is true, `dispatchGuest` is never called. This is intentional — impact text and guests are mutually exclusive per the original design. If you want both, pass `forceImpact: false` to suppress impact.

- [ ] **Step 7: Commit**

```powershell
git add index.html
git commit -m "feat(sequence): per-entry guest control (force/chance/cooldown/image)"
```

---

## Task 7: Implement `strikeBg` override

**Files:**
- Modify: `index.html` — `triggerShock()` heavy branch (the `showStrikeBg()` call), and add a conditional call in the medium branch.

After this task:
- `strikeBg: true` → strike-bg fires regardless of level (even on `medium` / `light`).
- `strikeBg: false` → strike-bg suppressed even on `heavy`.
- `strikeBg: undefined` → current behavior (heavy only).

- [ ] **Step 1: Replace the heavy-branch `showStrikeBg()` call**

Find in the heavy branch of `triggerShock`:

```js
                showLightning("heavy");
                showStrikeBg();
```

Replace with:

```js
                showLightning("heavy");
                if (entry.strikeBg !== false) showStrikeBg();
```

- [ ] **Step 2: Add an optional `showStrikeBg()` call in lower branches**

Find in the medium branch:

```js
                if (level === "medium") {
                    doShake("medium");
                    doPump("medium");
                    doFlash();
                    doBgFlash("medium");
                    showLightning("medium");
```

Replace with:

```js
                if (level === "medium") {
                    doShake("medium");
                    doPump("medium");
                    doFlash();
                    doBgFlash("medium");
                    showLightning("medium");
                    if (entry.strikeBg === true) showStrikeBg();
```

Find in the light branch:

```js
                if (level === "light") {
                    doShake("light");
                    showLightning("light");
                    if (Math.random() > 0.5) setTimeout(() => showLightning("light"), 80);
                    if (entry.forceImpact === true) showImpactText("medium", cr, cg, cb);
                    return;
                }
```

Replace with:

```js
                if (level === "light") {
                    doShake("light");
                    showLightning("light");
                    if (Math.random() > 0.5) setTimeout(() => showLightning("light"), 80);
                    if (entry.strikeBg === true) showStrikeBg();
                    if (entry.forceImpact === true) showImpactText("medium", cr, cg, cb);
                    return;
                }
```

- [ ] **Step 3: Verify in browser**

```powershell
Start-Process .\index.html
```

Watch for ~30 seconds. Expected: `electro` (heavy) still fires strike-bg as before. No console errors.

- [ ] **Step 4: Commit**

```powershell
git add index.html
git commit -m "feat(sequence): per-entry strikeBg override"
```

---

## Task 8: Sanity test — full normalized-entry test in console

**Files:** none modified — this is a verification-only task.

- [ ] **Step 1: Open browser console**

```powershell
Start-Process .\index.html
```

- [ ] **Step 2: Run the normalization snapshot test**

Paste in the DevTools console:

```js
console.table({
  minimal:  normalizeEntry({ word: "x" }),
  withDelay: normalizeEntry({ word: "x", delay: [400,400] }),
  full: normalizeEntry({
    word: "x", delay: [400,400], forceLevel: "heavy", forceImpact: false,
    impact: { m:["A"], h:["B"] }, impactChance: 0.5,
    typeSpeed: [120,0], holdBeforeFall: [1500,0],
    side: "left",
    guest: { force: "ship", chance: 1, ignoreCooldown: true, dogImage: "sbdg3.png" },
    pressureReset: true, pressureBoost: 5, strikeBg: true
  })
});
```

Expected: a table showing all three normalized entries with every documented field. `minimal` should have `delay: [700, 600]` and `guest: { force: null, chance: 0.45, ignoreCooldown: false, dogImage: null }`.

- [ ] **Step 3: Run a behavior probe**

Paste:

```js
// Force a left-side heavy with no impact, guaranteed dog
const fake = normalizeEntry({
  word: "test",
  forceLevel: "heavy",
  forceImpact: false,
  side: "left",
  guest: { force: "dog", ignoreCooldown: true, dogImage: "sbdg3.png" }
});
triggerShock(fake);
```

Expected: heavy shock with cyan-blue lightning (not pink), no impact text, and a dog image appears.

- [ ] **Step 4: Commit a NOTES file**

Create `docs/superpowers/plans/2026-05-24-sequence-format-NOTES.md` with:

```markdown
# SEQUENCE format — notes post-refactor

- `forceImpact` and `forceLevel` are independent; `forceLevel` still implicitly
  enables impact text unless `forceImpact: false` overrides.
- Impact text and guests are mutually exclusive per shock — by design.
- `strikeBg` works on any level when explicitly set; defaults to heavy-only.
- CSS only ships `hit-medium` and `hit-heavy` animations; `hit-light` etc. fall back to medium.
```

Then:

```powershell
git add docs/
git commit -m "docs(sequence): post-refactor notes"
```

---

## Task 9: Uncomment SEQUENCE and add 3 showcase enhancements

**Files:**
- Modify: `index.html` — the comment block at lines 655 and 745 (the `/* ... */` wrapping the bulk of entries)

After this task, the full sequence runs end-to-end with the new format.

- [ ] **Step 1: Remove the opening `/*`**

Read `index.html` lines 653-657 to confirm current shape:

```
              { word: "musique",       delay: [400, 400] , impact: { m: ["ELECTRO !"], h: ["ELECTRO !!!"] }, "forceLevel": "none" },
              { word: "electro",       delay: [400, 400] , impact: { m: ["ELECTRO !"], h: ["ELECTRO !!!"] }, "forceLevel": "heavy" },
/*
              { word: "pulse",         delay: [400, 400] , impact: { m: ["PULSE !"], h: ["PULSE !!!"] } },
```

Edit the line containing exactly `/*` (line 655) — replace it with an empty line. Be sure to match the exact line including any trailing whitespace.

Find:

```
              { word: "electro",       delay: [400, 400] , impact: { m: ["ELECTRO !"], h: ["ELECTRO !!!"] }, "forceLevel": "heavy" },
/*          
              { word: "pulse",         delay: [400, 400] , impact: { m: ["PULSE !"], h: ["PULSE !!!"] } },
```

Replace with:

```
              { word: "electro",       delay: [400, 400] , impact: { m: ["ELECTRO !"], h: ["ELECTRO !!!"] }, "forceLevel": "heavy" },

              { word: "pulse",         delay: [400, 400] , impact: { m: ["PULSE !"], h: ["PULSE !!!"] } },
```

- [ ] **Step 2: Remove the closing `*/`**

Find:

```
              { word: "**+. and we start again .+**", delay: [3000, 2000] , impact: { m: ["LOOP !"], h: ["AND WE START AGAIN !!!"] } },
*/
            ];
```

Replace with:

```
              { word: "**+. and we start again .+**", delay: [3000, 2000] , impact: { m: ["LOOP !"], h: ["AND WE START AGAIN !!!"] } },
            ];
```

- [ ] **Step 3: Showcase enhancement #1 — force impact on the "noise" subliminal**

The entry `{ word: "noise", delay: [3000, 2000], forceLevel: "heavy", impact: { m: ["...bon ?"], h: ["...bon ?"] } }` already forces heavy. No change needed — included as a reference of an already-good entry.

- [ ] **Step 4: Showcase enhancement #2 — force the spaceship on the "SUPERBON" punchline**

Find:

```
              { word: "SUPERBON",
                delay: [3500, 1500], forceLevel: "heavy",
                impact: { m: ["SUPERBON !"], h: ["SUPERBON !!!"] } },
```

Replace with:

```
              { word: "SUPERBON",
                delay: [3500, 1500], forceLevel: "heavy",
                impact: { m: ["SUPERBON !"], h: ["SUPERBON !!!"] },
                guest: { force: "ship", ignoreCooldown: true },
                forceImpact: false,           // laisse le ship occuper la scène
                pressureReset: true,
                strikeBg: true,
                side: "left" },
```

- [ ] **Step 5: Showcase enhancement #3 — force the dog on "respect."**

Find:

```
              { word: "respect.",             delay: [2500, 1500] , impact: { m: ["RESPECT !"], h: ["RESPECT !!!"] } },
```

Replace with:

```
              { word: "respect.",             delay: [2500, 1500] , impact: { m: ["RESPECT !"], h: ["RESPECT !!!"] },
                guest: { force: "dog", ignoreCooldown: true, dogImage: "sbdg3.png" },
                forceImpact: false },
```

- [ ] **Step 6: Showcase enhancement #4 — `forceImpact` without `forceLevel` on "vraiment ?!"**

Find:

```
              { word: "vraiment ?!",          delay: [1500, 1000] , impact: { m: ["VRAIMENT ?!"], h: ["INCROYABLE !!!"] } },
```

Replace with:

```
              { word: "vraiment ?!",          delay: [1500, 1000] , impact: { m: ["VRAIMENT ?!"], h: ["INCROYABLE !!!"] }, forceImpact: true },
```

This guarantees the impact text appears whatever level the shock rolls — demonstrating the decoupling.

- [ ] **Step 7: Verify full sequence runs**

```powershell
Start-Process .\index.html
```

Let the page run for at least 2 minutes. Expected:
- The full word sequence loops through (musique → electro → pulse → kick → … → and we start again).
- "SUPERBON" entry triggers the spaceship and no impact text (because `forceImpact: false`).
- "respect." entry triggers a dog (sbdg3.png).
- "vraiment ?!" always shows its impact text.
- No console errors at any point. (Watch for "Uncaught" or red entries.)

- [ ] **Step 8: Commit**

```powershell
git add index.html
git commit -m "feat(sequence): uncomment full sequence + 3 format showcases"
```

---

## Self-Review Checklist (executed by plan author)

**Spec coverage:**
- ✅ `forceImpact` flag decoupled from `forceLevel` (Task 5 + Task 9 Step 6)
- ✅ Niveau 0 default delay is random and not too long: `[700, 600]` → 700–1300 ms (Task 1)
- ✅ All documented fields exposed: `delay`, `forceLevel`, `forceImpact`, `impact`, `impactChance`, `typeSpeed`, `holdBeforeFall`, `side`, `guest.{force,chance,ignoreCooldown,dogImage}`, `pressureReset`, `pressureBoost`, `strikeBg` (Task 1)
- ✅ Backward compatibility: every existing entry continues to behave identically (Tasks 2–7 preserve semantics; only defaults are exposed)
- ✅ SEQUENCE uncommented (Task 9 Steps 1–2)
- ✅ Sequence adapted to showcase new features (Task 9 Steps 4–6)

**Placeholder scan:** No TBD/TODO/"add appropriate"/etc. — all code is complete.

**Type consistency:**
- `normalizeEntry` shape used identically in `tick`, `fall`, `triggerShock`, `showImpactText`, `shouldShowImpact`, `dispatchGuest`.
- `entry.guest.dogImage` flows through `dispatchGuest` → `showBgImage(overrideSrc)`.
- `entry.side` consumed only in `triggerShock` (the right place — color is a shock-time decision).

**Risks:**
- Task 2 leaves the file in a transiently broken state (calls `triggerShock(entry)` while the function still expects `forceLevel`). Task 4 fixes it. The plan explicitly tells the engineer not to browser-test between Tasks 2 and 4.
- Default delay change `[2200, 1800] → [700, 600]` is visible only on entries that omit `delay`. No active entry does, so no regression.

---

## Plan complete

Plan saved to `docs/superpowers/plans/2026-05-24-sequence-format-refactor.md`.
