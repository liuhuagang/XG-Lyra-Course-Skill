# UE5_Lyra学习指南_099_AI_如何理解拉力和减速效果

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.  
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)  
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)  

文章内容由小刚撰写,采用了以下多种方式:  
1.口述转文字  
2.AI重构  
3.参考引擎源码  
4.Lyra工程源码  
5.结合社区论坛各位大佬的解析  
你理解得很对！整个系统确实基于**三个检测框**，而**拉力和减速效果**有各自的应用条件和逻辑。让我详细解释：

## 一、三个检测框的理解

### 1. **TargetingReticleBounds（目标瞄准框）**
```cpp
// 最大的框，用于初步筛选目标
const FBox2D TargetingReticleBounds = OwnerData.ProjectReticleToScreen(
    Settings.TargetingReticleWidth.GetValue(),  // 默认1200
    Settings.TargetingReticleHeight.GetValue(), // 默认675
    ReticleDepth);
```
**作用**：相当于"搜索区域"，只有在这个框内的目标才会被系统考虑。这是一个**宽大的初步筛选区域**。

### 2. **AssistOuterReticleBounds（辅助外框）**
```cpp
// 中等框，开始应用辅助效果
const FBox2D AssistOuterReticleBounds = OwnerData.ProjectReticleToScreen(
    Settings.AssistOuterReticleWidth.GetValue(),   // 默认80
    Settings.AssistOuterReticleHeight.GetValue(),  // 默认80
    ReticleDepth);
```
**作用**：当目标进入这个区域，系统开始计算**拉力和减速效果**。这是**辅助效果的激活区域**。

### 3. **AssistInnerReticleBounds（辅助内框）**
```cpp
// 最小的框，应用更强的辅助效果
const FBox2D AssistInnerReticleBounds = OwnerData.ProjectReticleToScreen(
    Settings.AssistInnerReticleWidth.GetValue(),   // 默认20
    Settings.AssistInnerReticleHeight.GetValue(),  // 默认20
    ReticleDepth);
```
**作用**：当目标进入这个区域，系统应用**更强的辅助效果**。这是**精准瞄准区域**。

### 关键逻辑流程：
```cpp
// 判断目标在哪个区域
NewTarget.bUnderAssistInnerReticle = AssistInnerReticleBounds.Intersect(TargetScreenBounds);
NewTarget.bUnderAssistOuterReticle = AssistOuterReticleBounds.Intersect(TargetScreenBounds);

// 必须先在TargetingReticleBounds内
if (!TargetingReticleBounds.Intersect(TargetScreenBounds)) {
    continue;  // 不在目标瞄准框内，直接跳过
}
```

## 二、拉力效果（Pull）的应用条件

### **核心条件检查**：
```cpp
// AimAssistInputModifier.cpp 第477行
if (Settings.bApplyPull && bIsApplyingAnyInput && !FMath::IsNearlyZero(PullStrength)) {
    // 应用拉力效果
}
```

### **1. 系统开关条件**：
- `Settings.bApplyPull` = `true`（系统开启拉力功能）
- 目标必须在`AssistOuterReticleBounds`内且可见
  ```cpp
  // 第426行：遍历目标时检查
  if (Target.bUnderAssistOuterReticle && Target.bIsVisible) {
      // 计算拉力...
  }
  ```

### **2. 玩家输入条件**：
- **输入要求**：玩家必须有输入（移动或视角输入）
  ```cpp
  // 第466-475行：检查输入是否激活
  const bool bIsLookInputActive = (CurrentLookInputValue.SizeSquared() > FMath::Square(LookStickDeadzone));
  const bool bIsMoveInputActive = (CurrentMoveInputValue.SizeSquared() > FMath::Square(MoveStickDeadzone));
  const bool bIsApplyingLookInput = (bIsLookInputActive || !Settings.bRequireInput);
  const bool bIsApplyingMoveInput = (bIsMoveInputActive || !Settings.bRequireInput);
  const bool bIsApplyingAnyInput = (bIsApplyingLookInput || bIsApplyingMoveInput);
  ```

- **特殊情况**：如果`Settings.bRequireInput = false`，则**不需要玩家输入**也会应用拉力

### **3. 强度计算逻辑**：
```cpp
// 第682-714行：根据目标位置计算强度
if (Target.bUnderAssistInnerReticle) {
    // 内框：更强
    OutPullStrength = (bIsADS) ? 
        Settings.PullInnerStrengthAds.GetValue() :  // ADS模式：0.7
        Settings.PullInnerStrengthHip.GetValue();   // 腰射模式：0.6
} else if (Target.bUnderAssistOuterReticle) {
    // 外框：较弱
    OutPullStrength = (bIsADS) ? 
        Settings.PullOuterStrengthAds.GetValue() :  // ADS模式：0.4
        Settings.PullOuterStrengthHip.GetValue();   // 腰射模式：0.5
}

// 乘以目标权重
OutPullStrength *= Target.AssistWeight;
```

### **4. 特殊行为：侧移缩放**
```cpp
// 第486-492行：如果玩家没有主动转动视角但有侧移
if (!bIsApplyingLookInput && Settings.bApplyStrafePullScale) {
    float StrafePullScale = FMath::Abs(CurrentMoveInputValue.Y);  // 侧移幅度
    PullRotation.Yaw *= StrafePullScale;
    PullRotation.Pitch *= StrafePullScale;
}
```
**设计意图**：防止玩家直线跑过目标时，视角被突然拉向目标。

