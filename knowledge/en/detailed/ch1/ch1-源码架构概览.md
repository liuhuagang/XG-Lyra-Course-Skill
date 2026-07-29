# Source Architecture Overview

## Module Structure

The Lyra project consists of two main modules:

```
Lyra/
├── Source/
│   ├── LyraGame/          # Runtime module, core game logic
│   └── LyraEditor/        # Editor module, editor tools and customizations
└── Plugins/               # Plugin directory
    ├── CommonGame/
    ├── CommonUser/
    ├── CommonLoadingScreen/
    ├── GameSettings/
    ├── GameplayMessageRouter/
    ├── ModularGameplayActors/
    ├── UIExtension/
    ├── PocketWorlds/
    ├── AsyncMixin/
    ├── ControlFlows/
    └── GameFeatures/       # Game feature plugins
        ├── ShooterCore/
        ├── ShooterMaps/
        ├── ShooterExplorer/
        ├── TopDownArena/
        └── ShooterTests/
```

## LyraGame Module Directory Structure

```
LyraGame/
├── AbilitySystem/          # GAS ability system
├── Audio/                  # Audio system
├── Camera/                 # Camera system
├── Character/              # Character system
├── Cosmetics/              # Cosmetic system
├── Equipment/              # Equipment system
├── Feedback/               # Feedback system
├── GameFeatures/           # GameFeature plugins
├── GameModes/              # Game modes
├── Hotfix/                 # Hotfix
├── Input/                  # Input system
├── Interaction/            # Interaction system
├── Inventory/              # Inventory system
├── Messages/               # Message system
├── Player/                 # Player management
├── Replays/                # Replay system
├── Settings/               # Settings system
├── System/                 # Infrastructure
├── Teams/                  # Team system
├── UI/                     # UI system
├── Weapons/                # Weapon system
└── LyraGame.Build.cs       # Module build configuration
```

## Core Design Concepts

1. **Modularity**: Features are divided into modules and plugins, supporting on-demand loading
2. **GameFeature Plugins**: UE5's new project architecture approach, where plugin content can back-reference the main project
3. **Experience-driven**: Uses ExperienceDefinition data assets to replace traditional hardcoded GameMode
4. **Message-driven**: Decoupled communication between subsystems via GameplayMessageRouter
5. **Extensibility**: Core systems define interfaces and base classes, with concrete behavior provided by plugins

## Blueprint Structure Overview

Core framework Blueprints:
- `DefaultGameData` — Hardcoded default asset loaded at startup
- `GameInstance` — Global state management, inherits from C++ class
- Game Mode base class — Base class for multi-mode gameplay rules
- `Label` assets — Define packaging modules
- `InputConfig` — Manage enhanced input mappings

Character system Blueprints:
- Skeletal mesh and animation logic separated, driven by CurveMod for animation blending
- `CatMesh` component — Dynamic cosmetic system
- Stack-based camera management (`LyraCameraMode`), not spring arm implementation
