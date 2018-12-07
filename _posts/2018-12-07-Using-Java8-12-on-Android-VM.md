---
layout: post
title: 在 Android VM 上 使用 Java8-12
---


#### TLDR：“不是所有的Java8 Feature 都会被 D8 desugar”

---

在 `D8` 被引进作为替代 `dx` 的dex工具后，越来越多的人问我是不是可以在项目里用原生lambda来替换Retrolambda，或者问各种新特性在Android上的支持程度。


其实如果要正确地回答这个问题，不是能够用一句话说完的。
所以我整理了一下 Jake Wharton 的Slides，重点如下文便于大家查阅。

## Java8
### TLDR：真正能Natively用的特性

- Api Level 24及以上能用 interface methods (default methods or static methods) 和 Method reference
- Api Level 26及以上能用 lambda

### lambda

- Q: 在低Api Level上为什么不能 natively 用呢？
- A: 是因为 Native lambda 用到了 VM 指令 invoke-dynamic ，这个指令在低版本上不支持

---

- Q: 在低Api Level上怎么用呢？

- A1: 用 Retrolambda

- A2: 或者用 D8 的 desugar:  将 lambda 函数体抽取出来，提成一个静态方法；创建一个内部类，构造函数的参数实现 context capture，内部类里生成一个实例方法调用外部类的静态方法，最后原来调用 lambda 的地方调用 new 内部类的实例方法

---

- Q: 如果用了D8，且在D8的命令中指定 min-api 为26，是不是 lambda 就是 Native lambda 了呢？

- A: 不是！
是因为，Native lambda 采用的是 BootstrapMethod (也就是在 bytecode 第一次执行的时候，用来定义行为的，native lambda 生成的 invoke-dynamic method 就在这时定义)，BootstrapMethod 用到了 java/lang/invoke.* 一些类，可惜 Android runtime不提供这些类，所以还是不行，所以D8为了避免 runtime error ，还是 desugar 了 lambda

### LocalDateTime

LocalDateTime 的新用法要小心，D8不会帮你 Desugar ，LocalDateTime.now()编写时不会报任何错误，但是跑起来就不一定了，在 26以下会报 ClassNotFound 错

em.... 那就用 ThreeTenBP 吧 😄

### Interface methods

D8 会在编译后 Desugar 好，同样如果指定 --min-api 24 的话，D8 不会 desugar，使用 native interface methods

## Java9

- Sad fact : Java9 目前还没有被引进 Android SDK.
- Fun fact : ART 在运行时用 bytecode instructions 支持了某些Java9特性，例如 private interface methods.


**下面的新特性为了避免因为用得少而陌生，我先把例子写出来，然后再阐述是否可以在AndroidVM上使用。**

### Concise Try with Resources

如果你尝试在 Java 中读取一个流，然后关闭一个流，你应该会写很多 try catch

Java9 中可以这样写了：
```Java
import java.io.*;

class Java9TryWithResources {
  String effectivelyFinalTry(BufferedReader r) throws IOException {
    try (r) {
      return r.readLine();
    }
  }
}
```
**这个特性在 Java compiler 中实现，不需要特殊 VM 指令，所以 Android 里随便用**

### Anonymous Diamond

钻石语法 Java7 就有了，但是直到 Java9 才有匿名类的钻石语法，像这样：

```Java
import java.util.concurrent.*;

class Java9AnonymousDiamond {
  Callable<String> anonymousDiamond() {
    Callable<String> call = new Callable<>() {
      @Override public String call() {
        return "Hey";
      }
    };
    return call;
  }
}
```
**同样，这个特性在 Java compiler 中实现，不需要特殊 VM 指令，所以 Android 里随便用**

### Private interface methods

Java8 引入了 default methods & static methods for interfaces, 但是如果想要复用代码且不暴露给外界，必然需要 private methods，所以 Java9 引入了这个特性。

上面说到，D8 在给 Java8 的 interface default methods & static methods 的 desugar 和 non desugar 的情况，这里的 private methods 是一样的。


上面说过的这些特性虽然在Android SDK里包括，但是如果你 javac 后，再 D8，adb push 到 sdcard 上，用 `adb shell dalvikvm -cp /sdcard/path/to/classes.dex` 执行的话，会得到正确执行结果。

### String concatenation

