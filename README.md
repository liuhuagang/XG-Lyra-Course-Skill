# [UE5] 虚幻引擎游戏案例 Lyra 精讲

> 深度解析 Epic Games 官方示例项目 Lyra，掌握商业级游戏架构设计

[前往 B 站购买课程](https://www.bilibili.com/cheese/play/ss112001159)

---

## 课程简介

你是否曾好奇像《堡垒之夜》这样的顶级游戏是如何构建的？Epic Games 官方示例项目 **Lyra** 正是答案！Lyra 精讲课程为你带来最系统、最深入的 Lyra 项目解析教程，带你从零掌握商业级游戏架构设计。

| 课程信息 | 说明 |
|----------|------|
| 讲师 | 虚幻小刚（UAI 虚幻认证讲师、UE5 C++ 程序员、Steam 独立游戏 Gyra 制作人） |
| 课时 | 约 **500 课时 / 125+ 小时**，已完结 |
| 讲义 | **120+ 篇**手写讲义，逐课编写，覆盖全部 10 个章节 |
| 预估学习时间 | 1~3 个月（面向业内人士的虚幻引擎游戏开发高级课程） |
| 前置要求 | 扎实的虚幻引擎 UEC++ 基础（或约一年 UEC++ 开发经验） |

### 为什么选择这门课程？

- **项目拆解**：Lyra 是 Epic 官方提供的完整游戏模板，涵盖射击、多人联机、Gameplay Ability System（GAS）、UI 系统、AI、插件化架构等核心模块
- **实战导向**：不是简单的代码阅读，而是逐模块分析 + 实战优化，让你真正掌握可复用的开发模式
- **进阶必备**：适合有一定 UE5 基础，但想突破瓶颈、学习商业级游戏架构的开发者
- **节省时间**：Lyra 代码量庞大，自学容易迷失方向，本课程帮你直击核心，避免踩坑

### 学完你能做什么？

- 掌握 **GAS**（Gameplay Ability System）的深度应用，实现复杂的技能与 Buff 系统
- 理解 **模块化游戏设计**，学会用 GameFeature 插件动态加载玩法
- 构建 **高效多人联机架构**，优化同步与 Replication 策略
- 设计 **可扩展的 UI 系统**，掌握 CommonUI 与 UMG 的高级用法
- 学会 **虚幻引擎推荐的项目结构**，提升代码可维护性

### 适合谁学习？

- **UE5 中级开发者**：想进阶商业级游戏架构，提升工程能力
- **独立游戏团队**：希望借鉴 Epic 官方最佳实践，优化项目结构
- **技术策划 / TA**：深入理解 GAS、AI、多人联机等系统设计

---

## 交付物总览

购课学员可获得以下全部内容：

| 交付物 | 路径 | 说明 |
|--------|------|------|
| 课程讲义 | [docs/](../docs/) | 120+ 篇手写讲义（编号 001~122），125+ 小时课程内容逐课整理，覆盖全部 10 个章节 |
| AI 知识辅助 Skill | [.trae/skills/xg-lyra-course/](.trae/skills/xg-lyra-course/) | 面向开发实践的 AI 可交互知识库 |

---

## 课程讲义

`docs/` 目录包含 **120+ 篇课程讲义**（编号 001~122），与 **125+ 小时课程内容**一一对应，由讲师逐课编写。

覆盖范围从工程搭建起步，贯穿 10 个章节：Experience 框架、设置系统、UI 架构、角色与输入系统、GAS 能力系统、装备与武器系统、换装队伍与指示器，直到游戏流程与系统。每篇讲义对应一节课，完整的编号体系让你可以按学习进度精准定位到任意一节课的配套文档。

讲义格式包括 `.md` 与 `.docx`，均按编号顺序组织。

---

## xg-lyra-course Skill

`xg-lyra-course` 是本课程的 **AI 知识辅助工具**，将课程中 125+ 小时的讲解、10 个章节、82 篇知识文档进一步提炼为可交互的 Skill，帮助学员在开发实践中快速查阅 Lyra 架构。

Skill 定位在 **Trae / Cursor / Windsurf** 等 AI IDE 中使用，讲解课时你翻阅讲义，动手开发时 Skill 协助你快速定位代码和理解架构设计。

### Skill 内容

Skill 包含 1 篇主入口文档和 **9 篇系统参考文档**，覆盖 Lyra 的完整架构：

| 参考文档 | 覆盖系统 |
|----------|---------|
| [Experience 框架与加载流程](.trae/skills/xg-lyra-course/references/Experience框架与加载流程.md) | Experience 定义、3 阶段加载、GameFeature 激活 |
| [GAS 能力系统架构](.trae/skills/xg-lyra-course/references/GAS能力系统架构.md) | AbilitySet、ASC 输入激活、HealthComponent、AbilityCost |
| [InitState 初始化状态机](.trae/skills/xg-lyra-course/references/InitState初始化状态机.md) | 4 态状态机、组件注册方式 |
| [输入系统与 InputConfig](.trae/skills/xg-lyra-course/references/输入系统与InputConfig.md) | Tag→Action 映射、绑定管线 |
| [角色初始化与移动系统](.trae/skills/xg-lyra-course/references/角色初始化与移动系统.md) | PlayerSpawningManager、移动加速度 Polar 量化 |
| [装备与武器系统](.trae/skills/xg-lyra-course/references/装备与武器系统.md) | 三层装备设计、Spread/Heat 机制 |
| [UI 层栈架构](.trae/skills/xg-lyra-course/references/UI层栈架构.md) | 四层 UI 栈、HUDLayout |
| [动画系统概览](.trae/skills/xg-lyra-course/references/动画系统概览.md) | Lyra 动画架构 |
| [库存与消息系统](.trae/skills/xg-lyra-course/references/库存与消息系统.md) | 四层库存架构、GameplayMessageRouter、GamePhase |

### 使用示例

在 AI IDE 中配置本 Skill 后，你可以直接提问，例如：

> **"添加一个火箭筒武器需要怎么做？"**
>
> Skill 会自动检索装备系统的三层设计模式，给出从 EquipmentDefinition → EquipmentInstance → AbilitySet 的完整步骤和代码参考。

> **"解释一下 Lyra 中换装同步的机制"**
>
> Skill 会提取 Cosmetics 系统的双组件架构（Controller 授权 + Pawn FastArray 同步）并提供关键源码路径。

> **"在 Lyra 中添加一个新的 GameMode"**
>
> Skill 会引导你从 ExperienceDefinition 开始，配置 GameFeature 插件、默认 Pawn/Controller/HUD，并给出 5 步开发工作流。

---

## 许可证

本仓库中的课程知识内容（README、Skill 文档、课程讲义）仅供已购课学员学习和参考使用，不得用于商业再分发。

[前往 B 站购买课程](https://www.bilibili.com/cheese/play/ss112001159)
