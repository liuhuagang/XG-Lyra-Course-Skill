# UE5_Lyra学习指南_044_LyraButton

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.  
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)  
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)  

文章内容由小刚撰写,采用了以下多种方式:  
1.口述转文字  
2.AI重构  
3.参考引擎源码  
4.Lyra工程源码  
5.结合社区论坛各位大佬的解析  

- [UE5\_Lyra学习指南\_044\_LyraButton](#ue5_lyra学习指南_044_lyrabutton)
	- [概述](#概述)
	- [一次点击事件](#一次点击事件)
		- [Slate底层注册](#slate底层注册)
		- [SlateWidget](#slatewidget)
		- [外部调用](#外部调用)
		- [FGenericApplicationMessageHandler](#fgenericapplicationmessagehandler)
	- [UCommonButtonInternalBase](#ucommonbuttoninternalbase)
	- [UCommonButtonBase](#ucommonbuttonbase)
	- [可视化](#可视化)
		- [更新文本值](#更新文本值)
		- [更新Icon](#更新icon)
	- [代码](#代码)
		- [LyraButton](#lyrabutton)
		- [LyraActionWidget](#lyraactionwidget)
	- [总结](#总结)



## 概述
这节注意讲解了LyraButton和LyraActionWidget的应用.


## 一次点击事件

调用堆栈

``` txt
SButton::ExecuteOnClick() SButton.cpp:492
SButton::OnMouseButtonUp(const FGeometry &, const FPointerEvent &) SButton.cpp:419
SCommonButton::OnMouseButtonUp(const FGeometry &, const FPointerEvent &) CommonButtonTypes.cpp:65
??R<lambda_2>@?7??RoutePointerUpEvent@FSlateApplication@@QEAA?AVFReply@@AEBVFWidgetPath@@AEBUFPointerEvent@@@Z@QEBA@AEBVFArrangedWidget@@1@Z(const FArrangedWidget &,const FPointerEvent &) SlateApplication.cpp:5367
??$Route@VFReply@@VFToLeafmostPolicy@FEventRouter@@UFPointerEvent@@V<lambda_2>@?7??RoutePointerUpEvent@FSlateApplication@@QEAA?AV1@AEBVFWidgetPath@@AEBU4@@Z@@FEventRouter@@SA?AVFReply@@PEAVFSlateApplication@@VFToLeafmostPolicy@0@UFPointerEvent@@AEBV<lambda_2>@?7??RoutePointerUpEvent@2@QEAA?AV1@AEBVFWidgetPath@@AEBU4@@Z@W4ESlateDebuggingInputEvent@@@Z(FSlateApplication *,FToLeafmostPolicy,FPointerEvent,const <lambda_2> &,ESlateDebuggingInputEvent) SlateApplication.cpp:456
FSlateApplication::RoutePointerUpEvent(const FWidgetPath &, const FPointerEvent &) SlateApplication.cpp:5353
FSlateApplication::ProcessMouseButtonUpEvent(const FPointerEvent &) SlateApplication.cpp:5938
FSlateApplication::OnMouseUp(Type, TVector2<…>) SlateApplication.cpp:5894
FWindowsApplication::ProcessDeferredMessage(const FDeferredWindowsMessage &) WindowsApplication.cpp:3052
FWindowsApplication::DeferMessage(TSharedPtr<…> &, HWND__ *, unsigned int, unsigned long long, long long, int, int, unsigned int) WindowsApplication.cpp:3570
FWindowsApplication::ProcessMessage(HWND__ *, unsigned int, unsigned long long, long long) WindowsApplication.cpp:2725
[内联] WindowsApplication_WndProc(HWND__ *, unsigned int, unsigned long long, long long) WindowsApplication.cpp:1716
FWindowsApplication::AppWndProc(HWND__ *, unsigned int, unsigned long long, long long) WindowsApplication.cpp:1721
[内联] WinPumpMessages() WindowsPlatformApplicationMisc.cpp:116
FWindowsPlatformApplicationMisc::PumpMessages(bool) WindowsPlatformApplicationMisc.cpp:145
FEngineLoop::Tick() LaunchEngineLoop.cpp:5548
[内联] EngineTick() Launch.cpp:60
GuardedMain(const wchar_t *) Launch.cpp:189
LaunchWindowsStartup(HINSTANCE__ *, HINSTANCE__ *, char *, int, const wchar_t *) LaunchWindows.cpp:271
WinMain(HINSTANCE__ *, HINSTANCE__ *, char *, int) LaunchWindows.cpp:339

```

### Slate底层注册

这里直接看SButton即可.SCommonButton是其子类
``` cpp
FReply SCommonButton::OnMouseButtonDown(const FGeometry& MyGeometry, const FPointerEvent& MouseEvent)
{
	return IsInteractable() ? SButton::OnMouseButtonDown(MyGeometry, MouseEvent) : FReply::Handled();
}

```
``` cpp
FReply SButton::OnMouseButtonDown( const FGeometry& MyGeometry, const FPointerEvent& MouseEvent )
{
	FReply Reply = FReply::Unhandled();
	if (IsEnabled() && (MouseEvent.GetEffectingButton() == EKeys::LeftMouseButton || MouseEvent.IsTouchEvent()))
	{
		Press();
		PressedScreenSpacePosition = MouseEvent.GetScreenSpacePosition();

		EButtonClickMethod::Type InputClickMethod = GetClickMethodFromInputType(MouseEvent);
		
		if(InputClickMethod == EButtonClickMethod::MouseDown)
		{
			//get the reply from the execute function
			Reply = ExecuteOnClick();

			//You should ALWAYS handle the OnClicked event.
			ensure(Reply.IsEventHandled() == true);
		}
		else if (InputClickMethod == EButtonClickMethod::PreciseClick)
		{
			// do not capture the pointer for precise taps or clicks
			// 
			Reply = FReply::Handled();
		}
		else
		{
			//we need to capture the mouse for MouseUp events
			Reply = FReply::Handled().CaptureMouse( AsShared() );
		}
	}

	//return the constructed reply
	return Reply;
}


```
``` cpp
FReply SButton::ExecuteOnClick()
{
	PlayClickedSound();
	if (OnClicked.IsBound())
	{
		FReply Reply = OnClicked.Execute();
#if WITH_ACCESSIBILITY
		// @TODOAccessibility: This should pass the Id of the user that clicked the button but we don't want to change the regular Slate API just yet
		FSlateApplicationBase::Get().GetAccessibleMessageHandler()->OnWidgetEventRaised(FSlateAccessibleMessageHandler::FSlateWidgetAccessibleEventArgs(AsShared(), EAccessibleEvent::Activate));
#endif
		return Reply;
	}
	else
	{
		return FReply::Handled();
	}
}
```

这里的OnClicked是构造的时候传递进来的!
``` cpp
TSharedRef<SWidget> UCommonButtonInternalBase::RebuildWidget()
{
	MyButton = MyCommonButton = SNew(SCommonButton)
		.OnClicked(BIND_UOBJECT_DELEGATE(FOnClicked, SlateHandleClickedOverride))
}
```
``` cpp
class SCommonButton : public SButton
{
public:
	SLATE_BEGIN_ARGS(SCommonButton)
		: _Content()
		, _HAlign(HAlign_Fill)
		, _VAlign(VAlign_Fill)
		, _ClickMethod(EButtonClickMethod::DownAndUp)
		, _TouchMethod(EButtonTouchMethod::DownAndUp)
		, _PressMethod(EButtonPressMethod::DownAndUp)
		, _IsFocusable(true)
		, _IsInteractionEnabled(true)
	{}
	SLATE_DEFAULT_SLOT(FArguments, Content)
		SLATE_STYLE_ARGUMENT(FButtonStyle, ButtonStyle)
		SLATE_ARGUMENT(EHorizontalAlignment, HAlign)
		SLATE_ARGUMENT(EVerticalAlignment, VAlign)
		SLATE_EVENT(FOnClicked, OnClicked)
		SLATE_EVENT(FOnClicked, OnDoubleClicked)
		SLATE_EVENT(FSimpleDelegate, OnPressed)
		SLATE_EVENT(FSimpleDelegate, OnReleased)
		SLATE_ARGUMENT(EButtonClickMethod::Type, ClickMethod)
		SLATE_ARGUMENT(EButtonTouchMethod::Type, TouchMethod)
		SLATE_ARGUMENT(EButtonPressMethod::Type, PressMethod)
		SLATE_ARGUMENT(bool, IsFocusable)
		SLATE_EVENT(FSimpleDelegate, OnReceivedFocus)
		SLATE_EVENT(FSimpleDelegate, OnLostFocus)

		/** Is interaction enabled? */
		SLATE_ARGUMENT(bool, IsButtonEnabled)
		SLATE_ARGUMENT(bool, IsInteractionEnabled)
		SLATE_END_ARGS()

	void Construct(const FArguments& InArgs)
	{
		OnDoubleClicked = InArgs._OnDoubleClicked;

		SButton::Construct(SButton::FArguments()
			.ButtonStyle(InArgs._ButtonStyle)
			.HAlign(InArgs._HAlign)
			.VAlign(InArgs._VAlign)
			.ClickMethod(InArgs._ClickMethod)
			.TouchMethod(InArgs._TouchMethod)
			.PressMethod(InArgs._PressMethod)
			.OnClicked(InArgs._OnClicked)
			.OnPressed(InArgs._OnPressed)
			.OnReleased(InArgs._OnReleased)
			.IsFocusable(InArgs._IsFocusable)
			.Content()
			[
				InArgs._Content.Widget
			]);
			//....

	}
}

```
### SlateWidget

这个时候直接看顶层父类
``` cpp

/**
 * Abstract base class for Slate widgets.
 *
 * STOP. DO NOT INHERIT DIRECTLY FROM WIDGET!
 *
 * Inheritance:
 *   Widget is not meant to be directly inherited. Instead consider inheriting from LeafWidget or Panel,
 *   which represent intended use cases and provide a succinct set of methods which to override.
 *
 *   SWidget is the base class for all interactive Slate entities. SWidget's public interface describes
 *   everything that a Widget can do and is fairly complex as a result.
 * 
 * Events:
 *   Events in Slate are implemented as virtual functions that the Slate system will call
 *   on a Widget in order to notify the Widget about an important occurrence (e.g. a key press)
 *   or querying the Widget regarding some information (e.g. what mouse cursor should be displayed).
 *
 *   Widget provides a default implementation for most events; the default implementation does nothing
 *   and does not handle the event.
 *
 *   Some events are able to reply to the system by returning an FReply, FCursorReply, or similar
 *   object. 
 */
 /**
* 用于 Slate 组件的抽象基类。*
* 停止。切勿直接从组件中获取！*
* 继承：
*   Widget 并非旨在直接进行继承。相反，建议从 LeafWidget 或 Panel 类中进行继承，因为这些类代表了预期的使用场景，并提供了需要重写的简洁方法集。*
*SWidget 是所有交互式 Slate 实体的基类。SWidget 的公共接口描述了 Widget 所能执行的所有操作，因此其功能相当复杂。**
* 事件：
*   斯莱特中的事件是以虚拟函数的形式实现的，斯莱特系统会调用这些函数来通知某个控件有关重要事件的发生（例如按键操作），
*   或者向该控件询问一些信息（例如应显示什么样的鼠标光标）。*
*  Widget 为大多数事件提供了默认的实现方式；默认实现方式不会执行任何操作，也不会处理该事件。*
有些事件能够通过返回 FReply、FCursorReply 或类似的对象来向系统作出回应。*/
class SWidget
	: public FSlateControlledConstruction,
	public TSharedFromThis<SWidget>		// Enables 'this->AsShared()'
{
	/** See OnMouseButtonDown event */
	SLATECORE_API void SetOnMouseButtonDown(FPointerEventHandler EventHandler);

	/**
	 * The system calls this method to notify the widget that a mouse button was pressed within it. This event is bubbled.
	 *
	 * @param MyGeometry The Geometry of the widget receiving the event
	 * @param MouseEvent Information about the input event
	 * @return Whether the event was handled along with possible requests for the system to take action.
	 */
/**
系统会调用此方法来通知该控件，告知其鼠标按钮已在该控件内部被按下。此事件会进行传播。*
* @参数 MyGeometry：接收事件的控件的几何信息
* @参数 MouseEvent：有关输入事件的详细信息
* @返回 声明该事件是否已被处理，以及是否可能请求系统采取相应行动。*/
	SLATECORE_API virtual FReply OnMouseButtonDown(const FGeometry& MyGeometry, const FPointerEvent& MouseEvent);
}

```
现在主要关注SetOnMouseButtonDown的调用位置!

``` cpp
void SWidget::SetOnMouseButtonDown(FPointerEventHandler EventHandler)
{
	Private::FindOrAddMouseEventsMetaData(this)->MouseButtonDownHandle = EventHandler;
}


```

``` cpp
	/** Metadata associated with this widget. */
	TArray<TSharedRef<ISlateMetaData>> MetaData;

namespace Private
{
	TSharedPtr<FSlateMouseEventsMetaData> FindOrAddMouseEventsMetaData(SWidget* Widget)
	{
		TSharedPtr<FSlateMouseEventsMetaData> Data = Widget->GetMetaData<FSlateMouseEventsMetaData>();
		if (!Data)
		{
			Data = MakeShared<FSlateMouseEventsMetaData>();
			Widget->AddMetadata(Data.ToSharedRef());
		}
		return Data;
	}
}

```


``` cpp
FReply SWidget::OnMouseButtonDown(const FGeometry& MyGeometry, const FPointerEvent& MouseEvent)
{
	if (TSharedPtr<FSlateMouseEventsMetaData> Data = GetMetaData<FSlateMouseEventsMetaData>())
	{
		if (Data->MouseButtonDownHandle.IsBound() )
		{
			return Data->MouseButtonDownHandle.Execute(MyGeometry, MouseEvent);
		}
	}
	return FReply::Unhandled();
}
```


### 外部调用
``` cpp
bool FSlateApplication::ProcessMouseButtonDownEvent( const TSharedPtr< FGenericWindow >& PlatformWindow, const FPointerEvent& MouseEvent )
{

	// ...
	TempReply = InMouseCaptorWidget.Widget->OnMouseButtonDown(InMouseCaptorWidget.Geometry, Event);


}

```
``` cpp

FReply FSlateApplication::RoutePointerUpEvent(const FWidgetPath& WidgetsUnderPointer, const FPointerEvent& PointerEvent)
{
	TempReply = TargetWidget.Widget->OnMouseButtonUp( TargetWidget.Geometry, Event );

	// ...
}

```

### FGenericApplicationMessageHandler

``` cpp
class FGenericApplicationMessageHandler
{
public:
	virtual bool OnMouseDown( const TSharedPtr< FGenericWindow >& Window, const EMouseButtons::Type Button )
	{
		return false;
	}

	virtual bool OnMouseDown( const TSharedPtr< FGenericWindow >& Window, const EMouseButtons::Type Button, const FVector2D CursorPos )
	{
		return false;
	}

	virtual bool OnMouseUp( const EMouseButtons::Type Button )
	{
		return false;
	}

	virtual bool OnMouseUp( const EMouseButtons::Type Button, const FVector2D CursorPos )
	{
		return false;
	}
}

```

``` cpp
bool FSlateApplication::OnMouseUp( const EMouseButtons::Type Button, const FVector2D CursorPos )
{
	// convert left mouse click to touch event if we are faking it	
	if (IsFakingTouchEvents() && Button == EMouseButtons::Left)
	{
		bIsFakingTouched = false;
		return OnTouchEnded(PlatformApplication->Cursor->GetPosition(), 0, FSlateApplicationBase::SlateAppPrimaryPlatformUser, IPlatformInputDeviceMapper::Get().GetDefaultInputDevice());
	}

	FKey Key = TranslateMouseButtonToKey( Button );

	FPointerEvent MouseEvent(
		GetUserIndexForMouse(),
		CursorPointerIndex,
		CursorPos,
		GetLastCursorPos(),
		PressedMouseButtons,
		Key,
		0,
		PlatformApplication->GetModifierKeys()
		);

	return ProcessMouseButtonUpEvent( MouseEvent );
}

```



注意UCommonButtonBase里面还包了一个UCommonButtonInternalBase.
我们主要以点击事件为例简单介绍以下父类的这个控件
## UCommonButtonInternalBase
``` cpp
/** Custom UButton override that allows us to disable clicking without disabling the widget entirely */
/** 自定义 UButton 的重写版本，使我们能够禁用点击功能而不完全禁用该组件 */
UCLASS(Experimental, MinimalAPI)	// "Experimental" to hide it in the designer
class UCommonButtonInternalBase : public UButton
{
	GENERATED_UCLASS_BODY()

public:
	//....

protected:
	// UWidget interface
	COMMONUI_API virtual TSharedRef<SWidget> RebuildWidget() override;
	COMMONUI_API virtual void ReleaseSlateResources(bool bReleaseChildren) override;
	// End of UWidget interface

	COMMONUI_API virtual FReply SlateHandleClickedOverride();


	/** Cached pointer to the underlying slate button owned by this UWidget */
	TSharedPtr<class SCommonButton> MyCommonButton;
};

```

``` cpp
TSharedRef<SWidget> UCommonButtonInternalBase::RebuildWidget()
{
	MyButton = MyCommonButton = SNew(SCommonButton)
		.OnClicked(BIND_UOBJECT_DELEGATE(FOnClicked, SlateHandleClickedOverride))
		.OnDoubleClicked(BIND_UOBJECT_DELEGATE(FOnClicked, SlateHandleDoubleClicked))
		.OnPressed(BIND_UOBJECT_DELEGATE(FSimpleDelegate, SlateHandlePressedOverride))
		.OnReleased(BIND_UOBJECT_DELEGATE(FSimpleDelegate, SlateHandleReleasedOverride))
		.ButtonStyle(&GetStyle())
		.ClickMethod(GetClickMethod())
		.TouchMethod(GetTouchMethod())
		.IsFocusable(GetIsFocusable())
		.IsButtonEnabled(bButtonEnabled)
		.IsInteractionEnabled(bInteractionEnabled)
		.OnReceivedFocus(BIND_UOBJECT_DELEGATE(FSimpleDelegate, SlateHandleOnReceivedFocus))
		.OnLostFocus(BIND_UOBJECT_DELEGATE(FSimpleDelegate, SlateHandleOnLostFocus));

	MyBox = SNew(SBox)
		.MinDesiredWidth(MinWidth)
		.MinDesiredHeight(MinHeight)
		.MaxDesiredWidth(MaxWidth > 0 ? MaxWidth : FOptionalSize())
		.MaxDesiredHeight(MaxHeight > 0 ? MaxHeight : FOptionalSize())
		.HAlign(HAlign_Fill)
		.VAlign(VAlign_Fill)
		[
			MyCommonButton.ToSharedRef()
		];

	if (GetChildrenCount() > 0)
	{
		Cast<UButtonSlot>(GetContentSlot())->BuildSlot(MyCommonButton.ToSharedRef());
	}

	return MyBox.ToSharedRef();
}
```

``` cpp
FReply UCommonButtonInternalBase::SlateHandleClickedOverride()
{
	return Super::SlateHandleClicked();
}

```

``` cpp
FReply UButton::SlateHandleClicked()
{
	OnClicked.Broadcast();

	return FReply::Handled();
}
```

## UCommonButtonBase
UCommonButtonBase持有UCommonButtonInternalBase.
并将自己的点击处理绑定在UCommonButtonInternalBase身上!!!
``` cpp
bool UCommonButtonBase::Initialize()
{
	const bool bInitializedThisCall = Super::Initialize();

	if (bInitializedThisCall)
	{
		UCommonButtonInternalBase* RootButtonRaw = ConstructInternalButton();

		RootButtonRaw->SetClickMethod(ClickMethod);
		RootButtonRaw->SetTouchMethod(TouchMethod);
		RootButtonRaw->SetPressMethod(PressMethod);
		//Force the RootButton to not be focusable if it has a DesiredFocusWidgetName set which was stealing the focus and preventing DesiredFocusWidget from getting the FocusReceived event.
		RootButtonRaw->SetButtonFocusable(GetDesiredFocusWidgetName().IsNone() && IsFocusable());
		RootButtonRaw->SetButtonEnabled(bButtonEnabled);
		RootButtonRaw->SetInteractionEnabled(bInteractionEnabled);
		RootButton = RootButtonRaw;

		if (WidgetTree->RootWidget)
		{
			UButtonSlot* NewSlot = Cast<UButtonSlot>(RootButtonRaw->AddChild(WidgetTree->RootWidget));
			NewSlot->SetPadding(FMargin());
			NewSlot->SetHorizontalAlignment(EHorizontalAlignment::HAlign_Fill);
			NewSlot->SetVerticalAlignment(EVerticalAlignment::VAlign_Fill);
			WidgetTree->RootWidget = RootButtonRaw;

			RootButton->OnClicked.AddUniqueDynamic(this, &UCommonButtonBase::HandleButtonClicked);
			RootButton->HandleDoubleClicked.BindUObject(this, &UCommonButtonBase::HandleButtonDoubleClicked);
			RootButton->OnReceivedFocus.BindUObject(this, &UCommonButtonBase::HandleFocusReceived);
			RootButton->OnLostFocus.BindUObject(this, &UCommonButtonBase::HandleFocusLost);
			RootButton->OnPressed.AddUniqueDynamic(this, &UCommonButtonBase::HandleButtonPressed);
			RootButton->OnReleased.AddUniqueDynamic(this, &UCommonButtonBase::HandleButtonReleased);
		}
	}

	return bInitializedThisCall;
}


```
处理自己的点击逻辑
``` cpp
void UCommonButtonBase::HandleButtonClicked()
{
	// Since the button enabled state is part of UCommonButtonBase, UButton::OnClicked can be fired while this button is not interactable.
	// Guard against this case.
	if (IsInteractionEnabled())
	{
		// @TODO: Current click rejection method relies on click hold time, this can be refined. See NativeOnHoldProgress.
		// Also gamepad can indirectly trigger this method, so don't guard against pressed
		if (bRequiresHold && CurrentHoldProgress < 1.f)
		{
			return;
		}

		if (bTriggerClickedAfterSelection)
		{
			SetIsSelected(!bSelected, false);
			NativeOnClicked();
		}
		else
		{
			NativeOnClicked();
			SetIsSelected(!bSelected, false);
		}

		ExecuteTriggeredInput();
		HoldReset();
	}
}
```
``` cpp

void UCommonButtonBase::NativeOnClicked()
{
	if (!GetLocked())
	{
		BP_OnClicked();
		OnClicked().Broadcast();
		BroadcastFieldValueChanged(FFieldNotificationClassDescriptor::ClickEvent);
		if (OnButtonBaseClicked.IsBound())
		{
			OnButtonBaseClicked.Broadcast(this);
		}

		FString ButtonName, ABTestName, ExtraData;
		if (GetButtonAnalyticInfo(ButtonName, ABTestName, ExtraData))
		{
			UCommonUISubsystemBase* CommonUISubsystem = GetUISubsystem();
			if (GetGameInstance())
			{
				check(CommonUISubsystem);

				CommonUISubsystem->FireEvent_ButtonClicked(ButtonName, ABTestName, ExtraData);
			}
		}
	}
	else
	{
		BP_OnLockClicked();
		OnLockClicked().Broadcast();
		if (OnButtonBaseLockClicked.IsBound())
		{
			OnButtonBaseLockClicked.Broadcast(this);
		}
	}
}

```


## 可视化
``` cpp
UCLASS(Abstract, Blueprintable, MinimalAPI, ClassGroup = UI, meta = (Category = "Common UI", DisableNativeTick))
class UCommonButtonBase : public UCommonUserWidget
{
	/**
	 * Optionally bound widget for visualization behavior of an input action;
	 * NOTE: If specified, will visualize according to the following algorithm:
	 * If TriggeringEnhancedInputAction is specified, visualize it else:
	 * If TriggeringInputAction is specified, visualize it else:
	 * If TriggeredInputAction is specified, visualize it else:
	 * Visualize the default click action while hovered
	 */
	/**
	* 可选的用于输入操作可视化行为的组件；
	* 注意：如果指定，则将按照以下算法进行可视化：
	* 如果指定了“触发增强型输入操作”，则进行可视化，否则：
	* 如果指定了“触发输入操作”，则进行可视化，否则：
	* 如果指定了“触发的输入操作”，则进行可视化，否则：
	* 在鼠标悬停时可视化默认点击操作。*/
	UPROPERTY(BlueprintReadOnly, Category = Input, meta = (BindWidget, OptionalWidget = true, AllowPrivateAccess = true))
	TObjectPtr<UCommonActionWidget> InputActionWidget;
}
```
### 更新文本值
``` cpp
void ULyraButtonBase::RefreshButtonText()
{
	// 如果现在值是空的或者需要重写
	if (bOverride_ButtonText || ButtonText.IsEmpty())
	{
		// 读取输入控件的值
		if (InputActionWidget)
		{
			const FText ActionDisplayText = InputActionWidget->GetDisplayText();	
			if (!ActionDisplayText.IsEmpty())
			{
				UpdateButtonText(ActionDisplayText);
				return;
			}
		}
	}
	
	UpdateButtonText(ButtonText);	
}

```
### 更新Icon
``` cpp
// If there is an Enhanced Input action associated with this widget, then search for any
	// keys bound to that action and display those instead of the default data table settings.
	// This covers the case of when a player has rebound a key to something else
	
	// 如果此控件与增强输入操作相关联，那么就查找与该操作绑定的任何按键，并使用这些按键来显示数据表内容，而不是默认的数据表设置。
	// 这涵盖了玩家将某个按键重新绑定到其他操作的情况。
	if (AssociatedInputAction)
	{
		if (const UEnhancedInputLocalPlayerSubsystem* EnhancedInputSubsystem = GetEnhancedInputSubsystem())
		{
			TArray<FKey> BoundKeys = EnhancedInputSubsystem->QueryKeysMappedToAction(AssociatedInputAction);
			FSlateBrush SlateBrush;

			const UCommonInputSubsystem* CommonInputSubsystem = GetInputSubsystem();
			if (!BoundKeys.IsEmpty()
				&& CommonInputSubsystem
				&& UCommonInputPlatformSettings::Get()
				->TryGetInputBrush(SlateBrush, BoundKeys[0], CommonInputSubsystem->GetCurrentInputType(), CommonInputSubsystem->GetCurrentGamepadName()))
			{
				return SlateBrush;
			}
		}
	}
	
	return Super::GetIcon();

```
``` cpp
FSlateBrush UCommonActionWidget::GetIcon() const
{
	if (!IsDesignTime())
	{
		if (const UCommonInputSubsystem* CommonInputSubsystem = GetInputSubsystem())
		{
			return EnhancedInputAction && CommonUI::IsEnhancedInputSupportEnabled()
				? CommonUI::GetIconForEnhancedInputAction(CommonInputSubsystem, EnhancedInputAction)
				: CommonUI::GetIconForInputActions(CommonInputSubsystem, InputActions);
		}
	}
}
```

``` cpp
FSlateBrush CommonUI::GetIconForInputActions(const UCommonInputSubsystem* CommonInputSubsystem, const TArray<FDataTableRowHandle>& InputActions)
{
	TArray<FKey> Keys;
	for (const FDataTableRowHandle& InputAction : InputActions)
	{
		if (const FCommonInputActionDataBase* InputActionData = GetInputActionData(InputAction))
		{
			const FCommonInputTypeInfo& TypeInfo = InputActionData->GetCurrentInputTypeInfo(CommonInputSubsystem);
			Keys.Add(TypeInfo.GetKey());
		}
		else
		{
			return *FStyleDefaults::GetNoBrush();
		}
	}

	FSlateBrush SlateBrush;
	if (UCommonInputPlatformSettings::Get()->TryGetInputBrush(SlateBrush, Keys, CommonInputSubsystem->GetCurrentInputType(), CommonInputSubsystem->GetCurrentGamepadName()))
	{
		return SlateBrush;
	}

	return *FStyleDefaults::GetNoBrush();
}

```
``` cpp
bool UCommonInputPlatformSettings::TryGetInputBrush(FSlateBrush& OutBrush, const TArray<FKey>& Keys, ECommonInputType InputType, const FName GamepadName) const
{
	InitializeControllerData();

	for (const UCommonInputBaseControllerData* DefaultControllerData: GetControllerDataForInputType(InputType, GamepadName))
	{
		if (DefaultControllerData->TryGetInputBrush(OutBrush, Keys))
		{
			return true;
		}
	}

	return false;
}
```
在这里我们拿到我们配置的图片 修改了这个小控件的输入图标!
``` cpp
bool UCommonInputBaseControllerData::TryGetInputBrush(FSlateBrush& OutBrush, const TArray<FKey>& Keys) const
{
	if (Keys.Num() == 0)
	{
		return false;
	}

	if (Keys.Num() == 1)
	{
		return CommonUIUtils::TryGetInputBrushFromDataMap(OutBrush, Keys[0], InputBrushDataMap);
	}

	return CommonUIUtils::TryGetInputBrushFromKeySets(OutBrush, Keys, InputBrushKeySets);
}
```
## 代码
### LyraButton
``` cpp
/**
 * Lyra项目所使用的按钮基类
 * 该按钮在不处于激活状态时会自动隐藏。此外，如果与显示平台特定图标相关联，还会为通用操作部件框更新操作行为。
 * 
 */
UCLASS(Abstract, BlueprintType, Blueprintable)
class ULyraButtonBase : public UCommonButtonBase
{
	GENERATED_BODY()

public:
	UFUNCTION(BlueprintCallable)
	void SetButtonText(const FText& InText);
	
protected:
	// UUserWidget interface
	virtual void NativePreConstruct() override;
	// End of UUserWidget interface

	// UCommonButtonBase interface
	/** 用于根据按钮的状态更新相关输入操作控件（如果有）的辅助函数 */
	virtual void UpdateInputActionWidget() override;
	
	/** 当输入法发生变化时通过委托机制调用此方法 */
	virtual void OnInputMethodChanged(ECommonInputType CurrentInputType) override;
	// End of UCommonButtonBase interface

	void RefreshButtonText();

	// 交给蓝图子类重写文本
	UFUNCTION(BlueprintImplementableEvent)
	void UpdateButtonText(const FText& InText);

	// 交给蓝图子类重写风格
	UFUNCTION(BlueprintImplementableEvent)
	void UpdateButtonStyle();
	
private:
	
	// 提供按钮重写文本
	UPROPERTY(EditAnywhere, Category="Button", meta=(InlineEditConditionToggle))
	uint8 bOverride_ButtonText : 1;

	// 重写所使用的文本 受到bOverride_ButtonText的限制
	UPROPERTY(EditAnywhere, Category="Button", meta=( editcondition="bOverride_ButtonText" ))
	FText ButtonText;
};


```
### LyraActionWidget
``` cpp
/** An action widget that will get the icon of key that is currently assigned to the common input action on this widget */
/** 一个操作控件，它会获取当前分配给此控件通用输入操作的键的图标 */
UCLASS(BlueprintType, Blueprintable)
class ULyraActionWidget : public UCommonActionWidget
{
	GENERATED_BODY()

public:

	//~ Begin UCommonActionWidget interface
	// 通过我们在项目里面配置的ContronlData来实现输入图标的变化
	virtual FSlateBrush GetIcon() const override;
	//~ End of UCommonActionWidget interface

	/** The Enhanced Input Action that is associated with this Common Input action. */
	/** 与此通用输入操作相关联的增强型输入操作。*/
	UPROPERTY(BlueprintReadOnly, EditAnywhere)
	const TObjectPtr<UInputAction> AssociatedInputAction;

private:

	UEnhancedInputLocalPlayerSubsystem* GetEnhancedInputSubsystem() const;
	
};
```
## 总结
这节我们编写C++层面关于Lyra按钮的通用设置.它可以重写文字,样式,映射不同平台的输入图标等功能.
在蓝图中需要配置关于该按钮更多的拓展功能.