# Chordex — Developer Documentation

A single-file HTML/CSS/JS chord and scale reference app for Ukulele, Guitar, Bass, and Piano.

---

## Stack

- **Single file:** `ukulele-chords.html` (~89KB, ~1885 lines)
- **No build step, no dependencies** — pure vanilla JS
- **Fonts:** Nunito + Nunito Sans via Google Fonts
- **Hosting:** drop on Netlify, GitHub Pages, or any static host

---

## Design System

### Colors (CSS variables)
```css
--indigo:    #5b4fcf   /* primary accent */
--indigo2:   #7b6fdf   /* gradient end */
--indigo3:   #3d34a5   /* hover/dark accent */
--bg:        #f0f1f6   /* page background */
--surface:   #ffffff   /* cards */
--border:    #e8e9f0   /* card borders */
--text:      #1a1b2e   /* primary text */
--muted:     #8b8fa8   /* secondary text */
--radius:    18px      /* card radius */
--radius-sm: 12px      /* inner element radius */
```

### Diagram colors (hardcoded in JS)
```
#5b4fcf  — scale/chord dots (indigo)
#1a1b2e  — root note dots (dark)
#c8cad8  — strings and fret lines
#8b8fa8  — string name labels
```

---

## File Structure

```
<head>
  CSS (design system + all component styles)
  Google Fonts import
</head>
<body>
  .header-zone          — sticky gradient header
    .instrument-bar     — Ukulele / Guitar / Bass / Piano switcher
    .tabs               — Chords / Scales / Progressions / Finder
  
  #chords.page          — chord browser with filters + grid
  #scales.page          — scale fretboard/piano with root+type picker
  #progressions.page    — curated progression cards
  #finder.page          — tap-to-identify chord finder
  
  #detailOverlay        — bottom sheet for chord detail + inversions
  
  <script>              — all JS (~58KB)
</script>
</body>
```

---

## Instrument State

```js
let currentInstrument = 'ukulele'; // 'ukulele' | 'guitar' | 'bass' | 'piano'
```

Switch via `setInstrument(inst, el)` — resets all filters and re-renders.

### Open string pitches (semitones, C=0)
```js
const UKE_OPEN          = [7, 0, 4, 9];         // G C E A
const GTR_OPEN_PITCHES  = [4, 9, 2, 7, 11, 4];  // E A D G B E
const BASS_OPEN_PITCHES = [4, 9, 2, 7];          // E A D G
const GTR_NAMES  = ['E','A','D','G','B','E'];
const BASS_NAMES = ['E','A','D','G'];
```

---

## Chord Data

### Chord object shape
```js
{
  name:    'Am7',           // display name
  full:    'A Minor 7',     // full name shown in detail sheet
  type:    'm7',            // filter type key (see Types below)
  frets:   [0,0,0,0],       // fret per string (left→right), -1 = mute
  fingers: ['O','O','O','O'] // 'O'=open, 'x'=mute, '1'-'4'=finger
}
```

### Chord arrays
| Array | Instrument | Count | Notes |
|-------|-----------|-------|-------|
| `UKE_CHORDS` | Ukulele GCEA | 168 | Mathematically verified from tuning |
| `GTR_CHORDS` | Guitar EADGBe | 84 | Standard open/barre voicings |
| `BASS_CHORDS` | Bass EADG | 48 | Root/shell voicings |
| `PIANO_CHORDS` | Piano | 132 | Auto-generated from intervals |

Piano chords use a different shape:
```js
{ name, full, type, rootSemi: 0, intervals: [0,4,7] }
```

### Chord types
| Key | Name | Intervals |
|-----|------|-----------|
| `major` | Major | 0,4,7 |
| `minor` | Minor | 0,3,7 |
| `7th` | Dom 7 | 0,4,7,10 |
| `maj7` | Major 7 | 0,4,7,11 |
| `m7` | Minor 7 | 0,3,7,10 |
| `dim` | Diminished | 0,3,6 |
| `aug` | Augmented | 0,4,8 |
| `sus` | Sus 4 | 0,5,7 |
| `add9` | Add 9 | 0,4,7,14 |
| `6th` | 6th | 0,4,7,9 |
| `9th` | 9th | 0,4,7,10,14 |
| `m9` | Minor 9 | 0,3,7,10,14 |
| `m7b5` | Half Dim | 0,3,6,10 |
| `7aug` | Dom 7 Aug | 0,4,8,10 |

---

## Filter State

```js
let activeFilters = new Set(); // set of type keys e.g. {'minor','7th'}
let activeRoot    = 'all';     // 'all' | 'C' | 'C#' | 'D' | ...
let currentSearch = '';
```

