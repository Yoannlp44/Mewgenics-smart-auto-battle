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
│   │   ├── monk_abilities.gon.patch       Porcupine (tile_close_to_enemy + ai_base_score -5), Meditate (−99999 sleep 8t)
│   │   ├── tank_abilities.gon.patch       BarbedWire (tile_close_to_enemy + ai_base_score -5)
│   │   ├── colorless_abilities.gon.patch  Rest (must_heal_most_missing_health), Brace (-5), Block (-3)
│   │   ├── fighter_abilities.gon.patch    ChaosRampage/Enrage/Bloodzerk/Stoopzerk/ThinkTooHard2 (−99999), Counter (tile_close_to_enemy−5), Berserk/Juiced (−5), Exert (−10), Hurl (−10)
│   │   ├── hunter_abilities.gon.patch     ChaosShot/Pheromones (−99999), StakeOut (−20), SoothingShot (−10), Extend (tile_close_to_enemy−5)
│   │   ├── psychic_abilities.gon.patch    RealityScramble/ChaosSwap2/ExtraTurnQuestion/MindCrack/Glare/ThinkDeep (−99999), Pass (−20), BlindingFlash (−10), Supernova/MassManaLeech/MindBlast (−50)
│   │   ├── thief_abilities.gon.patch      CoinToss (−10), Cheat (−15), Pierce (tile−5), WindUp (−5), Jitter (−10), Outskirts2 (−5)
│   │   ├── mage_abilities.gon.patch       Absorb (must_heal), ManaMeld (−5), DealWithTheDevil/Corrupt/BlackMagic (−20), ChainLightning (−20), ForbiddenFlame (−10), ForbiddenFlood/Telefrag/Divide/Magnify (−5), ForbiddenFrost (−10)
│   │   ├── necromancer_abilities.gon.patch Flatline/SlitWrists/Seppuku/AbsorbSoul (−99999), Pestilence (−50), DemonicPact/ForbiddenFamine/Hush/Shriek (−20), BloodGeyser (−20), SoulTransfer/DonateBlood (−10), GigaDrain/TradeLife/LeechSwarm/BloodRain/WeAreOne (−5)
│   │   ├── druid_abilities.gon.patch      ChaChaSlide (−99999 friendly-fire), HardenSkin (tile_close_to_enemy + ai_base_score -5)
│   │   ├── medic_abilities.gon.patch      BornAgain (−99999 sleep 4t), Zealot (tile_close_to_enemy) [DivineShield persiste, pas 1-turn spam]
│   │   ├── butcher_abilities.gon.patch    TaintedOffering/Tryptophan (−99999), SkullBash/Consume/MyTurn/Butcher (−20), SelfMutilate/Fartoom/LightenTheLoad (−5/−10), Binge/TaintedOffering2/Tromp/Chonkwalk/Reflux/Cough/Track (−5)
│   │   ├── jester_abilities.gon.patch     Bump (−99999 displaces allies), PowerUp (−10)
│   │   └── tinkerer_abilities.gon.patch   Shockwave (−30), ShoddyJetpack/EjectButton (−10), FreshOffTheForge/UnreliableShield (−5)
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
danger_avoidance        (float — évite les tuiles dangereuses : 0.5 léger, 3-5 normal, 20+ extrême)
tall_grass              (float — attire vers TallGrassTile)
lava                    (float — attire vers LavaTile, NE PAS utiliser pour les joueurs)
count_nomove_in_eval    (bool)
consider_aggro_target_enemy (bool)
exclude_characters_tagged <tag>
```

### Tuiles du jeu — catalogue complet (tiles.gon)
```
ID  Nom               Classe moteur
 0  Empty             —
 1  Water             WaterTile
 2  Grass             GrassTile / TallGrassTile (15%) / BlankTile (5%)
 3  Tall Grass        TallGrassTile (80%) / GrassTile (15%) / BlankTile (5%)
 4  Fire              FireTile
 5  Ice               IceTile
 6  Lava              LavaTile
 7  Metal             MetalTile
 8  Rock              RockTile
 9  Creep             CreepTile
