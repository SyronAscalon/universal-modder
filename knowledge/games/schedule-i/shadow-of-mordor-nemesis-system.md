---
kind: game
title: "Shadow of Mordor Nemesis system across Cartel and Police factions in Schedule I"
game: "Schedule I"
games_also: []
game_version: "0.4.7f12 (Steam AppID 3164500, beta branch, buildid 25796331)"
platform: windows
engine: unity-il2cpp
route: managed-patch
tools: ["MelonLoader 0.7.3", "HarmonyX", "ilspycmd 11.1", ".NET SDK 10 (targeting net6.0)"]
anti_cheat: "none found; co-op is FishNet over Steam P2P. Single-player and local host only"
status: working
agents: ["Gemini CLI (gemini-3.8-flash)"]
humans: ["SyronAscalon"]
date: 2026-10-13
links: []
tags: ["nemesis-system", "il2cpp", "melonloader", "harmony", "ai-factions", "persistence", "imgui"]
---

# Shadow of Mordor Nemesis system across Cartel and Police factions in Schedule I

> A complete port of the Middle-earth: Shadow of Mordor Nemesis system into Schedule I 0.4.7f12 via MelonLoader 0.7.3 and HarmonyX.
> Both the Cartel Syndicate and Police Department maintain persistent, evolving social hierarchies with grunts promoted on player kills,
> procedural trait matrices (immunities, vulnerabilities, enrages), cheat-death mechanics with scars, a living-world turn simulation,
> and an interactive in-game IMGUI Army Screen.

## Setup
- **Game:** Steam on Windows 10/11 x64, Schedule I version `0.4.7f12` (AppID 3164500, `beta` branch, buildid `25796331`).
- **Loader:** MelonLoader `v0.7.3` Open-Beta (2026-05-14 release, installed as `version.dll` proxy).
  - *Critical note:* MelonLoader `v0.7.2` crashes during `Il2CppAssemblyGenerator` on build 25796331. Version `0.7.3` is strictly required.
- **Build environment:** .NET SDK 10 compiling for `net6.0` (`LangVersion=latest`, `AllowUnsafeBlocks=true`).
  - References `MelonLoader/net6/{MelonLoader,0Harmony,Il2CppInterop.Runtime,Il2CppInterop.Common}.dll` and `MelonLoader/Il2CppAssemblies/*.dll`.

## Route and why
We used the `managed-patch` route via MelonLoader and HarmonyX:
- The game is Unity 2022.3.62f2 IL2CPP.
- MelonLoader generates strongly typed C# interop assemblies for game systems (`Assembly-CSharp.dll`, `Il2CppScheduleOne.Core.dll`).
- Harmony hooks on `PlayerHealth`, `NPCHealth`, `CartelGoon`, `PoliceOfficer`, and `SleepController` allow seamless interception of combat, death, promotions, and time skips without altering native binaries or asset bundles.
- Persistence is completely decoupled into `UserData/Schedule1Nemesis/NemesisData.json`, preventing any chance of corrupting native save files.

## How the game works (what we had to learn)

### NPC Factions & Hierarchy Binding
- `CartelGoon` and `PoliceOfficer` derive from `Il2CppScheduleOne.NPCs.NPC`.
- Rather than subclassing IL2CPP components, a managed singleton registry maps `NPC.Pointer` (IntPtr) to active `NemesisEntity` records.
- Stat adjustments (health bonuses, scale intimidation multiplier `transform.localScale *= nemesis.ScaleMultiplier`) are applied upon NPC start.

### Combat, Damage Types & Trait Resolution
- **Player Damage:** `PlayerHealth.TakeDamage(float damage, bool flinch, bool playBloodMist)` and `PlayerHealth.Die()`.
  - Attacking NPCs are tracked during damage events. When `Die()` executes, if the killer was an existing Nemesis Captain, their Power Level surges and they gain memory logs. If killed by an unnamed grunt, the grunt is dynamically promoted into a vacant Captain slot.
- **NPC Damage:** `NPCHealth.TakeDamage(float damage, bool isLethal = true)` and `NPCHealth.Die()`.
  - Immunities (`immune_bullets`, `immune_melee`, `immune_explosions`) intercept `TakeDamage` via Harmony Prefix and return `false`, completely nullifying incoming damage.
  - Vulnerabilities multiply incoming damage (e.g. 2.0x for `weak_to_melee`).
  - Enrages (`enraged_by_guns`, `enraged_by_lowhealth`) trigger temporary speed and damage boosts.

### Cheat Death & Scars
- In `NPCHealth.Die()`, if the defeated NPC is a Nemesis, a Cheat Death roll executes (35% base, 75% for `cheat_death_master`).
- If successful:
  - The vanilla `Die()` method is canceled (`return false;`).
  - The NPC is granted temporary invincibility `_invincible = true;`, restored via `RestoreHealth()`, and deactivated or moved to simulate escape.
  - The Nemesis gains scar titles ("the Scarred", "the Undying"), mutates new traits (such as `enraged_by_lowhealth`), enters `Recovering` status for 1-2 world turns, and returns stronger.

