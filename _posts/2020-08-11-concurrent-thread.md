---
layout:     post
title: "深入剖析Thread"
subtitle: "Thread创建、生命周期、线程的启动与停止"
date:       2020-08-11 23:22:00
author:     "Autu"
header-img: "img/post-sample-image.jpg"
header-mask: 0.3
catalog:    true
tags:
  - 并发
  - 多线程
  - Thread
---



### 大纲

![1597159663195](http://123.57.45.66/images/autu_blog/concurrent/2020-08-11-concurrent-thread/1597159663195.png)



### 一、创建线程的3种方式

#### 1. 继承 `Thread` 类

我们看下Thread的源码：

![1596854087240](http://123.57.45.66/images/autu_blog/concurrent/2020-08-11-concurrent-thread/1596854087240.png)

接下来我们看下 `start()` 方法

![1596855318666](http://123.57.45.66/images/autu_blog/concurrent/2020-08-11-concurrent-thread/1596855318666.png)

从源码可以看出，**`Thread` 类本质上是实现了 `Runnable` 接口的一个实例**。

因此，线程启动的唯一方式就是通过 `Thread` 类的 `start()` 方法，`start()` 方法调用 `start0()` 方法， `start0()` 是个 **native 方法**，它会通过调用 `jvm` 方法，启动一个新线程。

我们来波实操：

```java
public class Test {
    public static void main( String[] args ) {
        for (int i = 0; i < 5; i++) {
            // 创建并开启线程
            new ThreadTest(i + "").start();
        }

    }
}
// 继承 Thread 类
public class ThreadTest extends Thread {
	// 定义指定线程名称的构造方法
    ThreadTest(String name){
        super(name);
    }

    @Override
    public void run() {
        // getName()方法 来自父类
        System.out.println("Thread :[ " + getName() + " ] run");
    }

}
```

下面我们看下执行结果：

```verilog
Thread :[ 1 ] run
Thread :[ 0 ] run
Thread :[ 2 ] run
Thread :[ 4 ] run
Thread :[ 3 ] run

Process finished with exit code 0
```



#### 2. 实现 `Runnable` 接口

同样我们也是先看下源码：

![1596858585229](http://123.57.45.66/images/autu_blog/concurrent/2020-08-11-concurrent-thread/1596858585229.png)

我们可以看到 `Runnable` 的内部只有一个**抽象方法 `run()`**。

因此：要启动线程就要实现 `Runnable` 接口并重写它的 `run()` 方法。

**由于java不支持多继承，如果自己的类已经继承了其他类，要启动线程就要实现 `Runnable` 接口并重写它的 `run()` 方法**

实操：

```java
public class Test {
    public static void main( String[] args ) {
        for (int i = 0; i < 5; i++) {
            // 创建实现了Runnable接口的实例
            RunnableTest runnableTest = new RunnableTest();
            // 启动线程，线程名为 i
            new Thread(runnableTest, i + "").start();
        }
    }
}

public class RunnableTest implements Runnable {

    @Override
    public void run() {
        // 打印线程名
        System.out.println("Runnable : [" + Thread.currentThread().getName() + "] run");
    }
}
```

执行结果：

```verilog
Runnable : [0] run
Runnable : [1] run
Runnable : [2] run
Runnable : [3] run
Runnable : [4] run

Process finished with exit code 0
```

`Runnable` 接口有 `@FunctionalInterface` 注解标识，说明Runnable也是一个**函数式接口**。那么我们如何使用函数式接口创建并启动线程呢？

直接上代码：

```java
public class Test {
    public static void main( String[] args ) {
        for (int i = 0; i < 5; i++) {
            new Thread(() -> {
                System.out.println("Runnable : [" + Thread.currentThread().getName() + "] run");
            }, i + "").start();
        }
    }
}
```

执行结果：

```verilog
Runnable : [0] run
Runnable : [1] run
Runnable : [2] run
Runnable : [3] run
Runnable : [4] run

Process finished with exit code 0
```



#### 3. `Callable/Future` ，带返回值

同样，我们还是来看下 `Callable` 的源码：

![1596868405660](http://123.57.45.66/images/autu_blog/concurrent/2020-08-11-concurrent-thread/1596868405660.png)

通过源码我们发现，`Callable` 同样是一个**函数式接口**，并且只有一个 `call()`  抽象方法。

我们既然也看了源码，那么 `Callable` 是如何将结果返回的呢？

我们先来实操下：

```java
public class Test {
    public static void main( String[] args ) throws ExecutionException, InterruptedException {
        ExecutorService executor = Executors.newSingleThreadExecutor();

        System.out.println("提交任务前：" + getStringDate());

        Future future = executor.submit(() -> {
            Thread.sleep(3000);
            return "Callable Run";
        });

        System.out.println("提交任务后，获取结果前：" + getStringDate());

        System.out.println("返回结果：" + future.get());

        System.out.println("获取结果后：" + getStringDate());
        executor.shutdown();
    }


    public static String getStringDate() {
        Date currentTime = new Date();
        SimpleDateFormat formatter = new SimpleDateFormat("HH:mm:ss");
        return formatter.format(currentTime);
    }
}
```

我们看下执行结果：

```verilog
提交任务前：14:48:39
提交任务后，获取结果前：14:48:39
返回结果：Callable Run
获取结果后：14:48:42

Process finished with exit code 0
```

在 `Callable` 中，sleep了3秒，通过执行结果，我们不难发现，在调用submit提交任务之后，主线程本来是继续运行了，但是运行到 `future.get()` 的时候就阻塞住了，一直等到任务执行完毕，拿到了返回的返回值，主线程才会继续运行。

**这里注意一下：**

* 阻塞性是因为调用 `get()` 方法时，任务还没有执行完，所以会一直等到任务完成，形成了阻塞。
* 任务是在调用 `submit` 方法时就开始执行了，如果在调用 `get()` 方法时，任务已经执行完毕，那么就不会造成阻塞。

下面我们在调用 `get()` 方法前，先睡4秒再取结果，就能马上取到结果。

```java
public static void main( String[] args ) throws ExecutionException, InterruptedException {
    ExecutorService executor = Executors.newSingleThreadExecutor();

    System.out.println("提交任务前：" + getStringDate());

    Future future = executor.submit(() -> {
        Thread.sleep(3000);
        return "Callable Run";
    });

    System.out.println("提交任务后：" + getStringDate());

    Thread.sleep(4000);
    System.out.println("已经睡了4秒,开始获取结果 "+getStringDate());

    System.out.println("返回结果：" + future.get());

    System.out.println("获取结果后：" + getStringDate());
    executor.shutdown();
}
```

执行结果：

```verilog
提交任务前：14:56:33
提交任务后：14:56:33
已经睡了4秒,开始获取结果 14:56:37
返回结果：Callable Run
获取结果后：14:56:37

Process finished with exit code 0
```

根据执行结果，我们可以看到，因为睡了4秒，任务已经执行完毕，所以get方法立马就得到了结果。



### 二、线程生命周期

#### 1. java中线程的6种状态

![1596958994589](http://123.57.45.66/images/autu_blog/concurrent/2020-08-11-concurrent-thread/1596958994589.png)

**Java语言定义了6种线程状态：**

- New：创建后尚未启动的线程处于这种状态。
- RUNNABLE：包括Running和Ready。线程开启 `start()` 方法，会进入该状态，在虚拟机内执行的。
- Waiting：无限的等待另一个线程的特定操作。
- Timed Waiting：有时限的等待另一个线程的特定操作。
- 阻塞（Blocked）：在程序等待进入同步区域的时候，线程将进入这种状态，在等待监视器锁。
- 结束（Terminated）：已终止线程的线程状态，线程已经结束执行。

**操作系统中线程状态：**

在操作系统中，一个线程真实存在的状态只有：

* ready：表示线程已经被创建，正在等待系统调度分配CPU使用权。
* running：表示获得了CPU使用权，正在进行运算。
* waiting：表示线程等待，让出CPU资源给其他线程使用

再加上 **新建状态** 和 **死亡状态**，操作系统一共5种运行状态



#### 2. 线程状态演示

首先看一段演示代码：

```java
public class ThreadStatusTest {
    
    public static void main(String[] args) {
        new Thread(() -> {
            while(true) {
                try {
                    TimeUnit.SECONDS.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        },"STATUS_01").start();  //阻塞状态

        new Thread(() -> {
            while(true) {
                synchronized (ThreadStatusTest.class) {
                    try {
                        ThreadStatusTest.class.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        },"STATUS_02").start(); //阻塞状态

        // 两个线程持有同一把锁
        new Thread(new BlockedDemo(),"BLOCKED-DEMO-01").start();
        new Thread(new BlockedDemo(),"BLOCKED-DEMO-02").start();

    }
    
    // 模拟Blocked状态
    static class BlockedDemo extends Thread {
        @Override
        public void run() {
            synchronized (BlockedDemo.class) {
                while(true) {
                    try {
                        TimeUnit.SECONDS.sleep(100);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }
    }
}
```

**查看线程状态步骤：**

* 运行main方法

* 在终端输入 `jps` 命令，查看java进程号，找到测试进程号。

  ![1596955706350](http://123.57.45.66/images/autu_blog/concurrent/2020-08-11-concurrent-thread/1596955706350.png)

* 通过 `jstack [进程号]` 命令查看jvm中当前时刻线程状态。

  ![1596955924525](http://123.57.45.66/images/autu_blog/concurrent/2020-08-11-concurrent-thread/1596955924525.png)



#### 3. 分析线程状态 

**STATUS_01 & STATUS_02** 

![1596958505667](http://123.57.45.66/images/autu_blog/concurrent/2020-08-11-concurrent-thread/1596958505667.png)

* STATUS_01 ：调用 `sleep` 方法，睡眠100s，因此线程状态为 `TIMED_WAITING` 。
* STATUS_02 ：在 `synchronized` 代码块内调用 `wait` 方法，因此线程状态为 `WAITING`。



**BLOCKED-DEMO-01 & BLOCKED-DEMO-02** 

![1596959626480](http://123.57.45.66/images/autu_blog/concurrent/2020-08-11-concurrent-thread/1596959626480.png)

这两个线程同时持有 `BlockedDemo.class` 锁，并且拿到锁后睡眠100s

* `BLOCKED-DEMO-01` 线程抢到锁时，线程状态为 `TIMED_WAITING` 。
* `BLOCKED-DEMO-02` 未抢到锁，因此线程状态为 `BLOCKED` 。



### 三、线程启动

![1596965482584](http://123.57.45.66/images/autu_blog/concurrent/2020-08-11-concurrent-thread/1596965482584.png)

通过前面 创建线程的方式 的学习，我们知道，创建线程是调用 `Thread` 类中本地方法 `start0()` 启动线程。那么本地线程是如何创建线程并执行线程呢？接下来我们详细剖析这个过程。

**`jvm` 层面调用并创建线程：**

* 通过本地方法 `start0()` 映射到 `JVM` 中的 `JVM_StartThread` 方法，并执行该方法。
* 在 `JVM_StartThread` 方法中，首先判断Java线程是否已经启动，如果已经启动过，则会抛异常；Java线程没有启动过，则开始创建Java线程。
* `JVM` 通过 `pthread_create()` 创建一个系统内核线程，并指定内核线程的初始运行地址，即一个方法指针。
* 在内核线程的初始运行方法中，利用 `JavaCalls` 模块，调用java线程的 `run()` 方法，开始java级别的线程执行。



### 四、线程停止

#### 1. Stop 方法终止

`thread.stop()` 来强行终止线程，但是stop方法是很危险的，就象突然关闭计算机电源，而不是按正常程序关机一样，可能会产生不可预料的结果。

`thread.stop()` 在结束一个线程时并**不会保证线程的资源正常释放**，因此，并**不推荐使用stop方法**来终止线程。



#### 2. Interrupt中断

当其他线程通过调用当前线程的 `thread.interrupt()` 方法，表示向当前线程打个招呼，告诉他可以中断线程的执行了。至于**什么时候中断，取决于当前线程自己**。 



****

**下面我们详细讲解 Interrupt中断方式**，讲解之前我们先了解几个概念：

> `interrupt()` ：将调用该方法的对象所表示的线程标记一个停止标记，并不是真的停止该线程。
>
> `interrupted()` ：获取**当前线程**的中断状态，并且会清除线程的状态标记。是一个是静态方法。
>
> `isInterrupted()` ：获**取调用该方法的对象所表示的线程的中断状态**，不会清除线程的状态标记。是一个实例方法。



首先我们来看一段代码：

```java
public class InterruptDemo extends Thread {

    public static void main(String[] args) {
        InterruptDemo interruptDemo = new InterruptDemo();
        interruptDemo.start();
        // 调用interrupt
		interruptDemo.interrupt();
    }

    @Override
    public void run() {
        System.out.println("线程开始执行时间：" + getStringDate());
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println("sleep结束时间：" + getStringDate());

        for (int i = 0; i < 10; i++) {
            System.out.println("Interrupt Test Demo " + i);
        }

    }
	// 时间工具
    public static String getStringDate() {
        Date currentTime = new Date();
        SimpleDateFormat formatter = new SimpleDateFormat("HH:mm:ss");
        return formatter.format(currentTime);
    }

}
```

执行结果：

```verilog
线程开始执行时间：22:51:40
java.lang.InterruptedException: sleep interrupted
	at java.lang.Thread.sleep(Native Method)
	at com.example.springbootthreaddemo.demo04.InterruptDemo.run(InterruptDemo.java:23)
sleep结束时间：22:51:40
Interrupt Test Demo 0
Interrupt Test Demo 1
Interrupt Test Demo 2
Interrupt Test Demo 3
Interrupt Test Demo 4
Interrupt Test Demo 5
Interrupt Test Demo 6
Interrupt Test Demo 7
Interrupt Test Demo 8
Interrupt Test Demo 9

Process finished with exit code 0
```

去掉 `interruptDemo.interrupt();` ，或将其注释，执行结果：

```verilog
线程开始执行时间：22:52:17
sleep结束时间：22:52:20
Interrupt Test Demo 0
Interrupt Test Demo 1
Interrupt Test Demo 2
Interrupt Test Demo 3
Interrupt Test Demo 4
Interrupt Test Demo 5
Interrupt Test Demo 6
Interrupt Test Demo 7
Interrupt Test Demo 8
Interrupt Test Demo 9

Process finished with exit code 0
```



##### ① `interrupt()` 

根据上述执行结果，我们发现 **`interrupt()` 的作用** 主要为两点：

- 设置一个线程终止的标记（共享变量的值 true）
- 唤醒处于阻塞状态下的线程

接下来，我们再看下 `jvm` 中 `interrupt` 相关 C++ 核心源码：

[os_linux.cpp](http://hg.openjdk.java.net/jdk7u/jdk7u/hotspot/file/677234770800/src/os/linux/vm/os_linux.cpp)

```c++
void os::interrupt(Thread* thread) {
	assert(Thread::current() == thread || Threads_lock->owned_by_self(),
		"possibility of dangling Thread pointer");
 
	//获取系统native线程对象
	OSThread* osthread = thread->osthread();

	if (!osthread->interrupted()) {
		osthread->set_interrupted(true); //设置中断状态为true
		//内存屏障，使osthread的interrupted状态对其它线程立即可见
		OrderAccess::fence();
		//_SleepEvent用于Thread.sleep,线程调用了sleep方法，则通过unpark唤醒
		ParkEvent * const slp = thread->_SleepEvent ;
		if (slp != NULL) slp->unpark() ;
	}
 
	//_parker用于concurrent相关的锁，此处同样通过unpark唤醒
	if (thread->is_Java_thread())
		((JavaThread*)thread)->parker()->unpark();
	//Object.wait()唤醒
	ParkEvent * ev = thread->_ParkEvent ;
	if (ev != NULL) ev->unpark() ;
 
}
```

看完 `jvm` 源码，果然和我们猜测一样。



##### ②`interrupted()` 

我们先看下 `jdk` 源码：

```java
public static boolean interrupted() {
    return currentThread().isInterrupted(true);
}
```



##### ③ `isInterrupted()` 

同样我们看下 `jdk` 源码：

```java
public boolean isInterrupted() {
    return isInterrupted(false);
}
```

`isInterrupted()` 方法 `jdk` 源码：

```java
private native boolean isInterrupted(boolean ClearInterrupted);
```



`isInterrupted` 是 `native` 方法，我们看下 `isInterrupted` 方法在 `jvm` 中的实现：

[thread.cpp](http://hg.openjdk.java.net/jdk7u/jdk7u/hotspot/file/677234770800/src/share/vm/runtime/thread.cpp)

```c++
bool Thread::is_interrupted(Thread* thread, bool clear_interrupted) {
	trace("is_interrupted", thread);
	debug_only(check_for_dangling_thread_pointer(thread);)
	// Note:  If clear_interrupted==false, this simply fetches and
	// returns the value of the field osthread()->interrupted().
	return os::is_interrupted(thread, clear_interrupted);
}
```

`linux` 平台对 `os::is_interrupted()` 的实现为：

[os_linux.cpp](http://hg.openjdk.java.net/jdk7u/jdk7u/hotspot/file/677234770800/src/os/linux/vm/os_linux.cpp)

```c++
bool os::is_interrupted(Thread* thread, bool clear_interrupted) {
	assert(Thread::current() == thread || Threads_lock->owned_by_self(),
		"possibility of dangling Thread pointer");
 
	OSThread* osthread = thread->osthread();
 
	bool interrupted = osthread->interrupted();  //获取线程中断状态
 
    // interrupted 是否设置线程中断状态，clear_interrupted 是否清除线程中断状态
	if (interrupted && clear_interrupted) {
		osthread->set_interrupted(false);  //清除线程中断状态，重置为false
		// consider thread->_SleepEvent->reset() ... optional optimization
	}
 
	return interrupted;
}
```

**小结：**

* `interrupted()` 和 `isInterrupted()` 在 `jvm` 层面均调用到 `Thread::is_interrupted` 方法
* 只是清除中断标记 `clear_interrupted` 的值不同
* `interrupted()` 清除中断标记为 **true** 
* `isInterrupted()` 清除中断标记为 **false** 



##### ④ 中断异常（ `InterruptedException` ），复位

至于在 **① `interrupt()`** 的案例中 interrupt 后，会**马上停止 sleep，并抛出异常**。这其中的原由是为何呢？

我们还是到 `jvm` 源码中去探索：

```c++
JVM_ENTRY(void, JVM_Sleep(JNIEnv* env, jclass threadClass, jlong millis))
 JVMWrapper("JVM_Sleep"); 

if (millis < 0) {
       THROW_MSG(vmSymbols::java_lang_IllegalArgumentException(), "timeout value is negative"); } 

//判断并清除线程中断状态，如果中断状态为true,抛出中断异常
if (Thread::is_interrupted (THREAD, true) && !HAS_PENDING_EXCEPTION) { 
      THROW_MSG(vmSymbols::java_lang_InterruptedException(), "sleep interrupted"); 
}
...
```

`Thread.sleep` 最终调用 `JVM_Sleep` 方法：

* 在该方法中，同样调用 `Thread::is_interrupted` ，清除中断状态。

* 在清除中断状态后，将线程中断中断复位，即将中断状态置为 **false**（ `set_interrupted(false)` ）。

* 最后抛出异常 `THROW_MSG` 



**小结：**

到这里我们就明白了，为什么sleep 3秒，马上停止sleep并抛出了 `InterruptedException` 异常。

* 调用 `interrupt()` 后，将线程中断标记置为 true，再通过 `unpark` 唤醒阻塞中的线程。
* 线程唤醒后，必然会调用并执行 `JVM_Sleep` 方法，因此，抛出异常。



#### 3.  中断标记 复位演示

我们还是先看一段代码：

```java
public class InterruptDemo extends Thread {

    public static void main(String[] args) throws InterruptedException {
        InterruptDemo interruptDemo = new InterruptDemo();
        interruptDemo.start();
        interruptDemo.interrupt();
    }

    @Override
    public void run() {
        System.out.println("线程开始执行时间：" + getStringDate());
        System.out.println("当前线程中断状态[ 1 ]：" + Thread.currentThread().isInterrupted());
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            // 可以不做处理、打印日志、抛出异常、继续中断
            e.printStackTrace();
            System.out.println("当前线程中断状态[ 2 ]：" + Thread.currentThread().isInterrupted());
            System.out.println("线程中断时间：" + getStringDate());

            // 继续中断
            Thread.currentThread().interrupt();

            System.out.println("当前线程中断状态[ 3 ]：" + Thread.currentThread().isInterrupted());
            return;
        }

        System.out.println("sleep结束时间：" + getStringDate());

        for (int i = 0; i < 10; i++) {
            System.out.println("Interrupt Test Demo " + i);
        }

    }

    public static String getStringDate() {
        Date currentTime = new Date();
        SimpleDateFormat formatter = new SimpleDateFormat("HH:mm:ss");
        return formatter.format(currentTime);
    }

}
```



正常执行，即将 `interruptDemo.interrupt();` 注释掉：

```verilog
线程开始执行时间：22:29:12
当前线程中断状态[ 1 ]：false
sleep结束时间：22:29:16
Interrupt Test Demo 0
Interrupt Test Demo 1
Interrupt Test Demo 2
Interrupt Test Demo 3
Interrupt Test Demo 4
Interrupt Test Demo 5
Interrupt Test Demo 6
Interrupt Test Demo 7
Interrupt Test Demo 8
Interrupt Test Demo 9

Process finished with exit code 0
```



程序执行中断逻辑结果：

```verilog
线程开始执行时间：22:33:50
当前线程中断状态[ 1 ]：true
java.lang.InterruptedException: sleep interrupted
	at java.lang.Thread.sleep(Native Method)
	at com.example.springbootthreaddemo.demo04.InterruptDemo04.run(InterruptDemo04.java:24)
当前线程中断状态[ 2 ]：false
线程中断时间：22:33:50
当前线程中断状态[ 3 ]：true

Process finished with exit code 0
```

通过结果我们可以看到：

* 执行 `interruptDemo.interrupt();` 后线程中断状态为 true，即中断状态[ 1 ] 。
* 随后便被复位，即中断状态[ 2 ] 。
* 当在catch代码块中再次中断后，线程中断状态变为 true，即中断状态[ 3 ] 。

**注意：** 

* `Object.wait` 、 `Thread.sleep` 、 `Thread.join` 会 抛 出 `InterruptedException` 
* 在当前线程catch中可以对中断做出相应的处理：抛出异常、记录日志、return/break、不做处理，具体方案自己把握。



再看一段代码，演示手动复位：

```java
public class InterruptDemo02 extends Thread {

    public static void main(String[] args) {

        InterruptDemo02 interruptDemo02 = new InterruptDemo05();
        interruptDemo02.start();
        interruptDemo02.interrupt();
    }


    @Override
    public void run() {
        for (int i = 0; i < 5; i++) {
            System.out.println(i);
            if (Thread.currentThread().isInterrupted()) {
                System.out.println("线程中断状态：" + Thread.currentThread().isInterrupted());
                // java中的复位
                Thread.interrupted();
            }
        }
    }
}
```



正常执行，将 `Thread.interrupted();` 注释，即不进行复位：

```verilog
0
线程中断状态：true
1
线程中断状态：true
2
线程中断状态：true
3
线程中断状态：true
4
线程中断状态：true

Process finished with exit code 0
```



复位结果：

```verilog
0
线程中断状态：true
1
2
3
4

Process finished with exit code 0
```



##### 为什么要复位

`Thread.interrupted()` 是属于当前线程的，是当前线程对外界中断信号的一个响应，表示自己已经得到了中断信号， 但不会立刻中断自己，具体什么时候中断由自己决定，让 外界知道在自身中断前，他的中断状态仍然是 false，这就 是复位的原因。



### 参考文档：

JVM之Java线程启动流程：<https://zhuanlan.zhihu.com/p/34414549>

JVM源码分析参考：<https://blog.csdn.net/qq_26222859/article/details/81112446>

java线程的中断的方式原理分析：<https://blog.csdn.net/oldshaui/article/details/106952102>
