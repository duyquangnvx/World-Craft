# Agent-Native GameCraft Toolkit — Design

Date: 2026-05-11
Status: Draft for review

## 1. Purpose

Agent-Native GameCraft Toolkit is a development toolkit for generating polished 2D casual games, puzzle games, and minigames through coding agents. It generalizes the World Craft idea beyond visual world construction by introducing an engine-neutral game contract language, genre recipes, validation tools, asset generation workflows, and engine-specific coding-agent skills.

The toolkit is not primarily an app/API at first. The initial interface is a coding agent extension/toolkit that can read and modify real game projects, run validators/tests, generate assets, implement gameplay code, and iterate on failures. A future app/API can wrap the same core toolkit.

Primary goals:

- Turn human GDDs or prompts into production-oriented 2D game contracts.
- Use one shared Source of Truth for Designer, Artist, and Coder agents.
- Support multiple casual game genres without creating a separate contract language per genre.
- Keep code generation LLM-driven, not rigid script-parser driven.
- Preserve asset-code alignment through strict IDs, manifests, schemas, and validation.
- Enable polished visual output: sprites, UI, animations, audio cues, VFX metadata, and game feel notes.

Non-goals for the first version:

- Full no-code consumer app.
- Multiplayer or backend-heavy games.
- 3D production pipeline.
- Fully deterministic compiler from contract to complete game.
- Supporting every engine equally from day one.

Recommended first target engine: Phaser 3 + TypeScript running on the web, because it provides the fastest validation loop for 2D casual games: instant browser preview, simple deployment, Playwright-based smoke tests, easy screenshot capture, and no heavyweight editor/runtime dependency.

Why Phaser TypeScript first:

- Web preview is immediate and shareable.
- Coding agents can run `npm test`, `npm run build`, and browser automation without opening a game editor.
- Playwright can validate launch, input, screenshots, console errors, and basic gameplay flows.
- TypeScript gives strong static feedback for generated code.
- Phaser maps well to 2D casual primitives: scenes, sprites, tweens, input, audio, tile/grid boards, and UI overlays.
- Generated games can be deployed quickly to static hosting.

---

## 2. High-Level Architecture

```text
Human Prompt / GDD
        │
        ▼
┌──────────────────────┐
│ Designer Agent        │
│ + Designer Skill      │
│ GDD → Contract        │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ Contract Validator    │
│ schema/ref/semantic   │
└──────────┬───────────┘
           │
           ▼
 docs/contract/
 Universal Game Contract
 Engine-neutral Source of Truth
           │
           ├───────────────────────────────┐
           │                               │
           ▼                               ▼
┌──────────────────────┐        ┌──────────────────────┐
│ Artist Agent          │        │ Coder Agent           │
│ contract → assets     │        │ contract → code       │
│ PNG/audio/meta        │        │ engine-specific       │
└──────────┬───────────┘        └──────────┬───────────┘
           │                               │
           └───────────────┬───────────────┘
                           ▼
                  Integration + QA Loop
                           │
                           ▼
                   Playable 2D Game
```

Core principle:

> The contract is not generated code. It is the shared, engine-neutral production specification that agents use to reason, generate, implement, validate, and iterate.

---

## 3. Key Concepts

### 3.1 GDD

The GDD is the human-readable design document. It can be loose, narrative, and product-oriented.

Example:

```text
Make a polished ocean-themed match-3 game for mobile portrait mode.
The player swaps shells, pearls, coral, and stars. Levels should feel juicy,
with satisfying match pops and clear goals.
```

The GDD is not the primary Source of Truth for agents. It is input to the Designer Agent.

### 3.2 Universal Game Contract

The contract is the structured Source of Truth for all downstream agents. It lives under:

```text
docs/contract/
```

It must include stable IDs, references, dimensions, rules, UI bindings, asset requirements, input mappings, feedback hooks, and validation-relevant metadata.

### 3.3 Designer Language

Designer Language is the contract DSL. It should be readable YAML/JSON, not a programming language. It describes what the game should contain and how it should behave, while leaving implementation details to engine-specific Coder Agents.

### 3.4 Genre Recipes

A genre recipe is a reusable design pattern or macro library used by the Designer Agent to produce a valid Universal Game Contract.

A recipe is not a separate contract language.

Examples:

