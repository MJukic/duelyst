# Kin to Royalty – Duelyst Core Port Guide

*Source mechanics documented from the CC0 Duelyst release; implementation guidance is original.*

## Table of Contents
- [Overview](#overview)
- [Systems to Port](#systems-to-port)
  - [Grid/Tiles & Movement](#gridtiles--movement)
  - [Turn Manager & Phases](#turn-manager--phases)
  - [Targeting System](#targeting-system)
  - [Effect System](#effect-system)
  - [Combat Resolution](#combat-resolution)
  - [Keyword Glossary](#keyword-glossary)
  - [Status & Buff Rules](#status--buff-rules)
- [Godot 4.5 Implementation Plan](#godot-45-implementation-plan)
  - [Core Service Classes](#core-service-classes)
  - [Signals and Events](#signals-and-events)
  - [Resource Data Patterns](#resource-data-patterns)
- [Mechanic Map](#mechanic-map)
- [Edge Cases](#edge-cases)
- [Testing](#testing)
- [Data Migration Notes](#data-migration-notes)
- [Multiplayer Determinism](#multiplayer-determinism)

## Overview
- **Board & Setup:** 9×5 grid with center reference constants for effect targeting.【F:app/common/config.js†L71-L99】
- **Mana Economy:** Players begin with 2 cores, cap at 9, and modifiers can adjust the live value.
  Hand limit is 6; starting hand is 5 with up to 2 mulligans.【F:app/common/config.js†L295-L303】
- **Draw Rules:** Each end phase creates up to one draw action (plus modifiers) and burns extras
  when the hand is full.【F:app/common/config.js†L369-L372】【F:app/sdk/cards/deck.coffee†L196-L229】
- **Replace:** Once per turn by default; modifiers adjust the quota.【F:app/sdk/cards/deck.coffee†L328-L343】
- **Turn Structure:** Authoritative queue resolves all actions, then handles end-turn draws,
  modifier duration ticks, and signature activation before the next start turn event.【F:app/sdk/gameSession.coffee†L1831-L1868】
- **Victory:** When all generals are dead the game ends—tie if both fall, otherwise the surviving
  player wins.【F:app/sdk/gameSession.coffee†L926-L981】

## Systems to Port
### Grid/Tiles & Movement
- Board exposes typed queries (units, tiles, spells) and filters by allegiance, radius, rows,
  and columns.【F:app/sdk/board.coffee†L14-L466】
- Movement paths use BFS over `CONFIG.MOVE_PATTERN_STEP`, respect obstructions, but allow allies
  and flyers to pass through before stopping on empty tiles.【F:app/sdk/entities/movementRange.coffee†L19-L76】
- Entities track remaining moves/attacks and exhaustion; battle pets ignore exhaustion gates.
  Movement increments also enforce "move before extra attacks" sequencing.【F:app/sdk/entities/entity.coffee†L42-L415】

### Turn Manager & Phases
- GameSession maintains an action stack, buffers events while effects resolve, and only advances
  phases once the queue empties. End turn draws occur before modifier duration updates and before
  the next player's start turn actions.【F:app/sdk/gameSession.coffee†L1831-L1868】
- Start/end events are published through the shared event bus; modifiers listen for
  `start_turn`, `end_turn`, duration change hooks, and cache update prompts.【F:app/common/event_types.js†L200-L339】【F:app/sdk/modifiers/modifier.coffee†L248-L284】

### Targeting System
- Attack range atlases precompute valid positions/targets per origin, accounting for LOS when
  needed and filtering friendlies.【F:app/sdk/entities/attackRange.coffee†L15-L195】
- Board helpers cover around/within radius, column/row scans, cardinal alignment, and team
  filtered sets (enemy/friendly, starting side, behind/in front).【F:app/sdk/board.coffee†L288-L600】
- Utility methods produce spawn positions from patterns, optional obstruction checks, and
  random non-conflicting allocations for multi-source effects.【F:app/common/utils/utils_game_session.coffee†L59-L199】

### Effect System
- Actions (move, attack, damage, die, apply/remove modifier) run through validation,
  execution, cleanup, and modifier hook phases.【F:app/sdk/actions/damageAction.coffee†L26-L101】【F:app/sdk/modifiers/modifier.coffee†L248-L395】
- Modifiers define aura behaviour, stacking model, attribute overrides, and trigger lifecycle
  with explicit context objects for serialization.【F:app/sdk/modifiers/modifier.coffee†L24-L227】
- Silence/Dispel removes removable modifiers in reverse order and flushes keyword caches.【F:app/sdk/cards/card.coffee†L1849-L1867】

### Combat Resolution
- Explicit attacks are `AttackAction` instances (damage equals current ATK) and increment
  attack counters when not implicit.【F:app/sdk/actions/attackAction.coffee†L22-L44】
- Damage processing floors multi-step adjustments (add, multiply, final add) and schedules
  death actions once HP ≤ 0.【F:app/sdk/actions/damageAction.coffee†L26-L101】
- Strikeback is attached via inherent modifier on all units, triggering when melee reach allows
  retaliation.【F:app/sdk/entities/unit.coffee†L35-L44】【F:app/sdk/modifiers/modifierStrikeback.coffee†L30-L64】
- Frenzy, Backstab, Provoke, Flying, Ranged, and Battle Pet modifiers extend combat behaviour
  (see Keyword Glossary).【F:app/sdk/modifiers/modifierFrenzy.coffee†L31-L71】【F:app/sdk/modifiers/modifierBackstab.coffee†L26-L55】【F:app/sdk/modifiers/modifierProvoke.coffee†L15-L33】【F:app/sdk/modifiers/modifierFlying.coffee†L5-L20】【F:app/sdk/modifiers/modifierRanged.coffee†L7-L22】【F:app/sdk/modifiers/modifierBattlePet.coffee†L38-L219】

### Keyword Glossary
- **Provoke:** Aura applying `Provoked` within radius 1; provoked units cannot move and must
  attack provoking enemies.【F:app/sdk/modifiers/modifierProvoke.coffee†L15-L33】【F:app/sdk/modifiers/modifierProvoked.coffee†L29-L48】
- **Flying:** Sets speed to `CONFIG.SPEED_INFINITE`; ignores pathing limits but still obeys
  landing restrictions.【F:app/sdk/modifiers/modifierFlying.coffee†L5-L20】【F:app/common/config.js†L966-L983】
- **Ranged:** Extends reach to full board (difference between ranged and melee constants).【F:app/sdk/modifiers/modifierRanged.coffee†L7-L22】【F:app/common/config.js†L972-L977】
- **Frenzy:** On explicit melee attack, schedules attacks on other adjacent enemies.【F:app/sdk/modifiers/modifierFrenzy.coffee†L35-L71】
- **Backstab:** Bonus damage from back arc or forced backstabbed status; disables strikeback.【F:app/sdk/modifiers/modifierBackstab.coffee†L33-L55】
- **Strikeback:** Counterattacks implicit when conditions are met.【F:app/sdk/modifiers/modifierStrikeback.coffee†L30-L64】
- **Battle Pet:** Automated move/attack planner plus player-control guard.【F:app/sdk/modifiers/modifierBattlePet.coffee†L38-L219】
- **Opening Gambit:** One-time trigger when played; buffers events before executing custom
  logic.【F:app/sdk/modifiers/modifierOpeningGambit.coffee†L9-L45】
- **Dying Wish:** Executes custom code when the unit's death action resolves.【F:app/sdk/modifiers/modifierDyingWish.coffee†L8-L31】

### Status & Buff Rules
- Modifiers support stacking, aura propagation, rebasing, fixed overrides, and per-turn durations
  (start/end). Duration updates fire only when queue is empty and phases change.【F:app/sdk/modifiers/modifier.coffee†L52-L395】【F:app/sdk/gameSession.coffee†L1831-L1868】
- Attribute buffs can be additive, absolute, rebased, or fixed, influencing recalculation order
  on the owning card.【F:app/sdk/modifiers/modifier.coffee†L37-L195】
- Cards expose helper methods to flush keyword caches after dispel/silence, ensuring state
  rebuilds deterministically.【F:app/sdk/cards/card.coffee†L1849-L1867】

## Godot 4.5 Implementation Plan
### Core Service Classes
```gdscript
class_name GridService
extends Node

signal tile_changed(pos: Vector2i)

var columns := 9
var rows := 5
var occupancy := {}

func is_on_board(pos: Vector2i) -> bool:
    return pos.x >= 0 and pos.y >= 0 and pos.x < columns and pos.y < rows

func get_entities(filter: Callable) -> Array:
    return occupancy.values().filter(filter)
```
```gdscript
class_name MovementRange
extends Resource

var move_vectors: Array[Vector2i] = [Vector2i(1, 0), Vector2i(-1, 0), Vector2i(0, 1), Vector2i(0, -1)]

func get_paths(grid: GridService, entity) -> Array:
    # BFS respecting flying/exhaustion flags
    return []
```
```gdscript
class_name AttackRange
extends Resource

func get_valid_targets(grid: GridService, entity, from_positions: Array) -> Array:
    return []
```
```gdscript
class_name TurnManager
extends Node

signal turn_started(player_id)
signal turn_ended(player_id)
signal phase_changed(name)

var queue: Array = []
var current_player := 0

func enqueue(action):
    queue.append(action)

func process_queue():
    while queue.size() > 0:
        var action = queue.pop_front()
        action.execute()
```
```gdscript
class_name EffectEngine
extends Node

signal before_action(action)
signal after_action(action)
signal action_invalidated(action, reason)

func execute(action):
    emit_signal("before_action", action)
    if action.validate():
        action.resolve()
        emit_signal("after_action", action)
    else:
        emit_signal("action_invalidated", action, action.error)
```
```gdscript
class_name Modifier
extends Resource

var is_stackable := true
var is_removable := true
var duration_start := 0
var duration_end := 0
var aura_radius := 0
var applied_entities: Array = []

func on_apply(target):
    applied_entities.append(target)

func on_remove(target):
    applied_entities.erase(target)
```
```gdscript
class_name UnitState
extends Resource

var atk := 0
var max_hp := 1
var speed := 2
var reach := 1
var moves_left := 1
var attacks_left := 1
var modifiers: Array[Modifier] = []

func can_move() -> bool:
    return moves_left > 0 and speed > 0
```
```gdscript
class_name CardData
extends Resource

@export var id := ""
@export var name := ""
@export var type := "Unit"
@export var mana_cost := 0
@export var keywords: Array[String] = []
@export var base_stats := {}
@export var abilities := []
```

### Signals and Events
- Emit Godot signals mirroring Duelyst phases: `before_action`, `after_action`, `turn_started`,
  `turn_ended`, `modifier_started`, `modifier_ended`, `unit_summoned`, `unit_died`, `card_drawn`,
  `card_burned`, `before_attack`, `after_attack`, `before_counter`, `after_counter`,
  `turn_timer_tick`.
- Map engine events to Godot: `modify_action_for_execution` → `before_action`; `action` →
  `after_action`; `modifier_add_aura`/`modifier_remove_aura` → custom `aura_updated`; `start_turn`
  and `end_turn` feed the TurnManager signals.【F:app/common/event_types.js†L200-L339】

### Resource Data Patterns
- Use `.tres` resources for `CardData`, `UnitState`, `ModifierConfig`, and `TargetPattern`.
- Example `UnitCard.tres`:
```gdscript
[gd_resource type="Resource" script="res://cards/unit_card.gd"]
[id="LYONAR_SILVERGUARD", name="Silverguard Knight", type="Unit", mana_cost=3,
 keywords=["Provoke", "Banding"], base_stats={"atk": 1, "max_hp": 5},
 abilities=["Give adjacent allies +1/+1 when summoned"]]
```
- Patterns should mirror constants like `PATTERN_WHOLE_BOARD`; encode as arrays of `Vector2i`
  exported from resources.【F:app/common/config.js†L981-L1015】

## Mechanic Map
| Duelyst Keyword/Ability | Godot Effect Class | Event Hooks |
| --- | --- | --- |
| Provoke | `ModifierProvokeEffect` | `before_action`, `action_invalidated` |
| Provoked | `StatusProvoked` | `before_action` |
| Flying | `ModifierFlight` | `movement_paths_requested` |
| Ranged | `ModifierRange` | `before_attack`, `targeting_query` |
| Frenzy | `ModifierFrenzy` | `before_attack`, `after_attack` |
| Backstab | `ModifierBackstab` | `before_attack` |
| Strikeback | `ModifierStrikeback` | `before_action` (attacks), `before_counter` |
| Battle Pet | `ModifierBattlePetAI` | `turn_started`, `after_action` |
| Opening Gambit | `ModifierOpeningGambit` | `unit_summoned` |
| Dying Wish | `ModifierDyingWish` | `unit_died`, `after_action` |
| Aura | `ModifierAuraEmitter` | `modifier_started`, `aura_updated` |
| Dispel | `DispelEffect` | `before_action`, `after_action` |
| Replace | `ReplaceService` | `turn_started`, `card_drawn` |

## Edge Cases
- **Simultaneous deaths:** Damage queues death actions after resolving total damage to ensure
  death triggers run once per entity.【F:app/sdk/actions/damageAction.coffee†L74-L101】
- **Tie:** Multiple generals dying yields draw; ensure Godot TurnManager checks all generals
  before assigning a winner.【F:app/sdk/gameSession.coffee†L959-L981】
- **Summon on blocked tile:** Use obstruction queries and queued spawn validation from
  `UtilsGameSession` when resolving teleport/summon effects.【F:app/common/utils/utils_game_session.coffee†L97-L199】
- **Out-of-range retarget:** Attack atlases filter valid targets; invalid attacks should be
  cancelled before resolution and raise an error message (match Provoked invalidation).【F:app/sdk/modifiers/modifierProvoked.coffee†L29-L48】
- **Dispel vs. aura:** Removing aura owner triggers modifier cleanup for sub-modifiers and
  flushes caches, so Godot should recalculate recipients after every `aura_updated` signal.【F:app/sdk/modifiers/modifier.coffee†L304-L379】
- **Transform vs. Dying Wish:** Trigger order ensures death effects fire on original entity before
  removal; transformations should preserve parent action indices for replay integrity.【F:app/sdk/modifiers/modifierDyingWish.coffee†L24-L31】【F:app/sdk/modifiers/modifier.coffee†L324-L335】
- **Hand/board caps:** Draw/burn actions respect `MAX_HAND_SIZE`; board spawn should queue burns
  or rejects when no valid tiles remain.【F:app/sdk/cards/deck.coffee†L196-L229】【F:app/sdk/board.coffee†L302-L317】

## Testing
- **Unit Tests:**
  - Grid pathfinding: deterministic BFS for various obstacles/flying states.
  - Targeting: attack atlas returns same sets for mirrored positions.
  - Turn sequencing: start/end events fire in correct order with queued actions.
  - Modifier stacking: additive, absolute, rebased, and fixed attributes produce expected stats.
  - Keyword behaviours: provoke invalidation, frenzy follow-up attacks, backstab bonus, battle pet
    automation, dying wish triggers.
- **Sample Given/When/Then:**
  - *Given* a unit with Provoke aura, *when* an enemy tries to move away, *then* action fails with
    provoked reason.【F:app/sdk/modifiers/modifierProvoked.coffee†L29-L48】
  - *Given* a backstabber behind an enemy, *when* it attacks, *then* damage equals base + bonus and
    no counterstrike occurs.【F:app/sdk/modifiers/modifierBackstab.coffee†L41-L55】
  - *Given* a battle pet with no targets, *when* turn starts, *then* it either moves toward the
    closest enemy or idles if provoked.【F:app/sdk/modifiers/modifierBattlePet.coffee†L38-L124】
  - *Given* a unit with stacked buffs, *when* a dispel resolves, *then* only non-removable modifiers
    remain and stats revert.【F:app/sdk/cards/card.coffee†L1849-L1867】
- **Replays:** log deterministic seeds for random spawn/target selection so automated tests can
  reproduce board states.【F:app/sdk/modifiers/modifierBattlePet.coffee†L55-L211】

## Data Migration Notes
- Export card/unit/spell records with fields: `id`, `name`, `type`, `mana_cost`, `attack`,
  `max_hp`, `keywords`, `description`, aligning with `Card` base class attributes.【F:app/sdk/cards/card.coffee†L41-L153】【F:exports/inventories/cards_catalog.csv†L1-L40】
- Map Duelyst keywords to Godot modifiers via the Mechanic Map.
- Store keyword arrays as string identifiers, letting Godot load matching `Modifier` resources.
- Hand/Deck metadata (`MAX_HAND_SIZE`, `CARD_DRAW_PER_TURN`) should live in a Godot `GameConfig`
  resource for easy tuning.【F:app/common/config.js†L295-L372】
- Serialize patterns (line, radius, whole-board) into dedicated resources to mirror Duelyst's
  `CONFIG.PATTERN_*` constants.【F:app/common/config.js†L981-L1015】

## Multiplayer Determinism
Keep action resolution deterministic: enforce ordered queues, seeded randomness (per turn or
per action), and record authoritative events for replay/rollback. Networking is out of scope,
so focus on serializable state snapshots after each resolved action, mirroring Duelyst's action
state record strategy.【F:app/sdk/modifiers/modifier.coffee†L635-L661】
