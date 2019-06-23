---
layout: post
title: Flutter的Event Loop、Future及Isolate
---

本文主要介绍Flutter中Event Loop以及如何在Flutter中做parallel processing.

## Event Loop

>First things first, everyone needs to bear in mind that Dart is Single Thread and Flutter relies on Dart.

>IMPORTANT
>Dart executes one operation at a time, one after the other meaning that as long as one operation is executing, it cannot be interrupted by any other Dart code.

跟AndroidVM类似，当你启动一个Flutter的App，那么就会系统就会启动一个DartVM
Flutter中的Dart VM启动后，那么一个新的Thread就会被创建，并且只会有一个线程，它运行在自己的Isolate中。

当这个Thread被创建后，DartVM会自动做以下3件事情：
* 初始化2个队列，一个叫“MicroTask”，一个叫“Event”，都是FIFO队列
* 执行 main() 方法，一旦执行完毕就
* 启动 Event Loop

Event Loop就像一个 infinite loop，被内部时钟来调谐，每一个tick，如果没有其他Dart Code在执行，就会做如下的事情（伪代码）：

```Dart
void eventLoop(){
    while (microTaskQueue.isNotEmpty){
        fetchFirstMicroTaskFromQueue();
        executeThisMicroTask();
        return;
    }

    if (eventQueue.isNotEmpty){
        fetchFirstEventFromQueue();
        executeThisEventRelatedCode();
    }
}
```

### MicroTask Queue
MicroTask Queue是为了非常短暂的asynchronously的内部操作来设计的。在其他Dart代码运行完毕后，且在移交给Event Queue前。
举个例子，我们经常需要在close一个resource以后，dispose掉一些handle，下面的这个例子里，scheduleMicroTask 可以用来做 dispose 的事情：
```Dart
    MyResource myResource;

    ...

    void closeAndRelease() {
        scheduleMicroTask(_dispose);
        _close();
    }

    void _close(){
        // The code to be run synchronously
        // to close the resource
        ...
    }

    void _dispose(){
        // The code which has to be run
        // right after the _close()
        // has completed
    }
```
这里，虽然`scheduleMicroTask(_dispose)`语句在`_close()`语句之前，但是由于上面说到的，“其他Dart代码执行完毕后”，所以`_close()`会先执行，然后执行 Event loop 的 microTask。

即使你已经知道 microTask 的执行时机，而且还学习了用`scheduleMicroTask`来使用 microTask，但是 microTask 也不是你常用的东西。就 Flutter 本身来说，整个 source code只引用了 `scheduleMicroTask()` 7次。

### Event Queue
Event Queue 主要用来处理当某些事件发生后，调用哪些操作，这些事件分为：

* 外部事件：
    * I/O
    * gesture
    * drawing
    * timers
    * streams
* futures

事实上，每当外部事件发生时，要执行的代码都是在 Event Queue里找到的。
只要当没有 MicroTask 需要run了，那么 Event Queue 就会从第一个事件开始处理

### Futures
当你创建了一个future的实例，实际上是做了以下几件事情：
* 一个 instance 被创建，放到一个内部的数组后重新排序，由dart管理
* 需要后续被执行的代码，被直接push到 Event Queue里面
* future立即同步返回一个状态`incomplete`
* 如果有其他 *synchronous* 代码，会先执行这些 sychronous 代码

Future和其他Event一样，会在EventQueue里被执行。
以下的例子用来说明Future和上面的Event执行过程一样
```Dart
void main(){
    print('Before the Future');
    Future((){
        print('Running the Future');
    }).then((_){
        print('Future is complete');
    });
    print('After the Future');
}
```
执行后，会得到如下输出：
```Shell
Before the Future
After the Future
Running the Future
Future is complete
```
我们来分步骤解释一下代码是如何执行的：
1. 执行print(‘Before the Future’)
2. 将function “(){print(‘Running the Future’);}” 添加到 event queue
3. 执行print(‘After the Future’)
4. Event Loop 取到第2步里说的代码，并且运行
5. 代码执行完毕后，它尝试找到 then 语句并运行

>A **Future** is **NOT** executed in parallel but following the regular sequence of events, handled by the **Event Loop**

### Async Methods
如果在任何一个方法的声明部分加上 `async` 后缀，那么你实际上在向dart表明：

* 该方法的结果是一个 future
* 如果调用时遇到一个await，那么它会同步执行，会把它所在的代码上下文给pause住
* 下一行代码会等待，直到上面的future(被await等的)结束

### Isolate
每个线程有自己的Isolate，你可以用`Isolate.spawn` 或者是 `compute` 来创建一个 Isolate
每个Isolate都有自己的Data，和Event loop
Isolate之间通过消息来进行沟通

