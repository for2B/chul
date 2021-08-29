---
layout:     post
title:      "checked exception、unchecked exception 与 Error"
subtitle:   "exception"
date:       2021-03-06
author:     "CHuiL"
header-img: "img/java-bg.png"
tags:
    - java
---


### checked exception

继承自Exception 的异常类，使用该异常的方法必须声明throws该异常，并且调用者需要try-catch它
```


public class CheckedExceptionTest extends Exception {
    public CheckedExceptionTest(String message){
        super("CheckedExceptionTest:"+message);
    }
}

    //必须throws
    private static void testCheckedException() throws CheckedExceptionTest {
        throw new CheckedExceptionTest("a");
    }
    
    //调用者必须try-catch
    public static void main(String[] args) {
        try {
            testCheckedException();
        }catch (CheckedExceptionTest exceptionTest){
            System.out.println(exceptionTest);
        }
    }
```


### unchecked exception
继承自RuntimeException ，使用该异常的方法不需要声明throws该异常，调用该方法也可以不用try-catch它

```
public class RuntimeExceptionTest extends RuntimeException {

    public RuntimeExceptionTest(String message) {
        super("RuntimeExceptionTest"+message);
    }
}
    //可以不throws，也可以throws
    private static void testRuntimeException() {
        throw new RuntimeExceptionTest("b");
    }
    //调用方可以不用try-catch,也可以调用；不catch会抛异常错误，catch后与exception处理一样。
    public static void main(String[] args) {
        testRuntimeException();
    }
```

其实unchecked exception应该算是exception中的一种特例，RuntimeException是继承自Exception的；而Exception继承Throwable接口；

### 这两者使用的场景是什么?
- 使用 checked 异常的情况是为了合理地期望调用者能够从中恢复，获取异常传播出去；显示的告诉调用者需要处理这种异常；
- 使用运行时异常来指示编程错误； 绝大多数运行时异常都表示操作违反了先决条件。违反先决条件是指使用 API 的客户端未能遵守 API 规范所建立的约定。例如，数组访问约定指定数组索引必须大于等于 0 并且小于等于 length-1 

如果你认为某个条件可能允许恢复，请使用 checked 异常；如果没有，则使用运行时异常。如果不清楚是否可以恢复，最好使用 unchecked 异常


### Exception与Error
Error和Exception都继承了Throwable类，在java中只有throwable类型的实例才可以被抛出和捕获；
- exception是程序正常运行中，可以预料到的意外情况，并且可以被捕获或者不捕获，通常不会导致程序不可用或不可恢复；
- Error指正常情况下不会出现的错误，Error一旦发生，一般都会导致程序处于非正常的，不可恢复的状态，因此即使他继承自throwable，也基本不需要捕获Error；
![image](/chuil/img/java/exception-1.png)
