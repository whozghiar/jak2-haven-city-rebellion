# Crimson Blue Guard & City Insurrection — Jak 2

<p align="center">
  <img src="https://img.shields.io/badge/OpenGOAL-Mod-blue.svg" alt="OpenGOAL Mod">
  <img src="https://img.shields.io/badge/Game-Jak%202-orange.svg" alt="Target Game">
  <img src="https://img.shields.io/badge/Status-Work%20in%20Progress-yellow.svg" alt="Work in Progress">
  <img src="https://img.shields.io/badge/AI--assisted-Modding-purple.svg" alt="AI Assisted">
</p>

<p align="center">
  <a href="#-english-version"><b>🇬🇧 English Version</b></a> &nbsp;•&nbsp; <a href="#-version-française"><b>🇫🇷 Version Française</b></a>
</p>

---

# 🇬🇧 English Version

> [!WARNING]
> ### ⚠️ Work in Progress — Stability Notice
> This mod is currently under **active development**. While fully playable, players and testers may encounter **occasional unexpected game crashes** (e.g. `exit status 5` / process allocation limits) due to the high density of concurrent combatants, process slot exhaustion under sustained heavy battle, or level streaming crossfades.
> Detailed health telemetry is periodically printed to the console terminal to help monitor heap memory and active process slots.