## 三、减速效果（Slow）的应用条件

### **核心条件检查**：
```cpp
// AimAssistInputModifier.cpp 第513行
if (Settings.bApplySlowing && bIsApplyingLookInput && !FMath::IsNearlyZero(SlowStrength)) {
    // 应用减速效果
}
```

### **1. 系统开关条件**：
- `Settings.bApplySlowing` = `true`（系统开启减速功能）
- 目标必须在`AssistOuterReticleBounds`内且可见

### **2. 玩家输入条件**：
- **必须要有视角输入**（玩家在主动转动视角）
  ```cpp
  const bool bIsApplyingLookInput = (bIsLookInputActive || !Settings.bRequireInput);
  ```
- 与拉力不同：**移动输入不触发减速**，只有视角输入触发

### **3. 强度计算逻辑**：
```cpp
// 第682-714行：根据目标位置计算强度
if (Target.bUnderAssistInnerReticle) {
    // 内框：更强减速
    OutSlowStrength = (bIsADS) ? 
        Settings.SlowInnerStrengthAds.GetValue() :   // ADS模式：0.7
        Settings.SlowInnerStrengthHip.GetValue();    // 腰射模式：0.6
} else if (Target.bUnderAssistOuterReticle) {
    // 外框：较弱减速
    OutSlowStrength = (bIsADS) ? 
        Settings.SlowOuterStrengthAds.GetValue() :   // ADS模式：0.4
        Settings.SlowOuterStrengthHip.GetValue();    // 腰射模式：0.5
}

// 乘以目标权重
OutSlowStrength *= Target.AssistWeight;
```

### **4. 动态减速增强**：
```cpp
// 第524-534行：如果启用动态减速
if (Settings.bUseDynamicSlow) {
    const FRotator BoostRotation = (RotationNeeded * (1.0f / DeltaTime));
    
    // 如果玩家转动方向与所需旋转方向一致，增强减速
    const float YawDynamicBoost = (BoostRotation.Yaw * FMath::Sign(CurrentLookInputValue.X));
    if (YawDynamicBoost > 0.0f) {
        SlowRates.Yaw += YawDynamicBoost;
    }
    
    const float PitchDynamicBoost = (BoostRotation.Pitch * FMath::Sign(CurrentLookInputValue.Y));
    if (PitchDynamicBoost > 0.0f) {
        SlowRates.Pitch += PitchDynamicBoost;
    }
}
```
**设计意图**：当玩家尝试追踪移动目标时，提供更强的"粘性"。

## 四、关键区别总结

| 特性 | 拉力效果 (Pull) | 减速效果 (Slow) |
|------|----------------|-----------------|
| **触发条件** | 目标在外框或内框 | 目标在外框或内框 |
| **输入要求** | 移动或视角输入 | **必须**有视角输入 |
| **作用方向** | **自动**拉向目标 | **减慢**玩家输入 |
| **感觉像** | 磁力吸附 | 粘性阻力 |
| **主要用途** | 帮助捕捉目标 | 帮助跟踪目标 |
| **强度变化** | 内框 > 外框 | 内框 > 外框 |
| **ADS vs 腰射** | ADS时内框更强 | ADS时内框更强 |
| **特殊行为** | 侧移时缩放强度 | 动态增强粘性 |

## 五、实际应用场景

### **场景1：目标快速横穿屏幕**
1. 目标进入`TargetingReticleBounds`（搜索区）
2. 目标进入`AssistOuterReticleBounds`（辅助区）
3. **拉力效果激活**：视角被轻微拉向目标
4. 如果玩家**开始转动视角**追目标：
   - **减速效果激活**：转动速度变慢，更容易跟上目标
5. 目标进入`AssistInnerReticleBounds`（精准区）：
   - **拉力增强**：更强力拉向目标
   - **减速增强**：转动更慢，精度更高

### **场景2：玩家直线跑过目标**
1. 目标进入辅助区
2. 玩家**没有转动视角**，但有**侧向移动**
3. **拉力效果被缩放**：根据侧移幅度减小拉力
4. **减速效果不激活**：因为没有视角输入
5. **结果**：视角不会被突然拉动，体验更自然

### **场景3：精确瞄准**
1. 目标在辅助区内
2. 玩家**微调**瞄准
3. **减速效果显著**：微调速度变慢，更容易精确瞄准
4. 如果目标移动：
   - **动态减速增强**：提供粘性，帮助跟踪

## 六、设计哲学

1. **拉力（Pull）** = **"捕捉"机制**
   - 帮助玩家**找到**目标
   - 更像是**自动校正**

2. **减速（Slow）** = **"粘性"机制**
   - 帮助玩家**保持**在目标上
   - 更像是**阻尼器**

3. **内外框分级** = **"渐进辅助"**
   - 外框：轻度辅助，不突兀
   - 内框：强力辅助，精准打击

这样的设计确保了：
- 新手玩家：获得明显的辅助，更容易命中
- 熟练玩家：辅助不干扰精细操作
- 所有玩家：体验自然，不觉得被"作弊"