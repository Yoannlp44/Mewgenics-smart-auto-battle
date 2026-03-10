# Smart_AutoBattle — Developer Guide

This mod makes cats play intelligently in auto-battle with class-specific AI behavior.

---

## Mod Objective

- Each class uses its own decision and movement presets.
- **Cats never hit allies** (via `damage_ally` penalty).
- Applied automatically at game start.

---

## File Structure

```
mods/Smart_AutoBattle/
├── agent.md                              # This file
├── data/                                 # Patch files
│   ├── abilities/                        # Ability AI patches by class
│   │   └── <class>_abilities.gon.patch  # All abilities patched with ai_base_score
│   ├── ai_presets/
│   │   ├── decision_presets.gon.patch   # Decision weights per class
│   │   └── move_presets.gon.patch       # Movement behavior per class
│   ├── characters/
│   │   └── player_cat.gon.patch         # Universal fallback (GenericBrain)
│   └── classes/
│       ├── classes.gon.patch            # Base classes (Fighter, Hunter, Tank, Mage, Medic, Thief, Colorless)
│       └── advanced_classes.gon.patch    # Advanced classes (Monk, Butcher, Druid, Tinkerer, Necromancer, Psychic, Jester)
└── dataGame/                            # GAME REFERENCE - NEVER MODIFY
    ├── abilities/                        # Ability definitions (read-only reference)
    ├── ai_presets/                      # Valid preset fields
    └── classes/                         # Class definitions
```

---

## How AI is Applied

### 1. Universal Fallback — `player_cat.gon.patch`

Replaces the default `brain` for ALL player cats:
```
PlayerCat.merge {
    ai {
        brain GenericBrain
        decision_weights smart_default
        move_weights keep_distance
    }
}
```

### 2. Class-Specific AI — `innate_passives` + `ReplaceBrain`

`innate_passives` auto-applies passives to all cats of a class at combat start.
`ReplaceBrain` is an inline passive that replaces the cat's `ai {}` block.

Pattern (identical for all classes):
```
ClassName.merge {
    innate_passives {
        ReplaceBrain {
            brain GenericBrain
            decision_weights smart_classname
            move_weights smart_classname_move
        }
    }
}
```

**Priority**: `innate_passives` + `ReplaceBrain` > `player_cat.gon.patch`

---

## Valid Field References

### decision_presets fields
```
damage_ally           heal_ally             kill_ally
damage_enemy          heal_enemy            kill_enemy
debuff_ally           buff_ally
debuff_enemy          buff_enemy
damage_self           heal_self             buff_self           debuff_self
damage_ally_corpse    damage_enemy_corpse
revive_ally_corpse    revive_enemy_corpse
spawn_object          spawn_object_distance_to_enemy    spawn_object_distance_to_ally
spawn_object_preferred_distance
negative_weight_scale spend_mana_scale
flat_cast_bonus
consider_total_damage     (bool)
consider_secondary_damage (bool)
accurate_knockback        (bool)
consider_overkill         (bool)
simple                    (bool)
consider_aoe              (bool)
```

### move_presets fields
```
distance_to_enemy       distance_to_ally        distance_to_character
distance_to_corpse      distance_to_aggro_target
distance_to_center      distance_to_water
cap_distance_to_enemy   cap_distance_to_ally    cap_distance_to_character
preferred_distance      (number, or: mov, reach, mov+N, mov-N, mov+reach)
total_distance_moved    cap_total_distance_moved
face_closest_enemy      face_aggro_target       face_camera
randomness
danger_avoidance        (float — 0.5 light, 3-5 normal, 20+ extreme)
tall_grass              (float — attracts to TallGrassTile)
lava                    (float — attracts to LavaTile, DO NOT use for players)
count_nomove_in_eval    (bool)
consider_aggro_target_enemy (bool)
exclude_characters_tagged <tag>
```

---

## Class Presets

