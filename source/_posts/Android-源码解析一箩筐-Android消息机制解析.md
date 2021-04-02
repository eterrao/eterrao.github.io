---
title: Android源码解析一箩-Android消息机制解析
date: 2021-02-17 14:21:54
tags: Android源码, Android
---

## 概述

面试说起 Android 消息机制一定会对 Handler 进行灵魂拷问，尽管作为老生常谈的话题，但真正理解 Handler 内部的实现，融会贯通，并且了解很多面试会考查到的细节才是我们真正的目的。

- Android 消息机制

- Handler 是什么
- Handler 的作用
- Handler 的面试题汇总

## Android 消息机制结构图

Android 中的线程模型是以 UI 主线程和子线程组成的，在 Android 中，UI 主线程和工作线程的的消息传递机制是依赖于 Handler 来实现的，我们先看看 Android 消息机制的原理图： 

![overview](/Users/rmy/work/images/overview.png)

<!-- more -->

## 关键类

谈起 Android 的消息循环机制，总免不了讨论源码实现，主要涉及到的有几个关键的类： 

| 类              | 作用                                                         |
| :-------------- | ------------------------------------------------------------ |
| Handler         | 用于处理和发送 Android 的消息，通过 Handler 可以实现主线程和其他子线程的消息通信。 |
| Looper          | 在 Looper 中有一个静态变量 `sThreadLocal`，在 `loop()` 方法中会先确认 Looper 处于哪条线程中，如果 `mylooper()` 获取的 `currentThread()` 不是 UI 主线程，则需要严格按照 Looper 的流程进行操作，否则会抛异常：`throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.")`。Looper 负责实现队列循环，维护消息循环，其中 `loop()` 方法内部由一个 `for(;;)` 循环从 MessageQueue 中取到 Message，然后通过内部名为 `target` 的 Handler 对象进行消息分发，参见 `loop() ` 部分的 `target.dispatchMessage()`调用。 |
| MessageQueue    | 消息队列，队列中的元素是 Message，由 Looper 维护。关键的几个方法： |
|                 | `enqueueMessage()` 消息入队，并且内部会调用 `nativeWake()` 通知循环体解除阻塞状态 |
|                 | `next(): Message` 将消息出列                                 |
|                 | `nativePollOnce()` 在队列为空时，将消息循环进行阻塞          |
| Message         | 作为队列中循环的数据载体，内部有关键变量 `target:Handler`，通过 `arg1/arg2/obj/data:Bundle` 几个变量来实现消息的传递。 |
| **ThreadLocal** |                                                              |
| IMESSENGER      | 在 Handler 中，还有一个变量 `mMessenger`，由 MessengerImpl 实现，MessengerImpl 继承自 IMessenger.Stub，关于 IMssenger 的实现，详情见 Binder 原理。 |
| HandlerThread   | 继承自 Thread，但内部由 Handler 实现，主要内部变量就是 `Handler` 和 `Looper` |

## Handler

### 关键方法

在 Handler 中主要的几种方法 xxxMessage()：

| 方法              | 作用                                                         |
| ----------------- | ------------------------------------------------------------ |
| sendMessage()     | 发送消息，实际调用的是 `enqueueMessage()`                    |
| enqueueMessage()  | 将消息塞入队列 MessageQueue                                  |
| removeMessage()   | 移除消息                                                     |
| obtainMessage()   | 从消息池(`sPool`)中获取 message，如果消息池中的 `message == null`，则返回 `new Message()` |
| handleMessage()   | 处理 Looper 分发过来的消息，该方法可以直接继承 Handler 实现，也可以通过 Handler.Callback 实现。 |
| dispatchMessage() | 分发消息，在这个方法中实际会调用 `handleMessage()`           |
| post(Runnable r)  | 实际调用的是 `sendMessage()`，关于 `runnable` 和 `callback` 之间有一个调用顺序需要注意，调用次序是：runnable == message > mCallback > Handler，具体在 `dispatchMessage()` 中实现，参见如下代码： |

```java
public void dispatchMessage(Message msg) {
  // 先调用的是 message, runnable 也会被封装成 message，
  if (msg.callback != null) {
    handleCallback(msg);
  } else {
    // 然后到 mCallback，如果 mCallback 为 null
    if (mCallback != null) {
      if (mCallback.handleMessage(msg)) {
        return;
      }
    }
    // 才到常规的 handleMessage(msg)
    handleMessage(msg);
  }
}
```

### [特别注意](https://blog.csdn.net/carson_ho/article/details/80175876)

> 线程`（Thread）`、循环器`（Looper）`、处理者`（Handler）`之间的对应关系如下：
>
> - 1个线程`（Thread）`只能绑定 1个循环器`（Looper）`，但可以有多个处理者`（Handler）`
> - 1个循环器`（Looper）` 可绑定多个处理者`（Handler）`
> - 1个处理者`（Handler）` 只能绑定1个1个循环器`（Looper）`
>
> ![示意图](/Users/rmy/work/images/aHR0cDovL3VwbG9hZC1pbWFnZXMuamlhbnNodS5pby91cGxvYWRfaW1hZ2VzLzk0NDM2NS02MWIzODdjMGU2NmVkOGVlLnBuZz9pbWFnZU1vZ3IyL2F1dG8tb3JpZW50L3N0cmlwJTdDaW1hZ2VWaWV3Mi8yL3cvMTI0MA-20210218231656337.png)

### 总结

Handler 的使用需要注意的几个点：

