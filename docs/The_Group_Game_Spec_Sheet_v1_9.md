# The Group Game — Project Spec Document
*For use at the start of any development session. Paste this into Claude before asking for changes.*

---

## What This Project Is

**The Group Game** is a single-file HTML character simulation where five characters are placed in a shared gathering and navigate a central conflict through a structured arc. The player right-clicks nodes on a map to connect them, triggering AI-generated narrative beats. The simulation unfolds through pressure, residue, and escalation across three acts until a climactic resolution scene surfaces five character-positioned approaches to the conflict.

The project runs as a **standalone HTML file opened directly in the browser**. It calls the Anthropic API (`claude-sonnet-4-6`) directly from the client. All logic, styles, and HTML live in one file: `the_group_game.html`.

**Design intent:** An experiential educational tool. Players learn what real conflict dynamics look like from the inside, and what genuinely works to address them. Philosophical dimensions emerge from behavior under pressure, not reflection.

**Current file size:** ~544KB, ~7,638 lines.

---

## Design System — "Ritual"

- **Colors:** Warm brown-black (`#0f0c0a`), gold `oklch(72% 0.11 68)`, harmonious blue/teal at consistent chroma/lightness. Cards: `#1e1916` fully opaque with gold top-border.
- **Typography:** Libre Baskerville + Space Mono. Global `letter-spacing:0.012em`, `word-spacing:0.04em`.
- **Light mode:** Warm parchment (`#f5f0e8`).
- **Background:** Warm radial gradient texture on setup screen via `::before`.
- **Section headers:** Hints stack below titles (column layout).

---

## Standalone Deployment

- Opened directly in browser as local HTML file
- API key stored in `localStorage` as `tgg_api_key`
- Budget tracked in `localStorage` as `tgg_budget`
- `throttledFetch` injects required Anthropic headers
- Audio at: `https://raw.githubusercontent.com/jakobbaron/The-Group-Game/main/`

---

## Setup Screen

Sections unlock sequentially. Readiness bar tracks completion.

**1. Setting** — `label`, `world`, `namingCulture`, `characterDomain`, `characterRoles`, `conflictTexture`. No `atmosphere` field.

**2. Characters (5)** — OCEAN system, wound territories, per-slot World toggle.

Consolidated **diversity constraints** (one `diversityConstraint` block per slot):
- Role diversity — max 1 information-handler per ensemble
- Conflict type diversity — max 2 of any `conflictType`
- Moral orientation diversity — max 2 of any orientation

**Wound field** appends: *"End with one plain sentence: how this wound shapes the way they move through difficulty — not what happened, not what they think about it, but what it makes them do differently than someone without it would."*

**Moral error field** appends: *"End with one plain sentence: how this misfire shapes the way they move through difficulty — what it makes them do differently than someone without this error would, specifically under pressure."*

**3. Conflict Seed** — `label`, `domain`, `environment`, `gatheringPremise`, `pressure`, `conflictVectors`. Max tokens: 1200.

15 full-phrase domains (`accountability` permanently removed).

`conflictVectors`: `umbrella` + `situations[2]` + `latent`. Generated from characters and setting, not domain alone. Avoid most probable version.

**Loading states** — all four generators (setting, character, conflict, relationships) cycle through context-appropriate messages during generation. On failure: clear loading, restore input form, show error.

**4. Relationships (10)** — max tokens 2000. Seeds facet system.

**5. Events** — no setup section. All emerge from play.

**Register:** Accessible / Standard / Complex via `registerInstruction()`.

---

## Map Screen

**Connection mechanic — right-click:**
- Right-click node A → enters connect mode with A selected
- Right-click node B → completes connection
- Esc → cancels
- Toolbar shows `.tb-hint#connect-hint` — updates to "⬡ Connecting..." state
- No connect button

**Node types:** `char`, `event`, `residue`, `conflict`, `relationship`, `intervention`, `mediator`

**Core mechanics:**
- Threshold framing for residue
- NARRATIVE NECESSITY: every interaction changes the gathering state
- Object-passing prohibition in events
- Two-connection event lifecycle (`connectionCount`)

**Event generation — vector-driven:**
- Event 1: situations A+B, commits to most live. `activeSituation: "A"|"B"|"both"`
- Event 2: situations + latent available. `activeSituation: "A"|"B"|"both"|"latent"`
- Event 3: all three, brings most prominent into contact. `activeSituation` required
- `activeSituation` persists to CFG export

