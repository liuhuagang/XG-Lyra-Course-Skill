# Editor Customization

## ULyraEditorEngine

[ULyraEditorEngine](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraEditor/LyraEditorEngine.h) inherits from `UUnrealEdEngine` and is responsible for Lyra customizations in the editor environment. Located in the `LyraEditor` module.

### FirstTickSetup

Initialization logic executed on the editor's first Tick:

```cpp
void ULyraEditorEngine::FirstTickSetup()
{
    if (bFirstTickSetup)
    {
        return;
    }
    bFirstTickSetup = true;

    // Force display of plugin folders in the content browser
    GetMutableDefault<UContentBrowserSettings>()->SetDisplayPluginFolders(true);
}
```

## SPathView bDisplayPluginFolders

### Background

`SPathView::CreateCompiledFolderFilter()` is the filter creation method for the content browser path view.

### UE 5.0 vs 5.6 Differences

**UE 5.0**: Reads the setting directly from `ContentBrowserSettings->GetDisplayPluginFolders()`.

**UE 5.6**: Adds instance configuration override logic, prioritizing `FContentBrowserInstanceConfig`:

```cpp
bool bDisplayPluginFolders = ContentBrowserSettings->GetDisplayPluginFolders();
if (FContentBrowserInstanceConfig* EditorConfig = GetContentBrowserConfig())
{
    bDisplayPluginFolders = EditorConfig->bShowPluginContent;
}
```

### LyraEditorEngine's Patch

`FirstTickSetup()` forcibly calls `SetDisplayPluginFolders(true)`, ensuring the content browser always displays plugin folders after the project loads. This is a lightweight editor customization approach that changes editor behavior without rewriting editor module code.
