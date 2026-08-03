> Thread类是Java多线程编程的基石，每个线程都对应一个Thread实例，由JVM进行管理和调度
> 
> Thread类实现了Runnable接口，下面先介绍一下Runnable接口

# Runnable

> ​Runnable​接口最大的意义在于它提供了一种标准协议，任何类只要实现了它的 run​方法，就表示其内部包含了一段可以在线程中执行的代码
> 
> ​Runnable​对象只负责定义“要做什么”（任务逻辑），而 Thread​对象负责“如何执行”（调度、管理）。这符合面向对象设计原则中的单一职责原则
> 
> 因为Java不支持多继承，有时候一个类已经继承了其他类，就无法再继承Thread​类，这时候想要实现多线程就可以实现Runnable​接口，也可以满足多线程的需求

以下是Runnable接口的源码，十分简洁
```java
@FunctionalInterface
public interface Runnable {
    /**
     * When an object implementing interface <code>Runnable</code> is used
     * to create a thread, starting the thread causes the object's
     * <code>run</code> method to be called in that separately executing
     * thread.
     * <p>
     * The general contract of the method <code>run</code> is that it may
     * take any action whatsoever.
     *
     * @see     java.lang.Thread#run()
     */
    public abstract void run();
}
```

---

# Thread

#### 初始化方式

1. 继承Thread类：直接继承Thread类并重写run方法
    ```Java
    class MyThread extends Thread {
        @Override
        public void run() {
            System.out.println("线程执行任务，继承自Thread类");
        }
    }
    // 使用
    MyThread myThread = new MyThread();
    myThread.start(); // 启动新线程
    ```
    
2. 实现Runnable接口（更推荐）：定义一个实现Runnable接口的类，将其实例作为参数传给Thread对象。这样做线程任务与Thread类本身解耦，更灵活
    ```Java
    class MyRunnable implements Runnable {
        @Override
        public void run() {
            System.out.println("线程执行任务，实现了Runnable接口");
        }
    }
    // 使用
    Thread thread = new Thread(new MyRunnable());
    thread.start();
    ```
    
3. 使用匿名内部类与Lambda表达式：对于简单任务，这两种方式可以使代码更简洁
    ```java
    
    // 匿名内部类 (基于Runnable)
    Thread thread1 = new Thread(new Runnable() {
        @Override
        public void run() {
            System.out.println("匿名内部类方式");
        }
    });
    
    // Lambda表达式 (最简洁)
    Thread thread2 = new Thread(() -> {
        System.out.println("Lambda表达式方式");
    });
    
    thread1.start();
    thread2.start();
    ```
    

#### 重要方法

> 掌握Thread类的核心方法是进行多线程编程的关键

- ​start()​与 run()​
	如前所述，start()​用于启动新线程，run()​定义任务逻辑。
    
- ​sleep(long millis)​
    让当前线程休眠指定的毫秒数，重要特性是休眠期间不会释放已持有的锁
    ```Java
    Thread sleepThread = new Thread(() -> {
        System.out.println("线程开始休眠2秒");
        try {
            Thread.sleep(2000); // 休眠2000毫秒
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("线程唤醒");
    });
    sleepThread.start();
    ```
    
- ​join()​
    等待指定线程执行完毕。比如主线程需要等待子线程t完成后再继续，可以在主线程中调用t.join()​。
    ```Java
    Thread worker = new Thread(() -> {
        System.out.println("子线程开始工作");
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("子线程工作完成");
    });
    
    worker.start();
    System.out.println("主线程等待子线程...");
    try {
        worker.join(); // 主线程在此等待worker线程结束
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    System.out.println("主线程继续执行");
    ```
    
- ​interrupt()​与中断检查
    用于向线程发送一个中断信号，这是一种协作式中断机制，并非强制停止线程。
    
    - 如果线程因调用sleep​, wait​, join​等方法而阻塞，会抛出InterruptedException​异常。
    - 如果线程正在运行，则只是设置其中断标志位。线程需要自己通过Thread.currentThread().isInterrupted()​或Thread.interrupted()​检查标志位并决定是否退出。
    ```Java
    Thread interruptibleThread = new Thread(() -> {
        while (!Thread.currentThread().isInterrupted()) { // 检查中断标志
            System.out.println("线程运行中...");
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                // 休眠时被中断会抛出异常，并清除中断标志
                System.out.println("捕获到中断异常，退出循环");
                Thread.currentThread().interrupt(); // 重新设置中断标志，以退出循环
            }
        }
        System.out.println("线程安全结束");
    });
    
    interruptibleThread.start();
    try {
        Thread.sleep(3000); // 主线程休眠3秒
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    interruptibleThread.interrupt(); // 3秒后中断子线程
    ```
    
- ​yield()​
    提示调度器当前线程愿意让出CPU，但调度器可以忽略此提示。它会使线程从运行状态回到就绪状态，但不会释放锁。
    
- ​setDaemon(boolean on)​
    将线程设置为守护线程（后台线程）。​JVM在所有用户线程（非守护线程）结束后就会退出，不会等待守护线程执行完毕​。此方法必须在start()​前调用。
    
    ```Java
    Thread daemonThread = new Thread(() -> {
        while (true) {
            System.out.println("守护线程在运行");
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    });
    
    daemonThread.setDaemon(true); // 设置为守护线程
    daemonThread.start();
    // 主线程（用户线程）结束后，即使daemonThread的循环未结束，JVM也会退出
    ```

