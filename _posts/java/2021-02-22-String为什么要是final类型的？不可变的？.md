---
layout:     post
title:      "String为什么要是final类型的？不可变的？"
subtitle:   "string"
date:       2021-02-22
author:     "CHuiL"
header-img: "img/java-bg.png"
tags:
    - java
---

#### 1.String为什么不可变？
```
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
    /** The value is used for character storage. */
    private final char value[];
}
```
这里final声明class，表示不可以被继承，同时value也是fianl，但是该value只是数组引用不可变，内部的元素还是可以变化的，所以这里其实主要是靠private，还有String里的方法很小心的没有对value里的元素进行变动，并且禁止继承防止子类破坏；

####  2.不变的好处
1. 字符串常量池，共享相同字符串。
2. 线程安全，因为不可写，所以不线程不安去的问题
3. 使用安全，避免因为引用导致字符串被意外修改；例如字符串作为参数传入方法，如果可变，被意外修改了，可能导致原来的字符串也被修改；又或者存入到set中，但是通过引用修改，导致set中存在两个一样的值等；



#### 3.String不是基本类型
基本类型有8个：boolean/char/byte/short/int/long/float/double；