10  Oil               OilTile
11  Toxic Sludge      ToxicTile
12  Shadow            ShadowTile
13  Glass Shards      GlassTile
14  Snow              SnowTile
15–18 Water Current   WaterTile_Current (N/S/E/W)
19  Dirt              DirtTile
20  Stalagmites       StalagmiteTile
21  Road              RoadTile
22  Brambles          BrambleTile
23  Flowers           FlowerTile
24  Tall Flowers      TallFlowerTile
25  Supercooled Water SupercooledWater
26  Glitch            GlitchTile
```

### Dangers environnementaux — ce qui est possible via les presets
- **`danger_avoidance <N>`** : couvre toutes les tuiles que le moteur considère dangereuses (lava, fire, toxic, verre, brambles...) et les OBJETS dangereux (SpikyRock → Thorns 5). Seul levier disponible pour éviter les pièges.
- **`tall_grass <N>`** : attire vers l'herbe haute (camouflage, utile Thief/Hunter)
- **`distance_to_water <N>`** : attire/repousse des tuiles d'eau

### Ce qui N'est PAS possible via les presets de mouvement
- Éviter spécifiquement les harpons (HarpoonTrap = objet, pas tuile — géré par le moteur)
- Éviter individuellement chaque type de tuile dangereuse (glace, toxic, feu...)
- Le champ `danger_avoidance` est un budget global — pas de granularité par type de danger

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

### NOTE -- Commentaires `//` dans les patches GON
- Les commentaires `//` **sont supportes** par le parseur GON, y compris en inline (ex: `damage_enemy 2  // commentaire`).
- Les fichiers vanilla du jeu en font usage massivement.
- **Pas de restriction sur les commentaires** -- seul le charset ASCII est obligatoire.

### ~~BUG CRITIQUE~~ CORRIGE -- Caracteres non-ASCII dans les fichiers .patch
- **Corrige le 2026-03-07.**
- Cause racine de l'erreur GON `"missing a '}' somewhere"` au lancement du jeu.
- Les fichiers vanilla GON sont 100% ASCII. Le parseur GON ne supporte pas UTF-8, meme dans les commentaires.
- Nos fichiers `.patch` contenaient des caracteres UTF-8 : tirets em (`--`), accents francais (`e`, `e`, `a`...), apostrophes typographiques.
- **Tous les fichiers .patch ont ete reedits en ASCII pur** (18 fichiers verifies a 0 octet non-ASCII). ✅
- **Regle** : uniquement des caracteres ASCII dans tous les fichiers `.gon.patch`.

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

### Corrections appliquées le 2026-03-07 (session 4) — danger_avoidance et terrain
Ajout de `danger_avoidance` à tous les presets de mouvement smart_* + `tall_grass` pour Thief, Druid, Hunter :

| Preset | `danger_avoidance` | Notes |
|---|---|---|
| smart_fighter_move | 1 | Fonce, accepte le risque |
| smart_hunter_move | 3 | Distance — évite les tuiles dangereuses + `tall_grass 1` |
| smart_tank_move | 0.5 | Tank — encaisse, évite juste la lave |
| smart_mage_move | 5 | Fragile — évite activement |
| smart_cleric_move | 5 | Doit rester en vie pour soigner |
| smart_thief_move | 3 | Mobile + `tall_grass 2` (embuscades) |
| smart_butcher_move | 1 | Enragé — accepte le risque |
| smart_psychic_move | 4 | Fragile, soutien |
| smart_colorless_move | 2 | Générique |
| smart_jester_move | 2 | Chaotique, limité |
| smart_druid_move | 3 | + `tall_grass 1` (affinité naturelle) |
| smart_monk_move | 1 | Fonce au contact |
| smart_necromancer_move | 5 | Fragile — ne doit pas marcher dans la lave |
| smart_tinkerer_move | 3 | Derrière les constructions |
- **Monk** : `buff_self` réduit de 4 → 1 (évite `Meditate` Sleep 8 tours) ✅
- **Tank** : `buff_self` et `buff_ally` réduits de 2 → 1 (évite Thorns/BarbedWire inutiles) ✅
- **Mage** : ajout de `consider_aoe true` ✅
- **Druid** : ajout de `consider_aoe true` (ChaChaSlide, DeathMetal) ✅
- **Thief** : `spend_mana_scale` réduit de .99 → .5 (évite spam CoinToss / boucle ReloadOnGainCoins) ✅

### Corrections appliquées le 2026-03-07 (session 2) — patches abilities
- **Monk** : `Porcupine`/`Porcupine2` → `tile_close_to_enemy` ; `Meditate`/`Meditate2` → `ai_base_score -99999` ✅
- **Tank** : `BarbedWire`/`BarbedWire2` → `tile_close_to_enemy` ✅

