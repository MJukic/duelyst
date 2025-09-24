# Kin to Royalty – Duelyst Data Schema Reference

## Overview
This document specifies JSON resource formats for porting Duelyst cards, effects, keywords, and
patterns into Godot resources. Fields align with the core `Card` properties and modifier
structures used in the CC0 release.【F:app/sdk/cards/card.coffee†L41-L153】【F:app/sdk/modifiers/modifier.coffee†L24-L227】

## JSON Schemas
### Card Base (applies to Units, Spells, Artifacts)
```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://k2r.example/schema/card.json",
  "type": "object",
  "required": ["id", "name", "type", "mana_cost"],
  "properties": {
    "id": {"type": "string"},
    "name": {"type": "string"},
    "type": {"enum": ["Unit", "Spell", "Artifact"]},
    "mana_cost": {"type": "integer", "minimum": 0},
    "faction": {"type": "string"},
    "rarity": {"type": "string"},
    "race": {"type": "string"},
    "keywords": {"type": "array", "items": {"type": "string"}},
    "description": {"type": "string"},
    "base_stats": {
      "type": "object",
      "properties": {
        "attack": {"type": "integer"},
        "max_hp": {"type": "integer"},
        "speed": {"type": "integer"},
        "reach": {"type": "integer"}
      }
    },
    "signature": {"type": "string"},
    "inherent_modifiers": {
      "type": "array",
      "items": {"$ref": "modifier.json"}
    },
    "opening_gambit": {"$ref": "effect.json"},
    "dying_wish": {"$ref": "effect.json"}
  }
}
```

### Unit Card Extension
```json
{
  "$id": "https://k2r.example/schema/unit_card.json",
  "allOf": [
    {"$ref": "card.json"},
    {
      "properties": {
        "type": {"const": "Unit"},
        "battle_pet": {"type": "boolean"},
        "provokes": {"type": "boolean"},
        "movement_profile": {"$ref": "movement.json"},
        "attack_profile": {"$ref": "attack.json"}
      }
    }
  ]
}
```

### Spell Card Extension
```json
{
  "$id": "https://k2r.example/schema/spell_card.json",
  "allOf": [
    {"$ref": "card.json"},
    {
      "properties": {
        "type": {"const": "Spell"},
        "targets": {"$ref": "target_pattern.json"},
        "effects": {"type": "array", "items": {"$ref": "effect.json"}}
      }
    }
  ]
}
```

### Keyword Definition
```json
{
  "$id": "https://k2r.example/schema/keyword.json",
  "type": "object",
  "required": ["id", "name", "definition"],
  "properties": {
    "id": {"type": "string"},
    "name": {"type": "string"},
    "definition": {"type": "string"},
    "applied_modifier": {"type": "string"},
    "parameters": {"type": "object"}
  }
}
```

### Modifier Context (`modifier.json`)
```json
{
  "$id": "https://k2r.example/schema/modifier.json",
  "type": "object",
  "required": ["type"],
  "properties": {
    "type": {"type": "string"},
    "is_stackable": {"type": "boolean"},
    "is_removable": {"type": "boolean"},
    "attribute_buffs": {
      "type": "object",
      "properties": {
        "attack": {"type": "integer"},
        "max_hp": {"type": "integer"},
        "speed": {"type": "integer"},
        "reach": {"type": "integer"}
      }
    },
    "aura": {"$ref": "aura.json"},
    "triggers": {
      "type": "array",
      "items": {"$ref": "effect.json"}
    }
  }
}
```

### Effect (`effect.json`)
```json
{
  "$id": "https://k2r.example/schema/effect.json",
  "type": "object",
  "required": ["id", "hook"],
  "properties": {
    "id": {"type": "string"},
    "hook": {"type": "string"},
    "conditions": {"type": "array", "items": {"type": "string"}},
    "operations": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["op"],
        "properties": {
          "op": {"type": "string"},
          "value": {}
        }
      }
    }
  }
}
```

### Target Pattern (`target_pattern.json`)
```json
{
  "$id": "https://k2r.example/schema/target_pattern.json",
  "type": "object",
  "required": ["shape"],
  "properties": {
    "shape": {"enum": ["Single", "Radius", "Line", "Column", "Row", "WholeBoard"]},
    "radius": {"type": "integer", "minimum": 0},
    "offsets": {
      "type": "array",
      "items": {
        "type": "array",
        "prefixItems": [
          {"type": "integer"},
          {"type": "integer"}
        ],
        "items": false
      }
    },
    "include_self": {"type": "boolean"},
    "allow_allies": {"type": "boolean"},
    "allow_enemies": {"type": "boolean"},
    "allow_tiles": {"type": "boolean"}
  }
}
```

