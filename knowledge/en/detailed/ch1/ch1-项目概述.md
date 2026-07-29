# Project Overview

## What is Lyra

Lyra is a modular game sample project developed by Epic Games that ships with UE5 updates. Its purpose is not to provide a complete game, but to demonstrate how various UE5 systems can be integrated, serving as a starting point for rapidly developing specific types of games.

Lyra uses a modular architecture where core systems are separated from Gameplay feature plugins, and different game modes are implemented through plugins.

## Features

- Modular architecture with core systems and plugins, updated regularly with UE5
- Cross-platform compatibility and extensibility, supporting multiplayer and cross-platform features
- Three game modes: Elimination, Control, Exploder party game
- Custom Gameplay Ability System (GAS)
- Niagara particle effects
- UMG widget-based UI
- Hand-crafted content: movement animations, assets, sounds, weapon systems, etc.

## Game Modes and Maps

| Game Mode | Description | Map Path |
|----------|------|---------|
| Control | Protect control points with teammates to increase score | /ShooterMaps/Maps/L_Convolution_Blockout |
| Elimination | Team deathmatch, eliminate enemies to earn enough points | /ShooterMaps/Maps/L_Expanse |
| Front End | Main menu interface | /Game/System/FrontEnd/Maps/L_LyraFrontEnd |
| Default Map | Test level for ShooterCore plugin features | /Game/System/DefaultEditorMap/L_DefaultEditorOverview |
| Shooting Gym | Shooting test | /ShooterCore/Maps/L_ShooterGym |
| Exploder | Top-down party game | /TopDownArena/Maps/L_TopDownArenaGym |

## Core Systems and Plugins

Lyra's Gameplay feature plugin system:

| Plugin | Description |
|------|------|
| Lyra Example Content | Shared materials, meshes, and other base assets |
| Shooter Core Content | Shooter game core elements: Gameplay logic, Gameplay abilities, AI, weapons, UI |
| ShooterMaps Content | Shooter game maps (open areas, winding areas) |
| TopDownArena Content | Top-down party game content |

The Experience system replaces traditional GameMode, dynamically loading gameplay modules through `ULyraExperienceDefinition` data assets. Each game mode independently derives from the same core base class, adjusting only win conditions and scoring methods.

## Download and Installation

1. Search for "Lyra" in the Epic Games Store, find "Lyra Starter Game"
2. Add to Library, download via the Epic Games Launcher
3. Install a compatible engine version (e.g., 5.6.0), check debug symbols
4. Create the project and name it "Lyra"
