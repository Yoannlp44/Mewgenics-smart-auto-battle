# Smart_AutoBattle — Agent Knowledge Base

Document de référence pour les sessions de développement IA du mod.  
Contient : architecture, découvertes, bugs connus, pistes d'amélioration.

---

## Objectif du mod

Faire jouer les chats en auto-battle avec une IA intelligente et **class-aware** :
- Chaque classe utilise ses propres presets de décision et de mouvement.
- **Aucun chat ne frappe jamais ses alliés** (pénalité `damage_ally` très négative).
- L'application est automatique dès le début, sans action du joueur.

---

## Structure des fichiers

```
mods/Smart_AutoBattle/
├── agent.md                               ← CE FICHIER
├── data/                                  ← FICHIERS DU MOD (patches)
│   ├── abilities/                         ← Patches d'abilities par classe (ai_base_score / custom_additional_ai_weight)
│   │   ├── monk_abilities.gon.patch       Porcupine (tile_close_to_enemy), Meditate (−99999 sleep 8t)
│   │   ├── tank_abilities.gon.patch       BarbedWire (tile_close_to_enemy)
│   │   ├── colorless_abilities.gon.patch  Rest (must_heal_most_missing_health), Brace (tile_close_to_enemy)
│   │   ├── thief_abilities.gon.patch      CoinToss (−10, évite boucle ReloadOnGainCoins)
│   │   ├── mage_abilities.gon.patch       Absorb (must_heal_most_missing_health), ManaMeld (−5)
│   │   ├── necromancer_abilities.gon.patch Flatline (−99999 self-die), BloodGeyser (−20), GigaDrain (−5)
│   │   ├── druid_abilities.gon.patch      ChaChaSlide (−99999 friendly-fire), HardenSkin (tile_close_to_enemy)
│   │   ├── medic_abilities.gon.patch      BornAgain (−99999 sleep 4t), Zealot (tile_close_to_enemy)
│   │   ├── butcher_abilities.gon.patch    SelfMutilate (−5), Fartoom (−10), LightenTheLoad (−10)
│   │   └── jester_abilities.gon.patch     Bump (−99999 displaces allies)
│   ├── ai_presets/
│   │   ├── decision_presets.gon.patch     14 presets (smart_default + 13 classes)
│   │   └── move_presets.gon.patch         13 move presets par classe
│   ├── characters/
│   │   └── player_cat.gon.patch           Fallback universel (GenericBrain + smart_default)
│   └── classes/
│       ├── classes.gon.patch              6 classes de base (Fighter, Hunter, Tank, Mage, Medic, Thief)
│       └── advanced_classes.gon.patch     7 classes avancées + Colorless
└── dataGame/                              ← REFERENCE JEU (ne jamais modifier)
    ├── abilities/                         ← Abilities de chaque classe (référence)
    ├── ai_presets/
    │   ├── decision_presets.gon           Tous les champs valides pour decision presets
    │   └── move_presets.gon               Tous les champs valides pour move presets
    ├── characters/
    │   └── player_cat.gon                 Définition de base du PlayerCat
    └── classes/
        ├── classes.gon                    Fighter, Hunter, Mage, Medic, Tank, Thief, Colorless
        └── advanced_classes.gon           Monk, Butcher, Druid, Tinkerer, Necromancer, Psychic, Jester
```

---

## Mécanisme central : comment l'IA est appliquée

### 1. Fallback universel — `player_cat.gon.patch`
Remplace le `brain` par défaut de TOUS les chats joueur.
```
PlayerCat.merge {
    ai {
        brain GenericBrain
        decision_weights smart_default
        move_weights keep_distance
    }
}
```
Garantit qu'aucun chat ne tape ses alliés même si la logique de classe échoue.

### 2. Application par classe — `innate_passives` + `ReplaceBrain`
`innate_passives` est un champ de définition de classe qui auto-applique des passives à tous les chats de la classe **dès le début du combat**.  
`ReplaceBrain` est une passive inline qui remplace le bloc `ai {}` du chat.

Pattern utilisé (identique pour toutes les classes) :
```
NomDeClasse.merge {
    innate_passives {
        ReplaceBrain {
            brain GenericBrain
            decision_weights smart_nomdepreset
            move_weights smart_nomdepreset_move
        }
    }
}
```

**Priorité** : `innate_passives` + `ReplaceBrain` > `player_cat.gon.patch` (override direct).

### 3. Les presets eux-mêmes
- `decision_presets.gon.patch` : définit les poids de décision (quelle action choisir).
- `move_presets.gon.patch` : définit le comportement de déplacement.
- Utiliser `.merge` car les fichiers de presets originaux existent déjà dans le jeu.