### Corrections appliquées (session 5) — anti-spam 1-turn buffs
Ajout de `ai_base_score -5` sur toutes les abilities avec `turns 1` + `expires_on_appliers_turn true` qui avaient seulement `tile_close_to_enemy` :

| Fichier | Abilities | Avant | Après |
|---|---|---|---|
| `tank_abilities.gon.patch` | BarbedWire/2 | `tile_close_to_enemy` | + `ai_base_score -5` ✅ |
| `monk_abilities.gon.patch` | Porcupine/2 | `tile_close_to_enemy` | + `ai_base_score -5` ✅ |
| `colorless_abilities.gon.patch` | Brace/2 | `tile_close_to_enemy` | + `ai_base_score -5` ✅ |
| `colorless_abilities.gon.patch` | Block/2 | `tile_close_to_enemy` | + `ai_base_score -3` ✅ (Shield persiste, pénalité plus légère) |
| `druid_abilities.gon.patch` | HardenSkin/2 | `tile_close_to_enemy` | + `ai_base_score -5` ✅ |

**Zealot (Medic)** : DivineShield persiste jusqu'à absorption (pas un buff 1-turn) → pas de spam possible → `tile_close_to_enemy` seul est correct ✅

### Corrections appliquées (session 6) — préférer l'offensif sur les buffs situationnels
**Problème confirmé :** Les buffs 1-tour sont cumulables (stacks), donc `avoid_redundant_debuffs` ne fonctionne pas — il faut `ai_base_score` négatif plus fort. `custom_additional_ai_weight` est un champ unique (non cumulable).

**Double approche :**

**1. `ai_base_score` relevé de -5 à -15 sur tous les buffs 1-tour stackables :**
| Fichier | Abilities | Score |
|---|---|---|
| `tank_abilities.gon.patch` | BarbedWire/2 | -15 ✅ |
| `monk_abilities.gon.patch` | Porcupine/2 | -15 ✅ |
| `colorless_abilities.gon.patch` | Brace/2 | -15 ✅ |
| `colorless_abilities.gon.patch` | Block/2 | -5 (Shield persiste) ✅ |
| `druid_abilities.gon.patch` | HardenSkin/2 | -15 ✅ |

**2. `buff_self` réduit à 0.3 dans les presets concernés :**
- `smart_tank` : `buff_self 1` → `0.3` + `damage_enemy 1→2` + `kill_enemy 5→8` (Tank trop passif)
- `smart_monk` : `buff_self 1` → `0.3`
- `smart_druid` : `buff_self 1` → `0.3`- **Colorless** : `Rest`/`Rest2` → `must_heal_most_missing_health` ; `Brace`/`Brace2` → `tile_close_to_enemy` ✅
- **Thief** : `CoinToss`/`CoinToss2` → `ai_base_score -10` ✅
- **Mage** : `Absorb`/`Absorb2` → `must_heal_most_missing_health` ; `ManaMeld`/`ManaMeld2` → `ai_base_score -5` ✅
- **Necromancer** : `Flatline`/`Flatline2` → `ai_base_score -99999` ; `BloodGeyser`/`BloodGeyser2` → `-20` ; `GigaDrain`/`GigaDrain2` → `-5` ✅
- **Druid** : `ChaChaSlide`/`ChaChaSlide2` → `ai_base_score -99999` ; `HardenSkin`/`HardenSkin2` → `tile_close_to_enemy` ✅
- **Medic** : `BornAgain` → `ai_base_score -99999` (sleep 4t) ; `Zealot`/`Zealot2` → `tile_close_to_enemy` ✅
- **Butcher** : `SelfMutilate`/`SelfMutilate2` → `-5` ; `Fartoom`/`Fartoom2` → `-10` ; `LightenTheLoad`/`LightenTheLoad2` → `-10` ✅
- **Jester** : `Bump`/`Bump2` → `ai_base_score -99999` ✅

### Notes techniques — patches abilities
- **`ChaosTeleport` (Mage)** : template `teleport`, pas de `damage_instance` → impossible à patcher via ce mécanisme
- **`ChaosSwap` (Psychic)** : template `swap`, pas de `damage_instance` → impossible à patcher via ce mécanisme. Seul `ChaosSwap2` est patchable (il a un `damage_instance` propre).
- **`BornAgain2` (Medic)** : version améliorée SANS le `self_damage Sleep` — ne pas patcher, c'est une bonne ability
- Les patches sont dans `data/abilities/<classe>_abilities.gon.patch`
- Format : `AbilityName.merge { damage_instance { ai_base_score N } }` ou `custom_additional_ai_weight <keyword>`

