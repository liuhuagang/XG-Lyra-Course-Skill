# Gyra Behavior Tree Supplement

## Overview

This section briefly covers supplementary uses of behavior trees in the Gyra project, including minion and boss behavior tree design, custom Tasks/Decorators, and Navigation Mesh-related techniques such as jump links.

## Minion Chase Behavior Tree

The minion behavior tree includes the following custom nodes:

- `BTT_SetMovementSpeed`: Custom Task for setting movement speed
- `BTT_Monster_MeleeAttack`: Melee attack Task

## Boss Phase Behavior Tree

The boss behavior tree includes more complex phase switching logic:

- Health detection: triggers ultimate skill below expected health threshold
- Teleport skill, fireball skill, zombie spawn skill
- Reuses the minion chase behavior tree as a sub-behavior tree

### Custom Decorators

- `BTDecorator_HealthCheck`: Detects health threshold, controls behavior tree branch switching
- `BTDecorator_CheckDistance`: Detects distance to target, controls skill release timing

### Custom Tasks

- `BTT_Lich_Transport`: Boss teleport Task

## Jump Links

Jump links define path points that AI can jump across, creating connections on the Navigation Mesh.

Reference doc: [Modifying the Navigation Mesh Overview](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/overview-of-how-to-modify-the-navigation-mesh-in-unreal-engine)

## Reference Source

The content in this section is an independent behavior tree implementation from the Gyra project, not present in the Lyra project source. Related content can be referenced in the AI system implementation in the Lyra project.