---

## Référence des champs valides

### decision_presets — champs confirmés
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

### move_presets — champs confirmés
```
distance_to_enemy       distance_to_ally        distance_to_character
distance_to_corpse      distance_to_aggro_target
distance_to_center      distance_to_water
cap_distance_to_enemy   cap_distance_to_ally    cap_distance_to_character
preferred_distance      (nombre, ou: mov, reach, mov+N, mov-N, mov+reach)
total_distance_moved    cap_total_distance_moved
face_closest_enemy      face_aggro_target       face_camera
randomness
danger_avoidance
tall_grass              lava
count_nomove_in_eval    (bool)
consider_aggro_target_enemy (bool)
exclude_characters_tagged <tag>
```

---

## Liste des classes et leurs presets

| Classe         | Fichier source          | decision preset      | move preset               |
|----------------|-------------------------|----------------------|---------------------------|
| Fighter        | classes.gon             | smart_fighter        | smart_fighter_move        |
| Hunter         | classes.gon             | smart_hunter         | smart_hunter_move         |
| Tank           | classes.gon             | smart_tank           | smart_tank_move           |
| Mage           | classes.gon             | smart_mage           | smart_mage_move           |
| Medic          | classes.gon             | smart_cleric         | smart_cleric_move         |
| Thief          | classes.gon             | smart_thief          | smart_thief_move          |
| Monk           | advanced_classes.gon    | smart_monk           | smart_monk_move           |
| Butcher        | advanced_classes.gon    | smart_butcher        | smart_butcher_move        |
| Druid          | advanced_classes.gon    | smart_druid          | smart_druid_move          |
| Tinkerer       | advanced_classes.gon    | smart_tinkerer       | smart_tinkerer_move       |
| Necromancer    | advanced_classes.gon    | smart_necromancer    | smart_necromancer_move    |
| Psychic        | advanced_classes.gon    | smart_psychic        | smart_psychic_move        |
| Jester         | advanced_classes.gon    | smart_jester         | smart_jester_move         |
| Colorless      | classes.gon             | smart_colorless      | smart_colorless_move      |
| (fallback)     | player_cat.gon          | smart_default        | keep_distance (jeu)       |

---

## Bugs connus / A corriger

### ~~BUG CRITIQUE~~ CORRIGÉ — Nom de classe `Colorless` vs `Collarless`
- **Corrigé le 2026-03-07.**
- `advanced_classes.gon.patch` ligne 74 : `Collarless.merge` → `Colorless.merge` ✅
- Presets renommés : `smart_collarless` → `smart_colorless`, `smart_collarless_move` → `smart_colorless_move` ✅

