#java 的Phaser,CountDownLatch,CyclicBarrier,ForkJoinPool
@[java,并发] 

[TOC]

-------------



##1.Phaser
Phaser 类是java 7中新增的一个同步辅助类，Phaser把多个线程协作执行的任务分成多个阶段（phase），编程时需要明确各个阶段的任务，每个阶段都有任意个参与者，线程可以随时注册并参与到某个阶段中，当一个阶段中所有线程都成功执行完成之后，Phaser的onAdvance()方法会被调用，可以通过覆盖添加自定义的处理逻辑，然后Phaser类会自动进入到下个阶段，如此循环，直到Phaser不再包含任何参与者。

###1.1几个常用方法
- register/bulkRegister :添加一个或多个参与者
- arrive:某个参与者完成任务后调用
- arriveAndDeregister:任务完成后取消自己的注册
- arriveAndAwaitAdvance:自己等待其他参与者完成，进入阻塞，直到Phaser成功进入下一阶段
- awaitAdvance()、awaitAdvanceInterruptibly()，等待phaser进入下个阶段，参数为当前阶段的编号，后者可以设置超时和处理中断请求。
- onAdvance(int phase, int registeredParties)方法。此方法有2个作用：1、当每一个阶段执行完毕，此方法会被自动调用，因此，重载此方法写入的代码会在每个阶段执行完毕时执行，相当于CyclicBarrier的barrierAction。2、当此方法返回true时，意味着Phaser被终止，因此可以巧妙的设置此方法的返回值来终止所有线程。

###1.2例子程序
如果我们有这样的需求：第二阶段的所有任务需要等待第一阶段的所有任务执行完才能继续执行，下面是示例代码。

```java
public class PhaserTest {
    public static void main(String[] args) {
        Phaser phaser = new Phaser(5);
        for(int i=0;i<5;i++){
            Task task = new Task(phaser);
            Thread thread = new Thread(task,"PhaserTest"+i);
            thread.start();
        }
    }
    static class Task implements Runnable{
        private final Phaser phaser;
        public Task(Phaser phaser){
            this.phaser=phaser;
        }
        @Override
        public void run() {
            System.out.println(Thread.currentThread().getName()+"任务执行完成，等待其他任务执行....");
            phaser.arriveAndAwaitAdvance();
            System.out.println(Thread.currentThread().getName()+"任务继续执行");
        }
    }
}
```

输出结果：
```
PhaserTest0任务执行完成，等待其他任务执行....
PhaserTest3任务执行完成，等待其他任务执行....
PhaserTest1任务执行完成，等待其他任务执行....
PhaserTest2任务执行完成，等待其他任务执行....
PhaserTest4任务执行完成，等待其他任务执行....
PhaserTest2任务继续执行
PhaserTest3任务继续执行
PhaserTest4任务继续执行
PhaserTest1任务继续执行
PhaserTest0任务继续执行
```

-------
官网提供一些例子，我们来看一下：

一个类似于CountDownLatch的功能，等待其他任务执行完，才执行下一个阶段

```java
 void runTasks(List<Runnable> tasks) {
   final Phaser phaser = new Phaser(1); // "1" to register self
   // create and start threads
   for (final Runnable task : tasks) {
     phaser.register();
     new Thread() {
       public void run() {
         phaser.arriveAndAwaitAdvance(); // await all creation
         task.run();
       }
     }.start();
   }

   // allow threads to start and deregister self
   phaser.arriveAndDeregister();
 }
```

下面是一个方法使得一系列线程在给定的迭代次数内重复执行，通过重写onAdvance方法。
```
 void startTasks(List<Runnable> tasks, final int iterations) {
   final Phaser phaser = new Phaser() {
     protected boolean onAdvance(int phase, int registeredParties) {
       return phase >= iterations || registeredParties == 0;
     }
   };
   phaser.register();
   for (final Runnable task : tasks) {
     phaser.register();
     new Thread() {
       public void run() {
         do {
           task.run();
           phaser.arriveAndAwaitAdvance();
         } while (!phaser.isTerminated());
       }
     }.start();
   }
   phaser.arriveAndDeregister(); // deregister self, don't wait
 }
```