Java9 之前的 String concat，都会在 bytecode 里转换成 StringBuilder，有多少次 concat 就有多少次append

Java9 的 concat 用了 invoke-dynamic 来委托给 `StringConcatFactory` 返回这些代码，也就是说，不管里怎么改 concat，实际上编译后的 bytecode 不会频繁发生改变，提高性能。


至此，我们知道 Java9 所有的 language feature 都被 D8 搞定了，虽然 Android SDK 没有引入任何 Java9的 class，但是仍然可以被 Google 魔改的 ART 和 D8 支持。

你只要用了 D8，且 -min-api 21 Java9 特性随便用，至于 Java8 么，呵呵，还要 -min-api 26 才能随便写，what a world!

## Java10

#### TLDR：很多新特性都是在 Java compiler 中实现的，所以和 Runtime 的支持与否无关

### Local variable type inference

var: 局部变量的类型推断

```Java
public class Main {

    public static void main(String[] args) {
    }

    List<String> localVariableTypeInference() {
        var url = new ArrayList<String>();
        return url;
    }
}
```

同样的，本地变量类型推断仍然是在 java compiler 里做的，所以 Android 里随便用

## Java11

### Type inference for lambda parameters

对 Java10 的类型推断的一个优化，看如下的代码实例，新旧用法对比：

```Java
public class Main {

    public static void main(String[] args) {
    
    }

    void lambdaParameterTypeInference() {
        // old ways
        Function<String, String> normal = (String param) -> param + param;
        // now
        Function<String, String> function = (var param) -> param;
    }
}
```

虽然这个特性看起来不起眼，因为大家平时用lambda的时候都会隐藏类型声明。但是如果是你想加一些 annotation，那就变的很方便：例如，上述代码如果需要在param上加@Nonnull注解，你不需要去翻阅代码去确认 param 的类型，直接把上述方法改为：

```Java
public class Main {

    public static void main(String[] args) {
    
    }

    void lambdaParameterTypeInference() {
        // old ways
        Function<String, String> normal = (String param) -> param + param;
        // now
        Function<String, String> function = (@NonNull var param) -> param;
    }
}
```

同样的，这个特性仍然是在 java compiler 里做的，所以 Android 里随便用 

### Nestmates

Java11 中Natively支持了嵌套类。
我们知道在以前在Java中编写嵌套的两个类，编译过后这两个类分别是两个class文件，VM里只是用命名规范来区别这两个类，例如如下Java类：
```Java
class Outer {
  private String hello = "hello";
  class Inner {
    String helloWorld() {
        return hello + "world!";
    }
  }
}
```
编译后就会变为：

```shell
$ javac *.java

$ ls
Outer.java  Outer.class  Outer$Inner.class
```

写过内部类的Handler的我们都知道，一旦内部类用了外部类的 method 或者 property 那么在编译时会 capture 外部类的实例，从而造成对外部类的强引用。
解决过OOM的人一定对此深恶痛绝，因为要解决这个问题要写很繁琐的 WeakReference。

在Java11中编译后的类会变为:

```shell
$ javac *.java

$ javap -v -p *.class
class Outer {
  private java.lang.String name;
}
NestMembers:
  Outer$Inner

class Outer$Inner {
  final Outer this$0;

  Outer$Inner(Outer);
    …

  java.lang.String sayHi();
    …
}
NestHost: class Outer
```

注意到`NestMembers`和`NestHost`属性，在VM中不再是通过命名来区别嵌套类了。这样，在VM级别 Outer 和 Inner 通过可见性关键词就可以互相访问 `package-private` and `private` 级别及以下的方法和属性。

**很遗憾，ART 不支持这个特性，所以得靠 D8 来 desugar，但是貌似D8没有对class文件中的`NestMembers`和`NestHost`进行支持，所以目前Android中不能用。**


## Java12

虽然Java12仍在预览版，不过我们可以通过 EA-build 来了解具体有哪些特性，到写这篇文章时：`expression switch` and `string literals`

expression switch:
```Java
 switch (s) {
      case "hello", "Hello" -> 0;
      case "world" -> 1;
      default -> s.length();
 }
```
string literals:
```Java
String script = `function hello() {
                    print('"Hello World"');
                 }
				
                 hello();
                `;
```
同样，上述特性都是 Java compiler 支持的，所以 Android 里随便用


