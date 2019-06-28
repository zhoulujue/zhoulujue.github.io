---
layout: post
title: 从源码了解Flutter的渲染基础：Widget/Element/RenderObject
---

如果你要使用Flutter开始构建一个页面，那么你可能从Widget开始一层一层搭建。

但是你曾经想过这些Widget是如何绘制到屏幕上的吗？很多同学应该会熟悉Flutter的三个树形结构：
* Widget Tree
* Element Tree
* RenderObject Tree

站在Flutter程序员的角度，打交道最多的应该是Widget了，那么下面我带领大家从Flutter源码读起来看一下一个Widget是如何绘制到屏幕上的。

## Opacity
首先我们来回忆一个熟悉的Widget：`Opacity`。

这个Widget一般用来给一个布局加上透明度，它只有一个参数 `opacity`，1.0表示完全不透明，0.0表示完全透明。

在源码里可以看到 Opacity 的继承关系：

> Opacity -> SingleChildRenderObjectWidget -> RenderObject -> Widget

我们比较一下最简单的 StatelessWidget/StatefulWidget :

> StatelessWidget -> Widget

> StatefulWidget -> Widget

可以看到`Opacity`多了`SingleChildRenderObjectWidget`和`RenderObject`这层关系，那这层关系到底是干什么的呢？我们接着往下看源码。

### createRenderObject 方法
通过看`RenderObject`的代码，我们知道只要继承了 `RenderObject`就需要实现`createRenderObject`方法。

```dart
@override
  RenderOpacity createRenderObject(BuildContext context) {
    return RenderOpacity(
      opacity: opacity,
      alwaysIncludeSemantics: alwaysIncludeSemantics,
    );
  }
```

可以发现其实最终返回了一个`RenderOpacity`

### RenderOpacity
大概看一下`RenderOpacity`的代码，可以知道这里实现了最后的所谓的“绘制”也就是`paint`的代码：

```dart
void paint(PaintingContext context, Offset offset) {
    if (child != null) {
      if (_alpha == 0) {
        return;
      }
      if (_alpha == 255) {
        context.paintChild(child, offset);
        return;
      }
      assert(needsCompositing);
      context.pushOpacity(offset, _alpha, super.paint);
    }
  }
```
这里的`PaintingContext`实际上是个Canvas，所以最重要的代码就是`context.pushOpacity`这个方法的调用，最终用上了 `_alpha`，而这个`_alpha`由最上层`Opacity`空间的唯一参数`opacity`计算而来。

所以我们来总结一下这里面的关系：

* `Opacity`直接继承`SingleChildRenderObjectWidget`
* `Widget`**不直接参与绘制**，仅仅存储绘制需要的信息
* `RenderOpacity` 在这里承担了实际的 layout/render 逻辑

### 小结
我们日常所用的Widget仅仅是一些配置，RenderObject做了实际的 layout/render 等工作。这一点和Android的 `View`、iOS的 `UIView` 有本质的区别，请注意。

在Flutter中，只要是`build()`方法被调用了，就会创建一堆Widget，这个是OK的，上面说过Widget仅仅是配置文件，所以即使频繁地 **创建/重建** 它们，是不会带来界面的刷新的。

回想一下，如果你要在Flutter里创建一个动画，一般来说，需要在指定的Widget之上套上一层XXXTransition，然后动画开始后用`animationController`来不断地让Widget刷新重建，但是整个页面不会因此重新刷新。

所以，Widget的重建和页面真正的重新渲染没有直接的联系，那么页面什么时候才会重新渲染呢，我们接着看源码。

## Element
上面我们以`Opacity`为例，从源码中看出了它的继承关系，并且推导出一些结论，这里面都没有涉及到`Element`，但实际上Element也十分重要。那什么是Element呢？源码里的注释如下：

>An instantiation of a Widget at a particular location in the tree.

我们知道，Widget是**immutable**的，也就是说一旦有了变化，Widget是反复重建的，你在代码里写的Widget的构造函数会被重新执行。那对应的，肯定得有一个东西来消化掉这些变化中的不变，来做cache。

让程序员不去保存Widgets是好的，因为他不会因为管理无数的Widget而且烦恼，一旦有变化就当整个页面都在变化，Widget的每一帧对应了一个State。这样，一个指定的state就能精确描述一个Widget该怎么展示。也就是说，程序员只要管理好状态，那么就能管理好整个页面变化的逻辑。

当一个Widget首次被创建的时候，那么这个Widget会过`Widget.createElement`inflate成一个`element`，挂在 element tree 上。

此后，当State发生变化，则Widget重建，但Element只会updates。

也就是说我们能大概有一个以下的一个流程：

程序员写Widget -> Widget形成Element -> Element构建 RenderObject -> RenderObject描述Canvas绘制 -> 交给FlutterEngine 光栅化**Rasterize**

回到我们今天的主角`Opacity`，从上面的分析，我么知道Opacity也是Widget，所以理应有对应的Element，但是上面看到为啥直接Widget就自己返回一个`RenderOpacity`呢？看起来是Widget自己创建了RenderObject。那么它的element是什么时候创建的呢？看源码：

```dart
@override
SingleChildRenderObjectElement createElement() => new SingleChildRenderObjectElement(this);
```
Opacity继承自`SingleChildRenderObjectWidget`，
这里面的`SingleChildRenderObjectElement`，实际上是只有一个child的Element。当第Widget第一次创建的时候，`createElement`会被调用，返回一个`SingleChildRenderObjectElement`

那它的 RenderObject 实际上是被这个 Element 创建的，看一下为什么：

```dart
SingleChildRenderObjectElement(SingleChildRenderObjectWidget widget) : super(widget);
```

这个`SingleChildRenderObjectElement`构造函数接受了一个`SingleChildRenderObjectWidget`参数，它负责创建一个`RenderObject`。那么我们看这个 renderobject 实际在哪里创建的：

```dart
class SingleChildRenderObjectElement {
    ...

    @override
  void mount(Element parent, dynamic newSlot) {
    super.mount(parent, newSlot);
    _child = updateChild(_child, widget.child, null);
  }

  ...
}
```
SingleChildRenderObjectElement里有个mount方法，看起来没啥，不过没事儿我们往父类里看：

```dart

@override
  void mount(Element parent, dynamic newSlot) {
    super.mount(parent, newSlot);
    _renderObject = widget.createRenderObject(this);
    assert(() { _debugUpdateRenderObjectOwner(); return true; }());
    assert(_slot == newSlot);
    attachRenderObject(newSlot);
    _dirty = false;
  }
```
终于看到magic了，所以在`mount`方法里，element调用了`widget.createRenderObject`，并attach。

注意，这些都发生在`mount`方法里，意味着只有element mounted的时候才会执行，所以只有一次。




---

参考文献：

[1. Flutter, what are Widgets, RenderObjects and Elements?](https://medium.com/flutter-community/flutter-what-are-widgets-renderobjects-and-elements-630a57d05208)

[2. The Layer Cake](https://medium.com/flutter-community/the-layer-cake-widgets-elements-renderobjects-7644c3142401)