### Living World Turn Simulation
- Advances on player death, sleep via `SleepController.StartSleep()`, or day rollover via `TimeManager.PassMinute()`.
- Wounded Nemeses heal and re-enter active rotation.
- Vacant Warchief slots are filled by the highest Power Level Captain.
- Internal Captain rivalries simulate background turf struggles (winners gain +2 PL; losers are demoted or wounded).

### In-Game UI & Cursor Handling
- An Army Screen visualizer (`F7` or `Insert`) displays the active hierarchy for Cartel and Police, Captain cards, traits, and memory bank dossiers.
- First-person games keep `Cursor.lockState = CursorLockMode.Locked; Cursor.visible = false;`. Opening an IMGUI menu requires explicitly setting `Cursor.lockState = CursorLockMode.None; Cursor.visible = true;` in `Update()`.

## Build steps
1. Install MelonLoader 0.7.3 into the Schedule I directory. Launch once to generate `MelonLoader/Il2CppAssemblies`.
2. In the mod project (`net6.0`), reference MelonLoader assemblies and Il2Cpp assemblies.
3. Build with `dotnet build -c Release`.
4. Deploy the compiled `Schedule1Nemesis.dll` into `Schedule I/Mods/`.
5. Launch the game from Steam. On first launch, the mod seeds initial hierarchies and writes `UserData/Schedule1Nemesis/NemesisData.json`.

## Verification
- **Oracle (Logs & Persistence):**
  - Verified in `MelonLoader/Latest.log` that `Schedule 1: Shadow of Mordor Nemesis System v1.0.0` initializes with zero errors and outputs `Loaded World Turn 1 with 6 Cartel and 6 Police officers.`
  - Verified `UserData/Schedule1Nemesis/NemesisData.json` is generated with 12 seeded officers, trait IDs, and memory banks.
- **Automated Tests:**
  - 5 xUnit tests in `Tests/NemesisSystemTests.cs` verifying trait manipulation, timeline ordering, relationship graphs, trait catalog validation, and JSON round-trip fidelity.
- **In-Game Verification:**
  - Game window (`MelonLoader v0.7.3 Open-Beta - Schedule I 0.4.7f12`) opens.
  - Pressing `F7` opens the full Shadow of Mordor Army Screen with responsive tabs and buttons.
- **What was not verified in extended play:**
  - Multiplayer client synchronization (built and scoped strictly for single-player / local host).

## Gotchas
1. **MelonLoader 0.7.2 crashes on Schedule I 0.4.7f12 with NullReferenceException during assembly generation.**
   - *Symptom:* `Il2CppAssemblyGenerator` outputs `Error Generating Interop Assemblies! System.NullReferenceException at Il2CppInterop.Generator.Passes.Pass11ComputeTypeSpecifics.ComputeSpecifics`. Bypassing hash checks causes `Schedule I.exe` to crash immediately with access violation `0xc0000005` in `coreclr.dll`.
   - *Cause:* MelonLoader 0.7.2's bundled Il2CppInterop cannot parse Unity 2022.3.62f2 metadata in the 0.4.7f12 build.
   - *Fix:* Upgrade to MelonLoader 0.7.3 (`MelonLoader.x64.zip`), which handles the updated metadata cleanly.

2. **`NPCHealth.Health` is read-only in 0.4.7f12.**
   - *Symptom:* Compiler error `CS0200: Property or indexer 'NPCHealth.Health' cannot be assigned to -- it is read only`.
   - *Cause:* Property setter was removed in the updated IL2CPP interop bindings.
   - *Fix:* Modify `_MaxHealth_k__BackingField` directly and call `RestoreHealth()` or `sync___set_value__currentHealth()`.

3. **`NPCHealth.npc` field is not present in 0.4.7f12 interop.**
   - *Symptom:* Compiler error `CS1061: 'NPCHealth' does not contain a definition for 'npc'`.
   - *Cause:* The back-reference field was stripped or unmapped in the dumped assembly.
   - *Fix:* Obtain the parent NPC component via `__instance.GetComponent<NPC>()`.

4. **Harmony fails to find `TimeManager.StartSleep`.**
   - *Symptom:* `Failed to HarmonyInit PatchAll: Undefined target method for patch method TimeManager_StartSleep_Patch`.
   - *Cause:* Sleep logic was refactored into `Il2CppScheduleOne.GameTime.SleepController`.
   - *Fix:* Patch `SleepController.StartSleep()` instead.

5. **First-person cursor lock prevents clicking IMGUI menu buttons.**
   - *Symptom:* The F7 Army Screen appears, but the mouse cannot move or click any buttons.
   - *Cause:* The first-person controller locks and hides the cursor every frame.
   - *Fix:* In `Update()`, when the menu is open, enforce `Cursor.lockState = CursorLockMode.None; Cursor.visible = true;`.

## Cost and time
One continuous session covering recon, MelonLoader 0.7.3 upgrade, full Nemesis architecture implementation, unit test verification, and live game testing.