### Multi-filter logic (smart combining)
When multiple types are selected, chords must satisfy **all** selected qualities:
```js
const TYPE_QUALITIES = {
  minor: ['minor','m7','m9','m7b5'],
  '7th': ['7th','m7','maj7','m9','7aug'],
  // etc.
};
// multi-select: chord.type must appear in EVERY selected filter's quality list
```

So `minor + 7th` → shows `m7`, `m9`. `minor + 9th` → shows `m9`.

### Root matching
```js
function matchesRoot(name, root) {
  if (root === 'all') return true;
  if (!name.startsWith(root)) return false;
  const next = name[root.length] || '';
  return !/[A-Gb#]/.test(next); // blocks C matching C#, B matching Bb
}
```

---

## Diagram Rendering

### Fret diagrams (uke / guitar / bass)
```js
buildFretDiagram(chord, W, H, fontSize, stringCount)
// → SVG string
// Auto-detects start fret (shows position marker if above fret 4)
// Dots show finger numbers, O = open, × = mute
```

### Piano diagrams
```js
buildPianoSVG(chord, W, H, isPreview)
// → full SVG string (white + black keys, one octave)
// Active tones = indigo, inactive = white/dark
```

### Scale fretboard
```js
buildScaleFretboard(scaleNotes, openPitches, strNames)
// → SVG string
// Wider frets (52px each), scrollable, position markers at 3,5,7,10,12
// Returns SVG for use inside a fixed-width viewBox
```

---

## Inversions

```js
let currentInversions = [];   // array of inversion objects
let currentInversionIdx = 0;
```

Inversion objects:
```js
// Fretboard instruments
{ frets: [0,2,2,1], fingers: ['O','2','3','1'], bassNote: 4, label: '1st Inversion' }

// Piano
{ tones: [4,7,0], bassNote: 4 }
```

Generated by:
- `getPianoInversions(chord)` — rotates chord tones
- `getFretInversions(chord)` — finds voicings with different bass notes

---

## Chord Finder

```js
let finderSelectedNotes   = new Set(); // pitch classes 0-11
let finderTappedPositions = new Set(); // 'stringIdx:fret' strings
```

**Flow:**
1. User taps fret → `tapFinderPosition(si, fret, pitch)` 
2. Rebuilds `finderSelectedNotes` from all tapped positions
3. `updateFinderResults()` checks every root × every type
4. Returns **exact** matches (all tones present) and **partial** matches (missing 1 tone, shown as "omit X")

Tap a note badge → `removeFinderPitch(pitch)` removes all positions producing that pitch.

---

## Scales

```js
let selectedScaleRoot = 0;    // semitone 0-11
let selectedScaleType = 'major';
```

Scale types defined in `SCALE_TYPES` array — 13 scales including all modes, pentatonics, blues, harmonic/melodic minor, chromatic.

Scale rendering dispatches to:
- `buildScaleFretboard()` for uke/guitar/bass
- `buildPianoScaleSVG()` for piano

---

## Progressions

Static array of 6 progressions. Each chord badge in a progression calls `showDetail(chordName)` so you can tap to see the fingering.

---

## Key Functions Reference

| Function | Purpose |
|----------|---------|
| `setInstrument(inst, el)` | Switch instrument, reset filters |
| `renderChords()` | Re-render chord grid with current filters |
| `filterType(type, el)` | Toggle type pill, update `activeFilters` |
| `filterRoot(root, el)` | Set root filter |
| `showDetail(name)` | Open detail sheet for a chord |
| `stepInversion(dir)` | Cycle through inversions (-1/+1) |
| `renderScale()` | Re-render scale diagram |
| `openFinder()` | Reset and show finder |
| `tapFinderPosition(si, fret, pitch)` | Register a fret tap in finder |
| `updateFinderResults()` | Match tapped notes against all chords |
| `showPage(id, tab)` | Switch between Chords/Scales/Progressions/Finder |

---

## To-Do / Ideas

- [ ] Chord audio playback (Web Audio API)
- [ ] Save favourite chords
- [ ] Custom tunings for guitar/bass
- [ ] More progressions
- [ ] Barre chord indicator on diagrams
- [ ] Export chord diagrams as PNG
- [ ] Dark mode
- [ ] PWA / installable app (add manifest + service worker)

---

## Deploying

Single file — no build needed.

```bash
# Netlify Drop
# Just drag ukulele-chords.html to netlify.com/drop

# GitHub Pages
# Rename to index.html, push to repo, enable Pages in Settings

# Local dev
open ukulele-chords.html   # macOS
# or
python3 -m http.server 8080
```

---

*Built with Claude — June 2026*
