# UE5_Lyra学习指南_092_如何构建子弹的散射

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.  
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)  
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)  

文章内容由小刚撰写,采用了以下多种方式:  
1.口述转文字  
2.AI重构  
3.参考引擎源码  
4.Lyra工程源码  
5.结合社区论坛各位大佬的解析  

- [UE5\_Lyra学习指南\_092\_如何构建子弹的散射](#ue5_lyra学习指南_092_如何构建子弹的散射)
	- [代码](#代码)
	- [概述](#概述)
	- [基本思想](#基本思想)
	- [具体分解](#具体分解)
		- [第一个旋转：远离中心 (FromCenterQuat)](#第一个旋转远离中心-fromcenterquat)
		- [第二个旋转：围绕中心轴 (AroundQuat)](#第二个旋转围绕中心轴-aroundquat)
	- [组合旋转的顺序](#组合旋转的顺序)
	- [几何可视化](#几何可视化)
	- [数学意义](#数学意义)
	- [实际应用](#实际应用)
	- [总结](#总结)


## 代码
``` cpp
FVector VRandConeNormalDistribution(const FVector& Dir, const float ConeHalfAngleRad, const float Exponent)
{
	if (ConeHalfAngleRad > 0.f)
	{
		// 弧度转换角度
		const float ConeHalfAngleDegrees = FMath::RadiansToDegrees(ConeHalfAngleRad);

		// consider the cone a concatenation of two rotations. one "away" from the center line, and another "around" the circle
		// apply the exponent to the away-from-center rotation. a larger exponent will cluster points more tightly around the center
		
		// 将该圆锥视为两个旋转的组合。一个旋转方向与中心线垂直，另一个旋转方向围绕圆周
		// 对远离中心的旋转应用指数运算。指数值越大，点就会更紧密地聚集在中心周围
		const float FromCenter = FMath::Pow(FMath::FRand(), Exponent);
		const float AngleFromCenter = FromCenter * ConeHalfAngleDegrees;
		const float AngleAround = FMath::FRand() * 360.0f;

		FRotator Rot = Dir.Rotation();
		FQuat DirQuat(Rot);
		FQuat FromCenterQuat(FRotator(0.0f, AngleFromCenter, 0.0f));
		FQuat AroundQuat(FRotator(0.0f, 0.0, AngleAround));
		FQuat FinalDirectionQuat = DirQuat * AroundQuat * FromCenterQuat;
		FinalDirectionQuat.Normalize();

		return FinalDirectionQuat.RotateVector(FVector::ForwardVector);
	}
	else
	{
		return Dir.GetSafeNormal();
	}
}
```
## 概述
这行代码将圆锥的生成分解为两个旋转操作，这是一个非常巧妙的数学建模方法。让我详细解释这个"两个旋转组合"的概念：

## 基本思想

将一个圆锥内的随机方向分解为：
1. **远离中心轴的旋转** - 控制偏离中心的角度
2. **围绕中心轴的旋转** - 控制圆周方向

## 具体分解

### 第一个旋转：远离中心 (FromCenterQuat)
```cpp
const float FromCenter = FMath::Pow(FMath::FRand(), Exponent);
const float AngleFromCenter = FromCenter * ConeHalfAngleDegrees;
FQuat FromCenterQuat(FRotator(0.0f, AngleFromCenter, 0.0f));
```

**作用**：
- 在**YOZ平面**内旋转向量（绕X轴）
- 控制向量偏离中心轴的程度
- `AngleFromCenter` 决定了偏离的角度大小

**指数分布的作用**：
- `FMath::Pow(FMath::FRand(), Exponent)` 让随机数偏向0或1
- `Exponent > 1`：结果偏向0 → 角度小 → 点聚集在中心
- `Exponent < 1`：结果偏向1 → 角度大 → 点分散在边缘
- `Exponent = 1`：均匀分布

### 第二个旋转：围绕中心轴 (AroundQuat)
```cpp
const float AngleAround = FMath::FRand() * 360.0f;
FQuat AroundQuat(FRotator(0.0f, 0.0, AngleAround));
```

**作用**：
- 围绕**原始方向轴**旋转（绕Z轴）
- 控制向量在圆锥截面圆周上的位置
- `AngleAround` 是0-360度之间的随机角度

## 组合旋转的顺序

```cpp
FQuat FinalDirectionQuat = DirQuat * AroundQuat * FromCenterQuat;
```

这个乘法顺序很重要：

1. **`FromCenterQuat`** - 先在局部坐标系中让向量偏离中心
   - 假设初始方向是 `FVector::ForwardVector` (1,0,0)
   - 经过这个旋转后，向量会在YOZ平面内偏离X轴

2. **`AroundQuat`** - 然后围绕原始方向轴旋转
   - 让偏离后的向量在圆锥截面的圆周上旋转

3. **`DirQuat`** - 最后将整个圆锥对齐到目标方向
   - 将局部坐标系中的圆锥变换到世界坐标系

## 几何可视化

想象这个过程：

1. **初始状态**：一个指向正前方(X轴)的向量
2. **第一步旋转**：向量向上或向下倾斜某个角度（形成经线）
3. **第二步旋转**：围绕原始轴线旋转某个角度（形成纬线）
4. **第三步变换**：将整个圆锥旋转到目标方向

## 数学意义

这种方法的好处：
- **解耦**：将三维随机方向生成分解为两个独立的二维问题
- **控制灵活**：可以分别控制径向分布和圆周分布
- **性能优化**：比直接三维随机采样更高效

## 实际应用

在游戏中的用途：
- **武器散射**：子弹在瞄准方向周围的随机分布
- **粒子效果**：粒子发射的锥形区域
- **AI视野**：NPC的视觉锥体检测

这种分解方法使得我们可以用简单的二维操作来生成复杂的三维锥形分布，同时通过指数参数精确控制分布的聚集程度。
## 总结
由AI解释得来,最好画图理解