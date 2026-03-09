# AD5 Prep App — Project Memory File
*Upload this file alongside AD5_Study_App.html at the start of each new session.*

---

## What this app is

A single-file HTML web application to prepare for the **EPSO AD5 competition (EPSO/AD/427/26)**, targeting the September/October 2026 exam. The user is based in Brussels.

The app runs entirely in the browser — no installation, no server. Just open the HTML file.

---

## Competition structure (what we're training for)

| Test | Questions | Time | Weight |
|------|-----------|------|--------|
| Verbal reasoning | 20 | 35 min | 35% |
| Numerical reasoning | 10 | 20 min | — |
| Abstract reasoning | 10 | 10 min | — |
| EU Knowledge | 30 | 40 min | — |
| Digital skills | 40 | 30 min | — |
| EUFTE essay | — | 40 min | — |

---

## App architecture

**Single HTML file** (~3,300 lines). No frameworks, no build step. Pure HTML + CSS + vanilla JS.

- Uses **Anthropic Claude API** (claude-sonnet-4-20250514) for AI tutoring
- User provides their own API key (stored in localStorage)
- All progress stored in **localStorage** (device-specific, no sync yet)

### Navigation / modules

| Nav ID | Page key | Handler | Notes |
|--------|----------|---------|-------|
| Dashboard | `dashboard` | `renderDashboard()` | Stats, streak, subject grid |
| Raisonnement Abstrait | `abstract` | `renderAbstractModule()` | Custom visual trainer, no API needed |
| Raisonnement Numérique | `numerical` | `renderNumericalModule()` | Has 2 tabs (see below) |
| Raisonnement Verbal | `verbal` | `renderTraining('verbal')` | Claude chat + PDF upload |
| Connaissances UE | `eu-knowledge` | `renderTraining('eu-knowledge')` | Claude chat + PDF upload |
| Compétences Digitales | `digital` | `renderTraining('digital')` | Claude chat |
| EUFTE (Essai) | `eufte` | `renderTraining('eufte')` | Claude chat + PDF upload |
| Ma Progression | `progress` | `renderProgress()` | History, per-subject scores |
| Paramètres | `settings` | `renderSettings()` | Name, exam date, custom instructions |

### State object

```js
state = {
  apiKey: '',
  currentPage: 'dashboard',
  currentSubject: null,
  messages: {},          // per-subject chat history
  uploadedFiles: {},     // per-subject uploaded PDF text content
  history: [],           // session log [{subject, mode, score, date, timestamp}]
  abstractHistory: [],   // abstract quiz results
  settings: {
    userName: '',
    examDate: '2026-09-15',
    customInstructions: '...',
    dailyGoal: 20
  },
  numericalTab: 'chat',  // 'chat' | 'library'
  fcFilter: 'Tous'       // flashcard category filter
}
```

Saved to `localStorage` key: `ad5prep_state`
API key saved separately: `ad5prep_apikey`
Flashcards saved separately: `ad5prep_flashcards`
Sidebar collapsed state: `ad5prep_sidebar_collapsed`

---

## Module details

### Abstract Reasoning (fully custom, no API needed)
- **Shape engine**: SVG-based, 10 shape types, 7 colors, fill/outline, rotation
- **10 hand-crafted questions**: series + 3×3 matrix puzzles, easy/medium/hard
- **3 modes**: Theory & Method / Practice (unlimited time) / Simulation (10 min timer)
- **Image upload feature**: user can upload PNG/JPG scans of real EPSO tests → Claude analyses the pattern via vision API
- Key functions: `absRender()`, `absStartQuiz()`, `renderAbsHome()`, `renderAbsQuiz()`, `renderAbsResults()`, `renderAbstractModule()` (entry point from main app), `renderAbsImageUpload()`, `practiceAbsImage()`, `analyzeAbsImage()`
- State: `absState` (separate from main `state`)
- Uploaded images: `absUploadedImages` array (session only, metadata in localStorage)

### Numerical Reasoning (2-tab module)
**Tab 1 — Chat**: Standard Claude chat with quick action buttons
**Tab 2 — Flashcard Library**:
- 20 pre-loaded cards covering: Pourcentages (5), Ratios & Proportions (3), Moyennes (2), Tableaux & Données (3), Index & Indices (2), Conversions (1), Séries temporelles (1), Estimation rapide (2), Probabilités (1), Stratégie EPSO (1)
- Each card has: `cat`, `title`, `concept`, `epso`, `example`, `traps`
- Stored as JSON in `localStorage` key `ad5prep_flashcards`
- Default cards defined in `DEFAULT_FLASHCARDS` array (JSON format, double-quoted)
- User can generate new cards via Claude API (async `generateFlashcard()`)
- User can delete cards, filter by category, open in modal, or "Approfondir" (jump to chat)
- Key functions: `renderNumericalModule()`, `switchNumericalTab()`, `renderFlashcardLibrary()`, `renderNumericalChat()`, `openFlashcard()`, `generateFlashcard()`, `askClaudeAboutCard()`, `getFcState()`, `saveFcState()`

