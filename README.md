# Android.Multithreading-notes
Android之线程和线程池的笔记~

## 开发艺术探索之Android的线程和线程池
> 在Android系统，线程主要分为主线程和子线程，**主线程处理和界面相关的事情，而子线程一般用于执行耗时操作** AsyncTask底层是**线程池**IntentService/HandlerThread底层是**线程**  

* 在Android中，线程的形态有很多种：
> * AsyncTask封装了线程池和Handler   
> * HandlerThread是具有消息循环的线程，内部可以使用handler
> * IntentService是一种Service，内部采用HandlerThread来执行任务，当任务执行完毕后IntentService会自动退出 由于它是一种Service，所以不容易被系统杀死

* 操作系统中，线程是操作系统调度的最小单元，同时线程又是一种受限的系统资源，其创建和销毁都会有相应的开销。同时当系统存在大量线程时，系统会通过时间片轮转的方式调度每个线程，因此线程不可能做到绝对的并发，除非线程数量小于等于CPU的核心数

### 1. 主线程和子线程

* 主线程主要处理界面交互逻辑，由于用户随时会和界面交互，所以主线程在任何时候都需要有较高响应速度，则**不能执行耗时的任务** 

* android3.0开始，网络访问将会失败并抛出NetworkOnMainThreadException这个异常，这样做是为了避免主线程由于被耗时操作所阻塞从而现ANR现象

### 2. Android中的线程形态

**2.1 AsyncTask** 

> AsyncTask是一种**轻量级的异步任务类**, 他可以在线程池中执行**后台任务**, 然后把执行的进度和最终的结果传递给**主线程并在主线程更新UI**. 从实现上来说. AsyncTask封装了**Thread和复杂的Handler**, 通过AsyncTask可以更加方便地执行后台任务,但是AsyncTask并不适合进行特别耗时的后台任务，对于**特别耗时的任务来说, 建议使用线程池**  

* AsyncTask就是一个抽象的泛型类. 这三个泛型的意义 如果不需要传递具体的参数, 那么这三个泛型参数可以用Void来代替   
> 1.Params：参数的类型     
  2.Progress：后台任务的执行进度的类型         
  3.Result：后台任务的返回结果的类型     

* 四个方法：
> * onPreExecute() **在主线程执行**, 在异步任务执行之前, 此方法会被调用, 一般可以用于做一些准备工作   
> * doInBackground()：**在线程池中执行**, 此方法用于执行异步任务, 参数params表示异步任务的输入参数. 在此方法中可以通过**自带的**publishProgress()方法来更新任务的进度, publishProgress()方法会调用onProgressUpdate()方法. 另外此方法需要返回计算结果给onPostExecute()-->UI线程了    
> * onProgressUpdate()/onPostExecute()：在**主线程执行**,在异步任务执行之后, 此方法会被调用, 其中result参数是**后台任务**的返回值, 即doInBackground()的返回值    

* AsyncTask在使用过程中有一些条件限制：  
> * AsyncTask的对象必须在**主线程中创建**
> * execute方法必须在**UI线程调用**
> * 串并串并：1.6之前, 是串行的  1.6的时候是并行的  3.0开始为了并发错误又串行了 但是也可以并行
 
**2.2 AsyncTask的工作原理** 

* AsyncTask是由两个线程池(SerialExecutor：串行执行器+THREAD_POOL_EXECUTOR)和一个Handler组成的 其中串行执行器是负责用于任务的排列的  而THREAD_POOL_EXECUTOR才是真正执行任务的线程池 而Handler的本质就是把线程再切换到主线程就是一种线程的切换

* 具体过程：首先将**params**参数封装成一个**FutrueTask**对象 然后这个玩意再交给**串行执行器的execute** 方法去处理 --> 也就是加入到**mTasks队列中**  如果这个时候没有在执行的任务 串行执行器就**会调用next方法区执行一个新的AsyncTask任务**  这个完了会继续其他的 所以也可以看出来**默认是串行的**

**2.3 HandlerThread** 

* 首先HandlerThread继承了Thread也就是个Thread 但是是一种可以使用Handler的Thread 具体实现是在Thread的run方法中调用 Looper.prepare和Looper.loop来创建和开启消息队列循环的 这样就可以在HandlerThread中使用Handler了

* 和Thread的区别：Thread中只执行一个耗时任务 而HandlerThread中有个消息队列 外部来通知这个Handler来具体执行哪个 但是内部只具体执行一个任务 另外因为这里的run是looper.loop所以是个无线循环的过程 所以不用的时候要记得quit

* Android中的应用场景就是IntentService

**2.4 IntentService** 

* 首先IntentService是一种特殊的Service 继承了Service 但是问题在于优先级远远远高于普通线程 所以适合一些高优先级的任务   
* 其次内部封装了HandlerThread 和 Handler

> 1.首先 onCreate方法自动创建一个new HandlerThread   
  2.在这个HandlerThread里的Looper里创建一个Handler -->mserviceHandler 这样内部封装的HandlerThread 和 Handler就全了   
  3.这个mserviceHandler发送的消息最后就会在HandlerThread中执行  
  
* 使用每次使用中IntentService其实都可以做后台任务 每次启动就是.startCommand(.start)被调用一次和Service一样 来处理后台的Intent

### 3. Android线程池

**3.1 ThreadPoolExecutor** 

**3.2 线程池的分类** 

* FixedThreadPool

> 通过Executor#newFixedThreadPool()方法来创建. 它是一种线程**数量固定**的线程池, 当线程处于空闲状态时, 它们并不会被回收, 除非线程池关闭了. 当所有的线程都处于活动状态时, 新任务都会处于等待状态, 直到有线程空闲出来. 由于FixedThreadPool**只有核心线程并且这些核心线程不会被回收**, 这意味着它能够更加快速地响应外界的请求.

* CachedThreadPool
> 通过Executor#newCachedThreadPool()方法来创建. 它是一种线程**数量不定**的线程池, 它**只有非核心线程**, 并且其**最大值线程数为Integer.MAX_VALU**E. 这就可以认为这个最大线程数为任意大了. 当线程池中的线程都处于活动的时候, 线程池会创建新的线程来处理新任务, 否则就会利用空闲的线程来处理新任务. 线程池中的空闲线程都有超时机制, 这个超时时长为60S, 超过这个时间那么空闲线程就会被回收.

* 和FixedThreadPool不同的是, CachedThreadPool的任务队列**其实相当于一个空集合**, 这将导致任何任务都会立即被执行, 因为在这种场景下> SynchronousQueue是无法插入任务的. SynchronousQueue是一个非常特殊的队列, 在很多情况下可以把它简单理解为一个无法存储元素的队列. 在实际使用中很少使用.这类线程比较适合执行大量的耗时较少的任务

* ScheduledThreadPool
> 通过Executor#newScheduledThreadPool()方法来创建. 它的**核心线程数量是固定的, 而非核心线程数是没有限制的**, 并且当非核心线程闲置时会立刻被回收掉. 这类线程池用于执行定时任务和**具有固定周期的重复任务**

* SingleThreadExecutor
> 通过Executor#newSingleThreadPool()方法来创建. 这类线程池内部**只有一个核心线程**, 它确保所有的任务都在**同一个线程中按顺序执行**这类线程池意义在于统一所有的外界任务到一个线程中, 这使得在这些任务之间不需要处理线程同步的问题
