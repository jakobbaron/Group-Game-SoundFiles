# The Group Game — Project Spec Document
*For use at the start of any development session. Paste this into Claude before asking for changes.*

---

## What This Project Is

**The Group Game** is a single-file HTML character simulation where five characters are placed in a shared gathering and navigate a central conflict through a structured arc. The player does not control characters directly — instead they draw connections between nodes on a map, triggering AI-generated narrative beats. The simulation unfolds through pressure, residue, and escalation across three acts until a climactic challenge scene resolves the conflict.

The project runs as a **standalone HTML file opened directly in the browser**. It calls the Anthropic API (`claude-sonnet-4-6`) directly from the client using an API key entered by the user on load. All logic, styles, and HTML live in one file: `the_group_game.html`. A companion development tool (`group_game_dev.jsx`) exists as a React artifact for testing individual AI calls in isolation.

**Design intent:** The simulation is a character simulation where philosophical dimensions emerge from behavior under pressure, not from reflection. Play should feel like observing real people in a genuinely difficult situation — drama produced by character action and decision, not by information asymmetry or document transfer.

---

## File Structure

There is one primary file. It is organized in this order:

1. `<style>` block — all CSS, including dark/light theme variables, setup screen styles, map screen styles, overlays, animations
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
   - AI generation functions (setting, characters, events, conflict, relationships, residue, rewrites, narrative beats, mediator system)
   - Challenge system (climactic overlay)
   - Act advancement logic
   - Panel/log UI
   - Export/Import functions (`exportSetup`, `importSetup`, `showJSONModal`, `captureCFG`)
   - Event binding (`bindMapEvents`)

---

## Standalone Deployment

