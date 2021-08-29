---
layout:     post
title:      "JavaAgent技术"
subtitle:   "JavaAgent技术"
date:       2021-02-24
author:     "CHuiL"
header-img: "img/java-bg.png"
tags:
    - java
---

### JavaAgent技术

启动时加载的JavaAgent是Jdk1.5Z之后引入的新特性，此特性为用户提供了在JVM将字节码文件读入内存之后，使用对应的字节流在java堆中生成一个Class对象之前，对其字节码进行修改的能力，而JVM也会使用用户修改过的字节码进行Class对象的创建。
##### 更具体的说
premain方法,会接受一个Instrumentation，它是就是**可以实现在方法插入额外的字节码从而达到收集使用中的数据到指定工具的目的。** 也就是在已有的类上附加（修改）字节码来实现增强的逻辑；

```
public class SkyWalkingAgent{ 
    public static void premain(String agentOps, Instrumentation inst){
        System.out.println("====premain 方法执行");
        inst.addTransformer(new SkyWalkingTransformer());
    }
}
```
而这里的SkyWalkingTransformer是实现了ClassFileTransformer接口的Transformer；该接口主要是transform方法
```
    byte[]
    transform(  ClassLoader         loader,
                String              className,
                Class<?>            classBeingRedefined,
                ProtectionDomain    protectionDomain,
                byte[]              classfileBuffer)
        throws IllegalClassFormatException;
```
从参数就可以看出，他接受当前载入的类的信息，然后返回增强后的字节码；
- ClassLoader:当前载入的Class的ClassLoader。
- String ClassName：当前载入的Class的名字。
- classfileBuffer:当前类以byte数组呈现的字节码数据
- 出参byte[]:增强后的类字节码；返回null表示没有修改；jvm会使用此字节码数据进行后续的流程。

当某个类的字节码被jvm载入内存之后，jvm会触发一个ClassFileLoadHook事件，JVM会一次遍历所有的instrumentation实例并执行其中所有的ClassFileTransformer的transform方法。
也就是说，我们的premain最主要的功能javaagent的入口，在类文件被载入jvm内存之前，将我们的增强逻辑添加到instrumentation，然后后续就可以在加载每个类文件之后进行类和类方法的增强。
![image](/chuil/img/java/java-agent-1.png)


## 参考
[深入理解Instrument(一)](https://www.throwable.club/2019/06/29/java-understand-instrument-first/)  
[Apache SkyWalking 实战]()