```text
match3_basic
memory_card
hidden_object
merge_grid
sliding_puzzle
endless_tapper
```

Each recipe defines expected modules, required rules, common UI, validation checks, polish expectations, and asset requirements.

### 3.5 Coder Agent

The Coder Agent is LLM-driven. It reads the contract and implements engine-specific code using its own skill/tooling. It is not a hardcoded parser/compiler.

For example:

```text
Universal Game Contract
        ↓
Coder Agent + Phaser TypeScript Skill
        ↓
Phaser scenes/systems/components/tests
```

---

## 4. Design Principles

1. **One contract language, many recipes**
   - Do not create one contract schema per genre.
   - Build universal primitives: game, scene, board, entity, component, state, event, rule, input, UI, asset, feedback, progression.

2. **Strict references, flexible implementation**
   - IDs, asset paths, entity refs, UI bindings, and rule names must be strict.
   - Implementation details remain flexible for Coder Agent reasoning.

3. **Agent-readable and human-reviewable**
   - YAML preferred for authoring.
   - JSON Schema or equivalent for validation.
   - Docs generated from schema where possible.

4. **Engine-neutral core, engine-specific adapters**
   - Contract should not mention Phaser classes directly.
   - Engine adapters translate concepts into Phaser/Godot/Unity/etc. patterns.

5. **Production polish as first-class contract data**
   - Visual style, animation intent, audio feedback, VFX, UI states, game feel, and accessibility are part of the contract.

6. **Validation before generation**
   - Designer output must be validated before Artist/Coder agents consume it.

7. **Iterative QA loop**
   - Generated output should be tested and revised by agents using runtime errors, visual QA, and contract consistency checks.

---

## 5. Contract Architecture

The Universal Game Contract has layered sections:

```text
┌────────────────────────────────────────────┐
│  Genre Recipe Metadata                     │
├────────────────────────────────────────────┤
│  Game/Product Metadata                     │
├────────────────────────────────────────────┤
│  Presentation: art, UI, audio, feedback    │
├────────────────────────────────────────────┤
│  Runtime Model: entities/components/state  │
├────────────────────────────────────────────┤
│  Gameplay Semantics: input/events/rules    │
├────────────────────────────────────────────┤
│  Progression: levels/goals/economy         │
├────────────────────────────────────────────┤
│  Asset Requirements + Output Manifests     │
├────────────────────────────────────────────┤
│  QA/Test Requirements                      │
└────────────────────────────────────────────┘
```

Recommended folder layout:

```text
docs/contract/
  game.yaml
  presentation.yaml
  assets.yaml
  entities.yaml
  rules.yaml
  ui.yaml
  levels/
    level_001.yaml
    level_002.yaml
  recipes/
    used_recipe.yaml
  qa.yaml
```

For small games, a single `game.contract.yaml` may be acceptable. For larger games, split files improve readability and reduce agent context pressure.

---

## 6. Universal Contract Sections

### 6.1 Game Metadata

Purpose: identify product, platform, orientation, target engine assumptions, and genre tags.

Example:

```yaml
game:
  id: ocean_match3
  title: Ocean Treasures
  genre_tags: [casual, puzzle, match3]
  target_platforms: [mobile]
  orientation: portrait
  target_engine: phaser_ts_web
  design_intent: >
    A polished ocean-themed match-3 minigame with juicy feedback,
    clear level goals, and simple mobile controls.
```

### 6.2 Presentation

Purpose: define art direction and polish requirements.

```yaml
presentation:
  resolution:
    design_size_px: [1080, 1920]
    safe_area: mobile_portrait
  art_style:
    format: 2d
    style: polished casual
    keywords: [glossy, bright, ocean, tactile]
    palette: ocean_bright
  animation_style:
    feel: juicy
    easing: snappy_soft
    reduced_motion: required
  audio_style:
    mood: cheerful_ocean
    sfx_density: medium
```

### 6.3 Assets

Purpose: define every required generated or sourced asset. This is the Artist Agent's primary input and the Coder Agent's asset dependency manifest.

```yaml
assets:
  - id: gem_shell
    type: sprite
    role: board_piece
    size_px: [96, 96]
    states: [idle, selected, matched]
    output_path: assets/sprites/gem_shell.png
    style_prompt: glossy pink seashell match-3 tile, transparent background

  - id: sfx_match_pop
    type: audio_sfx
    role: feedback
    duration_ms: 400
    output_path: assets/audio/sfx_match_pop.wav
```

