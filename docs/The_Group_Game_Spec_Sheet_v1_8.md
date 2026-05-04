# The Group Game — Project Spec Document
*For use at the start of any development session. Paste this into Claude before asking for changes.*

---

## What This Project Is

**The Group Game** is a single-file HTML character simulation where five characters are placed in a shared gathering and navigate a central conflict through a structured arc. The player does not control characters directly — instead they right-click nodes on a map to connect them, triggering AI-generated narrative beats. The simulation unfolds through pressure, residue, and escalation across three acts until a climactic resolution scene surfaces five character-positioned approaches to the conflict.

The project runs as a **standalone HTML file opened directly in the browser**. It calls the Anthropic API (`claude-sonnet-4-6`) directly from the client using an API key entered by the user on load. All logic, styles, and HTML live in one file: `the_group_game.html`.

**Design intent:** The simulation is an experiential educational tool — players learn what real conflict dynamics look like from the inside, and what genuinely works to address them. Philosophical dimensions emerge from behavior under pressure, not from reflection. Play should feel like observing real people in a genuinely difficult situation.

---

## File Structure

1. `<style>` block — all CSS, including Ritual design system variables, setup screen styles, map screen styles, overlays, animations
2. `<body>` — two major screens:
   - `#setup-screen` — setting/character/conflict/relationship configuration
   - `#map-screen` — the live simulation (hidden until setup completes)
3. `<script>` — all JavaScript, organized roughly as:
   - Budget tracking system
   - Audio system (sfx + setup music layers)
   - API key modal and management
   - Theme toggle
   - Setup constants (archetypes, wound territories, OCEAN system, setting pool, conflict archetypes, character domains)
   - Setup UI logic (slot building, generation, readiness checks, unlock sequencing)
   - Simulation state (`simulation`, `CFG`, `mapNodes`, `residueNodes`, etc.)
   - Map rendering (SVG rings, node placement, connections)
   - Drift engine (node movement, collision, absorption)
   - AI generation functions
   - Resolution library system
   - Act advancement logic
   - Panel/log UI
   - Export/Import functions
   - Event binding (`bindMapEvents`)

---

## Design System — "Ritual"

The visual design uses warm brown-black tones and Libre Baskerville + Space Mono typography.

**Color tokens:**
- `--void: #0f0c0a` — warm brown-black background
- `--gold: oklch(72% 0.11 68)` — primary accent
- `--blue: oklch(68% 0.10 240)`, `--teal: oklch(72% 0.10 175)`, etc. — all at consistent chroma/lightness
- Cards: `#1e1916` fully opaque, gold top-border accent, box shadows
- Light mode: warm parchment tones (`#f5f0e8` background)

**Typography:**
- Libre Baskerville + Space Mono (replaced Cinzel Decorative + Cormorant Garamond)
- Global `letter-spacing:0.012em`, `word-spacing:0.04em`

**Background:** Warm radial gradient texture on setup screen via `::before` pseudo-element.

**Section headers:** Hints stack below titles (column layout, not inline).

---

## Standalone Deployment

- Opened directly in browser as a local HTML file
- API key stored in `localStorage` as `tgg_api_key`
- Budget tracked in `localStorage` as `tgg_budget`
- `throttledFetch` injects required Anthropic headers
- Audio files hosted at: `https://raw.githubusercontent.com/jakobbaron/The-Group-Game/main/`

---

## Two Screens

### Setup Screen

Sections unlock sequentially. Readiness bar tracks completion.

**1. Setting** — generates `label`, `world`, `namingCulture`, `characterDomain`, `characterRoles`, `conflictTexture`. No `atmosphere` field.

**2. Characters (5)** — OCEAN system, wound territories, per-slot World toggle.

Character generation includes consolidated **diversity constraints** (all in one `diversityConstraint` block, injected per slot):
- **Role diversity** — max 1 information-handler per ensemble (archivist, clerk, journalist, etc.)
- **Conflict type diversity** — max 2 of any `conflictType` per ensemble
- **Moral orientation diversity** — max 2 of any orientation per ensemble

