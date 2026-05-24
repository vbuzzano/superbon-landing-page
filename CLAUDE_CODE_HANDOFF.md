# Superbon Landing Page — Claude Code Handoff

## Projet

Teaser landing page pour le groupe electro tech punk **Superbon** (`superbon.rocks` / `superbon.band`).  
Fichier de travail actif : **`index5.html`** (1400 lignes, single-file, no framework).

`index4.html` = version stable de référence (ne pas toucher).  
`index4 - RandomClashWord.html` = backup propre de index4 (source pour rebuilds Python si besoin).

---

## Architecture index5.html

### Visuels (CSS z-index stack)
```
ship        z:61   #bg-image (spaceship PNG)
dog         z:62   #bg-image2 (dog PNG)
logo-wrap   z:63   #logo-wrap
lightning   z:72   #lightning-overlay
impact-text z:88   #impact-text
```

### Flux principal
```
tick() → type 1 char (75+rnd(50)ms)
       → dernier char → setTimeout(fall, 700+rnd(400)ms)
fall() → triggerShock(entry.forceLevel) → IMMÉDIAT
       → setTimeout(550ms) → mi++, delay, tick()
```

### Format SEQUENCE
```javascript
{
  word:       "electro",           // requis
  delay:      [base_ms, range_ms], // optionnel, défaut [2200, 1800]
  impact:     { m: ["..."], h: ["..."] }, // optionnel, phrases medium/heavy
  forceLevel: "heavy"              // optionnel, force le niveau de choc
}
```

**forceLevels disponibles :**
| valeur | effet |
|---|---|
| `"none"` | restaure logo, mini-bolt 50% |
| `"whisper"` | shake léger + lightning léger |
| `"light"` | shake + lightning (double 50%) |
| `"medium"` | shake + pump + flash + bgFlash + lightning + impact/ship/dog |
| `"heavy"` | tout medium + vignette + strikeBg + impact/ship/dog |

Si absent → tirage aléatoire pondéré par `pressure` (variable globale qui s'accumule).

### Variables JS clés
```javascript
let mi = 0;              // index courant dans SEQUENCE
let ci = 0;              // index courant de char dans le mot
let pressure = 0;        // s'accumule, booste la proba de choc
let lastGuestAt = -99;   // index phrase du dernier ship/dog apparu
let nextGuest = 'ship'|'dog'; // alterne à chaque apparition
let shipBusy = false;    // flag anti-double ship
let dogBusy = false;     // flag anti-double dog
```

### triggerShock() — logique medium/heavy
```javascript
// medium et heavy ont la même logique :
if (forceLevel) {
    showImpactText("medium"|"heavy", cr, cg, cb); // toujours si forcé
} else if (Math.random() < 0.33) {
    showImpactText(...);                           // 1/3 aléatoire
} else if (mi - lastGuestAt >= 2 && Math.random() < 0.45) {
    if (nextGuest === 'ship') { showSpaceship(); } else { showBgImage(); }
    lastGuestAt = mi;
    nextGuest = (nextGuest === 'ship') ? 'dog' : 'ship';
}
```

---

## État actuel de index5.html

### Différences vs index4.html
1. `impact: { m: [...], h: [...] }` ajouté sur toutes les entrées SEQUENCE
2. Mobile CSS `@media (max-width: 768px)` : ship `-5vh`, dog `-8vh`
3. `let lastGuestAt = -99` et `let nextGuest` initialisé aléatoirement
4. Fix forceLevel : `if (forceLevel) { showImpactText() }` bypass le tirage
5. Système alternance chien/vaisseau (un seul à la fois, min 2 phrases entre apparitions)

### SEQUENCE active (le reste est commenté `/* ... */` lignes 655–745)
```javascript
{ word: "musique", delay: [400, 400], impact: {...}, forceLevel: "none" },
{ word: "electro", delay: [400, 400], impact: {...}, forceLevel: "heavy" },
// ← tout le reste commenté pour tests
```

---

## Tâche suivante : refactor SEQUENCE

### Objectif
Décommenter la SEQUENCE complète (~56 entrées) et réviser les `impact` texts + `forceLevel` pour que chaque mot ait du sens musical/artistique.

### Règles éditoriales
- `impact` doit être **en lien avec le mot** (ex: `word:"acid"` → `impact: { m:["ACID !"], h:["303 ACID !!!"] }`)
- `forceLevel` absent = aléatoire (comportement normal)
- `forceLevel: "heavy"` réservé aux mots-clés dramatiques de la narration (ex: "noise", "overdrive", "superbon")
- `delay` long (`[3000, 2000]`) = pause narrative intentionnelle

### Pitfalls Python à éviter si script de rebuild
- Toujours lire depuis `index4 - RandomClashWord.html` comme source propre
- Ne jamais utiliser `\n` dans les strings de remplacement (corruption binaire)
- Caractères accentués : utiliser `str.replace()` exact, pas regex
- Faire **un seul pass Python** qui applique tous les changements en une fois
- Vérifier avec `grep -c "binary"` après chaque opération

---

## Règles projet
- Tout le code, commentaires, docs : **anglais**
- Échanges avec l'utilisateur : **français**
- Réponses type **BLUF**
- Utilisateur : dev expérimenté (JS, Python, assembleur, etc.), HPI/TDAH