**Conflict node — single connection:**
- One connection triggers resolution library
- `conflictConnections < 1` gates trigger
- No progressive reveals

**Drift mechanics:**
- `confirm-overlay` — residue approaching conflict node (confirm/hold back)
- `absorb-overlay` — residue approaching event node (absorb/hold back)
- Both elements exist in HTML inside `#map-screen`

**Pressure arc:** `gatheringArcCtx()` pressure-aware, `pressureEngaged` flag.

**Facet system:** Relationship ledger tracks facet, subFacets, tension, interactions, miscommunications.

**Log panel:** Character list (`#log-chars`) hidden (`display:none`).

---

## Resolution Library (End Game)

Replaces mediator system entirely. No mediator textarea, submit button, or inquiry section.

**Trigger:** Single conflict connection → `triggerConclusion()` → `buildResolutionOverlay()` + `generateResolutionApproaches()`

**"What Happened"** — pulls from `conflictReveals` if populated, otherwise from `pressureCurrent`, recent `storyBeats`, and recent `establishedFacts`.

**Generation:** One call, `max_tokens: 3500`, five character-positioned approaches:
- `characterName`, `position` — kind of actor in this situation
- `principleName`, `principleSchool` — real named concept from conflict resolution theory
- `action` — specific move available to this position
- `plus` / `minus` — one sentence each
- `caseStudy` — 3-4 sentences specific to THIS simulation's events

Each approach from a categorically different domain of conflict resolution theory.

**UI:** Five expandable cards. Click header to expand case study. Return to Map button shown on both success and failure.

**Key functions:** `buildResolutionOverlay()`, `generateResolutionApproaches()`, `renderResolutionApproaches(approaches)`

**Removed (permanently):** `buildChallengeOverlay`, `generateConflictStatement`, `evaluateProposal`, `evaluateCharResponse`, `generateStakes`, `generateDecisionFrame`, `loadInquiry`, `generateConflictReveal`, `_generateConflictReveal`, `processRevealQueue`, `buildEventSlots`, `conflictRevealQueue/InFlight`

---

## ORIENT_TEXT (Player-Facing Strings)

Three registers: `accessible`, `medium`, `complex`. Each contains:
- `what` — what the simulation is
- `do` — what the player does (references right-click, resolution library)
- `watch` — what to attend to
- `actI/II/III/IV` — act labels. Act IV: "Round 4 — The Approaches" / "Act IV — Resolution Library"
- `confirmTitle` — "Explore the Resolution Library?"
- `confirmBody` — references resolution library, not mediation
- `abTitle/abBody/abYes/abNo/abHint` — absorb overlay strings (live)

**Removed keys:** `mediatorIntro`, `confirmProceed`, `hintEvents`, `stakesNote`, `submitBtn`, `chPromptLabel`, `chPromptHint`, `inquiryToggle`, `inquiryEyebrow`, `chLabelConflict`, `chLabelStakes`

---

## Audio System

**SFX pool (3 elements per sound, WAV):**

| File | Trigger | Notes |
|------|---------|-------|
| `click.wav` | Every mouse click (global) | 0.75 |
| `confirm.wav` | simStart | 0.7 |
| `error.wav` | Generation failure | 0.6 |
| `panel-open.wav` | Node panel opens | 0.5 |
| `panel-close.wav` | Panel close | 0.4 |
| `char-complete.wav` | Character preview renders | 0.8 |
| `setting-complete.wav` | Setting preview renders | 0.38, fade-in 600ms |
| `conflict-complete.wav` | Conflict card renders | 0.8 |
| `node-connect.wav` | Residue node appears | 0.8 |

`playWithFadeIn(key, targetVolume, durationMs)` — fade-in helper.

**Setup music layers (MP3, additive, synchronized):**

All start silently on first click. Fade in (3s) at trigger. All fade out (2.5s) on sim start.

| File | Trigger |
|------|---------|
| `setup-base.mp3` | First user click |
| `setup-setting.mp3` | Setting generates |
| `setup-characters.mp3` | First character generates |
| `setup-conflict.mp3` | Conflict generates |
| `setup-relationships.mp3` | Relationships generate |

---

## AI Calls

All use `model: 'claude-sonnet-4-6'`. All through `throttledFetch()`. All parsed via `safeParseJSON()` (newline sanitization included).