### Other modules (Verbal, EU Knowledge, Digital, EUFTE)
All use `renderTraining(subjectKey)` — standard Claude chat interface with:
- Mode selector (Quiz / Lecture / Simulation / Explain)
- Quick action buttons
- PDF upload (Verbal, EU Knowledge, EUFTE only)
- Chat area with message history

---

## UI / Design

- **Color scheme**: Navy (`#0f1b2d`), Gold (`#c9a84c`), Cream (`#faf6ee`)
- **Fonts**: Playfair Display (serif, titles) + DM Sans (body) + DM Mono (code/numbers)
- **Sidebar**: Collapsible (56px icons / 240px expanded), collapse state persisted
- **Responsive**: Mobile menu with overlay for small screens
- **Toast notifications**: `showToast(msg)` function

---

## API usage

All calls to `https://api.anthropic.com/v1/messages`

| Call site | max_tokens | Purpose |
|-----------|-----------|---------|
| Main chat (`sendMessage`) | **3000** | Full explanations, quizzes |
| Flashcard generation | 1000 | JSON-only response |
| Image analysis (`analyzeAbsImage`) | **2000** | Vision + reasoning |

Model: `claude-sonnet-4-20250514` everywhere.

---

## Known issues fixed in current version

- `revealAnswer is not defined` → renamed to `absRevealAnswer`, stale reference removed
- `missing } after property list` → flashcard strings had unescaped French apostrophes in single-quoted JS strings → converted all DEFAULT_FLASHCARDS to JSON double-quote format
- `navigate is not defined` → cascading failure from above syntax crash
- Backtick regex `/^```json?\n?/` inside template literal → replaced with `/(^[\s\S]*?({[\s\S]*})[\s\S]*$)/`
- `max_tokens: 1000` cutting explanations short → raised to 3000 for main chat

---

## What is NOT yet built (pending features)

### Priority (agreed, build next session)

- [ ] **Image discussion thread** — after Claude analyses an uploaded abstract image, user needs a full conversation thread to push back, correct errors, discuss. Same chat UI as other modules, anchored to the image. Scope: **Abstract Reasoning only for now** (can expand later).

- [ ] **Uploads feed creativity, not just analysis** — this is a fundamental architecture change agreed by user:
  - Current (wrong): upload image → Claude analyses *that specific image* once
  - Correct: upload image → Claude uses it as **inspiration to generate new CSS exercises** and expand theory
  - Current (wrong): upload PDF → Claude answers questions about that document
  - Correct: upload PDF → Claude **absorbs the methodology and theory**, then teaches it and creates exercises in that style
  - For images: primary output = **theory explanation of the pattern type + generation of new similar exercises**
  - For PDFs: primary output = **summary of what was learned + structured training** (not silent absorption)

- [ ] **PWA support** (Add to Home Screen on phone) — decision deferred, easy to add at end

- [ ] **Cross-device sync** — GitHub Gist approach proposed, decision deferred

- [ ] **PDF book ingestion (large)** — user has 700+ page EPSO prep books. Agreed architecture:
  - Files under ~80 pages: current upload works fine
  - Large books: need chunking + RAG, requires small backend (Supabase), or user splits PDF by chapter first (recommended short-term: use PDF24 to split by chapter, upload each chapter to its module)

- [ ] **More abstract reasoning questions** — only 10 currently

### Workflow clarification (important)
User will use the app daily for studying. When improvements are needed:
- **Do NOT** try to self-modify from within the app
- **Come back to Claude.ai**, upload both `AD5_Study_App.html` + `AD5_Project_Memory.md`, describe what's needed
- Claude.ai is the workshop; the HTML file is the daily tool

---

## Open decisions / user preferences

- PWA vs standalone phone app vs both: **not yet decided** (defer until app content is complete)
- Cross-device sync: **not yet decided**
- Discussion/correction chat: confirmed needed for **Abstract Reasoning first**, other modules later
- Upload purpose: confirmed as **creative input / theory expansion**, not one-shot document analysis

---

## How to continue in a new session

1. Upload both `AD5_Study_App.html` and this `AD5_Project_Memory.md`
2. Say: *"Here are my files. Please read the memory file and continue where we left off."*
3. Claude reads this file in ~30 seconds and is fully up to speed without re-reading all 3,300 lines of HTML

---

## Session 3 — Mobile UX Overhaul (March 2026)

### What was done
Full mobile-first redesign. Priority: app is primarily used on phone.

### Key changes made

**Bottom navigation bar (new)**
- Replaces hamburger menu entirely on mobile
- 5 tabs: Accueil / Abstrait / Numérique / Verbal / Plus (⋯)
- "Plus" tab opens a slide-up panel with: EU, Digital, Essai, Progrès, Réglages
- API key input also accessible inside the More panel on mobile
- Active state indicator (gold top border + gold text)
- `updateBottomNav(page)` called on every navigate()

**New JS functions**
- `updateBottomNav(page)` — syncs bottom nav active state
- `toggleMorePanel()` — opens/closes slide-up more panel
- `closeMorePanel()` — closes more panel + overlay
- `saveApiKeyMobile()` — saves API key from mobile input, syncs to desktop input

**Meta tags added**
- `apple-mobile-web-app-capable` (iOS Add to Home Screen)
- `apple-mobile-web-app-status-bar-style`
- `theme-color` for Android
- `viewport-fit=cover` for notch support

**Layout fixes**
- Content padding: 36px → 16px on mobile, with 80px bottom padding for nav bar
- Dashboard stats: 4 cols → 2x2
- Subject cards: 2 cols → 1 col  
- Progress grid: `repeat(3,1fr)` → `repeat(auto-fill, minmax(200px,1fr))`
- Flashcard grid: auto-fill → 1 col on mobile
- Filter bar: scrollable horizontally (no-wrap) on mobile
- Generate bar: row → column on mobile

**Abstract trainer mobile fixes**
- Mode cards: 3 cols → 1 col, horizontal layout with icon
- Shape cells: 80px → 56px on mobile
- Matrix: 80px cells → 64px cells
- Options grid: 4 cols → 2x2 on mobile
- Theory grid: 2 cols → 1 col

**Chat improvements**
- Send button: shows only "→" icon on mobile (hides text via `.send-text` span)
- `font-size: 16px` on textarea (prevents iOS auto-zoom on focus)
- Message bubbles: max-width 80% → 92% on mobile
- Toast: moved above bottom nav (bottom: 76px) on mobile

**Sidebar**: Hidden entirely on mobile (display:none). Only shown on desktop (>768px).

### Pending (next session)
1. Image discussion thread (chat anchored to uploaded abstract image)
2. Uploads as creative fuel (generate exercises from images/PDFs)
3. More abstract reasoning questions (only 10 currently)

---

## Session 3 continued — Image Discussion Thread

### What was done
Replaced the one-shot image analysis panel with a full persistent conversation thread.

### Architecture change
**Before**: `practiceAbsImage()` rendered a static div. Clicking quick-action buttons called `analyzeAbsImage()` which wiped the div and replaced it with a new single answer.

**After**: Full chat UI matching other modules. Image is embedded in the first API message of the thread (vision). Subsequent turns are text-only but Claude retains the visual context. Conversations persist in `absState.imageThreads[imageId]` as an array of `{role, content}` objects.

### New functions
- `absImageSend(id)` — main send function, builds API messages with image on first turn
- `absImageSendQuick(id, text)` — fills input and sends (used by quick-action buttons)
- `absImageRestoreThread(id)` — re-renders existing thread when returning to image
- `absImageHandleKey(event, id)` — Enter to send, Shift+Enter for newline
- `analyzeAbsImage(id, prompt)` — now just a wrapper calling `absImageSendQuick` (kept for compatibility)

### Key design decision
Image is sent **only in the first message** of each thread (not on every turn). This is intentional: Claude retains visual context across turns in a single API conversation via its context window. Sending the image every turn would be wasteful and hit token limits faster.

### Thread storage
`absState.imageThreads = { [imageId]: [{role, content}, ...] }`
Lives in memory only (not localStorage — images are base64 so localStorage would overflow).
Threads are lost on page reload. This is acceptable for now.

### UI layout
- Left column: image thumbnail + 4 quick-action buttons (vertical)
- Right column: chat thread + input row (same `.session-area` / `.session-input-row` classes)
- On mobile (≤768px): stacks to single column

### Pending (next session)
1. Uploads as creative fuel (generate exercises from images/PDFs)
2. More abstract reasoning questions (only 10 currently)

---

## Session 3 continued — Abstract Module: 4 new features

### 1. End session early button
In practice mode, a "Terminer la session →" button appears top-right (alongside the score counter). In simulation mode this space is taken by the timer, so the button doesn't appear there (by design — simulation should be uninterrupted).

### 2. Adaptive difficulty system
**Storage**: `localStorage` key `ad5prep_abs_perf` — object keyed by `"type_difficulty"` (e.g. `"series_hard"`, `"matrix_easy"`), each tracking `{correct, total}`.
**Recording**: `recordAbsPerf(q, isCorrect)` called in `absRevealAnswer()` after every answer.
**Selection**: `absSelectQuestionsAdaptive(count)` replaces the old random shuffle. Assigns weights inversely proportional to success rate (0% success → weight 3.0, 100% → weight 0.4). New/unseen questions get weight 1.5.
**Display**: `absGetPerfSummary()` shows a one-line breakdown by type+difficulty on the home screen.

### 3. Contest answer / discuss with Claude
After an answer is revealed, the feedback box now has two small buttons:
- **"💬 Discuter avec Claude"**: opens an inline chat anchored to that question. The system prompt gives Claude the full question context (type, correct answer, options, explanation). If the user argues convincingly, Claude can concede. Thread stored in `absState.contestThreads[qId]`.
- **"🚩 Signaler une erreur"**: flags the question ID in `localStorage` key `ad5prep_abs_flagged`. Button becomes disabled + red after clicking. (No API call — purely local flag for future reference.)

### 4. Batch analyze & style profile
**Button**: single "✦ Analyser le lot & améliorer l'App" button appears when ≥1 image is uploaded.
**Process**: sends up to 5 images in one API call. Claude analyzes each (pattern type, rules, difficulty, correct answer detection, traps) then writes a `[PROFIL DE STYLE]` section.
**Storage**: `localStorage` key `ad5prep_abs_style_profile` — object with `{summary, date, imageCount, fullAnalysis}`.
**Usage**: `absGetStyleInjection()` returns the profile as a string injected into the abstract module's system prompt (in `sendMessage` for subject key `'abstract'`).
**Correct answer detection**: Claude can read color-highlighted answers (e.g. green), infer from context, and will note confidence level when no answer is provided.
**Display**: stored profile shown on the abstract home screen with date + image count. "Réinitialiser" button clears it.

### Pending (next session)
1. Uploads as creative fuel for question generation (related to style profile — next step: generate new SVG questions inspired by the profile)
2. More abstract reasoning questions (only 10 currently)

---

## Session 3 continued — Bibliothèque (Documentation Library)

### What was built
A dedicated "Bibliothèque" section accessible from both sidebar and mobile More panel.

### Navigation
- Sidebar: new nav item "📁 Bibliothèque" under "Outils"
- Mobile More panel: new "📁 Biblio." item (6th item)
- Page title: "Bibliothèque"
- State: `state.libraryTab` stores active tab ('abstract' | 'numerical' | 'other')

### Structure
3 tabs: Abstrait (live) | Numérique (placeholder) | Autres (placeholder)

**Abstract tab contains:**
1. Upload zone — drag/drop or click, calls `processAbsFilesLib()` which re-renders library on completion
2. Style profile card — full summary text, date, image count, "Ré-analyser" button, "Réinitialiser" button
   - Explicit note: "Supprimer des images n'efface pas ce profil — il s'agit de la connaissance extraite, indépendante des sources."
3. Image grid — `repeat(auto-fill, minmax(200px, 1fr))`, hover border highlight, thread count badge (gold pill showing nb of discussion turns)
   - "💬 Discuter" → calls `practiceAbsImageFromLib(id)` → navigates to abstract module, then opens image practice
   - "🗑 Supprimer" → `deleteAbsImageLib(id)` → confirm dialog with profile-independence note → removes image + clears its thread from `absState.imageThreads`

### Abstract home screen (simplified)
- Upload zone still present (quick drop-in)
- Image grid REMOVED from abstract home — replaced by "Voir les N image(s) →" link pointing to library
- Batch analyze button still on abstract home when images exist
- Profile shown as compact card (first 180 chars + "...")

### Key functions
- `renderLibrary()` — main render
- `setLibraryTab(tab)` — switches tab, re-renders
- `renderLibraryAbstract(imgs, profile)` — abstract tab content
- `renderLibraryNumerical()` / `renderLibraryOther()` — placeholder tabs
- `handleAbsFilesLib()` / `dropAbsImageLib()` / `processAbsFilesLib()` — library-specific upload (re-renders library, not abstract home)
- `deleteAbsImageLib(id)` — deletes image + thread, shows confirm with note
- `practiceAbsImageFromLib(id)` — navigate('abstract') + setTimeout 100ms + practiceAbsImage(id)

### Pending
1. Generate new SVG questions from style profile ("uploads as creative fuel")
2. Simulation pause/resume (save timer + question state to localStorage)
3. More abstract reasoning questions (10 currently)

---

## Session 3 continued — Dynamic Question Bank (text questions)

### Architecture
Two-layer question bank:
- **Built-in**: 10 hardcoded SVG questions in `QUESTIONS[]` constant
- **Generated**: Claude-generated text questions in `localStorage` key `ad5prep_abs_generated`
- `absGetFullBank()` merges both at runtime
- `absSelectQuestionsAdaptive()` already updated to use full bank
- `absStartQuiz()` uses `absGetFullBank().length` for practice count (capped at 15)

### Generation
`absGenerateQuestions(count, silent)`:
- Calls Claude with: style profile (if available) + list of existing question titles (to avoid duplication) + count
- Prompts for text/unicode questions with `renderType: "text"`
- Robust JSON extraction via regex `\[[\s\S]*\]`
- Validates: title, options (array of 4 strings), answer (number), explanation, display field
- Assigns unique numeric IDs, adds `generated: true` + `generatedAt` date
- Saves to localStorage, returns count added

`absAutoGenerateIfNeeded()`: silently generates 3 questions if: API key set + style profile exists + bank < 18 total. Called at quiz start.

Manual button: "✦ Générer 5 nouvelles questions" on abstract home (only shown if API key set).
Clear button: small ✕ next to "N générées" stat (only shown if generated > 0).

### Text question format
```json
{
  "renderType": "text",
  "display": "▲ → ▲▲ → ▲▲▲ → ?",
  "options": ["▲▲▲▲", "▲▲", "■■■■", "●●●"],
  "answer": 0,
  "generated": true,
  "generatedAt": "05/03/2026"
}
```

### Rendering
`renderTextQuestion(q)`: monospace box with large Unicode display, letter-spacing, light "✦ Générée le..." label.
Options: plain text strings rendered large (22px) instead of SVG shapes.
Branch in `renderAbsQuiz`: `q.renderType === 'text'` → `renderTextQuestion(q)`, else SVG path.

### Style profile integration
Style profile summary injected into generation prompt as `[PROFIL DE STYLE EPSO APPRIS]` block. This is the "capability improvement" mechanism — the richer the profile, the better the generated questions match real EPSO patterns.

### ⚠️ PENDING: SVG question generation (deferred by user choice)
User agreed to text-first approach. SVG generation is a future phase.
**When raising this**: explain that SVG generation means Claude outputs JSON matching the shape renderer schema (`{type, color, fill, rotation}`). It's more fragile (needs validation/fallback) but produces visually identical questions to the built-in ones. User should decide if the added complexity is worth it at that point.
Pros: visual consistency, works offline after generation
Cons: more complex prompt engineering, higher error rate, needs schema validation layer


---

## Session 3 continued — #1 Sim pause/resume · #2 Feedback loop · #4 Verbal learning

### #1 — Simulation pause/resume
**Trigger**: `absShowHome()` and `navigate()` (when leaving abstract mid-simulation) both call `absSavePausedSimulation()`.
**Storage**: `localStorage` key `ad5prep_abs_sim_pause` — saves `{questions, currentQ, score, answers, timeLeft, savedAt}`.
**Resume**: on `absStartQuiz('simulation')`, checks `absPausedSimulation()`. If found, shows confirm dialog with stats (question N/10, time left, score). User confirms → `absResumeSimulation()` restores full state + restarts timer. User declines → `absClearPausedSimulation()` + fresh start.
**Banner**: on abstract home, if a paused simulation exists, a gold-bordered "Reprendre la simulation" card appears above the mode grid, showing current progress.
**Clear**: called after successful resume, and when user declines resume at start.

### #2 — Feedback → style profile loop
**Trigger**: after 4+ messages (2 exchanges) in a contest chat, a "✦ Extraire ce feedback pour améliorer les futures questions" button appears below the thread.
**`absLearnFromContest(qId)`**: sends the full conversation + current profile to Claude, asks it to extract 1-3 concrete actionable improvement points. Appends to profile with a `[Feedback utilisateur — date]` tag. Saves updated profile.
**Effect**: next time questions are generated, the profile includes the user's feedback (e.g. "éviter les alignements Unicode décalés", "les matrices hard doivent combiner 3 règles").
**UI**: button becomes "✓ Feedback intégré au profil de style" after success.

### #4 — Verbal learning loop
**Storage**: `localStorage` key `ad5prep_verbal_style_profile` — same structure as abstract profile.
**`verbalGetStyleInjection()`**: injects profile into verbal system prompt in `sendMessage()`.
**`renderVerbalLearningSection()`**: "✦ Améliorer ce module" collapsible section at top of verbal module containing:
  - Text paste area (textarea, 3000 char limit sent to API)
  - Image upload zone (drag/drop + click, up to 4 images, preview thumbnails with ✕ remove)
  - "✦ Analyser & améliorer le module verbal" button
  - Profile summary display (first 200 chars) once a profile exists
**`verbalAnalyzeAndLearn()`**: sends images + pasted text in one API call. Extracts `[PROFIL DE STYLE VERBAL]` section. Saves profile. Clears inputs. Re-navigates to verbal to refresh.
**Image memory**: `verbalUploadedImages[]` in-memory only (not persisted — just for the analysis call).
**Verbal quick actions added**: "📝 Question V/F/ITD" and "⏱ Mini-simulation" buttons.
**Note**: Verbal library tab in Bibliothèque still a placeholder — should be built when adding persistent image management for verbal (same pattern as abstract library tab).

### Remaining TODO (updated priority)
1. SVG question generation (deferred — see memory for trade-off brief)
2. Verbal library tab in Bibliothèque
3. EU Knowledge learning loop (same pattern as verbal)
4. Theory section updatable from style profile
5. More built-in abstract questions (10 currently)
6. Cross-device sync (requires backend)
7. PWA manifest (~15 min, cosmetic)

---

## Session 3 continued — SVG generation + Library tabs (Verbal/EU)

### SVG Question Generation (5-layer safety system)

**Layer 1 — Constrained prompt**
SVG prompt lists EXACT allowed values: shapes (circle, square, triangle, diamond, pentagon, hexagon, star, cross, arrow, semicircle), colors (blue, red, gold, green, purple, teal, white, dark), rotations (0/45/90/135/180/225/270/315). Includes a complete worked example for series AND matrix. One call, up to 5 questions.

**Layer 2 — Auto-repair dictionaries**
`COLOR_SYNONYMS`: yellow→gold, cyan→teal, black→dark, violet→purple, etc.
`SHAPE_SYNONYMS`: rectangle→square, oval→circle, rhombus→diamond, plus→cross, right_arrow→arrow, etc.
`repairShapeDef(def)`: normalizes type+color, returns `{repaired, repairs[]}`.

**Layer 3 — Strict validator**
`validateSVGQuestion(q)`: checks all fields, repairs inplace, returns `{valid, errors[], warnings[], autoRepairs[]}`.
Validates: title, instruction, explanation, type (series/matrix), difficulty (defaults medium), answer (0-3), options (array of 4), sequence (has null), matrix (3×3, exactly 1 null).

**Layer 4 — Render-time safety**
`svgShape()` default case draws a circle for unknown types — never crashes.
Quiz renderer handles renderType 'svg' same as built-in, 'text' same as before.

**Layer 5 — User-facing debug**
- Per-question badge: "🔷 SVG généré · date · 🔧 N réparation(s)" + "⚠️ Signaler" button
- `absReportQuestionIssue(id)`: opens contest chat pre-filled with structured bug report including auto-repairs applied
- `absShowGenerationReport()`: appears at top of abs-main when issues detected — shows log, repairs, rejections, "↺ Regénérer" button
- `absRetryRejected()`: re-runs generation for 2 questions

**Flow**
1. Try SVG generation → validate → if ≥1 accepted: save + report
2. If 0 SVG accepted: fallback to text generation → validate → save + report
3. If text also fails: show error toast
4. Toast summarizes: "X SVG + Y texte · Z réparées · W rejetées"
5. Debug report shown only if there were repairs, rejections, or SVG fallback

**Contest chat threshold lowered**: 2 messages (1 exchange) now triggers "✦ Extraire ce feedback" button (was 4).

### Library tabs (Verbal + EU)

`renderLibrary()` now has 4 tabs: Abstrait · Verbal · UE · Numérique

**Verbal tab (`renderLibraryVerbal`)**:
- Shows full verbal style profile if exists (summary, date, note about injection)
- "Réinitialiser" button → `clearVerbalStyleProfile()`
- If no profile: empty state with link to verbal module
- "→ Aller au module Verbal" button (where the actual upload lives)
- Note: individual file management for verbal = future feature

**EU tab (`renderLibraryEU`)**:
- Empty state with description
- "→ Aller au module UE" button
- EU profile + file management = future feature

**Numerical tab**: still placeholder

### Updated TODO
1. Numerical learning loop (same architecture as verbal)
2. EU learning loop
3. Individual file management in Library (verbal, EU, numerical)
4. Theory section updatable from style profile
5. More built-in abstract questions (still 10)
6. PWA manifest (~15 min)
7. Cross-device sync (requires backend)

---

## Session 4 — Image deduplication + analyzed tracking

### Problem solved
- Images uploaded multiple times (same file, different name/format) were all stored and re-analyzed
- Batch button always showed total image count even when all images already analyzed
- Profile was overwritten rather than enriched on re-analysis

### Fingerprinting & deduplication
`absImageFingerprint(dataUrl, fileSize)` → `"${fileSize}_${b64.substring(0,64)}"` — catches exact duplicates and re-saves regardless of filename/format.
On upload: if fingerprint matches any existing image → toast "⚠️ déjà importé — ignoré", skip.
Both `processAbsFiles()` and `processAbsFilesLib()` merged into shared logic with dedup check.

### Analyzed tracking
Each image has `analyzed: false` on upload, set to `true` after successful batch analysis.
`absAnalyzeBatch()` now: filters to `!i.analyzed` only → sends only new images → marks them `analyzed: true` → saves.
If 0 unanalyzed: toast "✓ Toutes les images ont été analysées — profil à jour", no API call.

### Profile now extends, not replaces
Existing profile included in analysis prompt as "PROFIL EXISTANT (à enrichir, ne pas effacer)".
`analyzedIds[]` array tracked in profile object.
`imageCount` accumulates across sessions.

### UI changes
- Abstract home button: "✦ Analyser N nouvelle(s) image(s)" (only unanalyzed count)
- Library grid: green ✓ Analysé badge / grey ⏳ Non analysé badge per card
- Analyzed images get green border in library
- All batch buttons context-aware: show count or "✓ tout analysé"

---

## Session 5 — Steps 1–6 (major feature session)

### Step 1 — Built-in abstract questions: 10 → 25
IDs 11–25 added to QUESTIONS array. Coverage:
- Series (11–18): rotation +45°, dual-cycle (form+colour), alternating fill+colour, 3-rule combined, rotation +90° cycle, colour+fill cycle, double-appearance pattern, 3-simultaneous-rule star
- Matrices (19–25): colour sudoku, form+fill by row/col, rotation +90° matrix, dual-rule (form+colour+fill), diagonal+fill, counting (multi-shape cells), 3-attribute combined

### Step 2 — Numerical learning loop
Storage: `ad5prep_numerical_style_profile`, `ad5prep_numerical_generated`
- `renderNumericalLearningSection()`: textarea paste + image upload + analyze button + profile preview — injected at top of numerical chat tab
- `numericalAnalyzeAndLearn()`: sends images+text to Claude, extracts [PROFIL DE STYLE NUMÉRIQUE], saves, re-renders
- `numericalGetStyleInjection()`: injects profile into numerical system prompt in sendMessage
- `generateTextQuestions('numerical', count, silent)`: generates JSON QCM with dataBlock (markdown table) + question + 4 options. Validates: title, question, options[4], answer 0-3, explanation
- `autoGenerateNumericalIfNeeded()`: silent generation if profile exists + bank < 10
- Quick-action buttons: "🎯 Quiz questions générées (N)" + "✦ Générer 5 questions numériques"
- Bank stats bar with clear button shown in numerical chat

### Step 3 — EU Knowledge learning loop
Storage: `ad5prep_eu_style_profile`, `ad5prep_eu_generated`
- Same architecture as numerical. EU-specific analysis prompt covers: institutions, traités, politiques, acronymes, pièges (dates proches, compétences partagées)
- `renderEuLearningSection()` injected into renderTraining for 'eu-knowledge'
- `euGetStyleInjection()` injected into sendMessage for eu-knowledge
- `generateTextQuestions('eu', count, silent)`: EU QCM with domain tag
- EU questions include "domain" field for future filtering

### Step 4 — Library: full file management for Verbal, Numerical, EU tabs
Shared helpers:
- `renderLibraryProfileCard(profile, clearFn, label)`: gold profile card or empty state
- `renderLibraryFileGrid(images, module, analyzeFn, deleteFn)`: file grid with ✓/⏳ badges, delete per file, smart batch button
- `renderLibUploadZone(module, icon, label)`: upload zone with drag/drop + click

Per-module image stores (localStorage):
- `ad5prep_verbal_lib_images`, `ad5prep_numerical_lib_images`, `ad5prep_eu_lib_images`
- Arrays: `verbalLibImages`, `numericalLibImages`, `euLibImages` — loaded at init via `loadLibImages()`
- Fingerprinting: same size+b64 dedup as abstract
- `analyzeLibImages(module)`: sends unanalyzed images to Claude, marks analyzed, updates profile, saves

Delete functions: `deleteVerbalLibImage(id)`, `deleteNumericalLibImage(id)`, `deleteEuLibImage(id)`
`rerenderLibraryTab()`: re-renders library if currently on that page

Library tabs now show: profile card + upload zone + file grid + generated questions count + clear button

### Step 5 — Theory section updatable from style profile
Storage: `ad5prep_abs_custom_theory` — `{html, date, profileDate}`
- "✦ Personnaliser avec mon profil" button appears in theory header if a style profile exists
- `absUpdateTheoryFromProfile()`: sends profile summary to Claude, asks for personalized pedagogical HTML (max 400w), saves, re-renders
- Custom theory shown as gold-bordered section ABOVE the standard 5-type reference (which stays)
- "↺ Théorie par défaut" button shown if custom theory exists → calls `resetAbsTheory()`

### Step 6 — Mistake pattern analysis
Storage: `ad5prep_abs_answer_log` — `{right:[], wrong:[]}` (capped 100 each)
Storage: `ad5prep_mistake_analysis` — `{summary, full, date, wrongCount, rightCount}`

Data collection:
- `recordAbsAnswer(q, chosenIndex, correct)`: called in `absRevealAnswer()` after every answer
- Records: qId, title, type, difficulty, renderType, chosen index, correct index, date

Analysis:
- `runMistakeAnalysis()`: sends last 30 wrong + 20 right answers to Claude with difficulty/type breakdown
- Claude returns [RÉSUMÉ] (2-3 sentences) + [ANALYSE DÉTAILLÉE] (4-6 structured points)
- Threshold: 5 wrong answers minimum to unlock

Display:
- `renderMistakeAnalysisPanel(analysis, log, canAnalyze, context)`: shared renderer
  - context='progress' → shows summary only + "→ Voir détail" link
  - context='module' → shows full analysis
- Progress page: "🔍 Analyse de mes erreurs" section between subject grid and history
- Abstract home: detailed panel below progression stats (appears after 5+ answers)
- Freshness indicator: shows "N nouvelle(s) erreur(s) depuis la dernière analyse"
- Refresh button when new errors accumulated; Clear button always

### Next TODO (updated)
1. Spaced repetition scheduler (was item A in new ideas)
2. Daily exam readiness score (was item B)
3. PWA manifest (~15 min)
4. Cross-device sync (requires backend)
5. Mistake analysis for Verbal/Numerical/EU (currently abstract-only)

---

## Session 6 — Spaced Repetition + PWA Manifest

### Spaced Repetition System (SM-2 algorithm)

Storage: `ad5prep_srs` — object keyed by question ID, each with `{interval, repetitions, easeFactor, nextReview, lastReview, qId}`.

**Core SM-2 algorithm** (`srsUpdateCard(qId, quality)`):
- quality 0-5 scale: 1=wrong, 3=correct with difficulty, 4=solid correct
- Correct: interval grows (1d → 3d → interval×easeFactor)
- Wrong: reset to interval 0, due immediately next session
- Ease factor adjusts per SM-2 formula, floor at 1.3

**Functions**:
- `getSrsData()` / `saveSrsData()` — localStorage persistence
- `srsUpdateCard(qId, quality)` — updates card schedule after each answer
- `srsQualityFromAnswer(isCorrect, wasHesitant)` — converts answer to SM-2 quality
- `srsDueQuestions()` — returns all due questions + up to 5 new unseen cards
- `srsStats()` — returns `{dueCount, reviewedToday, totalCards, unseenCount}`
- `renderSrsDueSection(context)` — renders gold "N questions à réviser" banner (dashboard or abstract)
- `absStartSrsReview()` — starts a quiz with only due questions (mode='srs')

**Integration points**:
- `absRevealAnswer()` calls `srsUpdateCard()` after every answer
- Dashboard shows SRS due section between stats grid and subject selector
- Abstract home shows SRS due section inside the mode grid area
- Quiz header shows "📅 Révision espacée" badge when in SRS mode
- SRS review sessions show "Terminer la session" button (like practice)

### PWA Manifest (inline)

Single-file PWA approach — manifest generated as Blob URL at page load:
- `name`: "AD5 Prep — EPSO Trainer"
- `short_name`: "AD5 Prep"
- `display`: "standalone"
- `background_color` / `theme_color`: "#0f1b2d"
- SVG icon (gold "5" on navy background) as data URL, 512x512
- Apple touch icon and favicon also added as inline SVG data URLs
- `apple-mobile-web-app-title` meta tag added

To install on phone: open file in Safari/Chrome → Share → "Add to Home Screen"

### Updated TODO
1. Daily exam readiness score
2. Cross-device sync (GitHub storage, when features stable)
3. Mistake analysis for Verbal/Numerical/EU (currently abstract-only)
4. Spaced repetition for Numerical/EU generated questions (currently abstract-only)
