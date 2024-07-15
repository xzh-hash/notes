## ThreadPoolExecutor原理

### 01.创建线程的几种方式

- 继承Thread

```java
public class ThreadPoolExecutorDemo {

    public static void main(String[] args) {
        MyThread myThread = new MyThread();
        myThread.start();

        System.out.println("主线程结束");
    }

    static class MyThread extends Thread{

        @Override
        public void run() {
            System.out.println("this is MyThread");
        }
    }
}
```



- 实现Runnable

```java
public class ThreadPoolExecutorDemo {

    public static void main(String[] args) {

        //1. 使用匿名内部类的写法
        Thread t0 = new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("子线程t0");
            }
        });
        t0.start();

        //2. 使用lambda表达式的写法
        Thread t1 = new Thread(()->{
            System.out.println("这是子线程t1");
        });
        t1.start();

        System.out.println("主线程结束");
    }


}

```





- 实现Callable

```java
public class ThreadPoolExecutorDemo {

    public static void main(String[] args) {

        //把Callable对象传给FutureTask
        FutureTask<String> futureTask = new FutureTask(new Callable() {
            @Override
            public Object call() throws Exception {
                return "子线程futuretask";
            }
        });

        Thread t3 = new Thread(futureTask);
        t3.start();

        //这里是个阻塞方法，会等待子线程返回result后，主线程才会继续执行
        try {
            String result = futureTask.get();
            System.out.println(result);
        } catch (Exception e) {
            e.printStackTrace();
        }

        System.out.println("主线程结束");
    }


}

```

### 02.为什么使用线程池

以上我们都是在需要的时候创建一个线程，这样的话如果我们服务器请求数量增大的时候，每次需要时候开启一个线程，会有如下副作用：

1.每个线程创建和销毁是需要额外的系统资源，如果频繁创建和销毁必然会导致消耗大量的系统资源，很多时候，我们的线程执行所耗费的资源可能比创建与销毁这个线程还要少。

2.每个线程运行时候也是需要消耗系统资源，如果我们不控制线程的数量，任意创建，那么系统中可能会线程泛滥，造成CPU频繁的在多线程之间来回切换，导致系统性能下降；另外也避免线程过多，耗尽系统资源。

### 03.ThreadPoolExecutor类层次

ThreadPoolExecutor是位于java.util.concurrent包中的一个线程池的实现，在JDK1.5时候引入的。专门帮助我们创建和管理线程。可以控制我们创建的线程总数，同时还可以重复利用已经存在的线程。

