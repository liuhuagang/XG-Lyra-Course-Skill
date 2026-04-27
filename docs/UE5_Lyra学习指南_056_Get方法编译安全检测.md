# UE5_Lyra学习指南_056_Get方法编译安全检测

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.  
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)  
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)  

文章内容由小刚撰写,采用了以下多种方式:  
1.口述转文字  
2.AI重构  
3.参考引擎源码  
4.Lyra工程源码  
5.结合社区论坛各位大佬的解析  

- [UE5\_Lyra学习指南\_056\_Get方法编译安全检测](#ue5_lyra学习指南_056_get方法编译安全检测)
  - [概述](#概述)
  - [宏定义解析：](#宏定义解析)
    - [分步解释：](#分步解释)
  - [核心作用：](#核心作用)
  - [典型使用场景：](#典型使用场景)
  - [对比直接字符串化的优势：](#对比直接字符串化的优势)
  - [关键点总结：](#关键点总结)


## 概述
在 Unreal Engine 5 (UE5) 中，宏 `GET_FUNCTION_NAME_STRING_CHECKED(ClassName, FunctionName)` 是一个**编译时安全检查工具**，主要用于将成员函数名转换为字符串，同时验证该函数是否真实存在于指定类中。它的设计目的是在保证安全性的同时避免运行时开销。

## 宏定义解析：
```cpp
#define GET_FUNCTION_NAME_STRING_CHECKED(ClassName, FunctionName) \
    ((void)sizeof(&ClassName::FunctionName), TEXT(#FunctionName))
```

### 分步解释：
1. **`&ClassName::FunctionName`**  
   - 尝试获取类 `ClassName` 中成员函数 `FunctionName` 的指针。
   - 如果 `FunctionName` **不存在**或**签名不匹配**，编译器会立即报错（静态检查）。

2. **`sizeof(...)`**  
   - 计算函数指针的大小（编译时操作）。
   - 仅用于通过编译器验证函数的存在性，不生成实际代码。

3. **`(void)sizeof(...)`**  
   - 显式忽略 `sizeof` 的结果（避免编译器警告未使用返回值）。

4. **逗号运算符 `(A, B)`**  
   - 按顺序执行表达式 `A` 和 `B`，但返回表达式 `B` 的结果。
   - 此处 `A` 是安全检查（`(void)sizeof(...)`），`B` 是字符串转换（`TEXT(#FunctionName)`）。

5. **`TEXT(#FunctionName)`**  
   - `#FunctionName`：将传入的符号 `FunctionName` **字符串化**（例如 `MyFunction` → `"MyFunction"`）。
   - `TEXT()`：UE 的字符串宏，确保生成兼容的 Unicode 字符串（如 `L"MyFunction"`）。

---

## 核心作用：
1. **编译时函数存在性验证**  
   - 如果 `ClassName` 中没有 `FunctionName`，**编译失败**（避免运行时崩溃）。
   - 示例：拼写错误（`Render` vs `Rander`）会立即被捕获。

2. **安全获取函数名字符串**  
   - 返回函数名的字符串形式（如 `TEXT("MyFunction")`）。

---

## 典型使用场景：
在 UE5 中常见于需要将函数名作为字符串传递的场合，例如：
```cpp
// 绑定委托时安全获取函数名字符串
FDelegateHandle = MyDelegate.AddUObject(
    MyObject,
    &UMyClass::MyFunction,
    GET_FUNCTION_NAME_STRING_CHECKED(UMyClass, MyFunction) // 作为调试信息
);

// 日志输出（带函数名验证）
UE_LOG(LogTemp, Warning, TEXT("Called %s"), 
    GET_FUNCTION_NAME_STRING_CHECKED(UMyClass, MyFunction));
```

---

## 对比直接字符串化的优势：
| 方法                          | 安全性 | 编译时检查 | 运行时开销 |
|-------------------------------|--------|------------|------------|
| `TEXT("MyFunction")`          | ❌ 易拼写错误 | ❌ 无检查 | 无         |
| `GET_FUNCTION_NAME_STRING_CHECKED` | ✅ 防拼写错误 | ✅ 强制验证 | 无         |

---

## 关键点总结：
- **零运行时开销**：所有操作在编译期完成。
- **防御性编程**：杜绝因函数名拼写错误或 API 变更导致的运行时崩溃。
- **UE 生态集成**：专为 UE 的反射/委托系统设计，尤其适合需要函数名字符串的场景。
- **类型安全**：比直接写字符串更可靠，是 UE 最佳实践之一。

以上回答来自DeepSeek