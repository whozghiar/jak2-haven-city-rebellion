# Haven City Rebellion — Jak 2

<p align="center">
  <img src="https://img.shields.io/badge/OpenGOAL-Mod-blue.svg" alt="OpenGOAL Mod">
  <img src="https://img.shields.io/badge/Game-Jak%202-orange.svg" alt="Target Game">
  <img src="https://img.shields.io/badge/Status-Work%20in%20Progress-yellow.svg" alt="Work in Progress">
  <img src="https://img.shields.io/badge/AI--assisted-Modding-purple.svg" alt="AI Assisted">
</p>

---

> [!WARNING]
> ### ⚠️ Work in Progress — Stability Notice
> This mod is currently under **active development**. While fully playable, players and testers may encounter **occasional unexpected game crashes** (e.g. `exit status 5` / process allocation limits) due to the high density of concurrent combatants, process slot exhaustion or heap memory fatigue under sustained heavy battle, or level streaming crossfades.
> Detailed health telemetry is periodically printed to the console terminal to help monitor heap memory and active process slots.

> [!NOTE]
> This mod moved from the `jak2/features/haven-city-rebellion` branch of [whozghiar/jak-project](https://github.com/whozghiar/jak-project) to this repository. Earlier releases stay installable from the launcher catalog.

## 📖 Overview
Adds the **Blue Crimson Guard** as its own standalone entity (`crimson-blue-guard`) with high-fidelity combat AI and custom textures, alongside the **City Insurrection** mode: a full-scale territorial civil war across Haven City between Baron Praxis's loyalist forces and the rebel blue guard insurgent faction.

- **Target Game:** Jak 2
- **Repository:** [`whozghiar/jak2-mod-haven-city-rebellion`](https://github.com/whozghiar/jak2-mod-haven-city-rebellion) — Standalone blue-guard traffic + City Insurrection territorial civil war.

### Mod Family
| Repository | Description |
|---|---|
| [`whozghiar/jak2-mod-blue-krimzon-guard`](https://github.com/whozghiar/jak2-mod-blue-krimzon-guard) | Blue guards replacing red guards, identical behaviour, optional grenade launcher |
| [`whozghiar/jak2-mod-peaceful-haven-city`](https://github.com/whozghiar/jak2-mod-peaceful-haven-city) | Neutral blue patrol **squads** — formation nav, mutual defense, friendly-fire immunity |
| **`whozghiar/jak2-mod-haven-city-rebellion`** *(this mod)* | Full **territorial civil war** — district zoning, autonomous inter-faction combat, 22-guard war zones, artillery grenade launchers, alert-free zones |

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
  - **War Zone (Industrial `ctyinda/b` by default, or any mix of districts toggled in the `War zones` submenu, up to All City):** An active battlefield where opposing factions hunt and engage each other on sight.
- **Autonomous Inter-Faction Warfare:** Blue and red/yellow guards engage at long range (~150m scan) with no police pursuit or wanted level triggered against Jak.
- **Civilian Evacuation:** Civilians, civilian hovercrafts, and ambient metalheads are automatically purged from the conflict zone to dedicate memory and process slots to the firefight.
- **Dynamic Faction Balancing (70% Loyalists / 30% Insurgents):** Street-level real-time balancing ensures loyalist forces maintain tactical superiority over the rebel forces.

### 💥 War-Zone Guard Density (22 Active Combatants)
- **Two Guard Pools Mobilized:** Pool 6 (`crimson-guard-1`) at 12 guards and Pool 4 (`crimson-guard-0`) at 10. Pool 7 (`crimson-guard-2`) can never spawn in Haven City, so it stays off.
- **Dense Spacing (`inv-density-factor 1.25`):** The engine's own dense preset, spawning combatants 4x denser than vanilla (5.0) across sidewalks and streets.
- **Continuous Battlefield Reinforcement (`fast-spawn #t`):** War zone losses are replenished in real-time frame-by-frame.

### 🎯 Overhauled Ranged Arsenal & Melee Minimization
- **0% Tasers:** Melee taser/baton guards are completely disabled in Insurrection mode.
- **Grenade Launchers for Red & Yellow Guards:** Loyalists are equipped with high-explosive grenade launchers (`vehicle-grenade`) featuring parabolic ballistic trajectories alongside standard pulse rifles.
- **Minimized Melee Attempts:**
  - Guards no longer abandon shooting to perform awkward rifle-butt swings.
  - The vanilla 10-meter shooting lockout is eliminated; guards fire at any range, including point-blank.
  - Initial reaction delay reduced from 1.0–3.0s down to 0.4–1.0s for faster fire.
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
*(AI-assisted)*
