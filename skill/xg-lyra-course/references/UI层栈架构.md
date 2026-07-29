# UI Layer Stack Architecture

## Overview

Lyra's UI system is built on the **CommonUI** plugin, using `UPrimaryGameLayout` to manage a four-layer Widget stack. Each layer displays and interacts independently. This architecture makes UI hierarchy management (HUD at the bottom, menus in the middle, modals at the top) a declarative configuration rather than manual management.

---

## Four-Layer Stack Structure

```
Modal (Modal Layer)
    ↑ Dialogs, confirmation boxes — BlockInput = true, blocks lower layer interaction
Menu (Menu Layer)
    ↑ Inventory, settings, store, map
GameMenu (Game Menu Layer)
    ↑ In-game pause menu (ESC menu)
Game (Game Layer)
    ↑ HUD, crosshair, health bar, cooldowns, kill feed
```

---

## PrimaryGameLayout

**File**: `Plugins/CommonGame/Source/Public/PrimaryGameLayout.h`

```cpp
UCLASS()
class UPrimaryGameLayout : public UCommonActivatableWidgetStack
{
    GENERATED_BODY()

public:
    // Four layer stack accessors
    UFUNCTION(BlueprintPure)
    UCommonActivatableWidgetStack* GetGameStack() const;

    UFUNCTION(BlueprintPure)
    UCommonActivatableWidgetStack* GetGameMenuStack() const;

    UFUNCTION(BlueprintPure)
    UCommonActivatableWidgetStack* GetMenuStack() const;

    UFUNCTION(BlueprintPure)
    UCommonActivatableWidgetStack* GetModalStack() const;

    // Get the PrimaryGameLayout for a specific player
    static UPrimaryGameLayout* GetPrimaryGameLayout(APlayerController* Controller);
};
```

---

## HUDLayout Structure

**File**: `Source/LyraGame/UI/LyraHUDLayout.h`

`ULyraHUDLayout` is the root Widget created by `ALyraHUD`, using CommonUI's Tier layout internally:

```
ULyraHUDLayout
    ├── TopLeft Tier (team info, player list)
    ├── TopCenter Tier (game mode info)
    ├── TopRight Tier (network status, FPS)
    ├── LowerLeft Tier (health, ammo, cooldowns)
    ├── LowerMiddle Tier (quick bar, current equipment)
    └── LowerRight Tier (kill feed, notifications)
```

HUD layout uses `UCommonBorder` as containers, controlling each Tier's visibility through GameplayTags.

---

## Widget Lifecycle

All UI Widgets inherit from `UCommonActivatableWidget` with a standard lifecycle:

```
CreateWidget → OnInitialized
    ↓
AddToStack → OnActivated
    ↓
(Widget active, receiving input)
    ↓
RemoveFromStack / Push another Widget → OnDeactivated
    ↓
Widget destroyed
```

---

## HUD Creation Flow

```
ALyraHUD::BeginPlay()
    → Creates ULyraHUDLayout Widget
    → AddToViewport and adds to Game Layer

ALyraHUD::OnPlayerStateChanged()
    → HUDLayout Tiers create/update child Widgets based on PlayerState
    → Example: LowerLeftTier creates HealthWidget and AmmoWidget
```

---

## Push/Pop Widget

```cpp
// Push HUD to Game layer
UPrimaryGameLayout* Layout = UPrimaryGameLayout::GetPrimaryGameLayout(Controller);
UCommonActivatableWidget* HUDWidget = CreateWidget<UMyHUDWidget>(Controller);
Layout->GetGameStack()->AddWidget(HUDWidget);

// Push inventory to Menu layer
UCommonActivatableWidget* InventoryWidget = CreateWidget<UMyInventoryWidget>(Controller);
Layout->GetMenuStack()->AddWidget(InventoryWidget);

// Push confirmation dialog to Modal layer
UCommonActivatableWidget* ConfirmWidget = CreateWidget<UMyConfirmDialog>(Controller);
Layout->GetModalStack()->AddWidget(ConfirmWidget);
```

---

## Input Blocking Rules

| Layer | Input Behavior |
|-------|----------------|
| Game | Game input (keyboard/mouse) passes through to Pawn |
| GameMenu | Game input blocked, menu input active |
| Menu | Game input blocked, menu input active |
| Modal | All input blocked, only modal widget receives input (BlockInput=true) |

The `BlockInput` feature is natively provided by CommonUI: when the Modal layer has a Widget, the input mode is automatically set to `UI Only`.

---

## Key Design Points

1. **Four-layer stack** — Naturally maps to game interaction hierarchy (HUD < pause menu < feature UI < dialogs)
2. **CommonUI native support** — ActivatableWidget's Push/Pop lifecycle reduces manual input mode management code
3. **Tier layout** — HUDLayout's internal Tier structure arranges HUD elements without relying on Absolute positioning
4. **GameplayTag-driven visibility** — Each Tier controls visibility through GameplayTags, unified with GAS's Tag system
5. **Singleton access** — `GetPrimaryGameLayout` looks up through LocalPlayer, ensuring each player has an independent UI layout
