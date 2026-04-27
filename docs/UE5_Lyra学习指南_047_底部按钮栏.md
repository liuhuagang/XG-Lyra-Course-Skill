# UE5_Lyra学习指南_047_底部按钮

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.  
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)  
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)  

文章内容由小刚撰写,采用了以下多种方式:  
1.口述转文字  
2.AI重构  
3.参考引擎源码  
4.Lyra工程源码  
5.结合社区论坛各位大佬的解析  

- [UE5\_Lyra学习指南\_047\_底部按钮](#ue5_lyra学习指南_047_底部按钮)
	- [概述](#概述)
		- [通过勾选默认回退操作](#通过勾选默认回退操作)
		- [通过注册时表明为底部按钮](#通过注册时表明为底部按钮)
		- [ULyraBoundActionButton](#ulyraboundactionbutton)
	- [总结](#总结)



## 概述
有一些操作需要以固定底部栏的方式进行实现.
UCommonBoundActionBar作为容器.
UCommonBoundActionButton作为按钮.
图片在Word文档中.
W_BottomActionBar.
W_BoundActionButton.
### 通过勾选默认回退操作
蓝图中勾选即可.
注意该固定栏的容器必须要实例化出来.并在容器中指定按钮的类.
Class-Defaults-Back:
--Is Back Handler
--Is Back Action Displayed in Action Bar.

### 通过注册时表明为底部按钮
``` cpp
void ULyraSettingScreen::NativeOnInitialized()
{
	Super::NativeOnInitialized();

	BackHandle = RegisterUIActionBinding(FBindUIActionArgs(BackInputActionData,
		true, FSimpleDelegate::CreateUObject(this, &ThisClass::HandleBackAction)));
	ApplyHandle = RegisterUIActionBinding(FBindUIActionArgs(ApplyInputActionData,
		true, FSimpleDelegate::CreateUObject(this, &ThisClass::HandleApplyAction)));
	CancelChangesHandle = RegisterUIActionBinding(FBindUIActionArgs(CancelChangesInputActionData,
		true, FSimpleDelegate::CreateUObject(this, &ThisClass::HandleCancelChangesAction)));
}
```
### ULyraBoundActionButton
用于监听输入方法改变时切换按钮样式
``` cpp
/**
 * 底部按钮
 * 用以切换按钮的样式,在输入方法改变时.
 */
UCLASS(MinimalAPI, Abstract, meta = (DisableNativeTick))
class ULyraBoundActionButton : public UCommonBoundActionButton
{
	GENERATED_BODY()
	
protected:
	UE_API virtual void NativeConstruct() override;

private:
	void HandleInputMethodChanged(ECommonInputType NewInputMethod);

	UPROPERTY(EditAnywhere, Category = "Styles")
	TSubclassOf<UCommonButtonStyle> KeyboardStyle;

	UPROPERTY(EditAnywhere, Category = "Styles")
	TSubclassOf<UCommonButtonStyle> GamepadStyle;

	UPROPERTY(EditAnywhere, Category = "Styles")
	TSubclassOf<UCommonButtonStyle> TouchStyle;
};

```

## 总结
因为比较简单,直接通过蓝图配置使用即可.