**Wound field** now appends: *"End with one plain sentence: how this wound shapes the way they move through difficulty — not what happened, not what they think about it, but what it makes them do differently than someone without it would."*

**Moral error field** now appends: *"End with one plain sentence: how this misfire shapes the way they move through difficulty — what it makes them do differently than someone without this error would, specifically under pressure."*

**3. Conflict Seed** — `label`, `domain`, `environment`, `gatheringPremise`, `pressure`, `conflictVectors`.

15 full-phrase domains (accountability permanently removed). `conflictVectors`: umbrella + two situations + latent.

**4. Relationships (10)** — max tokens 2000. Pre-sim relationship dynamics seed facet system.

**5. Events** — all three emerge from play.

**Register:** Accessible / Standard / Complex via `registerInstruction()`.

---

### Map Screen (The Simulation)

**Connection mechanic — right-click:**
- Right-click any node → enters connect mode with that node as A
- Right-click a second node → completes connection
- Esc → cancels connect mode
- Toolbar shows hint text ("Right-click any node to connect") that updates to "⬡ Connecting — right-click node B (Esc to cancel)" in connect mode
- No connect button — replaced with `.tb-hint` element

**Node types:** `char`, `event`, `residue`, `conflict`, `relationship`, `intervention`, `mediator`

**Connection types:**
- Character → Character: social psychology, relationship generation
- Character → Residue: character's orientation to social field *(parked)*
- Character → Event: direct physical involvement
- Character → Conflict: same with moral/ethical dimension
- Residue → Residue: sociology, social environment across dynamics
- Residue → Event: what social structures do to situations
- Residue → Conflict: same with ethical dimension

**Core mechanics:**
- Threshold framing for residue: *"Write it as a threshold — something has happened that makes what comes next genuinely different"*
- NARRATIVE NECESSITY: every interaction must leave the gathering in a different state
- Object-passing prohibition in events
- Two-connection event lifecycle

**Event generation — vector-driven:**
- Event 1: both situations, commits to most live one. `activeSituation: "A"|"B"|"both"`
- Event 2: both + latent. `activeSituation: "A"|"B"|"both"|"latent"`
- Event 3: all three, brings most prominent threads into contact. `activeSituation` required
- `activeSituation` persists to CFG export

**Conflict node — single connection:**
- One connection triggers the resolution library immediately
- `conflictConnections < 1` gates the trigger (changed from 3)
- No progressive reveals — umbrella surfaces directly shaped by play state

**Pressure arc:** `gatheringArcCtx()` pressure-aware, `pressureEngaged` flag.

**Facet system:** Relationship ledger tracks facet, subFacets, tension, interactions, miscommunications.

**Log panel:** Character list (`#log-chars`) is hidden (`display:none`).

---

## Resolution Library (End Game)

Replaces the old mediator/challenge system entirely.

**Trigger:** Single connection to conflict node → `triggerConclusion()` → `buildResolutionOverlay()` + `generateResolutionApproaches()`

**Generation:** One API call (`max_tokens: 2000`) producing five character-positioned approaches. Each approach:
- `characterName`, `position` — the kind of actor this person is in this situation
- `principleName`, `principleSchool` — real named concept from conflict resolution theory
- `action` — specific move available to someone in this position
- `plus` — one sentence: what this approach can reach
- `minus` — one sentence: what it costs or cannot reach
- `caseStudy` — 3-4 sentences applying the principle specifically to THIS simulation's events

**Diversity requirement:** Each approach must draw from a categorically different domain of conflict resolution theory. No two from the same school.

**UI:** Five expandable cards — collapse shows name, position, principle, school, one-line tradeoffs. Click header to expand case study and specific action.

**Key functions:** `buildResolutionOverlay()`, `generateResolutionApproaches()`, `renderResolutionApproaches(approaches)`

