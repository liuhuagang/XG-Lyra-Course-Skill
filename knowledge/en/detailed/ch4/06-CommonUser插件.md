# CommonUser Plugin

## Overview

The CommonUser plugin (`Plugins/CommonUser`) is Lyra's user management infrastructure, providing user initialization, login, and privilege checking functionality. It resides in the UE official plugin layer and is used directly by the Lyra project.

## Core Classes

### UCommonUserSubsystem

- **Header**: [CommonUserSubsystem.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Plugins/CommonUser/Source/CommonUser/Public/CommonUserSubsystem.h)
- **Inheritance**: `UGameInstanceSubsystem` → `UCommonUserSubsystem`
- **Responsibility**: Manages user initialization, login, and privilege checking

### UCommonUserInfo

- **Header**: Same file as above
- **Inheritance**: `UObject` → `UCommonUserInfo`
- **Responsibility**: Stores initialization information and state for a single user

### FCommonUserInitializeParams

- **Header**: Same file as above
- **Responsibility**: User initialization parameter structure

## Enum Types

### ECommonUserInitializationState

Defined in [CommonUserTypes.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Plugins/CommonUser/Source/CommonUser/Public/CommonUserTypes.h)

```
Unknown → DoingInitialLogin → DoingNetworkLogin
  → LoggedInOnline (success)
  → LoggedInLocalOnly (local only)
  → FailedToLogin (failure)
```

### ECommonUserPrivilege

| Value | Description |
|----|------|
| CanPlay | Can play |
| CanPlayOnline | Can play online |
| CanCommunicateViaTextOnline | Online text communication |
| CanCommunicateViaVoiceOnline | Online voice communication |
| CanUseUserGeneratedContent | Use user-generated content |
| CanUseCrossPlay | Cross-platform play |

### ECommonUserAvailability

- **Unknown**: Unknown
- **NowAvailable**: Now available
- **PossiblyAvailable**: Possibly available
- **CurrentlyUnavailable**: Currently unavailable
- **AlwaysUnavailable**: Always unavailable

### ECommonUserPrivilegeResult

- **Unknown** / **Available** / **UserNotLoggedIn** / **LicenseInvalid** / **VersionOutdated** / **NetworkConnectionUnavailable** / **AgeRestricted** / **AccountTypeRestricted** / **AccountUseRestricted** / **PlatformFailure**

### ECommonUserOnlineContext

- **Game**: Game context
- **Default**: Default
- **Service** / **ServiceOrDefault**: Service context
- **Platform** / **PlatformOrDefault**: Platform context
- **Invalid**: Invalid

## Initialization Flow

### Main API

| API | Description |
|-----|------|
| `TryToInitializeForLocalPlay` | Local play initialization |
| `TryToLoginForOnlinePlay` | Online login |
| `TryToInitializeUser` | Full user initialization |
| `ListenForLoginKeyInput` | Listen for login key input |
| `CancelUserInitialization` | Cancel initialization |
| `TryToLogOutUser` | Log out |
| `ResetUserState` | Reset state |

### FUserLoginRequest Internal State Machine

```
TransferPlatformAuthState
  → AutoLoginState (auto login)
    → LoginUIState (login UI)
      → PrivilegeCheckState (privilege check)
```

Each state uses `ECommonUserAsyncTaskState`:
- **NotStarted** → **InProgress** → **Done** / **Failed**

### FCommonUserInitializeParams Parameters

| Parameter | Default | Description |
|------|--------|------|
| LocalPlayerIndex | 0 | Local player index |
| ControllerId | -1 | Controller ID |
| PrimaryInputDevice | - | Primary input device |
| RequestedPrivilege | CanPlay | Requested privilege level |
| OnlineContext | Game | Online context |
| bCanCreateNewLocalPlayer | false | Whether to create a new LocalPlayer |
| bCanUseGuestLogin | false | Whether to use guest login |
| bSuppressLoginErrors | false | Whether to suppress login errors |
| OnUserInitializeComplete | - | Initialization complete callback |

## Callback Events

- `HandleIdentityLoginStatusChanged`: Identity login status change
- `HandleUserLoginCompleted`: User login complete
- `HandleControllerPairingChanged`: Controller pairing change
- `HandleNetworkConnectionStatusChanged`: Network connection status change
- `HandleOnLoginUIClosed`: Login UI closed
- `HandleCheckPrivilegesComplete`: Privilege check complete

## Online Context Cache

The CommonUser subsystem maintains three online context caches:
- **DefaultContextInternal**: Default context
- **ServiceContextInternal**: Service context
- **PlatformContextInternal**: Platform context

## Related Files

- [CommonUserSubsystem.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Plugins/CommonUser/Source/CommonUser/Public/CommonUserSubsystem.h)
- [CommonUserTypes.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Plugins/CommonUser/Source/CommonUser/Public/CommonUserTypes.h)
