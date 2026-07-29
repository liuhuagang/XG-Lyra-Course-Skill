# HUD Architecture and Layout System

## Overview

Lyra's HUD system is built on top of the CommonUI plugin's `UPrimaryGameLayout`, using a Layer Stack architecture to manage UI content with different priorities and purposes. The HUD itself does not inherit from the traditional `AHUD` but is entirely widget-based.

## Core Classes

### ULyraHUDLayout

- **Header**: [LyraHUDLayout.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/UI/LyraHUDLayout.h)
- **Inheritance**: `UCommonActivatableWidget` → `ULyraActivatableWidget` → `ULyraHUDLayout`
- **Responsibility**: Root HUD Widget, manages all HUD-level child Widgets

### UPrimaryGameLayout

- **Header**: [PrimaryGameLayout.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Plugins/CommonGame/Source/Public/PrimaryGameLayout.h)
- **Inheritance**: `UCommonUserWidget` → `UPrimaryGameLayout`
- **Responsibility**: UI layer stack container, manages Widget stacking across different layers

## Architecture Design

```
ULyraHUDLayout
  └── UPrimaryGameLayout
        ├── Game Layer (0)     — In-game HUD elements
        ├── Game Menu Layer (1) — In-game menus
        ├── Menu Layer (2)     — Main menu UI
        └── Modal Layer (3)    — Modal dialogs
```

### Layer Stack Management

- Each layer independently manages its Widget stack
- Higher layers automatically receive focus priority
- PushContentToLayer pushes a Widget into a specified layer
- Supports async loading (via `UAsyncAction_PushContentToLayerForPlayer`)

### HUD Initialization Flow

1. `ULyraHUDLayout` is created as the root HUD Widget
2. It holds a `UPrimaryGameLayout` as the layer stack container
3. Feature modules add Widgets to corresponding layers via `GameFeatureAction` when the feature activates

## Key API

- `PushContentToLayerForPlayer`: Push a specified Widget into a target layer
- `FindWidgetInLayer`: Find an existing Widget in a specified layer
- `RemoveWidgetFromLayer`: Remove a Widget from a layer

## Related Files

- [LyraHUDLayout.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/UI/LyraHUDLayout.h)
- [LyraHUD.cpp](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/UI/LyraHUD.cpp)
- [PrimaryGameLayout.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Plugins/CommonGame/Source/Public/PrimaryGameLayout.h)