# Executor

> 能够创建并管理线程池是 Executor​框架相较于直接使用 Thread​类的核心优势

| 特性对比  | 直接使用Thread类                 | Executor框架                                            |
| ----- | --------------------------- | ----------------------------------------------------- |
| 线程管理  | “野线程”：缺乏统一管理，创建和销毁开销大。      | 线程池：线程复用，降低资源消耗，提高响应速度。                               |
| 资源控制  | 难以控制：可无限制创建线程，易导致资源竞争或系统瘫痪。 | 可控并发数：有效控制最大并发线程数，提高系统资源利用率。                          |
| 功能丰富性 | 功能单一：主要提供基本的线程创建和运行功能。      | 功能强大：支持定时执行、定期执行、并方便地处理有返回值的任务。                       |
| 编程模型  | 任务与执行耦合：线程的创建和执行与任务逻辑绑定紧密。  | 任务提交与执行解耦：开发者只需提交任务（Runnable/Callable），由框架负责执行调度，更优雅。 |

####  深入了解Executor的线程池类型

- 固定大小线程池 (newFixedThreadPool​)
    - ​特点​：线程数量固定。
    - ​适用场景​：适用于负载较重、需要稳定线程数的服务器场景，可以控制资源消耗。
- 可缓存线程池 (newCachedThreadPool​)
    - ​特点​：线程数量可根据需求自动回收空闲线程、添加新线程。
    - ​适用场景​：适用于执行很多短期异步任务的小程序，或负载较轻的服务器。
- 单线程化线程池 (newSingleThreadExecutor​)
    - ​特点​：只有一个工作线程，所有任务按提交顺序执行。
    - ​适用场景​：需要保证任务顺序执行，并且不需要并发的情况。
- 定时/周期性线程池 (newScheduledThreadPool​)
    - ​特点​：支持定时及周期性任务执行。
    - 适用场景：需要执行延迟或定期任务，如轮询、心跳检测等，比 Timer​更安全强大

#### 核心组件简介

​Executor​框架不只是一个“线程池”，它是一套完整的体系，主要由以下核心组件构成：

- ​Executor​：最基础的接口，定义了执行任务的 execute​方法。
- ​ExecutorService​：继承了 Executor​，增加了更丰富的生命周期管理方法（如 shutdown​）、以及提交支持返回值的任务（submit​方法）的方法。
- ​Executors​：一个工具类（工厂类），上面提到的各种线程池都是通过它来创建的。
- ​ThreadPoolExecutor​：是线程池的核心实现类，提供了许多参数可供调整，允许开发者自定义线程池的详细行为。
- ​Future​与 Callable​：通过 Callable​接口定义的任务可以有返回值，提交任务后会返回一个 Future​对象，通过它可以获取任务执行结果或取消任务

##### 示例代码：

```Java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ExecutorDemo {
    public static void main(String[] args) {
        // 创建一个固定大小的线程池
        ExecutorService executor = Executors.newFixedThreadPool(3);
        
        // 提交Runnable任务给线程池执行
        for (int i = 0; i < 5; i++) {
            final int taskId = i;
            executor.execute(() -> { // 这里提交的就是一个Runnable
                System.out.println("执行任务: " + taskId + "，由线程: " + Thread.currentThread().getName());
            });
        }
        // 关闭线程池
        executor.shutdown();
    }
}
```

#### ThreadFactory

> ThreadFactory 是 Java 并发编程中一个用于标准化创建新线程的接口。它属于 java.util.concurrent​包，核心价值在于将线程的创建过程封装起来，让你能更精细地控制线程的属性，并提升代码的可维护性。

示例代码

```Java
public class NamedThreadFactory implements ThreadFactory {
    private final AtomicInteger threadNumber = new AtomicInteger(1);
    private final String namePrefix; // 名称前缀

    public NamedThreadFactory(String poolName) {
        this.namePrefix = poolName + "-thread-";
    }

    @Override
    public Thread newThread(Runnable r) {
        // 创建线程并设置名称
        Thread t = new Thread(r, namePrefix + threadNumber.getAndIncrement());
        t.setDaemon(false); // 设为非守护线程
        t.setPriority(Thread.NORM_PRIORITY);
        // 设置未捕获异常处理器，避免任务静默失败
        t.setUncaughtExceptionHandler((thread, throwable) -> {
            System.err.println("线程 " + thread.getName() + " 发生未捕获异常: " + throwable.getMessage());
        });
        return t;
    }
}

---

public class ThreadPoolExample {
    public static void main(String[] args) {
        // 1. 创建自定义线程工厂
        ThreadFactory customFactory = new NamedThreadFactory("MyAppPool");
        
        // 2. 创建线程池时传入自定义工厂
        ExecutorService executor = new ThreadPoolExecutor(
            5, // 核心线程数
            10, // 最大线程数
            60L, TimeUnit.SECONDS,
            new LinkedBlockingQueue<>(100),
            customFactory // 使用自定义工厂
        );
        
        // 3. 提交任务
        executor.submit(() -> {
            System.out.println("任务执行线程: " + Thread.currentThread().getName());
        });
        
        executor.shutdown();
    }
}
```