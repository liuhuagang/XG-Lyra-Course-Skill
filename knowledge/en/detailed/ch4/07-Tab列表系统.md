# Tab List System

## Overview

The Tab List system (`ULyraTabListWidgetBase`) is a component in Lyra's UI for managing tab page switching. It extends `UCommonTabListWidgetBase`, providing tab registration, content creation, and navigation functionality.

## Core Classes

### FLyraTabDescriptor

- **Header**: [LyraTabListWidgetBase.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/UI/Common/LyraTabListWidgetBase.h)
- **Responsibility**: Tab descriptor structure

| Field | Type | Description |
|------|------|------|
| TabId | FName | Tab unique identifier |
| TabText | FText | Tab display text |
| IconBrush | FSlateBrush | Tab icon |
| bHidden | bool | Whether hidden (hidden tabs are not registered) |
| TabButtonType | TSubclassOf<UCommonButtonBase> | Tab button widget class |
| TabContentType | TSubclassOf<UCommonUserWidget> | Tab content widget class |
| CreatedTabContentWidget | UWidget* | Created content widget instance (cached) |

### ILyraTabButtonInterface

- **Header**: Same as above
- **Responsibility**: Tab button interface, defines `SetTabLabelInfo(FLyraTabDescriptor)` method
- **Implementing Class**: Exposed to Blueprints via `ULyraTabButtonInterface` (UInterface)

### ULyraTabListWidgetBase

- **Header**: Same as above
- **Inheritance**: `UCommonTabListWidgetBase` → `ULyraTabListWidgetBase`
- **Responsibility**: Tab list Widget base class

## Tab Registration Flow

```
RegisterDynamicTab(FLyraTabDescriptor)
  ├── Check bHidden → Hidden tabs directly return true (skip registration)
  ├── Store in PendingTabLabelInfoMap.Add(TabId, TabDescriptor)
  └── Call RegisterTab(TabId, TabButtonType, CreatedTabContentWidget) [base class]
        └── HandleTabCreation_Implementation(FName TabId, UCommonButtonBase* TabButton)
              ├── Look up FLyraTabDescriptor:
              │     1. PreregisteredTabInfoArray (pre-registered tabs)
              │     2. PendingTabLabelInfoMap (dynamic tabs)
              └── If TabButton implements ILyraTabButtonInterface:
                    Execute_SetTabLabelInfo(TabButton, *TabInfoPtr)
                    → PendingTabLabelInfoMap.Remove(TabId)
```

## Tab Navigation

### Auto Input Listening

- Set `bAutoListenForInput = true`
- Internally calls `SetListeningForInput` → `UpdateBindings`
- Registers UIActionBindings for NextTab/PreviousTab

### Navigation Logic

```
SelectNextButton() / SelectPreviousButton():
  └── Recursive lambda: skip disabled buttons, supports bAllowWrap wrapping

SelectButtonAtIndex(ButtonIndex):
  └── ButtonToSelect->SetSelectedInternal(true)
        └── NativeOnSelected
              ├── BP_OnSelected (Blueprint event)
              └── OnSelectedChangedBase.Broadcast (broadcast)
```

## Related Files

- [LyraTabListWidgetBase.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/UI/Common/LyraTabListWidgetBase.h)
