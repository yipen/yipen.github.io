## 什么是JVM Shutdown Hook
Shutdown Hook是一种比较特殊的结构，一般用于在JVM关闭之前，需要执行的一些操作的时候。常见的，比如说你的程序退出时需要做一些清理工作的时候，就可以考虑使用Shutdown Hook。但是如果你的JVM是非正常退出的，比如接收了SIGKILL，这个时候就不能保证shutdown hook能够正常执行。

## 如何使用Shutdown Hook
Java中其实已经提供了相应的接口来实现，使用起来也非常方便。以下给出几个简单的例子：

``` java
public class ShutDownHookTest
{
  public static void main(String[] args)
  {
  
    Runtime.getRuntime().addShutdownHook(new Thread()
    {
      public void run()
      {
        System.out.println("Shutdown Hook now!");
      }
    });
    System.out.println("Going to exit");
  }
}
```
输出:
~~~
Going to exit
Shutdown Hook now!
~~~
以上例子可以看到，我们只需要给addShutdownHook里传入一个Thread对象，在run()中写入需要执行的操作。

``` java

class MyThread extends Thread {     
    public void run() {         
        System.out.println("Execute done");
        System.out.println("Shutdown hook now!");
    }
}
  
class ShutDownHookTest {
      
    public static void main(String[] args) {
        Runtime.getRuntime().addShutdownHook(new MyThread());
  
        for(int i = 1; i < 4; i++) 
            System.out.println("Execute：" + i);
    }
}
```
输出:
~~~
Execute：1
Execute：2
Execute：3
Execute done
Shutdown hook now!
~~~
以上例子可以清晰看到，当主程序执行完后，才会开始执行Shutdown Hook, JVM才会退出。

## Shutdown Hook的特性
### Shutdown Hook不能保证一定执行
如果JVM crashe了, Shutdown Hook并不能保证一定执行。例如如果收到了SIGKILL的时候，程序会立刻终止执行，JVM立刻退出，导致没有机会执行Shutdown Hook.如果调用了Runtime.halt()的情况下，也可以导致JVM在没有执行Shutdown Hook的时候直接退出。当然，如果一个程序正常结束，会在JVM退出去调用Shutdown Hook。如果JVM因为用户要求中断或者是接受到SIGTERM，也是会调用Shutdown Hook的。

### Shutdown Hook是可以被强制中断的
即使已经开始执行Shutdown Hook,也是可以被中断的，比如当接收到了SIGTERM，但是shutdown hook没有在一定时间内完成，也是会被强制中断，导致shutdown hook没有完整执行。所以一般在Shutdown hook中的操作都应该是可以快速执行完毕，不应该是一些long time的任务。

### Shutdown Hook可以有多个
一个程序中可以注册多个shutdown hook，但是JVM执行shutdown hook的时候是一个任意顺序，并且JVM执行hook的时候是并发的。

### Shutdown hook中不可以regist/unregister shutdown hook
如果这么做了，JVM会抛IllegalStateException。

### Shutdown hook可以被停止
如果一个Shutdown已经开始执行了，除了例如SIGKILL这样的外部干预，需要且只能通过Runtime.halt()中断。

### Shutdown hook需要安全权限

## 参考
http://docs.oracle.com/javase/1.5.0/docs/api/java/lang/Runtime.html#addShutdownHook(java.lang.Thread)