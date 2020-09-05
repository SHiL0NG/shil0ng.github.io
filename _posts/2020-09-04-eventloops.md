---
title: Event Loops
layout: post
---


An event loop is an infinite loop running on a single thread for handling events that occur during the lifetime of one or more connections. 

<!--more-->

Instead of assigning a thread from the thread pool for each connection, it is possible to use a single event loop working as a dispatcher to handle all incoming connections and then dispatch the tasks to corresponding worker event loops. These event loops run constantly on their own threads, thus eliminating the expenses of context switch.

### Organization

Although the Netty API seems to be very complicated, the classes/interfaces related to event loops are very well organized into two packages, `io.netty.util.concurrent` and `io.netty.channel` with a simple hierarchical dependency on `java.util.concurrent`.

The two Netty packages have a clear boundary of functionalities. The package `io.netty.util.concurrent` takes care of the threads and provides an abstraction of the thread model, while the `io.netty.channel` takes care of the `Channel` and `Selector`.


![Event Loop UML]({{ site.baseurl }}/images/uml_event_loop.png)


Starting from the root `Executor` to the end `NioEventLoop`, there is the inheritance path of abstract classes of executors, event loops on the left column, the interfaces in the middle, and executor groups and event loop groups on the right.

### Executor Framework

The Executor Framework provided in Java is important in understanding the Netty threading model. The executor framework decouples `task submission` from `task execution`, making it possible to implement and change our execution strategies. For example, we could create an `ImmediateExecutor` which runs the task immediately in the current thread.

```java

class ImmediateExecutor implements Executor {
     public void execute(Runnable r) {
         r.run();
     }
}
 
```

Or we could also create a new thread for each task, which is exactly the same as doing manually `new Thread(r).start()`


```java
class ThreadPerTaskExecutor implements Executor {
     public void execute(Runnable r) {
         new Thread(r).start();
     }
}
```


### EventExecutor and EventLoop


An `EventLoop` is basically an infinite loop running on a thread and takes care of the events submitted to it.


##### Thread Binding


The `SingleThreadEventExecutor` holds a reference to the thread which it is currently bound to, i.e. `thread`. If `inEventLoop` returns false, and it has no bound thread yet, it will start a new thread, update the `SingleThreadEventExecutor.thread` field as in `doStartThread`, and carry out the tasks in this newly bound thread.


The code here shows, where or not there's already an event loop thread, the task will always be added to the queue, the only difference is that, if the EventLoop is in state `ST_NOT_STARTED`, it will allocate a new thread and assign this thread to the EventLoop.


```java
    private void execute(Runnable task, boolean immediate) {
        boolean inEventLoop = inEventLoop();
        addTask(task);
        if (!inEventLoop) {
            startThread();
            ...
        }
        ...
    }
    
    private void startThread() {
        if (state == ST_NOT_STARTED) {
            if (STATE_UPDATER.compareAndSet(this, ST_NOT_STARTED, ST_STARTED)) {
                boolean success = false;
                try {
                    doStartThread();
                    success = true;
                } finally {
                    if (!success) {
                        STATE_UPDATER.compareAndSet(this, ST_STARTED, ST_NOT_STARTED);
                    }
                }
            }
        }
    }
    
    private void doStartThread() {
        assert thread == null;
        executor.execute(new Runnable() {
            @Override
            public void run() {
                thread = Thread.currentThread();
                try {
                    SingleThreadEventExecutor.this.run();
                    success = true;
                } catch (Throwable t) {
                    logger.warn("Unexpected exception from an event executor: ", t);
                } finally {
                    // cleanup, etc
                }
            }
        });
    }
    
    
```


In the `doStartThread` method of `SingleThreadEventExecutor`, we see `executor.execute(() -> {...}`. Thus depending on the executor, it decides how to bind an event loop to a thread. 



If the executor is a `ThreadPerTaskExecutor`, there will be a new thread created each time the `doStartThread` is called.



```java
// io.netty.util.concurrent.ThreadPerTaskExecutor
public final class ThreadPerTaskExecutor implements Executor {
    private final ThreadFactory threadFactory;

    public ThreadPerTaskExecutor(ThreadFactory threadFactory) {
        this.threadFactory = ObjectUtil.checkNotNull(threadFactory, "threadFactory");
    }

    @Override
    public void execute(Runnable command) {
        threadFactory.newThread(command).start();
    }
}
```

##### Task Handling

The `SingleThreadEventExecutor#run()` called in the `doStartThread` is an abstract method, so its subclasses can decide what they are going to do inside. For example, the `DefaultEventLoop` just keeps taking a task from the queue and runs it until `confirmShutDown.`



```java

    // io.netty.channel.DefaultEventLoop
    @Override
    protected void run() {
        for (;;) {
            Runnable task = takeTask();
            if (task != null) {
                task.run();
                updateLastExecutionTime();
            }

            if (confirmShutdown()) {
                break;
            }
        }
    }

```


And to deal with I/O operations, we use `NioEventLoop` which basically uses an `Selector` to `select`/`selectNow` the channels with available activities. These channels are registered to the event loop and the selector through `SingleThreadEventLoop#register(Channel).`



