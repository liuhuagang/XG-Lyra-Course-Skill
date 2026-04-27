# UE5_Lyra学习指南_014_泽_01_NearClipPlane

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)

文章内容由小刚撰写,采用了以下多种方式:

1.口述转文字
2.AI重构
3.参考引擎源码
4.Lyra工程源码
5.结合社区论坛各位大佬的解析

- [UE5\_Lyra学习指南\_014\_泽\_NearClipPlane](#ue5_lyra学习指南_014_泽_nearclipplane)
	- [概述](#概述)
		- [1. 定义](#1-定义)
		- [2. 作用](#2-作用)
		- [3. 典型配置位置](#3-典型配置位置)
		- [4. 默认值与调整](#4-默认值与调整)
		- [5. 注意事项](#5-注意事项)
		- [6. 示例配置](#6-示例配置)
	- [总结](#总结)



## 概述

在Unreal Engine 5 (UE5)的`DefaultEngine.ini`配置文件中，`NearClipPlane`（近裁剪平面）是一个与摄像机/视图相关的关键参数，其含义和作用如下：

---

### 1. 定义
`NearClipPlane`（通常以单位`厘米`计）定义了摄像机视锥体的**最近可见距离**。任何距离摄像机比该值更近的物体将不会被渲染（即被“裁剪”掉）。

---

### 2. 作用
- **性能优化**：避免渲染离摄像机过近的物体（如角色内部的几何体），减少不必要的计算。
- **视觉稳定性**：防止极端近距离下的渲染错误（如Z-fighting或深度缓冲精度问题）。
- **VR/特定场景适配**：在VR中可能需要调整该值以避免近处物体消失。

---

### 3. 典型配置位置
在`DefaultEngine.ini`中，通常位于以下配置段中：
```ini
[/Script/Engine.RendererSettings]
r.SetNearClipPlane=10.0  ; 默认值（单位：厘米）
```
或通过控制台命令动态修改：
```ini
r.SetNearClipPlane [值]
```

---

### 4. 默认值与调整
- **默认值**：通常为`10.0`（厘米），但可能因项目或引擎版本而异。
- **调整场景**：
  - **调大**：解决近处物体意外消失的问题（如角色第一人称视角下的武器）。
  - **调小**：需要渲染极近物体（如显微镜效果的场景），但可能引发深度精度问题。

---

### 5. 注意事项
- **与FarClipPlane的关系**：两者共同定义视锥体的深度范围（`NearClipPlane` ≤ `FarClipPlane`）。
- **动态修改**：可通过蓝图或C++调用`ConsoleCommand("r.SetNearClipPlane [值]")`实时调整。
- **平台差异**：移动端/VR可能需要更保守的值以适应性能或硬件限制。

---

### 6. 示例配置
```ini
[/Script/Engine.RendererSettings]
r.SetNearClipPlane=5.0  ; 将近裁剪平面设为5厘米
```

如果需要进一步优化或调试，可结合控制台命令`r.DebugNearClipPlane`可视化裁剪效果。

## 总结
以上回答来自DeepSeek.