Required asset fields:

- `id`
- `type`
- `role`
- `output_path`
- visual/audio size or duration where relevant
- style prompt or reference

### 6.4 Boards / Spaces

Purpose: describe spatial structures such as grids, boards, paths, or scenes.

```yaml
boards:
  - id: main_board
    type: grid
    size: [8, 8]
    cell_size_px: [96, 96]
    origin_anchor: center
    fill:
      piece_pool: [tile_shell, tile_pearl, tile_star, tile_coral]
      avoid_initial_matches: true
```

This primitive supports many genres:

- Match-3 board
- Memory card grid
- Merge grid
- Sliding puzzle grid
- Tile placement puzzle

### 6.5 Entities and Components

Purpose: define gameplay objects using an ECS-like model.

```yaml
entities:
  - id: tile_shell
    prefab: true
    components:
      - type: board_piece
        board_id: main_board
      - type: matchable
        group: shell
      - type: swappable
        mode: adjacent
      - type: renderable
        asset_ref: gem_shell
      - type: feedback_receiver
        feedback_refs: [match_pop, invalid_swap_shake]
```

Common components:

```text
renderable
clickable
draggable
board_piece
matchable
swappable
collectible
score_source
timer
health
spawner
obstacle
flippable
mergeable
movable
feedback_receiver
```

### 6.6 Input

Purpose: map player intent to game events.

```yaml
input:
  - id: swap_adjacent_tiles
    type: drag_or_tap_swap
    target_components: [board_piece, swappable]
    emits: player_swap_attempt
    constraints:
      - adjacent_only
```

### 6.7 Events and Rules

Purpose: express game logic semantically, without engine-specific code.

Rules should be declarative enough for validation, but not so rigid that they become a full programming language.

```yaml
rules:
  - id: valid_swap
    on: player_swap_attempt
    if:
      type: would_create_match
      min_count: 3
    do:
      - type: swap_pieces
      - type: emit
        event: after_swap
    else:
      - type: reject_swap
      - type: play_feedback
        feedback_id: invalid_swap_shake

  - id: resolve_matches
    on: after_swap
    do:
      - type: find_line_matches
        min_count: 3
        directions: [horizontal, vertical]
      - type: remove_matched
      - type: add_score
        per_piece: 10
      - type: apply_gravity
      - type: refill_board
      - type: emit
        event: board_settled
```

Coder Agent translates these into engine code. Validator checks references, missing events, impossible rules, and common genre requirements.

### 6.8 Goals and Progression

Purpose: describe win/loss conditions and level flow.

```yaml
goals:
  - id: collect_shells
    type: collect_pieces
    target_group: shell
    count: 20
    within_moves: 25

progression:
  level_sequence:
    - level_001
    - level_002
  unlock_policy: sequential
```

### 6.9 UI

Purpose: align UI implementation, asset generation, and gameplay state.

```yaml
ui:
  screens:
    - id: gameplay_hud
      layout: mobile_portrait_hud
      elements:
        - id: moves_counter
          type: text_counter
          binds_to: state.remaining_moves
        - id: score_counter
          type: text_counter
          binds_to: state.score
        - id: goal_tracker
          type: goal_tracker
          binds_to: goals.collect_shells
```

### 6.10 Feedback, Animation, VFX, Audio

Purpose: make polish explicit.

```yaml
feedback:
  - id: invalid_swap_shake
    type: animation
    target: selected_pieces
    motion: shake
    duration_ms: 180
    audio_ref: sfx_invalid

  - id: match_pop
    type: vfx_audio_combo
    target: matched_pieces
    vfx: pop_particles
    animation: scale_pop_fade
    audio_ref: sfx_match_pop
```

### 6.11 QA Requirements

Purpose: give QA agents and validators concrete success criteria.

```yaml
qa:
  required_checks:
    - contract_schema_valid
    - no_missing_asset_refs
    - no_missing_ui_bindings
    - level_goal_reachable
    - game_can_start
    - win_condition_triggerable
    - reduced_motion_supported
  playtest_scenarios:
    - id: complete_level_001
      expected_result: win_screen_shown
```

---

## 7. Genre Recipe System

