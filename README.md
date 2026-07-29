# [UE5] Unreal Engine Lyra Deep Dive

> In-depth analysis of Epic Games' official sample project Lyra, mastering commercial-grade game architecture design

[Buy the course on Bilibili](https://www.bilibili.com/cheese/play/ss112001159)

---

## Course Overview

Have you ever wondered how top-tier games like *Fortnite* are built? Epic Games' official sample project **Lyra** is the answer! The Lyra Deep Dive course provides the most systematic and in-depth Lyra project analysis, taking you from zero to mastering commercial-grade game architecture.

| Course Info | Details |
|-------------|---------|
| Instructor | Unreal Xiaogang (UAI Certified Unreal Instructor, UE5 C++ Programmer, Steam Indie Game Gyra Producer) |
| Lessons | Approximately **500 lessons / 125+ hours**, complete |
| Handouts | **120+** hand-written handouts, written per lesson, covering all 10 chapters |
| Estimated Study Time | 1~3 months (an advanced Unreal Engine game development course for industry professionals) |
| Prerequisites | Solid Unreal Engine UEC++ foundation (or approximately 1 year of UEC++ development experience) |

### Why Choose This Course?

- **Project Deconstruction**: Lyra is Epic's official complete game template, covering shooting, multiplayer, Gameplay Ability System (GAS), UI system, AI, plugin-based architecture, and other core modules
- **Practice-Oriented**: Not just code reading, but module-by-module analysis + hands-on optimization, allowing you to truly master reusable development patterns
- **Essential for Advancement**: Suitable for developers with some UE5 foundation who want to break through bottlenecks and learn commercial-grade game architecture
- **Save Time**: Lyra's codebase is massive and easy to get lost in when self-studying; this course helps you focus on the core and avoid pitfalls

### What You'll Learn

- Master deep application of **GAS** (Gameplay Ability System) to implement complex abilities and Buff systems
- Understand **modular game design**, learning to dynamically load gameplay with GameFeature plugins
- Build **efficient multiplayer architecture**, optimizing synchronization and Replication strategies
- Design **extensible UI systems**, mastering advanced usage of CommonUI and UMG
- Learn **Unreal Engine's recommended project structure** to improve code maintainability

### Who Is This For?

- **Intermediate UE5 Developers**: Looking to advance to commercial-grade game architecture and improve engineering skills
- **Indie Game Teams**: Wanting to learn from Epic's official best practices to optimize project structure
- **Technical Designers / TAs**: Seeking deep understanding of GAS, AI, multiplayer, and other system designs

---

## Deliverables Overview

Students who purchase the course receive all of the following:

| Deliverable | Path | Description |
|-------------|------|-------------|
| Course Handouts | [docs/](../docs/) | 120+ hand-written handouts (numbered 001~122), organized per lesson from 125+ hours of content, covering all 10 chapters |
| Granular Knowledge Documents | [knowledge/](../knowledge/) | 82 knowledge documents organized by chapter, serving as the knowledge source for the Skill, usable as standalone references |
| AI Knowledge Skill | [skill/xg-lyra-course/](skill/xg-lyra-course/) | An AI-interactive knowledge base for development practice |

---

## Course Handouts

The `docs/` directory contains **120+ course handouts** (numbered 001~122), corresponding one-to-one with **125+ hours of course content**, written per lesson by the instructor.

Coverage ranges from project setup through all 10 chapters: Experience framework, settings system, UI architecture, character and input system, GAS ability system, equipment and weapon system, cosmetics teams and indicators, through to game flow and systems. Each handout corresponds to one lesson, with a complete numbering system allowing you to precisely locate the accompanying document for any lesson as you progress.

Handouts are available in `.md` and `.docx` formats, organized by number.

---

## xg-lyra-course Skill

`xg-lyra-course` is the **AI knowledge assistant** for this course, further refining the 125+ hours of lectures, 10 chapters, and 82 knowledge documents into an interactive Skill, helping students quickly reference Lyra architecture during development practice.

The Skill is designed for use in **Trae / Cursor / Windsurf** and other AI IDEs. During lessons you read handouts; during hands-on development, the Skill helps you quickly locate code and understand architectural design.

### Skill Content

The Skill includes 1 main entry document and **9 system reference documents**, covering Lyra's complete architecture:

| Reference Document | System Coverage |
|-------------------|-----------------|
| [Experience Framework and Loading Flow](skill/xg-lyra-course/references/Experience框架与加载流程.md) | Experience definition, 3-stage loading, GameFeature activation |
| [GAS Ability System Architecture](skill/xg-lyra-course/references/GAS能力系统架构.md) | AbilitySet, ASC input activation, HealthComponent, AbilityCost |
| [InitState Initialization State Machine](skill/xg-lyra-course/references/InitState初始化状态机.md) | 4-state state machine, component registration |
| [Input System and InputConfig](skill/xg-lyra-course/references/输入系统与InputConfig.md) | Tag→Action mapping, binding pipeline |
| [Character Initialization and Movement System](skill/xg-lyra-course/references/角色初始化与移动系统.md) | PlayerSpawningManager, movement acceleration Polar quantization |
| [Equipment and Weapon System](skill/xg-lyra-course/references/装备与武器系统.md) | 3-layer equipment design, Spread/Heat mechanism |
| [UI Layer Stack Architecture](skill/xg-lyra-course/references/UI层栈架构.md) | 4-layer UI stack, HUDLayout |
| [Animation System Overview](skill/xg-lyra-course/references/动画系统概览.md) | Lyra animation architecture |
| [Inventory and Message System](skill/xg-lyra-course/references/库存与消息系统.md) | 4-layer inventory architecture, GameplayMessageRouter, GamePhase |

### Usage Examples

After configuring this Skill in an AI IDE, you can ask questions directly, for example:

> **"How do I add a rocket launcher weapon?"**
>
> The Skill will automatically retrieve the equipment system's 3-layer design pattern, providing the complete steps and code references from EquipmentDefinition → EquipmentInstance → AbilitySet.

> **"Explain the cosmetics synchronization mechanism in Lyra"**
>
> The Skill will extract the cosmetics system's dual-component architecture (Controller authorization + Pawn FastArray synchronization) and provide key source code paths.

> **"Add a new GameMode in Lyra"**
>
> The Skill will guide you starting from ExperienceDefinition, configuring the GameFeature plugin, default Pawn/Controller/HUD, and provide a 5-step development workflow.

If the Skill's AI interaction results are not as expected, the granular documents in the [knowledge/](../knowledge/) directory are provided as supplementary material — you can directly reference these knowledge documents, or use them together with handouts, source code, and other original materials as needed.

---

## License

The course knowledge content in this repository (README, Skill documents, course handouts) is provided for enrolled students' learning and reference use only, and may not be used for commercial redistribution.

[Buy the course on Bilibili](https://www.bilibili.com/cheese/play/ss112001159)
