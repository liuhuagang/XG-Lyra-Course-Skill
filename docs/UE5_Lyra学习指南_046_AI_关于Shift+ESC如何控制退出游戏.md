# UE5_Lyra学习指南_046_AI_关于Shift+ESC如何控制退出游戏

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.  
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)  
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)  

文章内容由小刚撰写,采用了以下多种方式:  
1.口述转文字  
2.AI重构  
3.参考引擎源码  
4.Lyra工程源码  
5.结合社区论坛各位大佬的解析  

- [UE5\_Lyra学习指南\_046\_AI\_关于Shift+ESC如何控制退出游戏](#ue5_lyra学习指南_046_ai_关于shiftesc如何控制退出游戏)
	- [概述](#概述)
		- [总体含义](#总体含义)
		- [格式解析（“伪JSON”格式）](#格式解析伪json格式)
		- [总结与类比](#总结与类比)
		- [重要提示](#重要提示)
	- [总结](#总结)



## 概述
在UE5中这个配置什么意思
DefaultEditorKeyBindings
[UserDefinedChords]
UserDefinedChords=~OpenBracket~~Quote~BindingContext~Quote~:~Quote~PlayWorld~Quote~,~Quote~CommandName~Quote~:~Quote~StopPlaySession~Quote~,~Quote~ChordIndex~Quote~:0,~Quote~Control~Quote~:false,~Quote~Alt~Quote~:false,~Quote~Shift~Quote~:true,~Quote~Command~Quote~:false,~Quote~Key~Quote~:~Quote~Escape~Quote~~CloseBracket~
好的，我们来详细解析这段配置的含义。这段代码是 Unreal Engine 编辑器配置文件（通常是 `DefaultEditor.ini` 或 `EditorPerProjectUserSettings.ini`）中的一部分，它用一种特殊的格式定义了一个**用户自定义的快捷键**。

### 总体含义

这段配置定义了一个快捷键组合：**Shift + Escape**，并将其绑定到了 **`StopPlaySession`** 命令上。这个命令的作用就是**停止当前的“在编辑器中播放”（Play-In-Editor, PIE）会话**。

换句话说，它实现了你上一个问题中想要的功能：**按下 Shift+ESC 来停止游戏模拟**。

---

### 格式解析（“伪JSON”格式）

Unreal Engine 的配置文件（.ini）使用一种自己的格式来存储复杂数据，它看起来很像 JSON，但用波浪号 `~` 代替了引号 `"`，并且去掉了一些逗号来避免与 .ini 文件的分隔符冲突。

如果我们把它“翻译”成标准的 JSON，它会看起来像这样，会清晰很多：

```json
{
  "BindingContext": "PlayWorld",
  "CommandName": "StopPlaySession",
  "ChordIndex": 0,
  "Control": false,
  "Alt": false,
  "Shift": true,
  "Command": false,
  "Key": "Escape"
}
```

现在，我们来逐一解释每个字段的意义：

| 字段名 | 值 | 解释 |
| :--- | :--- | :--- |
| **`BindingContext`** | `"PlayWorld"` | **绑定上下文**。这是最关键的概念之一。它指定了这个快捷键在**什么环境下生效**。`PlayWorld` 表示这个快捷键只在**游戏运行（PIE）模式**下有效。在普通的编辑器界面中，这个快捷键不会被触发。 |
| **`CommandName`** | `"StopPlaySession"` | **要执行的命令**。这是引擎内部定义的一个指令，其功能就是终止当前的游戏模拟会话。 |
| **`Key`****`** | `"Escape"` | **主按键**。这是快捷键的核心按键，这里是 `Escape` (ESC键)。 |
| **`Shift`** | `true` | **修饰键**。这表示需要同时按下 **Shift** 键。`true` 代表需要按下，`false` 代表不需要。 |
| **`Control`** | `false` | **修饰键**。表示不需要按下 Control (Ctrl) 键。 |
| **`Alt`** | `false` | **修饰键**。表示不需要按下 Alt 键。 |
| **`Command`** | `false` | **修饰键**（主要针对 macOS）。表示不需要按下 Command (⌘) 键。在Windows上通常无关。 |
| **`ChordIndex`** | `0` | **和弦索引**。这是一个高级用法，用于处理复杂的、多序列的快捷键（例如，先按`K`，再按`M`）。`0` 表示这是一个简单的单次按键组合，没有序列。 |

---

### 总结与类比

你可以把这段配置理解为：

> **“当处于游戏运行环境（`PlayWorld`）中时，如果用户按下 `Shift + Esc` 这个组合键，就执行 `StopPlaySession` 这个命令。”**

这正是在编辑器里运行游戏后，你用 **Shift+ESC** 来退出游戏模式的底层实现。

### 重要提示

1.  **文件位置**：这类用户自定义快捷键通常保存在 `[YourProjectDirectory]/Saved/Config/Windows/EditorPerProjectUserSettings.ini` 文件中。当你通过编辑器的偏好设置（Editor Preferences）图形化界面修改快捷键时，引擎最终就会将更改写入到类似这样的配置段中。
2.  **不要直接编辑**：虽然你可以直接修改 .ini 文件，但**强烈不建议**这样做。因为格式非常容易出错（例如，漏掉一个波浪号 `~` 或逗号 `,` 都会导致解析失败）。最好总是通过 **Edit -> Editor Preferences -> Keyboard Shortcuts** 菜单进行可视化设置，这样更安全可靠。
3.  **多个快捷键**：`[UserDefinedChords]` 部分可以包含多个这样的定义，每一条都定义了一个独立的快捷键。

所以，你提供的这段代码，正是UE5编辑器用来实现 **“Shift+ESC 停止游戏”** 功能的配置文件记录。


## 总结
以上来自deepSeek.