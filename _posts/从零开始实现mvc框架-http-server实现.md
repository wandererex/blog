---
title: 从零开始实现mvc框架-http server实现
date: 2020-10-09 15:33:37
tags: http, server, nio
---
## 前言
[Boomvc](https://github.com/StevenKin/Boomvc)完成已经有一段时间了，但拖延到现在才开始记录。写这篇文章主要是回忆和复盘一下思路。如题所讲，Boomvc是一个mvc框架，但是它自带http server功能，也就是说不需要tomcat之类的server，可以在一个jar包里启动而不需要其他的依赖，这就需要自己去写http server的实现，这一篇我就梳理一下实现。
## server接口
首先定义一个server接口
```
public interface Server {

    void init(Boom boom);

    void start();

    void stop();

}

```
这个接口可以有多种实现，可以从nio socket开始写，也可以用netty这样的非常好用的network层的框架实现。在这里我实现了一个简易版的TinyServer。
```
public class TinyServer implements Server {
    private static final Logger logger = LoggerFactory.getLogger(TinyServer.class);

    private Boom boom;

    private Ioc ioc;

    private MvcDispatcher dispatcher;

    private Environment environment;

    private EventExecutorGroup boss;

    private EventExecutorGroup workers;

    private Thread cleanSession;


    @Override
    public void init(Boom boom) {
        ...
        ...
    }

    @Override
    public void start() {
        this.boss.start();
        this.workers.start();
        this.cleanSession.start();
    }

    @Override
    public void stop() {
        this.boss.stop();
        this.workers.stop();
    }

}
```
在这里要关注这个地方
```
private EventExecutorGroup boss;

private EventExecutorGroup workers;
```
这是我抽象出来的表示线程组，一个EventExecuteGroup持有多个EventExecute，boss接受连接请求，workers执行业务逻辑。看一下EventExecuteGroup的实现。

```
public class EventExecutorGroup implements Task {

    private int threadNum;

    private List<EventExecutor> executorList;

    private int index;

    private ThreadFactory threadName;

    private EventExecutorGroup childGroup;

    private MvcDispatcher dispatcher;


    public EventExecutorGroup(int threadNum, ThreadFactory threadName, EventExecutorGroup childGroup, MvcDispatcher dispatcher, SessionManager sessionManager) {
        this.threadNum = threadNum;
        this.threadName = threadName;
        this.childGroup = childGroup;
        this.dispatcher = dispatcher;
        this.executorList = new ArrayList<>(this.threadNum);
        IntStream.of(this.threadNum)
                .forEach(i-> {
                    try {
                        this.executorList.add(new EventExecutor(this.threadName, this.childGroup, this.dispatcher, sessionManager));
                    } catch (IOException e) {
                        throw new RuntimeException(e);
                    }
                });
        this.index = 0;
    }

    public void register(SelectableChannel channel, int ops) throws ClosedChannelException {
        int index1 = 0;
        synchronized (this){
            index1 = this.index%this.threadNum;
            this.index++;
        }
        this.executorList.get(index1).register(channel, ops);
    }

    public void register(SelectableChannel channel, int ops, Object att) throws ClosedChannelException {
        int index1 = 0;
        synchronized (this){
            index1 = this.index%this.threadNum;
            this.index++;
        }
        this.executorList.get(index1).register(channel, ops, att);
    }

    @Override
    public void start() {
        this.executorList.forEach(e->e.start());
    }

    @Override
    public void stop() {
        this.executorList.forEach(e->e.stop());
    }
}
```
## EventExecutor
EventExecutor就是一个io线程，它持有一个selector，selector是Java NIO核心组件中的一个，用于检查一个或多个Channel（通道）的状态是否处于可读、可写。如此可以实现单线程管理多个channels,也就是可以管理多个网络链接。io线程就不断轮询这个selector，获取多个selector key，根据这个key的状态，比如accept，read，write执行不同的逻辑。在这里EventExecutor是有多个的，也就是说selector有多个，boss EventExecutorGroup只有一个EventExecutor，它负责accept连接请求，并把接受的连接注册到workers EventExecutorGroup里，由worker线程处理read和write。
```
public class EventExecutor {
    private static final Logger logger = LoggerFactory.getLogger(EventExecutor.class);

    private ThreadFactory threadName;

    private EventExecutorGroup childGroup;

    private Selector selector;

    private Thread ioThread;

    private MvcDispatcher dispatcher;

    private Runnable task;

    private Semaphore semaphore = new Semaphore(1);


    public EventExecutor(ThreadFactory threadName, EventExecutorGroup childGroup, MvcDispatcher dispatcher, SessionManager sessionManager) throws IOException {
        this.threadName = threadName;
        this.childGroup = childGroup;
        this.dispatcher = dispatcher;
        this.selector = Selector.open();
        this.task = new EventLoop(selector, this.childGroup, this.dispatcher, sessionManager, semaphore);
        this.ioThread = threadName.newThread(this.task);
    }

    public void register(SelectableChannel channel, int ops) throws ClosedChannelException {
        channel.register(this.selector, ops);
    }

    public void register(SelectableChannel channel, int ops, Object att) throws ClosedChannelException {
        /* 将接收的连接注册到selector上
        // 发现无法直接注册，一直获取不到锁
        // 这是由于 io 线程正阻塞在 select() 方法上，直接注册会造成死锁
        // 如果这时直接调用 wakeup，有可能还没有注册成功又阻塞了，可以使用信号量从 select 返回后先阻塞，等注册完后在执行
        */
        try {
            this.semaphore.acquire();
            this.selector.wakeup();
            channel.register(this.selector, ops, att);
        }catch (InterruptedException e){
            logger.error("", e);
        }finally {
            this.semaphore.release();
        }
    }

    public void start(){
        ((Task)this.task).start();
        this.ioThread.start();
    }

    public void stop(){
        ((Task)this.task).stop();
    }

}
```
selector轮询是在EventLoop这里实现的。
## EventLoop
```
public class EventLoop implements Runnable, Task {
    private static final Logger logger = LoggerFactory.getLogger(EventLoop.class);

    private Selector selector;

    private EventExecutorGroup childGroup;

    private MvcDispatcher dispatcher;

    private FilterMapping filterMapping;

    private volatile boolean isStart = false;

    private Semaphore semaphore;

    private SessionManager sessionManager;

    public EventLoop(Selector selector, EventExecutorGroup childGroup, MvcDispatcher dispatcher, SessionManager sessionManager, Semaphore semaphore) {
        ...
    }

    @Override
    public void run() {
        while(this.isStart){
            try {
                int n = -1;
                try {
                    n = selector.select(1000);
                    semaphore.acquire();
                } catch (InterruptedException e) {
                    logger.error("", e);
                } finally {
                    semaphore.release();
                }
                if(n<=0)
                    continue;
            } catch (IOException e) {
                logger.error("", e);
                continue;
            }
            Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
            while(iterator.hasNext()){
                SelectionKey key = iterator.next();
                iterator.remove();
                if(!key.isValid())
                    continue;
                try {
                    if (key.isAcceptable()) {
                        accept(key);
                    }
                    if (key.isReadable()) {
                        read(key);
                    }
                    if (key.isWritable()) {
                        write(key);
                    }
                }catch (Exception e){
                    if(key!=null&&key.isValid()){
                        try {
                            key.channel().close();
                        } catch (IOException e1) {
                            e1.printStackTrace();
                        }
                    }
                    logger.error("", e);
                }
            }
        }
    }

    private void accept(SelectionKey key) throws IOException {
        ServerSocketChannel serverSocketChannel = (ServerSocketChannel) key.channel();
        SocketChannel socketChannel = serverSocketChannel.accept();
        socketChannel.configureBlocking(false);
        this.childGroup.register(socketChannel, SelectionKey.OP_READ, new HttpProtocolParser(socketChannel));
    }

    private void read(SelectionKey key) throws Exception{
        ...
    }

    private void write(SelectionKey key) throws IOException {
        ...
    }

    @Override
    public void start() {
        this.isStart = true;
    }

    @Override
    public void stop() {
        this.filterMapping.distory();
        this.isStart = false;
    }

    public Semaphore semaphore(){
        return this.semaphore;
    }
}
```
这就是一个经典的nio程序模式，要注意这里
```
this.childGroup.register(socketChannel, SelectionKey.OP_READ, new HttpProtocolParser(socketChannel));
```
这就把接受的连接注册到其他selector了。
这里我用了一个nio程序的多reactor模式，主线程中EventLoop对象通过 select监控连接建立事件，收到事件后通过 Acceptor接收，将新的连接分配给某个子EventLoop。
子线程中的EventLoop完成 read -> 业务处理 -> send 的完整流程。这种模式主线程和子线程的职责非常明确，主线程只负责接收新连接，子线程负责完成后续的业务处理，并且使用多个selector，read，业务处理，write不会影响accept，这对于大量并发连接可以提高accept的速度，不会因业务处理使大量连接堆积，这里其实参考了netty的思想。如下图

![netty](/images/netty.jpeg)
## 遇到的坑
在写EventExecutor的register方法是，发现如果直接在selector上调用register的话，可能会造成死锁。因为selector被多个线程访问，当其中一个线程调用selector.select()方法时发生阻塞，这个线程会一直持有selector的锁，这时另一个线程的register方法会被阻塞。如果这时直接调用 wakeup，有可能还没有注册成功又阻塞了，可以使用信号量从 select 返回后先阻塞，等注册完后在执行。具体实现如下
```
try {
    this.semaphore.acquire();
    this.selector.wakeup();
    channel.register(this.selector, ops, att);
}catch (InterruptedException e){
    logger.error("", e);
}finally {
    this.semaphore.release();
}
```
```
try {
    n = selector.select(1000);
    semaphore.acquire();
} catch (InterruptedException e) {
    logger.error("", e);
} finally {
    semaphore.release();
}
```
这里semaphore就起到一个阻塞EventLoop在被唤醒时继续执行的作用，当注册完成时才继续执行。