A recipe helps the Designer Agent generate a complete contract for a genre. It is a template/checklist, not a separate contract schema.

Recipe file example:

```yaml
recipe:
  id: match3_basic
  genre_tags: [match3, casual, puzzle]
  required_contract_sections:
    - boards
    - entities
    - input
    - rules
    - goals
    - ui
    - feedback
  required_primitives:
    - grid_board
    - board_piece
    - matchable
    - swappable
    - gravity_fill
    - score_or_collection_goal
  required_rules:
    - valid_swap
    - resolve_matches
    - refill_board
    - check_goal
  recommended_feedback:
    - selection_highlight
    - invalid_swap_shake
    - match_pop
    - cascade_bonus
```

Candidate initial recipes:

1. `match3_basic`
2. `memory_card`
3. `hidden_object`
4. `merge_grid`
5. `sliding_puzzle`
6. `tap_reaction_minigame`

Recipe expansion flow:

```text
GDD / prompt
   ↓
Designer Agent selects recipe(s)
   ↓
Recipe checklist guides contract generation
   ↓
Universal Game Contract emitted
   ↓
Contract Validator validates schema + recipe-specific semantics
```

---

## 8. Agent Roles

### 8.1 Designer Agent

Inputs:

- Human prompt or GDD
- Available genre recipes
- Target engine capabilities
- Contract schema

Outputs:

- `docs/contract/*.yaml`
- Optional human summary of design decisions

Responsibilities:

- Choose or combine genre recipes.
- Convert human intent into Universal Game Contract.
- Use stable naming conventions.
- Ensure every gameplay concept has assets/UI/rules where needed.
- Avoid unsupported features based on capability registry.

Designer Agent skill should include:

- Contract schema guide
- Genre recipe guide
- Naming convention
- Required field checklist
- Examples
- Anti-patterns

### 8.2 Contract Validator

Inputs:

- `docs/contract/*.yaml`
- Schemas
- Recipe constraints
- Capability registry

Outputs:

- Validation report
- Machine-readable error list

Validation levels:

1. Schema validation
2. Reference validation
3. Semantic validation
4. Recipe-specific validation
5. Production readiness validation

Example errors:

```text
- asset_ref gem_shell_selected is used but not defined.
- goal collect_shells targets group shell, but no piece in main_board has group shell.
- match3_basic requires refill_board after remove_matched.
- UI element moves_counter binds to state.remaining_moves, but state is not declared.
```

### 8.3 Artist Agent

Inputs:

- Asset section of contract
- Presentation style
- Feedback/VFX requirements
- Asset manifest

Outputs:

```text
assets/sprites/*.png
assets/ui/*.png
assets/audio/*.wav
assets/vfx/*.png or metadata
assets/asset_manifest.generated.yaml
```

Responsibilities:

- Generate or source assets with exact IDs and output paths.
- Preserve dimensions and transparent backgrounds.
- Generate states required by contract.
- Write metadata for slicing, pivots, animation frames, and import settings.
- Run visual QA checks.

### 8.4 Coder Agent

Inputs:

- Full contract
- Generated asset manifest
- Engine-specific skill
- Existing engine project

Outputs:

- Engine project files
- Scenes/scripts/resources
- Tests or smoke-test scripts

Responsibilities:

- Implement gameplay based on rules and components.
- Wire generated assets by ID/path.
- Implement UI bindings.
- Implement feedback hooks.
- Run project/tests and fix errors.

The Coder Agent remains LLM-driven. It should reason from the contract and engine conventions, not merely transform YAML through hardcoded templates.

### 8.5 QA Agents / Tools

Recommended QA layers:

- Contract QA
- Asset QA
- Integration QA
- Runtime QA
- Visual polish QA
- Gameplay solvability QA

---

## 9. Tooling Surface

Initial toolkit should expose both agent skills and CLI commands.

Suggested CLI:

```text
gamecraft init
gamecraft design --gdd docs/GDD.md --recipe match3_basic
gamecraft validate-contract
gamecraft generate-assets
gamecraft code --engine phaser-ts
gamecraft test
gamecraft qa
gamecraft package
```

The CLI should be useful to agents and humans. Coding agents can call it during workflows.

Suggested package layout:

```text
gamecraft/
  schemas/
  recipes/
  validators/
  cli/
  skills/
    designer/
    artist/
    coder-phaser-ts/
    qa/
  examples/
    match3_ocean/
    memory_cards_animals/
```