再看一个需求：开启5个线程，一共3个阶段，第一个阶段打印1-100，第二个阶段打印100-1，第三个阶段打印1-100，三个阶段完成后，就停止所有线程。


下面是实例代码：

```java
public class NumberPrint {
    public static void main(String[] args) {
        Phaser phaser=new Phaser(5){
            @Override
            protected boolean onAdvance(int phase, int registeredParties) {
                System.out.println("阶段： "+this.getPhase()+"参数阶段："+phase);
                return this.getPhase() == 2;
            }
        };
        System.out.println("程序开始执行");
        for(int i=0;i<5;i++){
            new Thread(new Task(phaser,1)).start();
        }
        while(!phaser.isTerminated()){
            Thread.yield();//主线程一直等待
        }
        System.out.println("任务执行结束");
    }
    static class Task implements Runnable{
        private final Phaser phaser;
        private int start;
        public Task(Phaser phaser,int start){
            this.phaser = phaser;
            this.start = start;

        }

        @Override
        public void run() {
            while(!phaser.isTerminated()){
                for(int i=1;i<=100;i++){
                    System.out.printf("线程：%s,阶段：%s ,数字： %d\n",Thread.currentThread().getName(),phaser.getPhase(), i);
                }
                start=101-start;
                phaser.arriveAndAwaitAdvance();//等待其他线程达到终点，再一起进入下一个阶段
            }
        }
    }
}
```
-------

##2.ForkJoinPool
java7提供了ForkJoinPool来支持将一个任务拆分成多个"小任务"并行计算，再把多个"小任务"的结果合并成总的计算结果，ForkJoinPool是ExecutorService的实现类，因此是一种特殊的线程池。

###2.1构造器
- ForkJoinPool(int parallelism) ： 创建一个包含parallelism个并行线程的ForkJoinPool
- ForkJoinPool():以Runtime.availableProcessors()方法的返回值作为parallelism参数来创建ForkJoinPool

创建了ForkJoinPool后，就可以通过`submit(ForkJoinTask task)`和`invoke(ForkJoinTask task)`方法来执行指定任务了。其中ForkJoinTask 代表一个可以并行、合并的任务，ForkJoinTask 是一个抽象类，它有两个抽象之类：RecursiveTask、RecursiveAction,其中RecursiveTask代表有返回值的任务，而RecursiveAction代表没有返回值的任务。

###2.2常用方法
- fork(): 分解出子任务执行
- join(): 当准备就绪时返回计算结果。
- invoke(): 相当于fork()、join()的组合。

###2.3例子程序
打印1-300，将任务分解。

```java
public class ForkJoinTest {
    public static void main(String[] args) throws InterruptedException {
        ForkJoinPool pool=new ForkJoinPool();
        pool.submit(new PrintTask(0,300));
        pool.awaitTermination(2, TimeUnit.SECONDS);
        pool.shutdown();
    }

}
class PrintTask extends RecursiveAction{
    private static final int THRESHOLD=50;
    private int start;
    private int end;
    public PrintTask(int start,int end){
        this.start=start;
        this.end=end;
    }

    @Override
    protected void compute() {
        if(end-start<THRESHOLD){
            for(int i=start;i<end;i++){
                System.out.println(Thread.currentThread().getName()+"的数值："+i);
            }
        }else {
            int middle=(start+end)/2;
            PrintTask left=new PrintTask(start,middle);
            PrintTask right=new PrintTask(middle,end);
            left.fork();
            right.fork();
        }
    }
}

```