**Old system removed:** `buildChallengeOverlay`, `generateConflictStatement`, `evaluateProposal`, `generateStakes`, `generateDecisionFrame`, mediator textarea, submit button, inquiry section.

---

## Audio System

**SFX pool (3 elements per sound):**

| File | Trigger | Volume |
|------|---------|--------|
| `click.wav` | Every mouse click (global listener) | 0.75 |
| `confirm.wav` | simStart | 0.7 |
| `error.wav` | Generation failure | 0.6 |
| `panel-open.wav` | Node panel opens | 0.5 |
| `panel-close.wav` | Panel close | 0.4 |
| `char-complete.wav` | Character preview renders | 0.8 |
| `setting-complete.wav` | Setting preview renders | 0.38 (fade-in 600ms) |
| `conflict-complete.wav` | Conflict card renders | 0.8 |
| `node-connect.wav` | Residue node appears | 0.8 |

**Setup music layers (MP3, additive, synchronized):**

All five start silently on first click. Each fades in (3s) at trigger. All fade out (2.5s) on sim start.

| File | Trigger |
|------|---------|
| `setup-base.mp3` | First user click |
| `setup-setting.mp3` | Setting generates |
| `setup-characters.mp3` | First character generates |
| `setup-conflict.mp3` | Conflict generates |
| `setup-relationships.mp3` | Relationships generate |

`playWithFadeIn(key, targetVolume, durationMs)` — fade-in helper for setting complete and future use.

---

## AI Calls

All use `model: 'claude-sonnet-4-6'`. All through `throttledFetch()`. All parsed via `safeParseJSON()` (includes newline sanitization).

- `generateSetting()` — refusal handling, retry
- `generateChar(i)` — OCEAN + wound territory + diversity constraints. `max_tokens: 1200`
- `generateConflict()` — `conflictVectors`, 15-domain selection. `max_tokens: 1200`
- `generateAllRelationships()` — `max_tokens: 2000`
- `generateResidue(a, b)` — threshold framing, full context
- `performEventRewrite()` — source-aware, valence-conditional
- `generateSequentialEvent(idx)` — vector-driven, `activeSituation` required
- `generateConflictReveal(n)` — exists in code but no longer called
- `generateResolutionApproaches()` — five-approach resolution library. `max_tokens: 2000`
- `generateStoryBeat()`, `generateTestimony()`, `generateIntervention()`

---

## Key Data Structures

```
conflict object = {
  label, domain, environment,
  gatheringPremise, gatheringRoles: '',
  firstQuestion: '',  // alias for pressure
  pressure: '',
  pressureCurrent: '',
  conflictVectors: { umbrella, situations: ['',''], latent },
  input: '',
  generated, connectionsReceived, wasAuto
}

event object (CFG export) = {
  label, provocation, pressureType, friction,
  activeSituation,  // "A"|"B"|"both"|"latent"|""
  connectionCount, valence, outcome, layers, pending
}
```

CFG export includes: `_pressure`, `_pressureCurrent`, `_conflictInput`, `_conflictVectors`, `_conflictTexture`, `_setting`, `_register`, `_teams`, `_storyBeats`, `_charBeats`, `_establishedFacts`, `_conflictReveals`, `_act`, `_totalResidueCount`

---

## Architectural Decisions — Do Not Change Without Discussion

- **Single file.**
- **Model string:** Always `claude-sonnet-4-6`.
- **Standalone only.**
- **`safeParseJSON` is the only JSON parser.**
- **Register system** threads through all prompts via `registerInstruction()`.
- **Right-click connect** — do not re-add a connect button. The hint element is `#connect-hint`.
- **Single conflict connection** — `conflictConnections < 1`. Do not revert to 3.
- **Resolution library** — replaces mediator system entirely. Do not re-add mediator textarea or submit button.
- **`gatheringRoles` permanently removed.**
- **`pressure` replaces `firstQuestion`.** Alias kept for backward compat.
- **Object-passing prohibition in events.**
- **Threshold framing for residue.**
- **Intervention nodes have type `'intervention'`, not `'residue'`.**
- **Drift always active during simulation.**
- **Character colors fixed:** `['#c8a96e','#6e9ec8','#c86e6e','#8a8a6e','#9e6ec8']` — these are the warm-tone hex equivalents; oklch vars used for new elements.
- **`accountability` permanently removed as domain.**
- **`conflictVectors` is the dramatic spine.** All event generation receives it.
- **Diversity constraints** live in one consolidated block. Do not add separate hardcoded role/type constraints elsewhere.
- **Setup music layers start silently together.** Timing synchronization depends on all playing from the same start point.
- **Audio repo:** `https://raw.githubusercontent.com/jakobbaron/The-Group-Game/main/` — WAV for SFX, MP3 for music layers.