---

## 10. Capability Registry

Purpose: prevent Designer Agent from emitting contracts that target tooling cannot implement.

Example:

```yaml
engine_capabilities:
  target: phaser_ts_web
  supported:
    - grid_board
    - drag_or_tap_swap
    - 2d_sprites
    - ui_bindings
    - tween_animation
    - audio_sfx
    - save_local_progress
    - browser_preview
    - screenshot_testing
    - console_error_capture
    - static_web_export
  unsupported:
    - online_multiplayer
    - skeletal_2d_animation
    - procedural_3d
```

Designer Agent uses this registry before finalizing contract. Validator enforces it.

---

## 11. Production Readiness Requirements

For casual/puzzle/minigames, a production-oriented contract should cover:

- Main gameplay loop
- Win/loss condition
- Level start/end flow
- UI screens
- Input model
- Core state variables
- Asset list
- Animation/feedback hooks
- Audio cues
- Accessibility notes
- Save/progression assumptions
- QA scenarios

Minimum Definition of Done for generated games:

```text
- Game launches without runtime errors.
- Main level is playable.
- Player can win and lose or fail.
- Required assets exist and are wired.
- UI updates from game state.
- Core interactions have feedback.
- Contract validator passes.
- Smoke test passes.
```

---

## 12. Example: Match-3 Contract Slice

```yaml
game:
  id: ocean_match3
  title: Ocean Treasures
  genre_tags: [casual, puzzle, match3]
  orientation: portrait

recipe:
  id: match3_basic

presentation:
  resolution:
    design_size_px: [1080, 1920]
  art_style:
    style: polished casual
    keywords: [glossy, ocean, bright, tactile]

assets:
  - id: gem_shell
    type: sprite
    role: board_piece
    size_px: [96, 96]
    states: [idle, selected, matched]
    output_path: assets/sprites/gem_shell.png
  - id: gem_pearl
    type: sprite
    role: board_piece
    size_px: [96, 96]
    states: [idle, selected, matched]
    output_path: assets/sprites/gem_pearl.png

boards:
  - id: main_board
    type: grid
    size: [8, 8]
    cell_size_px: [96, 96]
    fill:
      piece_pool: [tile_shell, tile_pearl]
      avoid_initial_matches: true

entities:
  - id: tile_shell
    prefab: true
    components:
      - type: board_piece
        board_id: main_board
      - type: matchable
        group: shell
      - type: swappable
        mode: adjacent
      - type: renderable
        asset_ref: gem_shell

  - id: tile_pearl
    prefab: true
    components:
      - type: board_piece
        board_id: main_board
      - type: matchable
        group: pearl
      - type: swappable
        mode: adjacent
      - type: renderable
        asset_ref: gem_pearl

input:
  - id: swap_adjacent_tiles
    type: drag_or_tap_swap
    target_components: [board_piece, swappable]
    emits: player_swap_attempt
    constraints: [adjacent_only]

rules:
  - id: valid_swap
    on: player_swap_attempt
    if:
      type: would_create_match
      min_count: 3
    do:
      - type: swap_pieces
      - type: emit
        event: after_swap
    else:
      - type: reject_swap
      - type: play_feedback
        feedback_id: invalid_swap_shake

  - id: resolve_matches
    on: after_swap
    do:
      - type: find_line_matches
        min_count: 3
        directions: [horizontal, vertical]
      - type: remove_matched
      - type: add_score
        per_piece: 10
      - type: apply_gravity
      - type: refill_board
      - type: emit
        event: board_settled

goals:
  - id: collect_shells
    type: collect_pieces
    target_group: shell
    count: 20
    within_moves: 25

ui:
  screens:
    - id: gameplay_hud
      elements:
        - id: moves_counter
          type: text_counter
          binds_to: state.remaining_moves
        - id: goal_tracker
          type: goal_tracker
          binds_to: goals.collect_shells

feedback:
  - id: invalid_swap_shake
    type: animation
    target: selected_pieces
    motion: shake
    duration_ms: 180

qa:
  required_checks:
    - contract_schema_valid
    - no_missing_asset_refs
    - level_goal_reachable
    - game_can_start
    - win_condition_triggerable
```

---

## 13. Relationship to World Craft

World Craft currently centers on:

