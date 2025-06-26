# HarmonyOS Best Practices: Dynamic Component Creation

## Introduction

Hello everyone! I'm Jun Moxiao from the Qinglan Zhuma organization. Today, I'll share an analysis of dynamic component creation in HarmonyOS.

## What Problem Does Dynamic Component Creation Solve?

It addresses the issue of slow component loading by accelerating component rendering and creation.

## What's the Principle Behind It?

According to the official documentation, the ArkUI framework's dynamic UI operations support component pre-creation. This allows developers to create components outside the build lifecycle. Once created, these components can undergo property setting and layout calculations. When used during page loading, this significantly improves page response speed.

As shown in the diagram below, the component pre-creation mechanism utilizes idle time during animation execution to pre-create components and set properties. After the animation ends, properties and layouts are updated, saving component creation time and accelerating page rendering.

![Component Pre-creation Mechanism](https://img-1335237891.cos.ap-shanghai.myqcloud.com/%E9%B8%BF%E8%92%99%E6%9C%80%E4%BD%B3%E5%AE%9E%E8%B7%B5%E4%B9%8B%E7%BB%84%E4%BB%B6%E5%8A%A8%E6%80%81%E5%88%9B%E5%BB%BA%2F1.jpg)

## Dynamic Component Creation Uses FrameNode Custom Nodes

What performance advantages do FrameNode custom nodes offer?

1. **Reduced Custom Component Creation Overhead**: In declarative development paradigms, using ArkUI custom components to define each node in the node tree often leads to inefficient node creation. This is primarily because each node requires memory allocation in the ArkTS engine to store custom components and state variables. During node creation, operations like component ID, component closure, and state variable dependency collection must be performed. In contrast, using ArkUI's [FrameNode](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/arkts-user-defined-arktsnode-framenode) avoids creating custom component objects and state variable objects, eliminating the need for dependency collection, thus significantly improving component creation speed.

2. **Faster Component Updates**: In dynamic layout framework update scenarios, there's typically a UI component tree (TreeA) created from a tree-structured data model (ViewModelA). When updating TreeA with a new data structure (ViewModelB), although declarative development paradigms can achieve data-driven automatic updates, this process involves numerous diff operations, as shown below. For the ArkTS engine, executing diff algorithms on a complex component tree (depth exceeding 30 layers, containing 100-200 components) makes it nearly impossible to maintain full frame rates at 120Hz. However, using ArkUI's FrameNode extension, the framework can independently control the update process, achieving efficient pruning as needed. This is particularly beneficial for dynamic layout frameworks serving specific business needs, enabling rapid update operations.

   ![Component Update Process](https://img-1335237891.cos.ap-shanghai.myqcloud.com/%E9%B8%BF%E8%92%99%E6%9C%80%E4%BD%B3%E5%AE%9E%E8%B7%B5%E4%B9%8B%E7%BB%84%E4%BB%B6%E5%8A%A8%E6%80%81%E5%88%9B%E5%BB%BA%2F2.png)

3. **Direct Component Tree Manipulation**: Declarative development paradigms also face challenges in updating component tree structures. For example, moving a subtree from one child node to another cannot be done by directly adjusting component instance structural relationships—it requires re-rendering the entire component tree. With ArkUI's FrameNode extension, you can easily manipulate the subtree by operating on FrameNode and transplant it to another node, resulting in only local rendering refreshes and better performance.

![Component Tree Manipulation](https://img-1335237891.cos.ap-shanghai.myqcloud.com/%E9%B8%BF%E8%92%99%E6%9C%80%E4%BD%B3%E5%AE%9E%E8%B7%B5%E4%B9%8B%E7%BB%84%E4%BB%B6%E5%8A%A8%E6%80%81%E5%88%9B%E5%BB%BA%2F3.png)

## Implementation Steps

### 1. Create Custom Nodes

```typescript
// Parameter type
class Params {
  text: string
  img: ResourceStr

  constructor(text: string, img: ResourceStr) {
    this.text = text;
    this.img = img;
  }
}
// UI component
@Builder
function buildText(params: Params) {
  Column() {
    Text(params.text)
      .fontSize(20)
    Image(params.img)
      .width(30)
  }
}
```

### 2. Implement NodeController Class

```typescript
class TextNodeController extends NodeController {
  private textNode: BuilderNode<[Params]> | null = null;

  constructor() {
    super();
  }

  makeNode(context: UIContext): FrameNode | null {
    this.textNode = new BuilderNode(context);
    this.textNode.build(wrapBuilder<[Params]>(buildText), new Params('Hello', $r('app.media.startIcon')));
    return this.textNode.getFrameNode();
  }
}
```

**Note**: Use the BuilderNode's build method to construct the component tree. The build() method requires two parameters: the first is the global @Builder method wrapped by wrapBuilder(), and the second is the parameter object required by the @Builder method. If the @Builder method has no parameters or has default parameters, the second parameter of build() can be omitted.

### 3. Display Custom Nodes

```typescript
@Entry
@Component
struct Index {
  textNodeController: TextNodeController = new TextNodeController()

  build() {
    Column() {
      NodeContainer(this.textNodeController)
    }
  }
}
```

**Result**:

![Display Result](https://img-1335237891.cos.ap-shanghai.myqcloud.com/%E9%B8%BF%E8%92%99%E6%9C%80%E4%BD%B3%E5%AE%9E%E8%B7%B5%E4%B9%8B%E7%BB%84%E4%BB%B6%E5%8A%A8%E6%80%81%E5%88%9B%E5%BB%BA%2F4.jpg)

### 4. Dynamically Remove Components

NodeContainer nodes can be removed or displayed using conditional control statements:

```typescript
Column() {
  if (this.isShow) {
    NodeContainer(this.textNodeController)
  }
  Button('Click me')
    .onClick(() => {
      this.isShow = !this.isShow
    })
}
```

### 5. Dynamically Update Components

```typescript
replaceBuilderNode(newNode: BuilderNode<Object[]>) {
  this.textNode = newNode;
  this.rebuild();
}
```

```typescript
@Entry
@Component
struct Index {
  textNodeController: TextNodeController = new TextNodeController()
  @State isShow: boolean = true

  buildNewNode(): BuilderNode<[Params]> {
    let uiContext: UIContext = this.getUIContext();
    let textNode = new BuilderNode<[Params]>(uiContext);
    textNode.build(wrapBuilder<[Params]>(buildText), new Params('Create new node', $r('app.media.app_background')))
    return textNode;
  }

  build() {
    Column() {
      NodeContainer(this.textNodeController)
      Button('Click me')
        .onClick(() => {
          this.textNodeController.replaceBuilderNode(this.buildNewNode());
        })
    }
  }
}
```

**Note**: After clicking, the image will flicker because the node is deleted and then added again. If you use update() to update the custom node, the image flickering won't occur.

**Demo**:

```typescript
update() {
  if (this.textNode !== null) {
    this.textNode.update(new Params('Update', $r('app.media.background')));
  }
}
```

```typescript
@Entry
@Component
struct Index {
  textNodeController: TextNodeController = new TextNodeController()
  @State isShow: boolean = true

  buildNewNode(): BuilderNode<[Params]> {
    let uiContext: UIContext = this.getUIContext();
    let textNode = new BuilderNode<[Params]>(uiContext);
    textNode.build(wrapBuilder<[Params]>(buildText), new Params('Create new node', $r('app.media.app_background')))
    return textNode;
  }

  build() {
    Column() {
      NodeContainer(this.textNodeController)
      Button('Click to create new node')
        .onClick(() => {
          this.textNodeController.replaceBuilderNode(this.buildNewNode());
        })
      Button('Click to update node')
        .onClick(() => {
          this.textNodeController.update()
        })
    }
  }
}
```
        