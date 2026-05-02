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
   - Audio system
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
A generated world object that all downstream generation inherits from. Has: `label`, `world`, `namingCulture`, `characterDomain`, `characterRoles`, `conflictTexture`. The `atmosphere` field has been removed. `world` describes era, power structure, and the physical and social conditions that define daily life — not what people fight over. `conflictTexture` names pressures (scarcity, surveillance, debt, secrecy, speed, displacement) rather than conflict types. `characterRoles` describes types of people drawn to this situation, not just who works here.

The setting prompt explicitly prohibits: pre-loading dramatic tension, naming what people fight over, defaulting to generic civic spaces, placing information-handlers in document-heavy environments.

**2. Characters (5)** *(unlocks after setting)*
Each character has: `name`, `brief`, `tendency`, `assertiveness`, `cooperativeness`, `conflictType`, `openingLine`, `wound`, `moralOrientation`, `moralStrength`, `moralError`, `orthogonalValue`. Hidden from player: `wound`, `moralOrientation`, `moralStrength`, `moralError`.

The `brief` field: second sentence must feel like it could only be true of someone with these specific OCEAN dimensions — not someone in this job, someone with this exact profile.

Character generation uses the **OCEAN system** (Big Five personality model):
- OCEAN profile generated probabilistically (low 25% / medium 45% / high 30% per dimension)
- 20% chance of mismatch injection — one dimension flipped to its opposite, flagged as `_mismatchDim`
- Wound territory selected from annotated pool (36 entries, including 4 vocation-free entries) weighted by OCEAN correlation (70% weighted, 30% random)
- Profile and territory injected into generation prompt
- Model derives tendency, assertiveness, cooperativeness, conflictType from the profile
- Named or Titled naming mode available
- **Per-slot World toggle** — when active, grounds a user-described character in the generated setting's naming culture and world register

Character generation is framed as a character simulation, not a philosophical simulation. The `wound` and `orthogonalValue` fields use grounded present-tense endings rather than phenomenological abstraction.

**3. Conflict Seed** *(unlocks after all 5 characters)*
Has: `label`, `domain`, `environment`, `gatheringPremise`, `pressure`, `conflictVectors`.