###2.4使用过程
1. 创建一个ForkJoinPool
2. 实现一个任务类，继承了RecursiveTask或RecursiveAction,重写compute方法（一个有返回值，一个没有返回值），其中可以使用invokeAll(t1,t2)方法，或者使用fork()加上join()方法。
3. 在主线程中提交任务，可以通过execute(task)或者submit(task)。
4. 得到submit的返回结果Future,通过Future.get()方法获取结果,execute方法没有返回值。
5. 关闭线程池，pool.shutdown()

------
##3.CountDownLatch

一个同步辅助类，在完成一组正在正在其他线程中执行的操作之前，它允许线程一直等待。用给定的计数初始化CountDownLatch，由于调用了CountDown()方法，所以在当前计数到达0之前，await()方法会一直阻塞，之后会释放所有的等待线程，await()后的所有后续调用都将立即返回。

###3.1.重要方法
- countDown():递减锁存器的计数，如果计数到达0，会释放所有等待的线程。
- await():当前线程在锁存器达到计数为0之前一直等待，除非线程中断或者超出了指定时间。


###3.2.例子
考虑一个典型的场景：百米赛跑，选手全部就绪，等裁判发令，比赛开始，当所有选手哦到达终点时，比赛结束。

```
public class CountDownLatchTest {
    public static void main(String[] args) {
        final CountDownLatch begin = new CountDownLatch(1);//开始比赛的倒数锁
        final CountDownLatch end = new CountDownLatch(10);//结束比赛的倒数锁
        final ExecutorService executorService = Executors.newFixedThreadPool(10);//常见容量为10的线程池
        for(int i=0;i<10;i++){
            final int index=i+1;
            Runnable runner= new Runnable() {
                @Override
                public void run() {
                    try{
                        //比赛开始前等待
                        begin.await();
                        Thread.sleep((long)(Math.random()*1000));
                        System.out.printf("The %s player arrived\n",index);
                    }catch (InterruptedException e){
                        e.printStackTrace();
                    }finally {
                        //该选手达到终点，让结束的倒数锁计数减1
                        end.countDown();
                    }
                }
            };
            executorService.submit(runner);
        }
        System.out.println("Game Starts");
        begin.countDown();
        try {
            end.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("Game over");
        executorService.shutdown();
    }
}
```

###3.3使用说明
1.创建一个CountDownLatch
2.在线程中减少计数
3.在等待线程中调用await()方法等待其他线程执行完成

##4.CyclicBarrier

java提供的同步辅助类，它允许多个线程在某个集合点相互等待。

字面意思回环栅栏，通过它可以实现让一组线程等待至某个状态之后再全部同时执行。叫做回环是因为当所有等待线程都被释放以后，CyclicBarrier可以被重用。我们暂且把这个状态就叫做barrier，当调用await()方法之后，线程就处于barrier了。

当所有线程都到达await()方法时，就可以继续执行后面的代码块。
###4.1例子
假若有若干个线程都要进行写数据操作，并且只有所有线程都完成写数据操作之后，这些线程才能继续做后面的事情，此时就可以利用CyclicBarrier了：

```java
public class BarrierTst {
    public static void main(String[] args) {
        int N = 10;//等待线程数
        CyclicBarrier barriers = new CyclicBarrier(N);
        for(int i=0;i<N;i++){
            new Thread(new WriteTask(barriers)).start();
        }
    }
    static class WriteTask implements Runnable{
        private CyclicBarrier barrier;
        public WriteTask(CyclicBarrier barrier){
            this.barrier=barrier;
        }
        @Override
        public void run() {
            try{
	            //模拟写入过程
                System.out.printf("%s 执行第一阶段任务\n", Thread.currentThread().getName());
                Thread.sleep((long)(Math.random()*1000));
                barrier.await();
            }catch (Exception e){
                e.printStackTrace();
            }
            System.out.printf("%s 执行第二阶段任务 \n", Thread.currentThread().getName());
        }
    }
}
```