- `generateSetting()` — refusal handling, retry on failure
- `generateChar(i)` — OCEAN + wound territory + diversity constraints
- `generateConflict()` — `conflictVectors`, 15-domain. Max tokens: 1200. Cycles loading messages.
- `generateAllRelationships()` — max tokens 2000. Cycles loading messages in button text.
- `generateResidue(a, b)` — threshold framing, full context
- `performEventRewrite()` — source-aware, valence-conditional
- `generateSequentialEvent(idx)` — vector-driven, `activeSituation` required. Max tokens: 500.
- `generateResolutionApproaches()` — five-approach library. Max tokens: 3500.
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
- **Register system** via `registerInstruction()`.
- **Right-click connect** — no connect button. Hint element is `#connect-hint`.
- **Single conflict connection** — `conflictConnections < 1`.
- **Resolution library** — replaces mediator entirely.
- **`gatheringRoles` permanently removed.**
- **`pressure` replaces `firstQuestion`.** Alias kept.
- **Object-passing prohibition in events.**
- **Threshold framing for residue.**
- **Intervention nodes have type `'intervention'`, not `'residue'`.**
- **Drift always active.**
- **`accountability` permanently removed as domain.**
- **`conflictVectors` is the dramatic spine.**
- **Diversity constraints** in one consolidated block per char slot.
- **Setup music layers start silently together.**
- **Dead code policy:** Remove dead functions, CSS, and element references immediately. Do not leave commented stubs.
- **Audio repo:** `https://raw.githubusercontent.com/jakobbaron/The-Group-Game/main/`

---

## Known Fragile Areas

- **`safeParseJSON`** — always use it, never raw `JSON.parse`.
- **Setting generation refusals** — sanitizes and retries.
- **Residue drift collision** — sensitive to node placement changes.
- **Challenge trigger** — gated by `conflictConnections >= 1` and act state.
- **`throttledFetch` queue** — check `generating` flag before residue generation.
- **Panel open state** — use `removeMapNode(id)` helper.
- **Intervention node type** — handle wherever `type === 'residue'` is checked.
- **Section unlock sequencing** — setting → characters → conflict → relationships.
- **OCEAN mismatch injection** — preserve `_mismatchDim` handling.
- **Import context gap** — `storyBeats`/`charBeats` don't restore to live arrays.
- **`conflictVectors` on older setups** — pre-v1.6 have `null`. Handled gracefully.
- **`activeSituation` empty on Event 3** — model occasionally doesn't commit. Prompt needs tightening.
- **Avoiding/low-assertiveness characters** — disappear from CFGs. Not yet diagnosed.

---

## Current State

### What Is Working
- Full standalone deployment
- Full setup flow with loading states, failure recovery, diversity constraints
- Character generation: OCEAN, wound+behavioral append, moral error+behavioral append
- Conflict generation: `conflictVectors`, 15-domain, cycling loading messages
- Map simulation: residue, event rewrites, act advancement
- Vector-driven event generation with `activeSituation` persisting to CFG export
- Right-click connect with Esc cancel
- Resolution library: five approaches, expandable, Return to Map on failure
- "What Happened" pulls from live state when conflict reveals empty
- Confirm/absorb overlays restored and functional
- Pressure arc system
- Setup music layers (additive, synchronized)
- All player-facing text updated — no mediator references
- Ritual design system

### Known Open Issues
- **Resolution library token bump (3500) not yet validated** — fix in place, needs live test
- **`activeSituation` empty on Event 3** — needs stronger prompt constraint
- **Paper problem** — partially addressed via character changes, not validated
- **Avoiding/low-assertiveness character absence** — not diagnosed
- **`orthogonalValue` inert** — generated but not injected into residue/event context
- **Resolution pathway untested** — all events escalate in dataset
- **Relationship beats underdeveloped**
- **Import context gap** — `storyBeats`/`charBeats` don't restore to live arrays

---

## Development Principles

- **Read before touching.**
- **Syntax check after every change.** Extract script, `node --check`, copy to outputs only after clean.
- **Remove dead code immediately.** No stubs, no commented-out functions.
- **No overcorrection.** Behavioral instructions set floors, not ceilings.
- **Prompt changes are the primary lever.**
- **Preserve context threading:** `generatedSetting`, `charBeats`, `establishedFacts`, `relationshipLedger`, `nodeTension`.
- **Test with CFG dumps.**
- **Pattern analysis over single-run evaluation.**

---

*Last updated: May 2026*
*Version: v1.9*
*Primary file: `the_group_game.html`*
*Audio: `https://github.com/jakobbaron/The-Group-Game`*