Key architecture:
- `gatheringRoles` has been **removed**. Character stakes emerge from who they are (wound, tendency, professional position, relationships) — not from a pre-assigned dramatic role.
- `pressure` is one plain sentence: the physical, visible, time-bound condition bearing on everyone in the gathering right now.
- `gatheringPremise` describes the surface occasion — neutral, calendar-entry readable, not naming the conflict itself.
- The conflict `input` field (user's conflict hint) is the actual dramatic situation — surfaced in Event 1 during play, not announced upfront.
- `conflictVectors` is the dramatic spine of the conflict (see below).

**Conflict input UI:** Two paths — Describe or Auto-generate. In Describe mode, a single textarea serves both the setting description and the conflict hint (the redundant second box has been removed). In Auto-generate mode, an optional hint textarea remains.

**Conflict domain:** 15 full-phrase domains replace the original 6. `accountability` has been removed entirely — it defaulted too often and pulled generation toward a legal/evidentiary register. Current domain list:
- `consequence and reckoning`
- `power and its distribution`
- `truth and concealment`
- `loyalty and its limits`
- `endings and what they reveal`
- `value and invisibility`
- `identity and belonging`
- `forced choice and genuine dilemma`
- `divided self`
- `temporal`
- `structural and systemic`
- `epistemic`
- `sacrificial and tragic`
- `relational and care`
- `threshold and precedent`

**`conflictVectors` — the dramatic spine:**
Generated by `generateConflict()`. Three parts:
- `umbrella` — one sentence naming the overarching dramatic situation. Not a theme. A situation.
- `situations` — exactly two specific situations, each naming a character (or implied character), a decision or action they face or have made, and what is at stake for them specifically.
- `latent` — one sentence naming an undisclosed, orthogonal situation/condition/fact that is not the same material as either situation, but connects or reframes both when it emerges.

Generation instruction: derive vectors from these specific characters in this specific place — not from the domain alone. Avoid the most probable version of the conflict. The latent should feel inevitable in retrospect but not obvious from the surface.

**4. Relationships (10)** *(unlocks after conflict seed)*
One for each character pair. Has: `type`, `powerBalance`, `dynamic`. The dynamic names one structural fact about how these two people relate and one structural thing that is genuinely open. Max tokens: 2000. Pre-sim relationship dynamics seed the facet system on first interaction.

**5. Events** *(no longer in setup)*
All three events emerge from play. Events initialize as `[{pending:true},{pending:true},{pending:true}]` at simulation start.

**Register (difficulty):** Accessible / Standard / Complex. Threads through all AI prompts via `registerInstruction()`.

---

### Map Screen (The Simulation)

The map shows nodes on a canvas-like stage with three concentric rings. Characters sit on the outer ring. Events on the middle ring. Conflict at the center.

**Node types:**
- `char` — one per character (5 total), color-coded, on outer ring
- `event` — three event nodes on middle ring, generated during play
- `residue` — generated when two nodes connect; drifts autonomously; the primary narrative artifact
- `conflict` — central conflict node, revealed in three progressive stages
- `relationship` — pre-sim relationship node spawned between character pairs
- `intervention` — player-typed mediator actions (rose-violet, distinct from residue)
- `mediator` — player's presence node (violet, pulsing, placed below character ring)

**Connection types and their intended output:**
- **Character → Character** — social psychology; relationship generation
- **Character → Residue** — character's orientation to the social field (involvement appetite, self-efficacy, judgment of others). Produces a small non-connectable thought node that orbits the character. *[Parked — not yet implemented in v1.6]*
- **Character → Event** — direct physical involvement; how different people respond to situations
- **Character → Conflict** — same as above, with moral/ethical dimension
- **Residue → Residue** — sociology; what the social environment looks like across multiple dynamics. Two paths: shared parent (pattern recognition about one character) or different parents (rhyme, irony, or collision between distinct dynamics)
- **Residue → Event** — what social structures do to situations
- **Residue → Conflict** — same, with ethical dimension

**Core mechanics:**

- **Connect mode:** Player selects two nodes. A connection triggers `generateResidue()`. Output: `label`, `quote`, `observation`, `pressureTag`, `deltaA`, `deltaB`, `tensionDelta`, `logEntry`, `concept`, `miscommunication`, `stakes`, `collaborative`, `facet`, `subFacet`, `clarifies`, `mentionedParty`.
- **Residue user message (threshold framing):** *"What residue does this interaction leave? Write it as a threshold — something has happened between these two people that makes what comes next genuinely different depending on what anyone does."*
- **NARRATIVE NECESSITY:** Every interaction must leave the gathering in a different state. The change is already happening.
- **Residue nodes:** Each connection spawns a residue node that drifts across the stage. Connecting a residue to an event triggers `performEventRewrite()`. Connecting to the conflict node triggers a conflict reveal.
- **Event lifecycle:** Two connections per event — stage 1 rewrites/escalates, stage 2 resolves. Events track `valence` and `connectionCount`.
- **Object-passing prohibition:** Events must not advance the situation through documents, records, or objects changing hands. Drama comes from what a character says directly, decides openly, or does in front of others.
- **Intervention residues:** Have type `intervention`, not `residue`. Can connect to events and conflict node.

**Event generation (in-simulation) — vector-driven:**

All three events receive `CFG.conflict.conflictVectors` and are instructed to surface vectors sequentially:

- **Event 1** — receives both situations. Instructed to commit to whichever situation is most live given early play (recent residues, tension levels). Must output `activeSituation: "A"|"B"|"both"`. "Both" valid only when they are in direct collision, not running in parallel.
- **Event 2** — receives both situations + latent. Instructed to surface the second situation, or bring the latent into contact with what Event 1 established. The latent reframes, not adds. Must output `activeSituation: "A"|"B"|"both"|"latent"`.
- **Event 3** — receives all three. Instructed to bring the two most prominent threads into direct, irreversible contact. The umbrella should feel legible from what happens, not named. Must output `activeSituation`.

Event 1 also receives the conflict `input` field directly with the instruction to surface the true situation within the occasion. Events 2 and 3 use fixed ESCALATION and IRREVERSIBLE instructions. Object-passing prohibition applies to all events.

**Pressure arc system (`gatheringArcCtx`):**
The arc context injected into every residue and event generation call includes: current gathering phase, the `pressure` sentence, and `pressureEngaged` state. After a character first connects to an event node, `pressureEngaged` flips to `true`.

**Facet system:** Relationship ledger tracks `facet`, `subFacets`, `tension`, `interactions`, `miscommunications`. Facet is seeded on first interaction from the pre-sim relationship's open structural question.

**Established facts:** Running list (`_establishedFacts`) of what has been confirmed. Sources: `interaction`, `event-rewrite`, `conflict-reveal`.

---

## AI Calls

All AI calls use `model: 'claude-sonnet-4-6'`. All calls go through `throttledFetch()`. All responses parsed via `safeParseJSON()`. Token usage from every response is recorded by `recordUsage()`.

`safeParseJSON` now includes a step that collapses unescaped literal newlines inside JSON string values before parsing — prevents failures when model line-wraps inside string fields.

**Setup generation:**
- `generateSetting()` — generates full setting object. Handles refusals by sanitizing seed and retrying.
- `generateChar(i)` — generates one character. Injects OCEAN profile + wound territory.
- `generateConflict()` — generates conflict seed. `max_tokens: 1200`. Outputs `conflictVectors` in addition to existing fields. Prompt instructs: derive vectors from these specific characters in this specific place; avoid the most probable version; the latent should reframe, not add.
- `generateAllRelationships()` — generates all 10 relationships. Max tokens: 2000.

**Simulation generation:**
- `generateResidue(a, b)` — core connection handler. Threshold framing. Builds rich context from character profiles, relationship ledger, facet plan, established facts, charBeats, tension values, pressure arc.
- `performEventRewrite(eventNode, otherNode)` — rewrites event in place. Source-aware stage 1 instructions (residue vs. character connections). Stage 2 valence-conditional with forward deposit in `friction` field.
- `generateSequentialEvent(idx)` — generates events 1/2/3 during play. Receives full `conflictVectors` object with per-event sequencing instructions. `activeSituation` field required in response.
- `generateConflictReveal(n)` — three progressive reveals.
- `generateStoryBeat()` — periodic plain-language log entry (under 25 words).
- `generateTestimony(charNd)` — character's biased account to mediator.
- `generateIntervention(type, targetNd, playerText)` — mediator intervention residue.
- `openChallengeOverlay()` — final challenge scene.

---

## Key Data Structures

```
simulation = {
  characters: [null × 5],
  events: [{pending:true} × 3],
  conflict: null,
  relationships: {}
}

CFG = deep copy of simulation passed into map screen on start

conflict object = {
  label, domain, environment,
  gatheringPremise,
  gatheringRoles: '',       // always empty — removed from generation
  firstQuestion: '',        // alias for pressure — kept for backward compat
  pressure: '',             // physical, visible, time-bound condition
  pressureCurrent: '',      // live arc state — updated during play
  conflictVectors: {        // dramatic spine — generated by generateConflict()
    umbrella: '',           // overarching dramatic situation
    situations: ['', ''],   // two specific actionable situations
    latent: ''              // undisclosed orthogonal reframing fact
  },
  input: '',                // actual dramatic situation — surfaced in Event 1
  generated, connectionsReceived, wasAuto
}

generatedSetting = {
  label, world, namingCulture,
  characterDomain, characterRoles, conflictTexture,
  // atmosphere field removed
  poolEntry, userInput
}

mapNodes, residueNodes, relationshipLedger, interventionPool,
mediatorTestimonies, nodeTension, charBeats, storyBeats,
establishedFacts, conflictReveals, pressureEngaged
— all as previously documented
```

**CFG export (`captureCFG`)** includes: `_pressure`, `_pressureCurrent`, `_conflictInput`, `_conflictVectors`, `_conflictTexture`, `_setting`, `_register`, `_teams`, `_storyBeats`, `_charBeats`, `_establishedFacts`, `_conflictReveals`, `_act`, `_totalResidueCount`.

---

## Audio System

Sound files hosted at `https://raw.githubusercontent.com/jakobbaron/Group-Game-SoundFiles/main/`.

Uses HTML5 Audio element pools (3 elements per sound). Sounds initialize on first user click.

| File | Trigger |
|------|---------|
| `click.wav` | Generic button press |
| `confirm.wav` | Also used as `simStart` |
| `error.wav` | Generation failure, connection rejection |
| `panel-open.wav` | Node panel opens |
| `panel-close.wav` | Panel close button |
| `char-complete.wav` | Character preview renders |
| `setting-complete.wav.wav` | Setting preview renders |
| `conflict-complete.wav.wav` | Conflict card renders |
| `node-connect.wav.wav` | Residue node appears |
| `residue-complete.wav` | (pending) |
| `event-unlock.wav` | (pending) |
| `event-rewrite.wav` | (pending) |
| `conflict-reveal.wav` | (pending) |
| `act-advance.wav` | (pending) |
| `challenge-trigger.wav` | (pending) |

Note: some files were uploaded with double extensions (`.wav.wav`) — SOUNDS object uses actual filenames as uploaded.

---

## Architectural Decisions — Do Not Change Without Discussion

- **Single file.** Everything stays in one `.html` file.
- **Model string:** Always `claude-sonnet-4-6`.
- **Standalone only.** Do not attempt to run inside Claude's artifact environment.
- **`localStorage` usage:** API key (`tgg_api_key`), budget (`tgg_budget`). Game state is not saved between sessions.
- **`safeParseJSON` is the only JSON parser.**
- **Register system.** Three-tier register threads through all AI prompts via `registerInstruction()`.
- **Sequential event generation.** All three events generate during play. Vector-driven sequencing.
- **Two-connection event lifecycle.** Each event takes exactly two connections to close.
- **`generatedSetting` is the root context object.**
- **OCEAN system is the character generation foundation.**
- **`gatheringRoles` is permanently removed.**
- **`pressure` replaces `firstQuestion`.** `firstQuestion` exists only as backward-compat alias.
- **Conflict generates before characters have dramatic functions.**
- **Object-passing prohibition in events.**
- **Threshold framing for residue.**
- **Intervention nodes have type `'intervention'`, not `'residue'`.**
- **Drift is always active during simulation.**
- **Character colors are fixed:** `['#c8a96e','#6e9ec8','#c86e6e','#8a8a6e','#9e6ec8']`.
- **`accountability` is permanently removed as a domain.** Do not re-add it. Use the 15-domain list.
- **`conflictVectors` is the dramatic spine.** All event generation receives it. Do not bypass it for new event types.

---

## Known Fragile Areas

- **`safeParseJSON` failures** — AI occasionally returns malformed JSON. Newline sanitization added in v1.6. Any new AI call must use `safeParseJSON`.
- **Setting generation refusals** — sensitive seeds trigger content refusals. Sanitizes and retries automatically.
- **Residue drift collision** — drift engine has subtle interaction with node position tracking.
- **Challenge trigger timing** — gated by `conflictConnections === 3` and act state.
- **`throttledFetch` queue** — rapid interactions can stack API calls. Check `generating` flag before initiating residue generation.
- **Panel open state** — use `removeMapNode(id)` helper rather than removing nodes manually.
- **Intervention node type handling** — must be explicitly handled wherever `type === 'residue'` is checked.
- **Section unlock sequencing** — setting → characters → conflict → relationships.
- **OCEAN profile mismatch injection** — preserve `_mismatchDim` handling in any prompt changes.
- **Import storyBeats/charBeats** — not restored into live arrays on import.
- **Audio double extensions** — `setting-complete.wav.wav`, `conflict-complete.wav.wav`, `node-connect.wav.wav`.
- **`conflictVectors` on older setups** — setups generated before v1.6 will have `conflictVectors: null`. Event generation handles this gracefully (vectorCtx returns empty string when cv is null).
- **Domain selection tendency** — expanded domain list needs observation across varied inputs to confirm it ranges beyond accountability-adjacent domains.

---

## Current State

### What Is Working
- Full standalone deployment with API key modal and weekly budget tracker
- Full setup flow (setting → characters → conflict → relationships)
- Character generation with OCEAN system, wound territory weighting, per-slot World toggle
- Setting generation with atmosphere removal, refusal handling
- Conflict generation: `gatheringRoles` removed, `pressure` sentence, `gatheringPremise` as neutral surface occasion, `conflictVectors` (umbrella + situations + latent), 15-domain selection
- Conflict UI: describe mode uses single textarea (redundant hint box removed)
- Map simulation: residue generation, event rewrites, conflict reveals, act advancement, challenge scene
- In-simulation event generation: vector-driven sequencing with `activeSituation` field
- Pressure arc system: `gatheringArcCtx()` pressure-aware, `pressureEngaged` flag
- Mediator system: testimonies, interventions (Ask/Name/Propose)
- Pre-sim relationship facet seeding
- Register system across all AI calls
- Export Setup / Import Setup (pre-sim and in-sim CFG capture), includes `_conflictVectors`
- Audio system: UI sounds, generation completion sounds, error sounds

### Known Open Issues
- **Event vector wiring untested in live playthrough** — implemented but not yet validated with a CFG dump.
- **Conflict reveals not yet updated to synthesize vectors** — reveals still use the old three-stage progressive reveal without awareness of `conflictVectors`. Planned: reveals should synthesize umbrella from what has surfaced across events.
- **Character → Residue rework parked** — produces a small non-connectable thought node orbiting the character; prompt change to surface orientation to social field (involvement appetite, self-efficacy, judgment of others). Not yet implemented.
- **Domain selection needs observation** — expanded 15-domain list has not been tested across varied enough inputs to confirm it ranges away from accountability-adjacent domains reliably.
- **Resolution pathway untested** — every event in the dataset escalates; constructive/resolution pathway has not been meaningfully tested.
- **Paper problem reduced but present** — object-as-prop vs. object-as-vehicle distinction needs further enforcement.
- **Relationship beats underdeveloped** — `_charBeats` for relationships are sparse across all CFGs.
- **Import context gap** — `storyBeats` and `charBeats` do not restore into live arrays on import.
- **`gatheringPremise` timing** — currently generated at conflict generation time rather than emerging from play.

---

## Development Principles

- **Read before touching.** Always view the relevant code section before making changes.
- **Syntax check after every change.** Run `node --check` on the extracted script block. Copy to outputs only after a clean check.
- **No overcorrection.** Behavioral instructions should set floors, not ceilings.
- **Prompt changes are the primary lever.** Before changing JS logic, determine whether a prompt instruction change would solve it.
- **Preserve context threading.** `generatedSetting`, `charBeats`, `establishedFacts`, `relationshipLedger`, and `nodeTension` are the continuity mechanisms.
- **Test with CFG dumps.** After major changes, run a full playthrough and paste the exported CFG for analysis.
- **Pattern analysis over single-run evaluation.** Send multiple CFGs before drawing conclusions about systemic issues.

---

*Last updated: May 2026*
*Version: v1.6*
*Primary file: `the_group_game.html`*
*Dev tool: `group_game_dev.jsx`*
*Sound files: `https://github.com/jakobbaron/Group-Game-SoundFiles`*