```java
    @Override
    public ChannelFuture register(Channel channel) {
        return register(new DefaultChannelPromise(channel, this));
    }

    @Override
    public ChannelFuture register(final ChannelPromise promise) {
        ObjectUtil.checkNotNull(promise, "promise");
        promise.channel().unsafe().register(this, promise);
        return promise;
    }
```


### EventExecutorGoup and EventLoopGroup

It is now clear how a single event loop works, but how the multithread environment is created? how many event loops should be created? And how is a `Channel` assigned to the event loops?

##### EventLoop Creation

The answer is that Netty manages the `EventLoop`-postfixed classes within the `EventLoopGroup`-postfixed classes.


In the constructor of `MultithreadEventExecutorGroup`, these is **no thread pool** but an array of `EventExecutor`s (which is an interface of `EventLoop`) whose size is equal to the number of threads. And unless explicitly specified, the default `Executor` will be the `ThreadPerTaskExecutor`, which means, for each event loop there will be a new thread created for  it.


```java
    protected MultithreadEventExecutorGroup(int nThreads, Executor executor,
                                            EventExecutorChooserFactory chooserFactory, Object... args) {

        ...
        if (executor == null) {
            executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
        }

        ... 
        children = new EventExecutor[nThreads];

        for (int i = 0; i < nThreads; i ++) {
            boolean success = false;
            try {
                children[i] = newChild(executor, args);
                success = true;
            }
            ...
        }
        ...
   }
```


The `newChild()` is again an abstract method whose implementation is left to its subclasses.



```java
protected abstract EventExecutor newChild(Executor executor, Object... args) throws Exception;
```


In the `NioEventLoopGroup`, the `newChild` returns a `NioEventLoop.`


```java
    @Override
    protected EventLoop newChild(Executor executor, Object... args) throws Exception {
        EventLoopTaskQueueFactory queueFactory = args.length == 4 ? (EventLoopTaskQueueFactory) args[3] : null;
        return new NioEventLoop(this, executor, (SelectorProvider) args[0],
            ((SelectStrategyFactory) args[1]).newSelectStrategy(), (RejectedExecutionHandler) args[2], queueFactory);
    }
```

And in `DefaultEventLoopGroup`, it returns a `DefaultEventLoop.`

```java
    @Override
    protected EventLoop newChild(Executor executor, Object... args) throws Exception {
        return new DefaultEventLoop(this, executor);
    }
```

##### EventLoop Allocation

An event loop is allocated to a channel in a round-robin fashion.


```java
    @Override
    public ChannelFuture register(Channel channel) {
        return next().register(channel);
    }
    
    // io.netty.util.concurrent.DefaultEventExecutorChooserFactory.PowerOfTwoEventExecutorChooser
    @Override
    public EventExecutor next() {
        return executors[idx.getAndIncrement() & executors.length - 1];
    }
    
```
So that multiple channels can share one event loop. 


### Application


<!-- ![Reactor]({{ site.baseurl }}/img/flowchart_reactor.png) -->

In a typical Netty network server, we need two EventLoopGroups, one as the dispatcher (`bossGroup`) and another as the worker (`workerGroup`) to implement the `Reactor` architecture. The advantage here is that the dispatcher takes care of the incoming requests and dispatches them to the workers. The workers handle the traffic of established connections. In this way the application-specific logic is separated from the dispatcher implementation. Usually it's enough to use only one thread for the dispatcher.


```java
    
public class DiscardServer {
    
    private int port;
    
    public DiscardServer(int port) {
        this.port = port;
    }
    
    public void run() throws Exception {
        EventLoopGroup bossGroup = new NioEventLoopGroup(); // (1)
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap(); // (2)
            b.group(bossGroup, workerGroup)
             .channel(NioServerSocketChannel.class) // (3)
             .childHandler(new ChannelInitializer<SocketChannel>() { // (4)
                 @Override
                 public void initChannel(SocketChannel ch) throws Exception {
                     ch.pipeline().addLast(new DiscardServerHandler());
                 }
             })
             .option(ChannelOption.SO_BACKLOG, 128)          // (5)
             .childOption(ChannelOption.SO_KEEPALIVE, true); // (6)
    
            ChannelFuture f = b.bind(port).sync(); // (7)
    
            f.channel().closeFuture().sync();
        } finally {
            workerGroup.shutdownGracefully();
            bossGroup.shutdownGracefully();
        }
    }
    
    public static void main(String[] args) throws Exception {
        int port = 8080;
        if (args.length > 0) {
            port = Integer.parseInt(args[0]);
        }

        new DiscardServer(port).run();
    }
}

```





### Summary

Here's a brief summary of the Netty EventLoop framework, which is elegantly designed, organized and implemented.  

1. You can bind several channels/sockets to one event loop.
2. An event loop is bound to exactly one thread that never changes.
3. Tasks can be submitted to the event loop for immediate or scheduled execution.
4. No thread pool is used.


### References

1. [Netty.docs: User guide for 4.x](https://netty.io/wiki/user-guide-for-4.x.html#wiki-h3-5)
2. [GitHub - netty/netty: Netty project - an event-driven asynchronous network application framework](https://github.com/netty/netty)
3. [Netty精粹之基于EventLoop机制的高效线程模型 - Float_Luuu的个人空间 - OSCHINA](https://my.oschina.net/andylucc/blog/618179)
4. [Netty源码探究2：线程模型 - 知乎](https://zhuanlan.zhihu.com/p/103704402)