---

## Known Fragile Areas

- **`safeParseJSON` failures** — newline sanitization added. Always use it.
- **Setting generation refusals** — sanitizes and retries automatically.
- **Residue drift collision** — drift engine sensitive to node placement changes.
- **Challenge trigger timing** — now gated by `conflictConnections >= 1` and act state.
- **`throttledFetch` queue** — check `generating` flag before residue generation.
- **Panel open state** — use `removeMapNode(id)` helper.
- **Intervention node type** — must be handled wherever `type === 'residue'` is checked.
- **Section unlock sequencing** — setting → characters → conflict → relationships.
- **OCEAN mismatch injection** — preserve `_mismatchDim` handling.
- **Import context gap** — `storyBeats` and `charBeats` don't restore to live arrays.
- **`conflictVectors` on older setups** — pre-v1.6 setups have `conflictVectors: null`. Handled gracefully.
- **`activeSituation` empty on Event 3** — model occasionally doesn't commit. Prompt constraint needs tightening.
- **Avoiding/low-assertiveness characters** — tend to disappear from CFGs. Lucius Varro problem. Not yet diagnosed.

---

## Current State

### What Is Working
- Full standalone deployment
- Full setup flow with diversity constraints (role, conflictType, moralOrientation)
- Character generation: OCEAN, wound+behavioral append, moral error+behavioral append, diversity enforcement
- Conflict generation: `conflictVectors`, 15-domain selection
- Map simulation: residue, event rewrites, conflict reveal, act advancement
- Vector-driven event generation with `activeSituation` persisting to CFG export
- Right-click connect mechanic with Esc cancel
- Resolution library: five character-positioned approaches, expandable case studies
- Pressure arc system
- Mediator system (testimonies, interventions) — still in code, not the end game
- Setup music layers (additive, synchronized, fade in/out)
- SFX: global click, setting fade-in, all others
- Ritual design system: warm colors, Libre Baskerville, opaque cards, texture

### Known Open Issues
- **Resolution library not yet validated post-fix** — was broken, now fixed, needs live test
- **`activeSituation` empty on Event 3** — needs stronger prompt constraint
- **Avoiding/low-assertiveness character absence** — disappear from play, not diagnosed
- **`orthogonalValue` not injected into generation** — generated but inert
- **Paper problem** — reduced via character changes, not validated in playthrough yet
- **Resolution pathway untested** — all events escalate
- **Relationship beats underdeveloped**
- **Import context gap** — `storyBeats`/`charBeats` don't restore to live arrays
- **`gatheringPremise` timing** — generated at conflict time, planned to move to late generation

---

## Development Principles

- **Read before touching.** Always view relevant code before changes.
- **Syntax check after every change.** Extract script, run `node --check`, copy to outputs only after clean.
- **No overcorrection.** Behavioral instructions set floors, not ceilings.
- **Prompt changes are the primary lever.**
- **Preserve context threading:** `generatedSetting`, `charBeats`, `establishedFacts`, `relationshipLedger`, `nodeTension`.
- **Test with CFG dumps.**
- **Pattern analysis over single-run evaluation.**

---

*Last updated: May 2026*
*Version: v1.8*
*Primary file: `the_group_game.html`*
*Audio: `https://github.com/jakobbaron/The-Group-Game`*
