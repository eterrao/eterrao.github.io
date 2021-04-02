---
title: 'Java: hashCode() 有几种实现方式？'
date: 2021-03-10 13:02:54
tags: Java
---

搞清楚这个问题之前需要先解决这些问题：

- 什么是 Hash 和 HashCode
- 为什么要使用 HashCode
- 实现 HashCode 的几种方式
- HashCode 的应用场景
- HashCode 相关
  - 判断相等需要重写 `equalis()` 和 `hashCode()`
  - HashMap 中的 hashCode(Key key) 实现方式



什么是 HashCode