| Class       | Source File       | Decision Preset   | Move Preset           |
|-------------|-------------------|-------------------|-----------------------|
| Fighter     | classes.gon       | smart_fighter     | smart_fighter_move    |
| Hunter      | classes.gon       | smart_hunter      | smart_hunter_move     |
| Tank        | classes.gon       | smart_tank        | smart_tank_move       |
| Mage        | classes.gon       | smart_mage        | smart_mage_move       |
| Medic       | classes.gon       | smart_cleric      | smart_cleric_move     |
| Thief       | classes.gon       | smart_thief       | smart_thief_move      |
| Monk        | advanced_classes.gon | smart_monk     | smart_monk_move       |
| Butcher     | advanced_classes.gon | smart_butcher   | smart_butcher_move    |
| Druid       | advanced_classes.gon | smart_druid     | smart_druid_move      |
| Tinkerer    | advanced_classes.gon | smart_tinkerer  | smart_tinkerer_move   |
| Necromancer | advanced_classes.gon | smart_necromancer | smart_necromancer_move |
| Psychic     | advanced_classes.gon | smart_psychic   | smart_psychic_move    |
| Jester      | advanced_classes.gon | smart_jester    | smart_jester_move     |
| Colorless   | classes.gon       | smart_colorless   | smart_colorless_move  |
| (fallback)  | player_cat.gon    | smart_default     | keep_distance (game)  |

---

## Ability Patching

### Format
```gon
AbilityName.merge {
    damage_instance {
        ai_base_score N              // Direct score modifier
        // OR
        custom_additional_ai_weight <keyword>  // can be use with ai_base_score or not
    }
}
```

### ai_base_score Guidelines

| Score  | Usage |
|--------|-------|
| 0      | Normal priority (useful attacks/buffs) |
| -5 to -20 | Slight penalty (situational/self-damage) |
| -50    | Heavy penalty (AoE that might hit allies) |
| -99999 | Disabled (dangerous to team) |

### custom_additional_ai_weight Keywords
- `tile_close_to_enemy` — Only use near enemies
- `tile_close_to_enemy_soft` — Soft preference near enemies  
- `must_heal_most_missing_health` — Only when hurt
- `toss_towards_buddy`
- `toss_far`
- `magnetize_favorlineup`
- `tile_has_no_known_traps`
- `avoid_redundant_debuffs_strict`
- `target_farthest`
- `target_closest`
- `pyrophina_throw_to_lava`
- `tile_close_to_enemy_soft`
- `tile_exists`
- `favor_tile_far_away`
- `no_coins_on_map`
- `favor_enemy_already_moved`
- `avoid_redundant_debuffs`
- `enemy_is_webbed`
- `favor_tile_far_away`
- `one_charmed_enemy_at_a_time`
- `must_target_buddy`
- `must_not_target_buddy`
- `toss_farthest`
- `moonhead_punchself`
- `moonhead_use_if_cracked`
- `toss_towards_bottomleft`
- `dont_target_rock`
- `tutorial_boulderdrop_miss_if_one_left`
- `pills_only`
- `no_redundant_formchange`
- `dybbuk_possession`
- `thiefcat_coinroll`
- `thiefcat_roll`
- `non_bramble_tile_close_to_enemy`
- `spread_grenades`
- `avoid_target_map_top`
- `teslacoil_priorities`

### CRITICAL: `custom_additional_ai_weight` MUST be prefixed
```gon
// CORRECT
custom_additional_ai_weight tile_close_to_enemy

// WRONG - causes GON error on launch
tile_close_to_enemy
```



---

## Development Rules

1. **Never modify `dataGame/`** — Read-only reference only.
2. **Always use `.merge`** for patches (never redefine objects entirely).
3. **ASCII only in all `.patch` files** — GON parser does not support UTF-8.
4. **Use `//` comments freely** — Supported by GON parser.
5. **Verify exact names** in `dataGame/classes/` before writing patches.
6. **Verify fields** in `dataGame/ai_presets/` before adding new fields.
7. **The `player_cat.gon.patch` fallback is sacred** — Never weaken it. Last defense against friendly fire.

---