> [!NOTE]
> This mod moved from the `jak2/features/haven-city-rebellion` branch of [whozghiar/jak-project](https://github.com/whozghiar/jak-project) to this repository. Earlier releases stay installable from the launcher catalog.

## 📖 Overview
Adds the **Blue Crimson Guard** as its own standalone entity (`crimson-blue-guard`) with high-fidelity combat AI and custom textures, alongside the **City Insurrection** mode: a full-scale territorial civil war across Haven City between Baron Praxis's loyalist forces and the rebel blue guard insurgent faction.

- **Target Game:** Jak 2
- **Repository:** [`whozghiar/jak2-mod-haven-city-rebellion`](https://github.com/whozghiar/jak2-mod-haven-city-rebellion) — Standalone blue-guard traffic + City Insurrection territorial civil war.

### Branch Family
| Branch | Description |
|---|---|
| `jak2/features/blueguard-traffic` | Base: `crimson-blue-guard` entity, faithful combat AI, ambient city-traffic spawning, modular hook layer |
| `jak2/features/city-peaceful` | Neutral blue patrol **squads** — formation nav, mutual defense, friendly-fire immunity |
| **`jak2/features/city-insurrection`** *(this branch)* | Full **territorial civil war** — district zoning, autonomous inter-faction combat, 60-guard density, artillery grenade launchers, alert-free zones |
| `jak2/features/blueguard` | Integration of both modes (mutually exclusive at runtime via debug menu) |

---

## ✨ Key Features

### 🔵 Standalone Custom Entity (`crimson-blue-guard`)
- **Native GOAL Type:** A standalone subclass of `crimson-guard` with its own mesh and textures, not a simple global texture swap. Standard red and yellow guards spawn alongside.
- **Visual & Audio Fidelity:** Preserves 100% of authentic animations, voice lines, sound effects, collision, and death effects (native purple particle disintegration and knockdown physics).
- **Independent Faction Logic:** Passive towards Jak by default; does not join general police alerts against him. Defends itself without raising the city-wide alarm.

### ⚔️ City Insurrection Mode

**The mod ships OFF.** It is enabled in-game with **L3 + SELECT** under `Mods ▸ crimson-blueguard-insurrection ▸ Enable` (works in retail boot, no debug mode required).
With it **off, Haven City is byte-for-byte stock Jak 2** — no blue guards, retail guard density,
retail alerts and guard combat. The `War zones` submenu (district bitmask, default Industrial) and
`War music` picker sit in the same submenu. Toggling the mod re-rolls / purges the city traffic on
the spot; the choice persists across level reloads.

When enabled:
- **Territorial District Zoning:**
  - **Slums (`ctysluma/b/c`):** Insurgent stronghold. 100% blue rebel guards, no police gunships, alarm-free haven.
  - **Loyalist Districts:** Baron Praxis control. 100% red and yellow loyalist police with vanilla enforcement.
  - **War Zone (Industrial `ctyinda/b` by default, or selectable via Debug Menu / All City):** An active battlefield where opposing factions hunt and engage each other on sight.
- **Autonomous Inter-Faction Warfare:** Blue and red/yellow guards engage at long range (~150m scan) with no police pursuit or wanted level triggered against Jak.
- **Civilian Evacuation:** Civilians, civilian hovercrafts, and ambient metalheads are automatically purged from the conflict zone to dedicate memory and process slots to the firefight.
- **Dynamic Faction Balancing (70% Loyalists / 30% Insurgents):** Street-level real-time balancing ensures loyalist forces maintain tactical superiority over the rebel forces.

### 💥 Maximum Guard Density (60 Active Combatants)
- **All Three Guard Pools Mobilized:** Exploits the OpenGOAL traffic engine architecture to its limit by mobilizing Pool 4 (`crimson-guard-0`), Pool 6 (`crimson-guard-1`), and Pool 7 (`crimson-guard-2`) at 20 guards each.
- **Ultra-Dense Spacing (`inv-density-factor 0.1`):** Spawns combatants 50x denser than vanilla across sidewalks and streets.
- **Continuous Battlefield Reinforcement (`fast-spawn #t`):** War zone losses are replenished in real-time frame-by-frame.

### 🎯 Overhauled Ranged Arsenal & Melee Minimization
- **0% Tasers:** Melee taser/baton guards are completely disabled in Insurrection mode.
- **Grenade Launchers for Red & Yellow Guards:** Loyalists are equipped with high-explosive grenade launchers (`vehicle-grenade`) featuring parabolic ballistic trajectories alongside standard pulse rifles.
- **Minimized Melee Attempts:**
  - Guards no longer abandon shooting to perform awkward rifle-butt swings.
  - The vanilla 10-meter shooting lockout is eliminated; guards fire at any range, including point-blank.
  - Initial reaction delay reduced from 1.0–3.0s down to 0.2–0.5s for immediate fire.
  - Standoff flanking positioning: guards maintain an arc of ~6.5m (rifle) to ~9m (grenade launcher).
  - Guard bumping collisions in dense streets no longer trigger melee states.

### 📊 Real-Time Terminal Telemetry Logs
- Health metrics logged to the terminal every 5 seconds:
  `[INS-METRICS] Heap: ... | Slots: ... | Combatants (act/tot): ... | P4(...) P6(...) P7(...) | Zone: conflict`

---

## 🎥 Demonstration Video

[![Demonstration Video](https://img.youtube.com/vi/9zsszh1OukM/maxresdefault.jpg)](https://youtu.be/9zsszh1OukM)

▶️ **[Watch the demonstration video on YouTube](https://youtu.be/9zsszh1OukM)**

---

## 🚀 Step-by-Step Guide to Run the Mod

### 1. Select the Active Game
```bash
task set-game-jak2
```

### 2. Binary Compilation
Required once if C++ engine tools have changed:
```bash
task build-release-game
```

### 3. Asset Extraction
Required once to extract the custom model and textures into `GAME.fr3`:
```bash
task extract
```

### 4. Launch the Game
```bash
task boot-game
```
*(Or launch via the OpenGOAL REPL using `task repl`, then compile and run with `(mi)` and `(boot-game)`).*

---

## 📖 Technical Documentation
For complete technical notes, engine modifications, and architecture:
- 📄 [`docs/modding/current_mod/blue_guard_reskin_readme.md`](docs/modding/current_mod/blue_guard_reskin_readme.md)

---

# 🇫🇷 Version Française

> [!WARNING]
> ### ⚠️ En Cours de Développement — Avis de Stabilité
> Ce mod est actuellement en **cours de développement actif**. Bien que pleinement fonctionnel, les joueurs et testeurs peuvent faire face à des **crashs inopinés** (ex. `exit status 5` / saturation de processus) en raison de la très forte densité d'entités, de la fatigue de la mémoire de tas (heap) lors de combats intenses prolongés ou des transitions rapides de quartiers.
> Des métriques de santé sont affichées régulièrement dans les logs du terminal pour surveiller l'état de la mémoire et des slots disponibles.

## 📖 Présentation du Mod
Ce mod introduit le **Garde Crimson Bleu** en tant qu'entité autonome (`crimson-blue-guard`) dotée d'une IA de combat avancée et de textures dédiées, ainsi que le mode **City Insurrection** : une guerre civile territoriale à grande échelle dans Haven City opposant les forces loyalistes du Baron Praxis à l'insurrection des gardes bleus.

- **Jeu Ciblé :** Jak 2
- **Dépôt :** [`whozghiar/jak2-mod-haven-city-rebellion`](https://github.com/whozghiar/jak2-mod-haven-city-rebellion) — Garde bleu autonome + mode guerre civile City Insurrection.

### Famille de Branches
| Branche | Description |
|---|---|
| `jak2/features/blueguard-traffic` | Base : entité `crimson-blue-guard`, IA de combat fidèle, spawn dans le trafic ambiant, hooks modulaires |
| `jak2/features/city-peaceful` | **Escouades** de patrouille bleues neutres — formation, défense mutuelle, immunité aux tirs alliés |
| **`jak2/features/city-insurrection`** *(cette branche)* | **Guerre civile territoriale** — zonage par quartier, affrontements autonomes, densité de 60 gardes, tirs d'artillerie au lance-grenades |
| `jak2/features/blueguard` | Intégration des deux modes (mutuellement exclusifs au runtime via le menu debug) |

---

## ✨ Fonctionnalités Clés

### 🔵 Nouvelle Entité Autonome (`crimson-blue-guard`)
- **Type GOAL Dédié :** Vrai sous-type de `crimson-guard` avec son propre maillage 3D et ses textures personnalisées (pas un simple swap global). Les gardes rouges et jaunes continuent d'apparaître normalement.
- **Fidélité Totale :** Conserve l'intégralité des animations, voix, bruitages, collisions et effets de mort (désintégration violette et chutes après projection).
- **Faction Indépendante :** Neutre envers Jak par défaut ; ne se joint pas aux alertes policières de la ville. Se défend si attaqué directement sans déclencher d'alarme générale.

### ⚔️ Mode City Insurrection (Guerre Civile)

**Le mod est livré DÉSACTIVÉ.** Il s'active en jeu avec **L3 + SELECT** dans `Mods ▸ crimson-blueguard-insurrection ▸ Enable` (fonctionne en boot normal, sans mode debug).
Désactivé, **Abriville est identique au Jak 2 d'origine** — aucun garde bleu, densité et alertes
d'origine, combats de gardes d'origine. Les sous-menus `War zones` (masque de quartiers, Industriel
par défaut) et `War music` sont dans le même sous-menu. La bascule re-tire / purge le trafic
immédiatement ; le choix persiste au rechargement des niveaux.

Une fois activé :
- **Zonage Territorial des Quartiers :**
  - **Slums / Bas-fonds (`ctysluma/b/c`) :** Bastion rebelle. 100% de gardes bleus, aucun vaisseau de police, zone refuge sans alerte.
  - **Quartiers Loyalistes :** Contrôle total de Praxis. 100% de gardes rouges et jaunes avec maintien de l'ordre d'origine.
  - **Zone de Conflit (Zone Industrielle `ctyinda/b` par défaut, sélectionnable dans le menu ou All City) :** Champ de bataille urbain où les factions s'affrontent à vue.
- **Combats Inter-Factions Autonomes :** Les gardes bleus et rouges/jaunes se repèrent à longue distance (~150m) et combattent sans impacter le niveau de recherche de Jak.
- **Évacuation des Civils :** Civils, véhicules civils et têtes-de-métal sont purgés de la zone de guerre afin de dédier les ressources mémoire aux combattants.
- **Équilibrage Dynamique (70% Loyalistes / 30% Insurgés) :** Régulation en direct dans la rue garantissant la supériorité numérique des forces loyalistes.

### 💥 Densité Maximale des Gardes (60 Combattants Actifs)
- **Mobilisation des 3 Pools du Moteur :** Exploite la limite architecturale du moteur de trafic OpenGOAL en mobilisant les Pools 4, 6 et 7 à 20 gardes chacun (60 gardes simultanés).
- **Espacement Ultra-Serré (`inv-density-factor 0.1`) :** Les gardes apparaissent 50 fois plus rapprochés le long des rues.
- **Réapprovisionnement Continu (`fast-spawn #t`) :** Les pertes sont comblées instantanément à chaque image.

### 🎯 Arsenal Rénové & Minimisation du Corps à Corps
- **0% de Tasers :** Les armes de corps à corps (bâtons/tasers) sont totalement supprimées en mode Insurrection.
- **Lance-Grenades pour les Gardes Rouges et Jaunes :** Tir de projectiles explosifs (`vehicle-grenade`) en cloche balistique parabolique et tirs au fusil d'assaut.
- **Minimisation des Tentatives de Mêlée :**
  - Fin des coups de crosse intempestifs qui interrompent les tirs.
  - Suppression de l'interdiction de tir sous 10 mètres du code vanilla (tirs autorisés à bout portant).
  - Réaction quasi-immédiate (0,2 à 0,5 s au lieu de 1 à 3 s).
  - Maintien d'une distance tactique en arc de cercle (~6,5 m au fusil, ~9 m au lance-grenades).
  - Les bousculades entre gardes dans les foules denses ne déclenchent plus de coups de crosse dans le vide.

### 📊 Télémétrie en Direct (Logs Terminal)
- Métriques affichées toutes les 5 secondes dans la console :
  `[INS-METRICS] Heap: ... | Slots: ... | Combatants (act/tot): ... | P4(...) P6(...) P7(...) | Zone: conflict`

---

## 🎥 Démonstration Vidéo

[![Démonstration Vidéo](https://img.youtube.com/vi/9zsszh1OukM/maxresdefault.jpg)](https://youtu.be/9zsszh1OukM)

▶️ **[Regarder la vidéo de démonstration sur YouTube](https://youtu.be/9zsszh1OukM)**

---

## 🚀 Guide Pas à Pas pour Lancer le Mod

### 1. Sélectionner le Jeu Actif
```bash
task set-game-jak2
```

### 2. Compilation des Binaires
```bash
task build-release-game
```

### 3. Extraction des Données (Assets)
```bash
task extract
```

### 4. Lancer le Jeu
```bash
task boot-game
```
*(Ou via le REPL OpenGOAL avec `task repl`, puis `(mi)` et `(boot-game)`).*

---

## 📖 Documentation Technique
Pour l'audit technique complet, l'architecture et les modifications apportées au moteur :
- 📄 [`docs/modding/current_mod/blue_guard_reskin_readme.md`](docs/modding/current_mod/blue_guard_reskin_readme.md)

---
*(AI-assisted)*