The game is deployed as a standalone HTML file opened directly in the browser (not run inside Claude's artifact environment). This is required for:
- External audio loading from GitHub (CSP blocks external media in the artifact sandbox)
- Direct Anthropic API calls without proxy headers

**API key management:**
- On first load, a modal prompts for an Anthropic API key (`sk-ant-...`)
- The key is stored in `localStorage` as `tgg_api_key`
- `throttledFetch` injects `x-api-key`, `anthropic-version`, and `anthropic-dangerous-direct-browser-access` headers on every Anthropic API call
- A ⚿ API Key button appears in both the setup screen and the simulation toolbar to clear and re-enter the key
- The modal validates key format on load — an invalid stored key is cleared automatically

**Weekly budget tracker:**
- Tracks actual token usage from API response `usage` fields
- Converts to USD using Sonnet 4 pricing ($3/M input tokens, $15/M output tokens)
- Running total stored in `localStorage` as `tgg_budget` with a weekly auto-reset timestamp
- Soft warning threshold and optional hard cap (both configurable)
- `$X.XXX this week` display in the toolbar; click to open budget modal
- Manual reset available in the budget modal

---

## Two Screens

### Setup Screen

The player configures the simulation before it begins. Sections unlock sequentially — each section requires the previous to be complete. Readiness bar tracks completion across all sections.

The setup screen has Export Setup / Import Setup buttons (bottom bar) and a ⚿ API Key button for key management without entering simulation.

**Export Setup / Import Setup:**
- Export captures full setup state: characters, conflict, relationships, setting, plus diagnostic fields (`_pressure`, `_conflictInput`, `_conflictTexture`, `_conflictVectors`, `_teams`)
- Import restores all setup state to editable form — each section unlocks, characters show previews, relationships rebuild, conflict card renders. Events are always reset to `[{pending:true},{pending:true},{pending:true}]` on import
- Both accept CFG snapshots (in-simulation exports) or setup exports

**1. Setting** *(unlocks first)*
A generated world object that all downstream generation inherits from. Has: `label`, `world`, `namingCulture`, `characterDomain`, `characterRoles`, `conflictTexture`. The `atmosphere` field has been removed. `world` describes era, power structure, and the physical and social conditions that define daily life — not what people fight over. `conflictTexture` names pressures rather than conflict types. `characterRoles` describes types of people drawn to this situation, not just who works here.

**2. Characters (5)** *(unlocks after setting)*
Each character has: `name`, `brief`, `tendency`, `assertiveness`, `cooperativeness`, `conflictType`, `openingLine`, `wound`, `moralOrientation`, `moralStrength`, `moralError`, `orthogonalValue`. Hidden from player: `wound`, `moralOrientation`, `moralStrength`, `moralError`.

Character generation uses the **OCEAN system** (Big Five personality model):
- OCEAN profile generated probabilistically (low 25% / medium 45% / high 30% per dimension)
- 20% chance of mismatch injection — one dimension flipped to its opposite, flagged as `_mismatchDim`
- Wound territory selected from annotated pool (36 entries, including 4 vocation-free entries) weighted by OCEAN correlation
- Named or Titled naming mode available
- **Per-slot World toggle** — grounds a user-described character in the generated setting's naming culture and world register

**3. Conflict Seed** *(unlocks after all 5 characters)*
Has: `label`, `domain`, `environment`, `gatheringPremise`, `pressure`, `conflictVectors`.

- `pressure` — one plain sentence: physical, visible, time-bound condition bearing on everyone
- `gatheringPremise` — neutral surface occasion, calendar-entry readable
- `input` — the actual dramatic situation, surfaced in Event 1
- `conflictVectors` — the dramatic spine (see below)

**Conflict input UI:** In Describe mode, a single textarea serves both purposes (redundant hint box removed). In Auto-generate mode, an optional hint textarea remains.

**Conflict domain:** 15 full-phrase domains. `accountability` permanently removed. Current list:
- `consequence and reckoning` | `power and its distribution` | `truth and concealment`
- `loyalty and its limits` | `endings and what they reveal` | `value and invisibility`
- `identity and belonging` | `forced choice and genuine dilemma` | `divided self`
- `temporal` | `structural and systemic` | `epistemic` | `sacrificial and tragic`
- `relational and care` | `threshold and precedent`

**`conflictVectors` — the dramatic spine:**
- `umbrella` — overarching dramatic situation. Not a theme. A situation.
- `situations` — exactly two specific actionable situations with named characters, decisions, and stakes
- `latent` — undisclosed, orthogonal fact that connects or reframes both situations when it emerges

Generation: derive from these specific characters in this specific place. Avoid the most probable version. `max_tokens: 1200`.

**4. Relationships (10)** *(unlocks after conflict seed)*
One for each character pair. Max tokens: 2000. Pre-sim relationship dynamics seed the facet system on first interaction.

**5. Events** *(no longer in setup)*
All three events emerge from play. Initialize as `[{pending:true},{pending:true},{pending:true}]`.

**Register (difficulty):** Accessible / Standard / Complex. Threads through all AI prompts via `registerInstruction()`.

---

### Map Screen (The Simulation)

**Node types:** `char`, `event`, `residue`, `conflict`, `relationship`, `intervention`, `mediator`

**Connection types and intended output:**
- **Character → Character** — social psychology; relationship generation
- **Character → Residue** — character's orientation to social field. *[Parked — not yet implemented]*
- **Character → Event** — direct physical involvement; how people respond to situations
- **Character → Conflict** — same, with moral/ethical dimension
- **Residue → Residue** — sociology; social environment across multiple dynamics
- **Residue → Event** — what social structures do to situations
- **Residue → Conflict** — same, with ethical dimension

**Core mechanics:**
- **Connect mode:** Player selects two nodes → `generateResidue()` → threshold framing
- **NARRATIVE NECESSITY:** Every interaction must leave the gathering in a different state
- **Event lifecycle:** Two connections per event — stage 1 rewrites/escalates, stage 2 resolves
- **Object-passing prohibition:** Events must not advance the situation through documents or objects changing hands

**Event generation — vector-driven:**
- **Event 1** — receives both situations. Commits to whichever is most live. `activeSituation: "A"|"B"|"both"`
- **Event 2** — receives both situations + latent. Surfaces second situation or brings latent into contact. `activeSituation: "A"|"B"|"both"|"latent"`
- **Event 3** — all three. Brings two most prominent threads into direct irreversible contact. `activeSituation` required.

`activeSituation` is stored in the event object and persists to CFG export.

**Pressure arc system:** `gatheringArcCtx()` is pressure-aware. `pressureEngaged` flips true when character first connects to event node.

**Facet system:** Relationship ledger tracks `facet`, `subFacets`, `tension`, `interactions`, `miscommunications`.

**Established facts:** Running list (`_establishedFacts`). Sources: `interaction`, `event-rewrite`, `conflict-reveal`.

---

## Audio System

Sound files hosted at `https://raw.githubusercontent.com/jakobbaron/The-Group-Game/main/`.

**SFX — HTML5 Audio pool (3 elements per sound):**

| File | Trigger |
|------|---------|
| `click.wav` | Generic button press |
| `confirm.wav` | Also used as simStart |
| `error.wav` | Generation failure |
| `panel-open.wav` | Node panel opens |
| `panel-close.wav` | Panel close |
| `char-complete.wav` | Character preview renders |
| `setting-complete.wav` | Setting preview renders |
| `conflict-complete.wav` | Conflict card renders |
| `node-connect.wav` | Residue node appears |

Pending (null): `residue-complete`, `event-unlock`, `event-rewrite`, `conflict-reveal`, `act-advance`, `challenge-trigger`

**Setup music layers — additive vertical layering:**

All five layers start playing silently (volume 0) on first user click. Each fades in (3s) when its trigger fires. All fade out (2.5s) when the start button is clicked.

| File | Trigger |
|------|---------|
| `setup-base.mp3` | First user click (if on setup screen) |
| `setup-setting.mp3` | Setting generates |
| `setup-characters.mp3` | First character generates |
| `setup-conflict.mp3` | Conflict generates |
| `setup-relationships.mp3` | Relationships generate |

Key functions: `_initMusicLayers()`, `_startAllLayersSilent()`, `_fadeMusic(audio, targetVol, durationMs)`, `startSetupLayer(key, targetVol)`, `fadeOutAllSetupMusic(durationMs)`.

All layers loop. Synchronized by starting together at volume 0 — no timing drift.

---

## AI Calls

All AI calls use `model: 'claude-sonnet-4-6'`. All calls go through `throttledFetch()`. All responses parsed via `safeParseJSON()` (includes newline sanitization). Token usage recorded by `recordUsage()`.

**Setup generation:**
- `generateSetting()` — max tokens default. Handles refusals.
- `generateChar(i)` — injects OCEAN profile + wound territory.
- `generateConflict()` — max tokens: 1200. Outputs `conflictVectors`.
- `generateAllRelationships()` — max tokens: 2000.

**Simulation generation:**
- `generateResidue(a, b)` — threshold framing, full context
- `performEventRewrite(eventNode, otherNode)` — source-aware, valence-conditional
- `generateSequentialEvent(idx)` — vector-driven, `activeSituation` field required
- `generateConflictReveal(n)` — three progressive reveals
- `generateStoryBeat()` — under 25 words
- `generateTestimony(charNd)`, `generateIntervention(type, targetNd, playerText)`
- `openChallengeOverlay()` — final challenge scene

---

## Key Data Structures

```
conflict object = {
  label, domain, environment,
  gatheringPremise, gatheringRoles: '',
  firstQuestion: '',  // alias for pressure
  pressure: '',
  pressureCurrent: '',
  conflictVectors: {
    umbrella: '',
    situations: ['', ''],
    latent: ''
  },
  input: '',
  generated, connectionsReceived, wasAuto
}

event object (in CFG export) = {
  label, provocation, pressureType, friction,
  activeSituation,   // "A"|"B"|"both"|"latent" — which vector this event surfaces
  connectionCount, valence, outcome, layers, pending
}
```

CFG export includes: `_pressure`, `_pressureCurrent`, `_conflictInput`, `_conflictVectors`, `_conflictTexture`, `_setting`, `_register`, `_teams`, `_storyBeats`, `_charBeats`, `_establishedFacts`, `_conflictReveals`, `_act`, `_totalResidueCount`.

---

## Architectural Decisions — Do Not Change Without Discussion

- **Single file.** Everything stays in one `.html` file.
- **Model string:** Always `claude-sonnet-4-6`.
- **Standalone only.** Do not run inside Claude's artifact environment.
- **`localStorage`:** API key (`tgg_api_key`), budget (`tgg_budget`).
- **`safeParseJSON` is the only JSON parser.**
- **Register system** threads through all AI prompts via `registerInstruction()`.
- **Sequential event generation.** Vector-driven. `activeSituation` required.
- **Two-connection event lifecycle.**
- **`generatedSetting` is the root context object.**
- **OCEAN system is the character generation foundation.**
- **`gatheringRoles` permanently removed.**
- **`pressure` replaces `firstQuestion`.** Alias kept for backward compat.
- **Conflict generates before characters have dramatic functions.**
- **Object-passing prohibition in events.**
- **Threshold framing for residue.**
- **Intervention nodes have type `'intervention'`, not `'residue'`.**
- **Drift always active during simulation.**
- **Character colors fixed:** `['#c8a96e','#6e9ec8','#c86e6e','#8a8a6e','#9e6ec8']`
- **`accountability` permanently removed as domain.**
- **`conflictVectors` is the dramatic spine.** All event generation receives it.
- **Setup music layers start silently together.** Do not start layers at different times — timing synchronization depends on all playing from the same start point.
- **Audio files at:** `https://raw.githubusercontent.com/jakobbaron/The-Group-Game/main/` — WAV for SFX, MP3 for music layers.

---

## Known Fragile Areas

- **`safeParseJSON` failures** — newline sanitization added. Always use `safeParseJSON`.
- **Setting generation refusals** — sanitizes and retries automatically.
- **Residue drift collision** — drift engine sensitive to node placement changes.
- **Challenge trigger timing** — gated by `conflictConnections === 3` and act state.
- **`throttledFetch` queue** — check `generating` flag before residue generation.
- **Panel open state** — use `removeMapNode(id)` helper.
- **Intervention node type handling** — must be handled wherever `type === 'residue'` is checked.
- **Section unlock sequencing** — setting → characters → conflict → relationships.
- **OCEAN profile mismatch injection** — preserve `_mismatchDim` handling.
- **Import context gap** — `storyBeats` and `charBeats` don't restore to live arrays.
- **`conflictVectors` on older setups** — pre-v1.6 setups have `conflictVectors: null`. Handled gracefully.
- **Domain selection tendency** — 15-domain list needs more observation across varied inputs.
- **Music layer sync** — all layers must start together at volume 0. If `_startAllLayersSilent()` fails silently, layers will be out of sync when faded in.

---

## Current State

### What Is Working
- Full standalone deployment with API key modal and weekly budget tracker
- Full setup flow (setting → characters → conflict → relationships)
- Character generation with OCEAN system, wound territory weighting, per-slot World toggle
- Setting generation with refusal handling
- Conflict generation: `conflictVectors`, 15-domain selection, describe mode UI fix
- Map simulation: residue generation, event rewrites, conflict reveals, act advancement, challenge scene
- Vector-driven event generation with `activeSituation` field persisting to CFG export
- Pressure arc system
- Mediator system (testimonies, interventions)
- Pre-sim relationship facet seeding
- Register system across all AI calls
- Export/Import (pre-sim and in-sim), includes `_conflictVectors`
- Audio: SFX system, setup music layers (additive, synchronized, fade in/out)

### Known Open Issues
- **Conflict reveals not updated to synthesize vectors** — reveals still use old three-stage progressive reveal without awareness of `conflictVectors`. Planned: reveals should synthesize umbrella from what has surfaced across events.
- **Character → Residue rework parked** — produces small non-connectable thought node orbiting character. Not yet implemented.
- **Domain selection needs observation** — needs more varied test runs.
- **Resolution pathway untested** — all events in dataset escalate.
- **Paper problem reduced but present.**
- **Relationship beats underdeveloped.**
- **Import context gap** — `storyBeats` and `charBeats` don't restore to live arrays.
- **`gatheringPremise` timing** — generated at conflict time rather than emerging from play.
- **Connect button intermittent failure** — occasional, not root-cause diagnosed. Low priority.

---

## Development Principles

- **Read before touching.** Always view the relevant code section before making changes.
- **Syntax check after every change.** Extract script block, run `node --check`, copy to outputs only after clean.
- **No overcorrection.** Behavioral instructions should set floors, not ceilings.
- **Prompt changes are the primary lever.**
- **Preserve context threading.** `generatedSetting`, `charBeats`, `establishedFacts`, `relationshipLedger`, `nodeTension`.
- **Test with CFG dumps.**
- **Pattern analysis over single-run evaluation.**

---

*Last updated: May 2026*
*Version: v1.7*
*Primary file: `the_group_game.html`*
*Dev tool: `group_game_dev.jsx`*
*Audio: `https://github.com/jakobbaron/The-Group-Game`*