### Aura (`aura.json`)
```json
{
  "$id": "https://k2r.example/schema/aura.json",
  "type": "object",
  "properties": {
    "radius": {"type": "integer", "minimum": 0},
    "include_self": {"type": "boolean"},
    "include_allies": {"type": "boolean"},
    "include_enemies": {"type": "boolean"},
    "include_generals": {"type": "boolean"},
    "filters": {
      "type": "object",
      "properties": {
        "card_ids": {"type": "array", "items": {"type": "string"}},
        "races": {"type": "array", "items": {"type": "string"}},
        "modifiers": {"type": "array", "items": {"type": "string"}}
      }
    },
    "granted_modifiers": {
      "type": "array",
      "items": {"$ref": "modifier.json"}
    }
  }
}
```

### Movement Profile (`movement.json`)
```json
{
  "$id": "https://k2r.example/schema/movement.json",
  "type": "object",
  "properties": {
    "speed": {"type": "integer", "minimum": 0},
    "vectors": {
      "type": "array",
      "items": {
        "type": "array",
        "prefixItems": [
          {"type": "integer"},
          {"type": "integer"}
        ],
        "items": false
      }
    },
    "flying": {"type": "boolean"}
  }
}
```

### Attack Profile (`attack.json`)
```json
{
  "$id": "https://k2r.example/schema/attack.json",
  "type": "object",
  "properties": {
    "reach": {"type": "integer", "minimum": 0},
    "needs_los": {"type": "boolean"},
    "custom_pattern": {"$ref": "target_pattern.json"},
    "strikeback": {"type": "boolean"},
    "frenzy": {"type": "boolean"}
  }
}
```

## Example Instances
### Unit Card – Silverguard Knight
```json
{
  "id": "LYONAR_SILVERGUARD",
  "name": "Silverguard Knight",
  "type": "Unit",
  "mana_cost": 3,
  "keywords": ["ModifierProvoke", "ModifierBandingAttack"],
  "base_stats": {"attack": 1, "max_hp": 5},
  "inherent_modifiers": [
    {"type": "ModifierProvoke", "aura": {"radius": 1, "include_enemies": true}}
  ]
}
```
*Source data: Duelyst card catalog entry with provoke keyword.*【F:exports/inventories/cards_catalog.csv†L1-L40】【F:app/sdk/modifiers/modifierProvoke.coffee†L15-L33】

### Unit Card – Argeon General
```json
{
  "id": "GENERAL_ARGEON",
  "name": "Argeon Highmayne",
  "type": "Unit",
  "mana_cost": 0,
  "base_stats": {"attack": 2, "max_hp": 25},
  "signature": "LYONAR_BLOODBOUND"
}
```
*Generals are zero-cost units with high HP.*【F:exports/inventories/cards_catalog.csv†L1-L8】

### Spell Card – Example Buff
```json
{
  "id": "SONGHAI_INTERNAL_BUFF",
  "name": "Inner Focus",
  "type": "Spell",
  "mana_cost": 0,
  "targets": {"shape": "Single", "allow_allies": true},
  "effects": [
    {
      "id": "READY_TARGET",
      "hook": "after_resolution",
      "operations": [{"op": "set_status", "value": "Ready"}]
    }
  ]
}
```
*Spell data mirrors Duelyst's single-target buffs that refresh a friendly unit.*【F:app/sdk/cards/deck.coffee†L188-L229】

### Keyword – Battle Pet
```json
{
  "id": "ModifierBattlePet",
  "name": "Battle Pet",
  "definition": "Acts automatically at the start of its controller's turn.",
  "applied_modifier": "ModifierBattlePet",
  "parameters": {"automatic": true}
}
```
*Definition links to automated action planner.*【F:app/sdk/modifiers/modifierBattlePet.coffee†L38-L219】

### Effect – Frenzy Follow-up
```json
{
  "id": "FRENZY_SWEEP",
  "hook": "before_attack",
  "conditions": ["attack_is_explicit", "target_within_melee"],
  "operations": [
    {"op": "queue_attack", "value": {"scope": "adjacent_enemies"}}
  ]
}
```
*Represents the queued extra strikes of Frenzy.*【F:app/sdk/modifiers/modifierFrenzy.coffee†L35-L71】

### Target Pattern – Whole Board
```json
{
  "shape": "WholeBoard",
  "offsets": [],
  "include_self": true,
  "allow_allies": true,
  "allow_enemies": true
}
```
*Direct translation of `CONFIG.WHOLE_BOARD_RADIUS` helpers.*【F:app/common/config.js†L981-L1015】

### Aura – Provoke Field
```json
{
  "radius": 1,
  "include_self": false,
  "include_allies": false,
  "include_enemies": true,
  "granted_modifiers": [
    {"type": "ModifierProvoked", "is_removable": true}
  ]
}
```
*Matches the aura emitted by Provoke-bearing units.*【F:app/sdk/modifiers/modifierProvoke.coffee†L27-L33】【F:app/sdk/modifiers/modifierProvoked.coffee†L29-L48】

