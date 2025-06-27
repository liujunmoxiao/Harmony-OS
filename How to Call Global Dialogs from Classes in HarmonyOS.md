# How to Call Global Dialogs from Classes in HarmonyOS

## Introduction

I'm sure many of you have been using `@CustomDialog` for custom dialogs in your development, writing a bunch of initialization code on each page, and finding it particularly difficult to call from within classes. Today, I'd like to share how to use [`promptAction.openCustomDialog`](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V13/js-apis-promptaction-V13#promptactionopencustomdialog11) to call global dialogs from within classes in HarmonyOS.

## Implementation Guide

### Step 1: Store the Context in EntryAbility (Required)

![Context Storage](https://img-1335237891.cos.ap-shanghai.myqcloud.com/HarmonyOS%E4%B8%AD%E5%A6%82%E4%BD%95%E5%9C%A8%E7%B1%BB%E4%B8%AD%E8%B0%83%E7%94%A8%E5%85%A8%E5%B1%80%E7%9A%84%E5%BC%B9%E7%AA%97%EF%BC%9F%2FPixPin_2025-06-13_15-01-46.png)

```typescript:EntryAbility.ets
onWindowStageCreate(windowStage: window.WindowStage): void {
  windowStage.loadContent('pages/Index', (err) => {
    window.getLastWindow(this.context).then((data: window.Window) => {
      let uiContext = data.getUIContext();
      AppStorage.setOrCreate<UIContext>('UIContext', uiContext);
    });
  });
}
```

### Step 2: Create a Builder (Customize the Style)

![Builder Creation](https://img-1335237891.cos.ap-shanghai.myqcloud.com/HarmonyOS%E4%B8%AD%E5%A6%82%E4%BD%95%E5%9C%A8%E7%B1%BB%E4%B8%AD%E8%B0%83%E7%94%A8%E5%85%A8%E5%B1%80%E7%9A%84%E5%BC%B9%E7%AA%97%EF%BC%9F%2FPixPin_2025-06-13_15-19-55.png)

```typescript:test.ets
export class Params {
  text: ResourceStr = ""

  constructor(text: ResourceStr) {
    this.text = text;
  }
}

@Builder
export function testTextBuilder(params: Params) {
  Column() {
    Text(params.text)
      .textAlign(TextAlign.Center)
      .fontSize(20)
      .borderRadius(6)
      .backgroundColor('rgba(24, 25, 26, 0.85)')
      .fontColor($r('sys.color.comp_background_list_card'))
      .padding(16)
  }
}
```

### Step 3: Define a Global Class

![Global Class Definition](https://img-1335237891.cos.ap-shanghai.myqcloud.com/HarmonyOS%E4%B8%AD%E5%A6%82%E4%BD%95%E5%9C%A8%E7%B1%BB%E4%B8%AD%E8%B0%83%E7%94%A8%E5%85%A8%E5%B1%80%E7%9A%84%E5%BC%B9%E7%AA%97%EF%BC%9F%2FPixPin_2025-06-13_15-21-07.png)

```typescript
export class DialogModel {
  static showDialog(message: string) {
    if (message != '') {
      let timeId: number = -1
      // Get the context
      let uiContext = AppStorage.get('UIContext') as UIContext
      let promptAction = uiContext.getPromptAction();
      let contentNode = new ComponentContent(uiContext, wrapBuilder(testTextBuilder), new Params(message));
      promptAction.openCustomDialog(contentNode, {
        isModal: false,
        onWillAppear: () => {
          timeId = setTimeout(() => {
            promptAction.closeCustomDialog(contentNode)
          }, 2000)
        },
        onDidDisappear: () => {
          clearTimeout(timeId)
        },
        keyboardAvoidMode: KeyboardAvoidMode.NONE,
        showInSubWindow: true,
      })
    }
  }
}
```

### Step 4: Implementation

```typescript
@Entry
@Component
struct Index {
  build() {
    Column() {
      Button('Click me for a dialog')
        .onClick(() => {
          DialogModel.showDialog('Here it is!')
        })
    }
  }
}
```

## Result

![Demo Result](https://img-1335237891.cos.ap-shanghai.myqcloud.com/HarmonyOS%E4%B8%AD%E5%A6%82%E4%BD%95%E5%9C%A8%E7%B1%BB%E4%B8%AD%E8%B0%83%E7%94%A8%E5%85%A8%E5%B1%80%E7%9A%84%E5%BC%B9%E7%AA%97%EF%BC%9F%2FPixPin_2025-06-13_15-22-37.png)

## Limitations

1. The dialog closes when navigating back (not an issue for toast notifications)
2. Continuous clicks will keep showing dialogs (some requirements may need to close the previous dialog when opening a new one)
3. You might want to exit the page stack without closing the dialog   