```Dart
Isolate.spawn(
  aFunctionToRun,
  {'data' : 'Here is some data.'}, 
);
```

```Dart
compute(
  (paramas) {
    /* do something */
  },
  {'data' : 'Here is some data.'},
);  
```
当你在一个Isolate中，创建了一个新的Isolate，然后要和新的Isolate进行沟通，那么就需要`SendPort`和`ReceivePort`。
为了能沟通，两个Isolate必须要互相知晓对方的port：
* 本地 Isolate 通过 `SendPort` 来收/发消息，官方起的名字真的是有些让人困惑。
* 当你创建一个Isolate时，就需要给 `spawn` 方法传递一个 `ReceivePort` 的实例，后续会用这个port来收/发消息，同时也会通过这个port把本地 Isolate的sendport返回

找了一个例子，感受一下
```Dart
//
// The port of the new isolate
// this port will be used to further
// send messages to that isolate
//
SendPort newIsolateSendPort;

//
// Instance of the new Isolate
//
Isolate newIsolate;

//
// Method that launches a new isolate
// and proceeds with the initial
// hand-shaking
//
void callerCreateIsolate() async {
    //
    // Local and temporary ReceivePort to retrieve
    // the new isolate's SendPort
    //
    ReceivePort receivePort = ReceivePort();

    //
    // Instantiate the new isolate
    //
    newIsolate = await Isolate.spawn(
        callbackFunction,
        receivePort.sendPort,
    );

    //
    // Retrieve the port to be used for further
    // communication
    //
    newIsolateSendPort = await receivePort.first;
}

//
// The entry point of the new isolate
//
static void callbackFunction(SendPort callerSendPort){
    //
    // Instantiate a SendPort to receive message
    // from the caller
    //
    ReceivePort newIsolateReceivePort = ReceivePort();

    //
    // Provide the caller with the reference of THIS isolate's SendPort
    //
    callerSendPort.send(newIsolateReceivePort.sendPort);

    //
    // Further processing
    //
}
```
两个 Isolate 都有了各自的port，那么它们就可以开始互发消息了:

本地Isolate向新Isolate发消息并回收结果：

```Dart
Future<String> sendReceive(String messageToBeSent) async {
    //
    // We create a temporary port to receive the answer
    //
    ReceivePort port = ReceivePort();

    //
    // We send the message to the Isolate, and also
    // tell the isolate which port to use to provide
    // any answer
    //
    newIsolateSendPort.send(
        CrossIsolatesMessage<String>(
            sender: port.sendPort,
            message: messageToBeSent,
        )
    );

    //
    // Wait for the answer and return it
    //
    return port.first;
}
```

本地Isolate被动收消息，还记得上面的`spawn`方法吗，第一个参数是 `callbackFunction`这个方法就是用来收结果的：

```Dart
//
// Extension of the callback function to process incoming messages
//
static void callbackFunction(SendPort callerSendPort){
    //
    // Instantiate a SendPort to receive message
    // from the caller
    //
    ReceivePort newIsolateReceivePort = ReceivePort();

    //
    // Provide the caller with the reference of THIS isolate's SendPort
    //
    callerSendPort.send(newIsolateReceivePort.sendPort);

    //
    // Isolate main routine that listens to incoming messages,
    // processes it and provides an answer
    //
    newIsolateReceivePort.listen((dynamic message){
        CrossIsolatesMessage incomingMessage = message as CrossIsolatesMessage;

        //
        // Process the message
        //
        String newMessage = "complemented string " + incomingMessage.message;

        //
        // Sends the outcome of the processing
        //
        incomingMessage.sender.send(newMessage);
    });
}

//
// Helper class
//
class CrossIsolatesMessage<T> {
    final SendPort sender;
    final T message;

    CrossIsolatesMessage({
        @required this.sender,
        this.message,
    });
}
```

#### Isolate 的销毁
如果创建的Isolate不再使用，那么最好是能将其release掉：
```Dart
void dispose(){
    newIsolate?.kill(priority: Isolate.immediate);
    newIsolate = null;
}
```

#### Single-Listener Streams
实际上Isolate之间的沟通是通过 “Single-Listener” Streams 来实现的。


### compute 
上面说过创建 Isolate的方式，其中 compute 适合于创建以后执行任务，而且完成任务后你不希望有任何沟通。
compute是个function：

* spawn 一个 Isolate
* 运行一个callback，传递一些data，返回结果
* 在callback执行完毕时，kill掉isolate

### 适合 Isolate 的一些场景

* JSON 解析
* encryption 加解密
* 图像处理，比如 cropping
* 从网络加载图片

如何挑选呢？

一般来说：

* 如果一个方法耗时几十毫秒，用 Future
* 如果一个操作需要几百毫秒了，那么就用 Isolate


