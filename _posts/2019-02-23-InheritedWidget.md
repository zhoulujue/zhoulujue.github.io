---
layout: post
title: Flutter中InheritedWidget使用的最佳实践
---

本文主要介绍InheritedWidget的设计目的、用法以及推荐的最佳实践

# InheritedWidget 

Flutter的Widget层级可以做得非常深，在Widget层级间传递数据会让效率变得很低，也会多处很多bolierplate代码。

例如下面的`accountId`，`scopeId`，如果MyWidget肯本用不到它们，`accountId`, `scopeId`还是要作为MyWidget的final参数，因为它们要作为构造函数的参数接受参数值并向`MyOtherWidget`传递，先得非常冗余，因为MyWidget完全没有必要知道这两个参数。

```dart
class MyPage extends StatelessWidget {
  final int accountId;
  final int scopeId;
  
  MyPage(this.accountId, this.scopeId);
  
  Widget build(BuildContext context) {
    return new MyWidget(accountId, scopeId);
  }
}

class MyWidget extends StatelessWidget {
  final int accountId;
  final int scopeId;
  
  MyWidget(this.accountId, this.scopeId);
  
  Widget build(BuildContext context) {
    // somewhere down the line
    new MyOtherWidget(accountId, scopeId);
    ...
  }
}

class MyOtherWidget extends StatelessWidget {
  final int accountId;
  final int scopeId;
  
  MyOtherWidget(this.accountId, this.scopeId);
  
  Widget build(BuildContext context) {
    // rinse and repeat
    ...
```

通过引进InheritedWidget，我们就能解决这个困扰。

InheritedWidget的思想和redux的思想差不多，所谓`accountId`，`scopeId`实际上是这些widget的状态，用户交互或其他事件会触发Action，Action产生新的State，Widget接受新的State生成新的RenderNode挂载到Widget Tree上，完成一次渲染。

那怎么让这些参数在不同层级之间传递呢？答案是将State往上放，也就是俗称`state up lifting`

例如下面的代码就是通过插入一个`InheritedWidget`将`accountId`和`scopeId`存储在InheritedWidget描述的这一层级中，所有这一层级和下面的层级都能访问这两个参数。

```dart
class MyInheritedWidget extends InheritedWidget {
  final int accountId;
  final int scopeId;

  MyInheritedWidget(accountId, scopeId, child): super(child);
  
  @override
  bool updateShouldNotify(MyInheritedWidget old) =>
    accountId != old.accountId || scopeId != old.scopeId;
}

class MyPage extends StatelessWidget {
  final int accountId;
  final int scopeId;
  
  MyPage(this.accountId, this.scopeId);
  
  Widget build(BuildContext context) {
    return new MyInheritedWidget(
      accountId,
      scopeId,
      const MyWidget(),
     );
  }
}

class MyWidget extends StatelessWidget {

  const MyWidget();
  
  Widget build(BuildContext context) {
    // somewhere down the line
    const MyOtherWidget();
    ...
  }
}

class MyOtherWidget extends StatelessWidget {

  const MyOtherWidget();
  
  Widget build(BuildContext context) {
    final myInheritedWidget = MyInheritedWidget.of(context);
    print(myInheritedWidget.scopeId);
    print(myInheritedWidget.accountId);
    ...
```

注意：

1. `InheritedWidget`的`child`是const修饰的构造函数，这样做的目的是让child能缓存
2. `accountId`或者`scopeId`值更新的时候，MyInheritedWidget会被重新创建，但是它的child不一定会被创建，取决于是否用到了`accountId`或者`scopeId`，上面的这个例子中`MyOtherWidget`会被rebuild，但是`MyWidget`不会被rebuild
3. 如果tree被其他事件触发rebuild，例如orientation changes，`InheritedWidget`会被rebuild，但是child同样不一定被rebuild，因为`accountId`和`scopeId`没变。

为了让InheritedWidget更加高效，推荐下面的最佳实践：

## Keep inherited widgets small

尽量让你的Context语义尽可能单一，这样Flutter渲染时才能更细粒度的判断哪些Widget需要rebuild哪些需要复用，否则就失去了`InheritedWidget`的意义。

**要这样**

```dart
class TeamContext {
  int teamId;
  String teamName;
}

class StudentContext {
  int studentId;
  String studentName;
}
 
class ClassContext {
  int classId;
  ...
}
```

**不要这样**

```dart
class MyAppContext {
  int teamId;
  String teamName;
  
  int studentId;
  String studentName;
  
  int classId;
  ...
}
```

## 使用const方式来构造child

如果没有const，`InheritedWidget`的整个child都不会缓存，选择性rebuild整个子tree就不会生效。

## 管理好Context的Scope

因为Flutter的整个App是一个Tree，所以`InheritedWidget`可以放在任意一个层级，这样显然不利于人脑来理解并管理，所以必须大家规约好，例如规定`InheritedWidget`只接受`Scaffold`作为child，这样所有的`InheritedWidget`最大粒度是Page，便于人脑理解及阅读。

## 不要跨路由访问context

Flutter眼里没有page不page的，所有东西就是widget，如果使用`Navigation.push`一个Widget请注意route的下一个widget没有继承上一个widget的InheritedWidget的Context，你需要明确将参数传递，而不是将InheriedWidget往上提。