- 在 Handler 中避免内存泄漏需要使用弱引用 `WeakReference<Context>`
- 自定义 Looper 时，需要注意 `prepare() ` 和  `loop()` 的配合调用，并且 `prepare()` 只能被调用一次

### 面试问题汇总

- 为什么要用 `Handler`消息传递机制

  在 Android 中为了保证 UI 主线程不被阻塞，Android 提供了 Handler 来操作 UI 线程和工作线程之间的消息通信的机制，这里的信息包括通知 UI 线程执行某个事件或者传递进行线程之间的数据传递。

- Handler 是怎么和线程绑定的（Looper）

- 为什么 Looper.loop() 要设计成死循环

- 为什么主线程一直在死循环却不会占用大量CPU消耗？

  > 对于线程即是一段可执行的代码，当可执行代码执行完成后，线程生命周期便该终止了，线程退出。而对于主线程，我们是绝不希望会被运行一段时间，自己就退出，那么如何保证能一直存活呢？简单做法就是可执行代码是能一直执行下去的，死循环便能保证不会被退出，例如，binder线程也是采用死循环的方法，通过循环方式不同与Binder驱动进行读写操作，当然并非简单地死循环，无消息时会休眠。但这里可能又引发了另一个问题，既然是死循环又如何去处理其他事务呢？通过创建新线程的方式。真正会卡死主线程的操作是在回调方法`onCreate/onStart/onResume`
  >
  > 主线程的死循环一直运行是不是特别消耗CPU资源呢？ 其实不然，这里就涉及到`Linux pipe/epoll`机制，简单说就是在主线程的`MessageQueue`没有消息时，便阻塞在loop的`queue.next()`中的`nativePollOnce()`方法里，此时主线程会释放CPU资源进入休眠状态，直到下个消息到达或者有事务发生，通过往pipe管道写端写入数据来唤醒主线程工作。这里采用的epoll机制，是一种IO多路复用机制，可以同时监控多个描述符，当某个描述符就绪(读或写就绪)，则立刻通知相应程序进行读或写操作，本质同步I/O，即读写是阻塞的。 所以说，主线程大多数时候都是处于休眠状态，并不会消耗大量CPU资源。 Gityuan–Handler (Native层)

- 为什么 Looper 死循环却不会引起 ANR？

- 究竟是什么导致主线程卡死？

- 主线程的消息循环机制是什么（死循环如何处理其它事务）？

- ActivityThread 的动力是什么？（ActivityThread执行Looper的线程是什么）

- Handler 是如何能够线程切换，发送Message的？（线程间通讯）

- 子线程有哪些更新 UI 的方法。

- 子线程中 Toast，showDialog 的方法（和子线程不能更新UI有关吗）

- 如何处理 Handler 使用不当导致的内存泄露？

- 在子线程中创建Handler报错是为什么?

### 参考文章

- [Android 异步通信：图文详解Handler机制工作原理](https://blog.csdn.net/carson_ho/article/details/80175876)

### 在子线程中创建Handler报错是为什么?

我们实现以下代码：

```java
new Thread(new Runnable() {
            @Override
            public void run() {
                new Handler() {
                    @Override
                    public void handleMessage(Message msg) {
                        super.handleMessage(msg);
                    }
                };
            }
        }).start();
```

执行后发现如下异常：

```shell
2021-02-23 16:08:32.403 7059-7142/com.raomengyang.lib.aospviewer E/AndroidRuntime: FATAL EXCEPTION: Thread-3
    Process: com.raomengyang.lib.aospviewer, PID: 7059
    java.lang.RuntimeException: Can't create handler inside thread Thread[Thread-3,5,main] that has not called Looper.prepare()
        at android.os.Handler.<init>(Handler.java:207)
        at android.os.Handler.<init>(Handler.java:119)
        at com.raomengyang.lib.aospviewer.MainActivity$1$1.<init>(MainActivity.java:45)
        at com.raomengyang.lib.aospviewer.MainActivity$1.run(MainActivity.java:45)
        at java.lang.Thread.run(Thread.java:919)
```

```
Can't create handler inside thread Thread[Thread-3,5,main] that has not called Looper.prepare()
```

出自：

```java
public Handler(Callback callback, boolean async) {
				// … 省略
        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread " + Thread.currentThread()
                        + " that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```

发现这里会调用 Looper.myLooper();

跟踪进去看 myLooper()

```java
/**
* Return the Looper object associated with the current thread.  Returns
* null if the calling thread is not associated with a Looper.
*/
public static @Nullable Looper myLooper() {
	return sThreadLocal.get();
}

```

发现 `sThreadLocal.get()` 这个方法被调用了，而且是可以为 null 的，证明 sThreadLocal.get() 为 null，导致 myLooper() 为 null。

深入看看这个方法的实现：

```java
/**
     * Returns the value in the current thread's copy of this
     * thread-local variable.  If the variable has no value for the
     * current thread, it is first initialized to the value returned
     * by an invocation of the {@link #initialValue} method.
     *
     * @return the current thread's value of this thread-local
     */
public T get() {
  Thread t = Thread.currentThread();
  ThreadLocalMap map = getMap(t);
  if (map != null) {
    ThreadLocalMap.Entry e = map.getEntry(this);
    if (e != null) {
      @SuppressWarnings("unchecked")
      T result = (T)e.value;
      return result;
    }
  }
  return setInitialValue();
}
```

```
ThreadLocalMap
```

那为什么主线程这个方法返回的不是 null 呢？