### Incertitude — `innate_passives` sur classes de base
- Les classes avancées (`advanced_classes.gon`) utilisent `innate_passives` nativement (Monk, Druid, Tinkerer, Psychic).
- Les classes de base (`classes.gon`) n'en ont **pas** nativement.
- Le merge devrait fonctionner (le moteur GON supporte l'ajout de nouveaux blocs via `.merge`).
- **A confirmer en jeu** si Fighter, Hunter, etc. ont bien leur IA spécifique.
- En cas d'échec : le fallback `smart_default` dans `player_cat.gon.patch` assure quand même qu'ils ne tapent pas les alliés.

### Monk — conflit avec `innate_passives` existant
- `Monk` a déjà `innate_passives { MonkStances [...] }` dans le jeu.
- Notre patch fait `Monk.merge { innate_passives { ReplaceBrain {...} } }`.
- Le comportement du `.merge` sur un bloc `innate_passives` existant est : **fusion** (les deux passives coexistent) — ce qui est le comportement souhaité.
- **A confirmer** que `MonkStances` n'est pas écrasé.

---

## Pistes d'amélioration (prochaines sessions)

### Corrections appliquées le 2026-03-07 (session 1)
- **Colorless** : `Collarless.merge` → `Colorless.merge` + renommage des presets ✅
- **Monk** : `buff_self` réduit de 4 → 1 (évite `Meditate` Sleep 8 tours) ✅
- **Tank** : `buff_self` et `buff_ally` réduits de 2 → 1 (évite Thorns/BarbedWire inutiles) ✅
- **Mage** : ajout de `consider_aoe true` ✅
- **Druid** : ajout de `consider_aoe true` (ChaChaSlide, DeathMetal) ✅
- **Thief** : `spend_mana_scale` réduit de .99 → .5 (évite spam CoinToss / boucle ReloadOnGainCoins) ✅

### Corrections appliquées le 2026-03-07 (session 2) — patches abilities
- **Monk** : `Porcupine`/`Porcupine2` → `tile_close_to_enemy` ; `Meditate`/`Meditate2` → `ai_base_score -99999` ✅
- **Tank** : `BarbedWire`/`BarbedWire2` → `tile_close_to_enemy` ✅
- **Colorless** : `Rest`/`Rest2` → `must_heal_most_missing_health` ; `Brace`/`Brace2` → `tile_close_to_enemy` ✅
- **Thief** : `CoinToss`/`CoinToss2` → `ai_base_score -10` ✅
- **Mage** : `Absorb`/`Absorb2` → `must_heal_most_missing_health` ; `ManaMeld`/`ManaMeld2` → `ai_base_score -5` ✅
- **Necromancer** : `Flatline`/`Flatline2` → `ai_base_score -99999` ; `BloodGeyser`/`BloodGeyser2` → `-20` ; `GigaDrain`/`GigaDrain2` → `-5` ✅
- **Druid** : `ChaChaSlide`/`ChaChaSlide2` → `ai_base_score -99999` ; `HardenSkin`/`HardenSkin2` → `tile_close_to_enemy` ✅
- **Medic** : `BornAgain` → `ai_base_score -99999` (sleep 4t) ; `Zealot`/`Zealot2` → `tile_close_to_enemy` ✅
- **Butcher** : `SelfMutilate`/`SelfMutilate2` → `-5` ; `Fartoom`/`Fartoom2` → `-10` ; `LightenTheLoad`/`LightenTheLoad2` → `-10` ✅
- **Jester** : `Bump`/`Bump2` → `ai_base_score -99999` ✅

### Notes techniques — patches abilities
- **`ChaosTeleport` (Mage)** : template `teleport`, pas de `damage_instance` → impossible à patcher via ce mécanisme
- **`BornAgain2` (Medic)** : version améliorée SANS le `self_damage Sleep` — ne pas patcher, c'est une bonne ability
- Les patches sont dans `data/abilities/<classe>_abilities.gon.patch`
- Format : `AbilityName.merge { damage_instance { ai_base_score N } }` ou `custom_additional_ai_weight <keyword>`

### Corrections appliquées le 2026-03-07 (session 3) — abilities colorless communes
Les abilities `Colorless` sont communes à toutes les classes (équipables par n'importe qui).
Patchs ajoutés dans `data/abilities/colorless_abilities.gon.patch` :

**Sleep :**
- `CatNap` → `must_heal_most_missing_health` (Cleanse + Sleep 1) ✅
- `CatNap2` : override damage_instance = `AllStatsUp 1` seulement, **pas de sleep** — non patchée intentionnellement

**Near-enemy buffs :**
- `Block`/`Block2` → `tile_close_to_enemy` (Shield inutile sans ennemi) ✅
- `GainThorns`/`GainThorns2` → `tile_close_to_enemy_soft` (Thorns permanent, moins urgent) ✅

**Risque létal :**
- `RussianRoulette`/`RussianRoulette2` → `ai_base_score -99999` (1/6 chance de mort) ✅

**Friendly-fire / aléatoire :**
- `LotteryShottery`/`LotteryShottery2` → `ai_base_score -99999` (cible aléatoire incluant alliés) ✅
- `Metronome`/`Metronome2` → `ai_base_score -99999` (action complètement aléatoire) ✅

**Disorder self-infligé :**
- `ForbiddenFart`/`ForbiddenFart2` → `ai_base_score -50` (inflige toujours un disorder sur soi) ✅

**Anti-alliés :**
- `ManaDrain`/`ManaDrain2` → `ai_base_score -99999` (vole le mana des alliés) ✅
- `HealBolt` (v1) → `ai_base_score -99999` (peut soigner les ennemis — pas de `DontHealEnemies`) ✅
  - `HealBolt2` : a `DontHealEnemies 1` et `Conditional_Enemy` — non patchée intentionnellement

**Action inutile :**
- `WasteTime` → `ai_base_score -99999` (coûte 1 mana, ne fait rien) ✅
  - `WasteTime2` : donne `Charge 1` — légèrement utile, non patchée

### Abilities colorless analysées mais non patchées (intentionnel)
- **ChaosTeleport** : pas de `damage_instance` → impossible
- **CatNap2** : override damage_instance = `AllStatsUp 1` seulement, pas de sleep
- **HealBolt2** : a `DontHealEnemies` et stun conditionnel sur ennemis seulement
- **WasteTime2** : donne `Charge 1`, légèrement utile
- **Flex/Flex2** : le Brace est dans `self_damage`, pas `damage_instance` ; le knockback dans `damage_instance` est utile en combat
- **ForbiddenFart** : pénalisé à -50 (pas -99999) car c'est gratuit et les dégâts AoE sont réels
- **RussianRoulette** : -99999 car risque de mort, mais si un joueur choisit cet item c'est intentionnel

### Priorité haute

1. ~~**Corriger le bug `Collarless` → `Colorless`**~~ — FAIT ✅

2. ~~**Patcher les abilities problématiques avec `ai_base_score` / `custom_additional_ai_weight`**~~ — FAIT ✅
   - Toutes les 10 classes patchées (Monk, Tank, Colorless, Thief, Mage, Necromancer, Druid, Medic, Butcher, Jester)

3. **Affiner les presets de décision par classe**
   - Étudier les capacités réelles de chaque classe (via `dataGame/abilities/`) pour ajuster les poids.
   - Exemple : le Medic avec `RangedHeal` doit favoriser `heal_ally` à grande distance → peut-être augmenter `heal_ally` encore plus.
   - Exemple : le Thief a beaucoup de capacités `debuff_enemy` → revoir `debuff_enemy` weight.

3. **Affiner les presets de mouvement**
   - Le Mage utilise AoE → il devrait se tenir à distance suffisante pour ne pas inclure ses alliés dans la zone. Envisager `preferred_distance mov+2`.
   - Le Necromancer avec `distance_to_corpse -2` : tester si cela fonctionne correctement ou crée un comportement aberrant.

### Priorité moyenne

4. **Tester le Tinkerer**
   - Le Tinkerer a une `innate_passives` native très complexe (`TinkererBasicAttackSwitching`).
   - Vérifier que `ReplaceBrain` ne casse pas le système de craft/throw.
   - Si problème : peut-être ne pas utiliser `ReplaceBrain` pour le Tinkerer, et se reposer sur le fallback.

5. **Gérer le Jester**
   - Le Jester a une palette de classes aléatoire — son comportement IA devrait peut-être être plus générique/chaotique.
   - Envisager `smart_default` ou un preset spécial "omnivore".

6. ~~**Ajouter un preset pour `Colorless`/chats sans classe**~~
   - Bug corrigé, preset `smart_colorless` actif. ✅

### Priorité basse

7. **Intégration avec les items et mutations**
   - Certains items donnent des capacités supplémentaires. Les presets actuels ne tiennent pas compte des synergies d'items.
   - Solution possible : utiliser `flat_cast_bonus` pour favoriser l'utilisation des capacités sur les attaques basiques.

8. **Modes d'IA configurables**
   - Idée : permettre via un item ou une passive de choisir entre plusieurs "modes" (agressif, défensif, support) en runtime.
   - Nécessiterait une passive custom qui appelle `ReplaceBrain` avec un preset différent.

9. **Presets pour les ennemis spéciaux**
   - Certains ennemis sont des copies de chats joueurs — ils utilisent aussi `PlayerCat`.
   - Vérifier que le mod ne rend pas les ennemis trop intelligents.

---

## Règles de développement

- **Ne jamais modifier `dataGame/`** — lecture seule, référence uniquement.
- **Toujours utiliser `.merge`** pour les patches (ne pas redéfinir entièrement les objets).
- **Vérifier les noms exacts** dans `dataGame/classes/classes.gon` et `advanced_classes.gon` avant d'écrire un patch.
- **Vérifier les champs** dans `dataGame/ai_presets/` avant d'ajouter un nouveau champ.
- **Le fallback `player_cat.gon.patch` est sacré** — ne jamais l'affaiblir. C'est la dernière ligne de défense contre le tir ami.

---

## Comment tester

1. Lancer le jeu avec le mod activé.
2. Démarrer un combat en auto-battle.
3. Vérifier :
   - [ ] Les chats ne tapent jamais leurs alliés.
   - [ ] Le Medic soigne ses alliés au lieu d'attaquer.
   - [ ] Le Fighter fonce au contact.
   - [ ] Le Hunter reste à distance.
   - [ ] Le Tank reste près de ses alliés.
   - [ ] Le Mage ne se positionne pas dans ses propres AoE.
   - [ ] Le Necromancer se dirige vers les cadavres.
   - [ ] Le Tinkerer place ses tourelles sans bloquer ses alliés.
4. Surveiller la console de logs du jeu pour des erreurs de chargement GON.
