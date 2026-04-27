# UE5_Lyra学习指南_066_AI_角色移动模式的打包传输

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.  
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)  
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)  

文章内容由小刚撰写,采用了以下多种方式:  
1.口述转文字  
2.AI重构  
3.参考引擎源码  
4.Lyra工程源码  
5.结合社区论坛各位大佬的解析  

- [UE5\_Lyra学习指南\_066\_AI\_角色移动模式的打包传输](#ue5_lyra学习指南_066_ai_角色移动模式的打包传输)
	- [代码](#代码)
	- [概述](#概述)
	- [函数功能详解：](#函数功能详解)
		- [1. 处理非自定义移动模式（MovementMode != MOVE\_Custom）：](#1-处理非自定义移动模式movementmode--move_custom)
		- [2. 处理自定义移动模式：](#2-处理自定义移动模式)
	- [位操作原理：](#位操作原理)
	- [网络同步中的应用：](#网络同步中的应用)
	- [示例：](#示例)
	- [总结](#总结)


## 代码

``` cpp
namespace PackedMovementModeConstants
{
	const uint32 GroundShift = FMath::CeilLogTwo(MOVE_MAX);
	const uint8 CustomModeThr = 2 * (1 << GroundShift);
	const uint8 GroundMask = (1 << GroundShift) - 1;
}

uint8 UCharacterMovementComponent::PackNetworkMovementMode() const
{
	if (MovementMode != MOVE_Custom)
	{
		ensureMsgf(GroundMovementMode == MOVE_Walking || GroundMovementMode == MOVE_NavWalking, TEXT("Invalid GroundMovementMode %d."), GroundMovementMode.GetValue());
		const uint8 GroundModeBit = (GroundMovementMode == MOVE_Walking ? 0 : 1);
		return uint8(MovementMode.GetValue()) | (GroundModeBit << PackedMovementModeConstants::GroundShift);
	}
	else
	{
		return CustomMovementMode + PackedMovementModeConstants::CustomModeThr;
	}
}

```
``` cpp
	static FORCEINLINE uint32 CeilLogTwo( uint32 Arg )
	{
		// if Arg is 0, change it to 1 so that we return 0
		Arg = Arg ? Arg : 1;
		return 32 - CountLeadingZeros(Arg - 1);
	}

```
``` cpp
	static FORCEINLINE uint32 CountLeadingZeros(uint32 Value)
	{
		// return 32 if value is zero
		unsigned long BitIndex;
		_BitScanReverse64(&BitIndex, uint64(Value)*2 + 1);
		return 32 - BitIndex;
	}

```

## 概述
这个函数 `PackNetworkMovementMode()` 是用于**将角色的移动模式压缩打包成一个字节（uint8）**，以便在网络同步时高效传输。这是网络游戏开发中常见的优化技术，用于减少带宽占用。

## 函数功能详解：

### 1. 处理非自定义移动模式（MovementMode != MOVE_Custom）：
```cpp
if (MovementMode != MOVE_Custom)
{
    // 验证地面移动模式的有效性
    ensureMsgf(GroundMovementMode == MOVE_Walking || GroundMovementMode == MOVE_NavWalking, 
               TEXT("Invalid GroundMovementMode %d."), GroundMovementMode.GetValue());
    
    // 将行走模式编码为0，导航行走编码为1
    const uint8 GroundModeBit = (GroundMovementMode == MOVE_Walking ? 0 : 1);
    
    // 将主移动模式和地面模式位组合成一个字节
    return uint8(MovementMode.GetValue()) | (GroundModeBit << PackedMovementModeConstants::GroundShift);
}
```

### 2. 处理自定义移动模式：
```cpp
else
{
    // 自定义模式从特定阈值开始编码
    return CustomMovementMode + PackedMovementModeConstants::CustomModeThr;
}
```

## 位操作原理：

函数使用位操作来在一个字节中编码多个信息：
- **低几位**：存储主移动模式（如Walking, Flying, Swimming等）
- **特定高位**：存储地面移动模式的细分类型（Walking vs NavWalking）

## 网络同步中的应用：

这个打包后的字节会通过网络发送，在接收端会有对应的 `UnpackNetworkMovementMode()` 函数来解包还原完整的移动状态信息。

## 示例：
假设：
- `MovementMode = MOVE_Walking` (值为1)
- `GroundMovementMode = MOVE_NavWalking` (GroundModeBit = 1)
- `GroundShift = 4`

那么打包结果：`1 | (1 << 4) = 17`

这样用一个字节就同时编码了主移动模式和地面移动子模式两种信息。

这种技术在大规模多人游戏中非常重要，可以显著减少网络带宽使用。
## 总结
以上回答来自DeepSeek.