```text
prompt → scene plan JSON → assets → Godot scene
```

Agent-Native GameCraft generalizes this into:

```text
GDD/prompt → universal game contract → assets + engine code + QA loop
```

Major differences:

- World Craft contract is scene/world/layout oriented.
- GameCraft contract includes gameplay loops, rules, UI, goals, feedback, progression, QA.
- World Craft uses an app/API-like pipeline.
- GameCraft starts as an agent-native toolkit and can later be wrapped by app/API.
- World Craft builds visualization.
- GameCraft targets playable, polished 2D casual games.

---

## 14. Recommended Implementation Phases

### Phase 1: Contract Foundation

Deliverables:

- Universal Game Contract draft schema
- YAML examples for match3 and memory card
- Basic validator: schema + references
- Designer Agent skill draft

Success criteria:

- Designer Agent can produce a valid contract from a short GDD.
- Validator catches missing IDs, missing asset refs, and invalid sections.

### Phase 2: Phaser TypeScript Web Vertical Slice

Deliverables:

- Phaser TypeScript Coder Agent skill
- Vite-powered web project template
- Match-3 example implementation
- Asset manifest format
- Browser smoke test command
- Playwright screenshot and interaction tests

Success criteria:

- Contract-driven match-3 project launches in a browser with `npm run dev` or `npm run preview`.
- Board, swap, match resolution, goal, UI, and feedback work.
- Playwright can load the game, perform a basic interaction, capture a screenshot, and verify no console/runtime errors.

### Phase 3: Artist Pipeline

Deliverables:

- Asset generation skill
- Sprite spec validation
- Import metadata
- Visual QA checks

Success criteria:

- Generated assets match contract dimensions and output paths.
- Coder Agent can wire assets without manual path fixing.

### Phase 4: More Recipes

Deliverables:

- Memory card recipe
- Hidden object recipe
- Merge grid recipe
- Recipe-specific validators

Success criteria:

- Same Universal Contract supports multiple game types.

### Phase 5: CLI and App/API Wrapper

Deliverables:

- Stable CLI commands
- Optional local web UI
- Optional hosted API wrapper

Success criteria:

- App/API calls the same core toolkit rather than duplicating logic.

---

## 15. Risks and Mitigations

### Risk: Contract becomes too generic and vague

Mitigation:

- Keep strict required fields for IDs, references, dimensions, rules, state, and UI bindings.
- Use recipe-specific validators.

### Risk: Contract becomes a programming language

Mitigation:

- Rules remain semantic and declarative.
- Coder Agent handles implementation details.
- Avoid arbitrary expressions in v1.

### Risk: Artist and Coder outputs diverge

Mitigation:

- Asset IDs and paths are strict.
- Generated asset manifest must be validated against contract.
- Coder Agent consumes generated manifest, not guessed paths.

### Risk: Too many genres too early

Mitigation:

- Start with match-3 as the stress test.
- Add memory card as a simpler second genre.
- Do not add more until validator and adapter patterns stabilize.

### Risk: Visual polish remains weak

Mitigation:

- Treat feedback, animation, UI states, and audio as first-class contract sections.
- Add visual QA and style consistency checks.

---

## 16. Open Decisions

1. Contract format: YAML only, or YAML authoring with JSON compiled form?
2. First engine: Phaser TypeScript only, or Phaser TypeScript + Godot later?
3. Asset generation backend: image model API, local model, sourced asset library, or hybrid?
4. How strict should rule semantics be in v1?
5. Should Designer Agent output a single contract file first, then split later?

Recommended decisions for v1:

- YAML authoring, JSON Schema validation.
- Phaser 3 + TypeScript + Vite for v1; add Godot later only after the contract and validator stabilize.
- Hybrid asset pipeline: generated assets + optional curated library.
- Rule semantics strict enough for validation, flexible enough for LLM implementation.
- Single-file contract examples first, multi-file layout for real projects.

---

## 17. Final Recommendation

Build GameCraft as an agent-native capability layer first:

```text
Core = Universal Game Contract + Recipes + Validators + Agent Skills + Engine Adapter
Interface v1 = Coding agent toolkit + CLI
Interface v2 = App/API wrapper
```

This gives the fastest path to discovering the right contract abstraction, validating real game generation, and achieving polished 2D output without prematurely building product infrastructure.
