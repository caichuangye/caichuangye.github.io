---
title: Exception和Error
date: 2016-05-14 22:54:37
tags:
- Throwable
- Java
categories: Java
---

Java中Exception和Error都继承自Throwable。
#### Exception
除了RuntimeException及其子类，Exception及其子类都是checked 异常，checked的异常需要显式的try catch或throws声明。

#### Error
Error及其子类都是unchecked，unchecked的异常不需要显式的try catch和throws声明。Error一般视为非常严重的系统级问题，比如OutOfMemoryError、StackOverflowError等，当出现这些问题时说明系统出现了严重的问题，建议应用程序终止.