### Corrections appliquées (session 8) -- bug custom_additional_ai_weight
**Cause racine confirmee :** `tile_close_to_enemy` sans `custom_additional_ai_weight` devant = erreur GON au lancement.

Fichiers corrigés :
- `thief_abilities.gon.patch` : `Pierce`/`Pierce2` -- ajout de `custom_additional_ai_weight` devant `tile_close_to_enemy` ✅
- `fighter_abilities.gon.patch` : `Counter`/`Counter2` -- ajout de `custom_additional_ai_weight` devant `tile_close_to_enemy` ✅

Tous les autres fichiers utilisant `tile_close_to_enemy` avaient deja le prefixe correct (hunter, medic, druid, tank, monk, colorless).

### Corrections appliquées (session 7) — patches abilities Fighter, Hunter, Psychic + complétion de toutes les classes

**Nouveaux fichiers créés :**
- `fighter_abilities.gon.patch` — ChaosRampage, Enrage, Bloodzerk, Stoopzerk, ThinkTooHard2 → `-99999` ; Counter → `tile_close_to_enemy −5` ; Berserk, Juiced → `-5` ; Exert, Hurl → `-10`
- `hunter_abilities.gon.patch` — ChaosShot, Pheromones → `-99999` ; StakeOut → `-20` ; SoothingShot → `-10` ; Extend → `tile_close_to_enemy −5`
- `psychic_abilities.gon.patch` — RealityScramble, ChaosSwap2, ExtraTurnQuestion, MindCrack, Glare, ThinkDeep → `-99999` ; Pass → `-20` ; BlindingFlash → `-10` ; MindBlast, Supernova, MassManaLeech → `-50`
- `tinkerer_abilities.gon.patch` — Shockwave → `-30` ; ShoddyJetpack, EjectButton → `-10` ; FreshOffTheForge, UnreliableShield → `-5`

**Fichiers existants complétés :**
- `thief_abilities.gon.patch` — ajout Cheat (`−15`), Pierce (`tile_close_to_enemy −5`), WindUp (`−5`), Jitter (`−10`), Outskirts2 (`−5`)
- `mage_abilities.gon.patch` — ajout DealWithTheDevil, Corrupt, BlackMagic, ChainLightning (`−20`) ; ForbiddenFlame, ForbiddenFrost (`−10`) ; ForbiddenFlood, Telefrag, Divide, Magnify (`−5`)
- `necromancer_abilities.gon.patch` — ajout SlitWrists, Seppuku, AbsorbSoul (`−99999`) ; Pestilence (`−50`) ; DemonicPact, ForbiddenFamine, Hush, Shriek (`−20`) ; SoulTransfer, DonateBlood (`−10`) ; TradeLife, LeechSwarm, BloodRain, WeAreOne (`−5`)
- `butcher_abilities.gon.patch` — ajout TaintedOffering v1, Tryptophan v1 (`−99999`) ; SkullBash v1, Consume, MyTurn, Butcher, Tryptophan2 (`−20`) ; SkullBash2 (`−10`) ; TaintedOffering2, Tromp, Chonkwalk, Reflux, Cough, Track, Binge (`−5`)
- `jester_abilities.gon.patch` — ajout PowerUp (`−10`)

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
   - Toutes les 14 classes patchées (Fighter, Hunter, Psychic, Monk, Tank, Colorless, Thief, Mage, Necromancer, Druid, Medic, Butcher, Jester, Tinkerer)

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

- **Commentaires `//` valides partout** -- y compris inline (`valeur // commentaire`). Les fichiers vanilla du jeu en font usage massivement.
- **ASCII pur obligatoire** dans tous les fichiers `.gon.patch` -- le parseur GON ne supporte pas UTF-8, meme dans les commentaires. Pas d'accents, pas de tirets em, pas de guillemets typographiques. Remplacer : `e` accent -> `e`, `--` (em dash) -> `--`, apostrophes typographiques -> droites, etc.
- **`custom_additional_ai_weight` obligatoire devant `tile_close_to_enemy`** -- ecrire `tile_close_to_enemy` seul dans un `damage_instance` cause une erreur GON au lancement. Format correct :
  `custom_additional_ai_weight tile_close_to_enemy`
- **Ne jamais modifier `dataGame/`** -- lecture seule, reference uniquement.
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
