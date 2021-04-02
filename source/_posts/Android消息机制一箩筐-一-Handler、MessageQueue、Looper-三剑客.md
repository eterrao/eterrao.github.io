---
title: Android消息机制一箩筐<一> Handler、MessageQueue、Looper 三剑客
date: 2021-02-23 17:10:49
tags: Android消息机制, Handler, 面试
---

## 概述

```java
public static void loop() {
    // 获取线程的 Looper，如果 Looper 为 null，则抛异常
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    // 从 Looper 中获取 MessageQueue
    final MessageQueue queue = me.mQueue;
    // 开启循环体
    for (;;) {
        // 从 MessageQueue 中取 Message
        Message msg = queue.next(); // might block
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }
        final long dispatchStart = needStartTime ? SystemClock.uptimeMillis() : 0;
        final long dispatchEnd;
      
        try {
            // 调用 message 中的 target:handler.dispatchMessage()，进行消息分发
            msg.target.dispatchMessage(msg);
        } catch (Exception exception) {
            throw exception;
        } finally {
            ThreadLocalWorkSource.restore(origWorkSource);
        }
        msg.recycleUnchecked();
    }
}
```

