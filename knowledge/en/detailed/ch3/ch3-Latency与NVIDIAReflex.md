# Latency and NVIDIA Reflex

## Overview

Lyra integrates NVIDIA Reflex low-latency technology, injecting latency markers via the ILatencyMarkerModule interface at key stages of the rendering pipeline to mark timestamps, enabling precise measurement and visualization of end-to-end latency.

## ILatencyMarkerModule

NVIDIA Reflex's modular latency marker interface, defining the injection method and marker types for latency markers.

### Latency Marker Injection

Time stamps are injected at specified stages of the rendering pipeline via `SetCustomLatencyMarker`:

```
Specific stage → SetCustomLatencyMarker(markerType)
  → NvAPI_D3D_SetLatencyMarker(..., markerType, frameID)
  → NVIDIA Reflex driver records timestamp
```

### Custom Marker Types (Recommended Range)

| Marker Type | Recommended Value Range |
|----------|-----------|
| NVIDIA Reflex Built-in Markers | 0x0000 ~ 0x00FF |
| Application Custom Markers | 0x1000 and above |

Custom markers must be bound to a specific frameID to ensure latency measurement aligns with rendered frames.

## LatencyFlashIndicators

NVIDIA Reflex latency visualization debugging tool, indicating latency status at each pipeline stage through color flashing:

| Color | Corresponding Stage | Meaning |
|------|---------|------|
| Red | SlateTick | UI logic update latency |
| Green | SlatePaint | UI rendering draw latency |
| Blue | RenderThread | Render thread processing latency |
| Yellow | GPU | GPU rendering latency |

### ELatencyStage Enum

Defines the pipeline stages for latency tracking:

| Stage | Description |
|------|------|
| Input Sampling | From input device to game logic |
| Game Logic | Game thread processes input |
| Render Submission | Render thread submits render commands |
| GPU Execution | GPU executes render commands |
| Display Output | Final frame delivered to display |

## Integration in Lyra

Latency-related functionality in ULyraSettingsLocal:

- `FLyraPerformanceStatCache`: Performance stat cache, collects latency data from each stage
- Latency markers are part of performance stats, controlled by the PerfStatDisplayState system for HUD display

## Latency Options in Settings UI

Latency-related settings are registered via `ULyraGameSettingRegistry`, allowing users to choose whether to display latency indicators and performance stat charts in the settings interface.