![image-20220609211040732](https://woniumd.oss-cn-hangzhou.aliyuncs.com/java/yangguang/20220609211046.png)

### 04.ThreadPoolExecutor七个参数

```java
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) 
```

1.corePoolSize:  核心线程数

2.maximumPoolSize:  最大线程数

3.keepAliveTime:  线程的空闲时间

4.unit:  空闲时间的单位（秒、分、小时等等）

5.workQueue:  等待队列

6.threadFactory:  线程工厂

7.handler:  拒绝策略

​	--抛出异常

​	--提交任务的线程自己执行

​	--丢弃任务

​    --丢弃等待队列中等待最久的任务

### 05.ThreadPoolExecutor线程增长策略

问题：

1. 当向一个新创建的线程池中提交任务时候，线程池何时创建新线程、何时把任务放入等待队列、何时执行拒绝策略？

2. 当一个线程池达到最大线程数以后，没有任务执行，哪些线程会被销毁？

3. 线程池中的最大线程数可以设置为多少？

4. 线程池中的等待队列使用有界队列还是无界队列？

5. 线程池中如何来复用一个线程？

#### 5.1.任务提交流程

![image-20220609222015253](https://woniumd.oss-cn-hangzhou.aliyuncs.com/java/yangguang/20220609222015.png)

#### 5.2.代码演示

```java
//execute方法
public class TestThreadPoolExecutor {

    private static volatile boolean flag = true;

    public static void main(String[] args) throws InterruptedException {
        ThreadPoolExecutor executor = new ThreadPoolExecutor(
                2, 4,
                20, TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(6),
                Executors.defaultThreadFactory(),
//                new ThreadPoolExecutor.AbortPolicy()  //默认拒绝策略，直接丢弃任务并抛出异常
//                new ThreadPoolExecutor.CallerRunsPolicy() //提交任务的线程自己执行这个任务
                new ThreadPoolExecutor.DiscardOldestPolicy() //从队列中删除等待时间最久的任务，不抛出异常
//                new ThreadPoolExecutor.DiscardPolicy() //直接抛弃最新的任务，不抛出异常

        );

        for (int i = 0; i < 15; i++) {
            try {
                executor.execute(new Mythread("t" + i));
                Thread.sleep(1000);
            } catch (Exception e) {
                System.out.println("exception：" + Thread.currentThread().getName() + "::" + e.getMessage());
//                如果抛出异常说明已经触发了拒绝策略，此时修改flag，让所有任务执行完毕
                flag = false;

            } finally {
                System.out.println("正在执行任务的线程数:" + executor.getActiveCount()
                        + ":当前池中的线程总数:" + executor.getPoolSize()
                        + ":设置的核心线程数:" + executor.getCorePoolSize()
                        + ":设置的最大线程数:" + executor.getMaximumPoolSize()
                        + ":池中达到过的最大线程数:" + executor.getLargestPoolSize()
                        + ":已经完成任务数:" + executor.getCompletedTaskCount()
                        + ":等待队列中的任务数:" + executor.getQueue().size()
                );
            }
        }
        System.out.println("===========" + new Date() + "============");
        for (int i = 0; i < 10; i++) {
            Thread.sleep(2000);
            System.out.println("正在执行任务的程数:" + executor.getActiveCount()
                    + ":当前池中的线程总数:" + executor.getPoolSize()
                    + ":设置的核心线程数:" + executor.getCorePoolSize()
                    + ":设置的最大线程数:" + executor.getMaximumPoolSize()
                    + ":池中达到过的最大线程数:" + executor.getLargestPoolSize()
                    + ":已经完成任务数:" + executor.getCompletedTaskCount()
                    + ":等待队列中的任务数:" + executor.getQueue().size());

            System.out.println("*****" + new Date() + "*****");

        }

        //如果执行到这里flag还是true，那么说明执行的拒绝策略为DiscardOldestPolicy或者
        //此时把flag设置为false，让所有的线程把任务执行完毕
        if(flag){
            flag=false;
            Thread.sleep(1000);
            System.out.println("========所有任务都执行完后，再次查看线程池的状态：==========");
            for (int i = 0; i < 5; i++) {
                System.out.println("&&&&&&&&&&&&" + new Date() + "&&&&&&&&&&&&");
                Thread.sleep(6000);
                System.out.println("正在执行任务的程数:" + executor.getActiveCount()
                        + ":当前池中的线程总数:" + executor.getPoolSize()
                        + ":设置的核心线程数:" + executor.getCorePoolSize()
                        + ":设置的最大线程数:" + executor.getMaximumPoolSize()
                        + ":池中达到过的最大线程数:" + executor.getLargestPoolSize()
                        + ":已经完成任务数:" + executor.getCompletedTaskCount()
                        + ":等待队列中的任务数:" + executor.getQueue().size());



            }

        }

        executor.shutdown();

    }


    static class Mythread implements Runnable {

        private String taskName;

        public Mythread(String taskName) {
            this.taskName = taskName;
        }

        @Override
        public void run() {
            //如果main线程执行这个任务，说明已经执行了拒绝策略，CallerRunsPolicy，
            // 此时可以把flag设置为false,让所有的线程把任务执行完毕（因为已经看到了拒绝策略的执行）
            if ("main".equals(Thread.currentThread().getName())) {
                flag = false;
            }
            //main线程以外 的其他线程执行到这里都进行死循环，为了观察线程池如何触发拒绝策略
            while (flag) {

            }
            System.out.println("thread:" + Thread.currentThread().getName() + "::" + "taskName:" + taskName + ":+执行完毕:");
        }
    }
}

```

### 06.ThreadPoolExecutor的状态

#### 6.1.状态以及描述

```properties
     *   RUNNING:  正常运行状态，接收新的任务，并且从等待队列中获取任务获取任务并处理
     *   SHUTDOWN: 不再接收新任务, 但是会继续处理等待队列中的任务
     *   STOP:     不接收新任务，也不会继续处理等待队列中的任务，而且中断正在执行的任务
     *   TIDYING:  所有任务都已经结束,工作线程数字为0,这是个过渡的状态，将会运行terminated()把      *				线程池状态修改为TERMINATED
     *   TERMINATED: terminated() 钩子函数已经执行完毕，线程池终止
```

#### 6.2.状态之间的转换

```properties
     * RUNNING -> SHUTDOWN
     *    当调用了shutdown() 方法，线程池就会转为shutdown状态
     * (RUNNING or SHUTDOWN) -> STOP
     *    当调用了shutdownNow()
     * SHUTDOWN -> TIDYING
     *    当等待队列为空，工作线程数为0，线程池就会从shutdown转为tidying
     * STOP -> TIDYING
     *    当等待队列为空，线程池就会从stop转为tyding
     * TIDYING -> TERMINATED
     *    当terminated()这个钩子函数被调用，线程池由tidying转为terminated状态
```



### 07.ThreadPoolExecutor的属性

```java
    //ctl是整个线程池的状态控制参数，这个32位的整数可以用来表示两个状态
    //1.线程池运行状态 ----用整型的最高3位来表示
    //2.线程池中工作线程数量 ----用整型的低29位来表示
	private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
	//根据Integer中的定义Integer.SIZE=32，这里COUNT_BITS是29
    private static final int COUNT_BITS = Integer.SIZE - 3;
	//最大工作线程数，这里用的是(2^29)-1
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

    //以下是线程池的运行状态，保存在ctl的高3位
    //1110 0000 0000 0000 0000 0000 0000 0000
    private static final int RUNNING    = -1 << COUNT_BITS;
	//0000 0000 0000 0000 0000 0000 0000 0000
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
	//0001 0000 0000 0000 0000 0000 0000 0000
    private static final int STOP       =  1 << COUNT_BITS;
	//0010 0000 0000 0000 0000 0000 0000 0000
    private static final int TIDYING    =  2 << COUNT_BITS;
	//0011 0000 0000 0000 0000 0000 0000 0000
    private static final int TERMINATED =  3 << COUNT_BITS;

	//是否允许核心线程超时后被销毁，这个值如果设置为true的话，
	//线程池中的核心线程在idle超时后也会被销毁
	private volatile boolean allowCoreThreadTimeOut;
```

### 08.ThreadPoolExecutor方法

```java
execute(Runnable r):没有返回值，仅仅是把一个任务提交给线程池处理

submit(Runnable r):返回值为Future类型，当任务处理完毕后，通过Future的get()方法获取返回值时候，得到的是null

submit(Runnable r,Object result):返回值为Future类型，当任务处理完毕后，通过Future的get()方法获取返回值时候，得到的是传入的第二个参数result

submit(Callable c):提交一个可以有返回值的线程任务，返回值为Future类型，当任务处理完毕后，通过Future的get()方法获取返回值
    
prestartCoreThread():启动一个线程，等待任务，如果核心线程数已经达到，那么这个方法返回false，否则返回true
    
prestartAllCoreThreads():启动所有的核心线程，返回启动成功的核心线程数    

shutdown():关闭线程池，不接受新任务，但是等待队列中的任务处理完毕才能真正关闭

shutdownNow():立即关闭线程池，不接受新任务，也不再处理等待队列中的任务，同时中断正在执行的线程
    
allowCoreThreadTimeOut(boolean v):设置核心线程池是否可以被销毁，当传入true的时候，核心线程池在空闲时间到达阈值后也会被销毁
    
getActiveCount():获取正在执行任务的线程数
    
getPoolSize():获取当前池中的线程总数

getCorePoolSize():获取设置的核心线程数

getMaximumPoolSize():获取设置的最大线程数

getLargestPoolSize():获取池中达到过的最大线程数

getCompletedTaskCount():获取已经完成任务数

getQueue():获取等待队列

setCorePoolSize(int corePoolSize):设置核心线程数
    
setKeepAliveTime(long time, TimeUnit unit):设置线程的空闲时间
    
setMaximumPoolSize(int maximumPoolSize):设置最大线程数
    
setRejectedExecutionHandler(RejectedExecutionHandler rh):设置拒绝策略
    
setThreadFactory(ThreadFactory tf):设置线程工厂

beforeExecute(Thread t, Runnable r):任务执行之前的钩子函数，这是一个空函数，使用者可以继承ThreadPoolExecutor后重写这个方法，实现其中的逻辑
    
afterExecute(Runnable r, Throwable t):任务执行之后的钩子函数，这是一个空函数，使用者可以继承ThreadPoolExecutor后重写这个方法，实现其中的逻辑
```



代码演示：

```java
public class ThreadPoolExecutorDemo {

    public static void main(String[] args) throws InterruptedException, ExecutionException {

        ThreadPoolExecutor executor = new ThreadPoolExecutor(
                2, 4,
                10, TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(6),
                Executors.defaultThreadFactory(),
                new ThreadPoolExecutor.AbortPolicy()  //默认拒绝策略，直接丢弃任务并抛出异常
        );


        //步骤1
        System.out.println("线程池刚刚创建好，还没有提交任务，那么应该没有任何活动线程");
        System.out.println("线程池活动线程数："+executor.getActiveCount());

        //步骤2
        System.out.println("当调用了prestartAllCoreThreads之后，会启动所有的核心线程，达到核心线程数的设置");
        executor.prestartAllCoreThreads();
        System.out.println("当调用了prestartAllCoreThreads之后,活动线程数："+executor.getActiveCount());

        Thread.sleep(1000);//休眠1秒钟等待核心线程启动完毕

        //步骤3：提交一个有返回值的任务，返回的结果应该等于我们传入的第二个参数：ok
        Future<String> future = executor.submit(new Runnable() {
            @Override
            public void run() {
                System.out.println("子线程：" + Thread.currentThread().getName());
            }
        }, "ok");

        String result = future.get();
        System.out.println("线程返回结果是："+result);

        //步骤5：演示提交Callable类型的任务，并获取返回结果
        List<Future<String>> futureList = new ArrayList();
        for (int i = 0; i < 14; i++) {
            final int index = i;
            Future<String> f = executor.submit(new Callable<String>() {
                @Override
                public String call() throws Exception {
                    return "返回值序号是："+index;
                }
            });
            //把每个提交的任务的返回的Future放入列表中
            futureList.add(f);
        }
        //循环Future列表获取所有的任务结果
        for (Future<String> ft : futureList) {
            System.out.println(ft.get());
        }



        //步骤4：动态修改最大线程数为6，并提交12个任务,查看线程池的最大线程数
        //在修改最大线程数前，先打印一下原来的最大线程数设置
        System.out.println("原最大线程数："+executor.getMaximumPoolSize());
        executor.setMaximumPoolSize(6);
        System.out.println("修改后的最大线程数："+executor.getMaximumPoolSize());
        for (int i = 0; i < 12; i++) {
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    //这里让线程一直死循环，就是希望线程在线程池中不能复用，
                    // 所以线程池一直会创建新的线程，一直达到最大线程数
                    while(true){}
                }
            });
        }
        System.out.println("活动线程数："+executor.getActiveCount());


        //步骤5：线程池默认只会销毁非核心线程，但是如果allowCoreThreadTimeOut(true)，那么核心线程也会被销毁，此时线程池也就自动关闭了
        //这里可以对比把这个值设置true和false时候，线程池最后是否会关闭

//        executor.allowCoreThreadTimeOut(false);//线程池最后不会关闭
        executor.allowCoreThreadTimeOut(true);//线程池最后会关闭

        //提交12个任务给线程池，让线程数量达到最大线程数
        for (int i = 0; i < 10; i++) {
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    try {
                        Thread.sleep(100);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            });
        }
        System.out.println(new Date()+":@@@"+"工作线程数："+executor.getActiveCount()+":核心线程数："+executor.getCorePoolSize());
        Thread.sleep(500);//等待0.5秒钟后查看活动工作线程数
        System.out.println(new Date()+":@@@"+"工作线程数："+executor.getActiveCount()+":核心线程数："+executor.getCorePoolSize());
        Thread.sleep(500);//等待0.5秒钟后查看活动工作线程数
        System.out.println(new Date()+":@@@"+"工作线程数："+executor.getActiveCount()+":核心线程数："+executor.getCorePoolSize());
        Thread.sleep(500);//等待0.5秒钟后查看活动工作线程数
        System.out.println(new Date()+":@@@"+"工作线程数："+executor.getActiveCount()+":核心线程数："+executor.getCorePoolSize());
        Thread.sleep(500);//等待0.5秒钟后查看活动工作线程数
        System.out.println(new Date()+":@@@"+"工作线程数："+executor.getActiveCount()+":核心线程数："+executor.getCorePoolSize());
        Thread.sleep(500);//等待0.5秒钟后查看活动工作线程数
    }
}
```



### 09.源码跟踪

通过跟踪源码来查看如何销毁线程，如何复用一个线程，以及解释前面的线程如何增长，一直到执行拒绝策略



### 10.Executors

Executors是一个java.util.concurrent包中的工具类，可以方便的为我们创建几种特定的线程池。

- FixedThreadPool:具有固定线程数量的线程池

线程池中的线程数量是固定的，当提交给线程池的任务数量大于池中的线程数量后，任务会在等待队列中排队，此处等待队列使用的是无界队列。当线程处理完一个任务后，会从等待队列中获取新的任务处理。

构造方法：

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
```



```java
public class ThreadPoolExecutorDemo {

    public static void main(String[] args) {
        //创建持有5个线程的线程池
        ExecutorService pool = Executors.newFixedThreadPool(5);

        for (int i = 0; i < 10; i++) {
            //向线程池中提交10个执行任务
            pool.execute(new Runnable() {
                @Override
                public void run() {
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.println("线程：" + Thread.currentThread().getName());
                }
            });
        }
        //关闭线程池
        pool.shutdown();
    }


}
```



- CachedThreadPool:线程数量可以动态伸缩的线程池

如果有新任务提交进来，只要没有空闲的线程处理，就会创建新的线程并处理。

构造方法如下：

核心线程数为0，最大线程数为Integer.MAX_VALUE，等待队列是个同步队列，不能存放任何元素，只是起到传递任务给工作线程的作用。

```java
    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```

使用样例：

```java
public class ThreadPoolExecutorDemo {

    public static void main(String[] args) {
        //创建线程数量课动态伸缩线程池
        ExecutorService pool = Executors.newCachedThreadPool();

        for (int i = 0; i < 30; i++) {
            //向线程池中提交30个执行任务
            pool.execute(new Runnable() {
                @Override
                public void run() {
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.println("线程：" + Thread.currentThread().getName());
                }
            });
        }
        //关闭线程池
        pool.shutdown();
    }


}

```



- SingleThreadPool:单个线程的线程池

这个线程池核心线程数和最大线程数都是1，队列是一个LinkedBlockingQueue的无界队列

构造函数：

```java
    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
```



使用样例：

```java

public class ThreadPoolExecutorDemo {

    public static void main(String[] args) {
        //创建单个线程的线程池
        ExecutorService pool = Executors.newSingleThreadExecutor();

        for (int i = 0; i < 10; i++) {
            //向线程池中提交10个执行任务
            pool.execute(new Runnable() {
                @Override
                public void run() {
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.println("线程：" + Thread.currentThread().getName());
                }
            });
        }
        //关闭线程池
        pool.shutdown();
    }


}
```



- ScheduledThreadPool:可以运行定时任务的线程池



构造函数：

```java
    public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }

    public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue());
    }
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), defaultHandler);
    }
```



执行样例：

```java
public class ThreadPoolExecutorDemo {

    public static void main(String[] args) {
        //创建一个具有定时任务功能的线程池，注意：这里的类型是ScheduledExecutorService
        ScheduledExecutorService pool = Executors.newScheduledThreadPool(2);


            //向线程池中提交1个执行任务
            System.out.println("提交任务时间："+new Date());
            pool.schedule(new Runnable() {
                @Override
                public void run() {
                    System.out.println("线程：" + Thread.currentThread().getName());
                    System.out.println("线程执行任务时间："+new Date());
                }
            },5,TimeUnit.SECONDS);//这里的第二个参数5和第三个参数放在一起意思是提交任务到线程										池后，要等待5秒再开始执行
    }
}

```



