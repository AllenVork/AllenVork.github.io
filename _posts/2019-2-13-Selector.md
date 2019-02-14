---
layout:     post
title:      NIO-Selector
subtitle:   解析 Selector 源码
header-img: img/android3.png
author:     Allen Vork
catalog: true
tags:
    - java
---

## Sample
```java
selector = Selector.open();  
  
ServerSocketChannel ssc = ServerSocketChannel.open();  
ssc.configureBlocking(false);  
ssc.socket().bind(new InetSocketAddress(port));  
  
ssc.register(selector, SelectionKey.OP_ACCEPT);  
  
while (true) {  
  
    // select()阻塞，等待有事件发生唤醒  
    int selected = selector.select();  
  
    if (selected > 0) {  
        Iterator<SelectionKey> selectedKeys = selector.selectedKeys().iterator();  
        while (selectedKeys.hasNext()) {  
            SelectionKey key = selectedKeys.next();  
            if ((key.readyOps() & SelectionKey.OP_ACCEPT) == SelectionKey.OP_ACCEPT) {  
                // 处理 accept 事件  
            } else if ((key.readyOps() & SelectionKey.OP_READ) == SelectionKey.OP_READ) {  
                // 处理 read 事件  
            } else if ((key.readyOps() & SelectionKey.OP_WRITE) == SelectionKey.OP_WRITE) {  
                // 处理 write 事件  
            }  
            selectedKeys.remove();  
        }  
    }  
}
```
下面开始详细讲解 selector 的核心代码。

## Selector.open()    
```java
public abstract class Selector implements Closeable {

    public static Selector open() throws IOException {
        return SelectorProvider.provider().openSelector();
    }
}
```
继续查看 SelectorProvider.provider()：
```java
    public static SelectorProvider provider() {
        synchronized (lock) {
            if (provider != null)
                return provider;
            return AccessController.doPrivileged(
                new PrivilegedAction<SelectorProvider>() {
                    public SelectorProvider run() {
                            if (loadProviderFromProperty())
                                return provider;
                            if (loadProviderAsService())
                                return provider;
                            provider = sun.nio.ch.DefaultSelectorProvider.create();
                            return provider;
                        }
                    });
        }
    }
```
它通过 DefaultSelectorProvider.create() 来创建一个 SelectorProvider：    
```java
public class DefaultSelectorProvider {

    public static SelectorProvider create() {
        return new WindowsSelectorProvider();
    }
}

public class WindowsSelectorProvider extends SelectorProviderImpl {
    public WindowsSelectorProvider() {
    }

    public AbstractSelector openSelector() throws IOException {
        return new WindowsSelectorImpl(this);
    }
}
```
可以看出 SelectorProvider.provider().openSelector() 是创建了一个 WindowsSelectorImpl：    
```java
final class WindowsSelectorImpl extends SelectorImpl {
    ...
    private PollArrayWrapper pollWrapper = new PollArrayWrapper(8);
    private final Pipe wakeupPipe = Pipe.open(); // 创建管道
    private final int wakeupSourceFd; // source 端的文件描述符
    private final int wakeupSinkFd;

    WindowsSelectorImpl(SelectorProvider var1) throws IOException {
        super(var1);
        // 获得上面创建好的管道的 source 端的文件描述符
        this.wakeupSourceFd = ((SelChImpl)this.wakeupPipe.source()).getFDVal();
        SinkChannelImpl var2 = (SinkChannelImpl)this.wakeupPipe.sink();
        var2.sc.socket().setTcpNoDelay(true);

        // 获得上面创建好的管道的 sink 端的文件描述符
        this.wakeupSinkFd = var2.getFDVal();

        // 将 source 文件描述符放到 pollWrapper 中
        this.pollWrapper.addWakeupSocket(this.wakeupSourceFd, 0);
    }
}
```
它首先会通过 Pipe.open 打开一个管道，然后获得该通道的 source 端和 sink 端的文件描述符，并将 source 端的文件描述符存到 pollWrapper 中。我们来看下 Pipe.open 做了什么：   

```java
public abstract class Pipe {

    public abstract Pipe.SourceChannel source();

    public abstract Pipe.SinkChannel sink();

    public static Pipe open() throws IOException {
        return SelectorProvider.provider().openPipe();
    }
}
```
可以看出它也是调用的 SelectorProvider.provider()，通过上面可知它最终调用的是 WindowsSelectorProvider 的 openPipe 方法：    
```java
public abstract class SelectorProviderImpl extends SelectorProvider {

    public Pipe openPipe() throws IOException {
        return new PipeImpl(this);
    }
}

class PipeImpl extends Pipe {
    private static final int NUM_SECRET_BYTES = 16;
    private static final Random RANDOM_NUMBER_GENERATOR = new SecureRandom();
    private SourceChannel source;
    private SinkChannel sink;

    PipeImpl(SelectorProvider var1) throws IOException {
        try {
            AccessController.doPrivileged(new PipeImpl.Initializer(var1));
        } catch (PrivilegedActionException var3) {
            throw (IOException)var3.getCause();
        }
    }
}
```
可以看出，整个过程最终调用的是 PipeImpl 的 Initializer 方法：
```java
    private class Initializer implements PrivilegedExceptionAction<Void> {
        private final SelectorProvider sp;
        private IOException ioe;

        private Initializer(SelectorProvider var2) {
            this.ioe = null;
            this.sp = var2;
        }

        public Void run() throws IOException {
            // 初始化管道，创建管道的 source 和 sink 端。下面会详细讲
            PipeImpl.Initializer.LoopbackConnector var1 = new PipeImpl.Initializer.LoopbackConnector();
            var1.run();
            ...
        }

        // 下面详细讲
        private class LoopbackConnector implements Runnable {
            private LoopbackConnector() {
            }

            // 创建2个本地的 SocketChannel，然后链接（通过写随机数来做两个 socket 的链接校验）。
            // 2个 SocketChannel 分别实现了管道的 source 和 sink 端。source 端由前面的 WindowsSelectorImpl 
            // 放到 PollWrapper 中。
            public void run() {
                ServerSocketChannel var1 = null;
                SocketChannel var2 = null;
                SocketChannel var3 = null;

                try {
                    ByteBuffer var4 = ByteBuffer.allocate(16);
                    ByteBuffer var5 = ByteBuffer.allocate(16);
                    InetAddress var6 = InetAddress.getByName("127.0.0.1");// 本机地址

                    assert var6.isLoopbackAddress();

                    InetSocketAddress var7 = null;

                    while(true) {
                        if (var1 == null || !var1.isOpen()) {
                            // 将ServerSocketChannel绑定本地环回地址，端口号为 0
                            var1 = ServerSocketChannel.open();
                            var1.socket().bind(new InetSocketAddress(var6, 0));

                            // 根据本机地址 var6 和 ServerSocketChannel 的端口号创建套接字地址 var7
                            var7 = new InetSocketAddress(var6, var1.socket().getLocalPort());
                        }

                        // 建立连接
                        var2 = SocketChannel.open(var7); 

                        // 往 ByteBuffer var4 中填充随机数
                        PipeImpl.RANDOM_NUMBER_GENERATOR.nextBytes(var4.array());

                        do {
                            var2.write(var4); // 将 var4 写到 SocketChannel 中
                        } while(var4.hasRemaining());

                        var4.rewind();
                        var3 = var1.accept(); // 获得 sink 端的 SocketChannel

                        do {
                            var3.read(var5); // 将 var2 传过来的数据读到 var5 中
                        } while(var5.hasRemaining());

                        var5.rewind();
                        if (var5.equals(var4)) { // 校验成功
                            // 通过2个 SocketChannel 创建管道的 source 和 sink 端
                            PipeImpl.this.source = new SourceChannelImpl(Initializer.this.sp, var2);
                            PipeImpl.this.sink = new SinkChannelImpl(Initializer.this.sp, var3);
                            break;
                        }

                        var3.close();
                        var2.close();
                    }
                } catch (IOException var18) {
                    try {
                        if (var2 != null) {
                            var2.close();
                        }

                        if (var3 != null) {
                            var3.close();
                        }
                    } catch (IOException var17) {
                        ;
                    }

                    Initializer.this.ioe = var18;
                } finally {
                    try {
                        if (var1 != null) {
                            var1.close();
                        }
                    } catch (IOException var16) {
                        ;
                    }

                }

            }
        }
    }
```
可以看出它是创建2个 SocketChannel，然后创建一个16个字节的 ByteBuffer，并用随机数来填充。然后 var2 来发送该数据，var3 来接收。如果发送的数据和接收的数据相同，则校验成功，即可通过这两个 SocketChannel 来创建管道的 source 和 sink 端。    
这样 Pipe.open() 就分析完了，再接着看将管道的 source 端放到 pollWrapper 是做了什么：    
```java
final class WindowsSelectorImpl extends SelectorImpl {
    // 创建 PollArrayWrapper
    private PollArrayWrapper pollWrapper = new PollArrayWrapper(8);

    WindowsSelectorImpl(SelectorProvider var1) throws IOException {
        super(var1);
        ...
        // 将管道的 source 端文件描述符放进去
        this.pollWrapper.addWakeupSocket(this.wakeupSourceFd, 0);
    }
}

class PollArrayWrapper {
    private AllocatedNativeObject pollArray;
    long pollArrayAddress;
    private static final short FD_OFFSET = 0;
    private static final short EVENT_OFFSET = 4;
    static short SIZE_POLLFD = 8;
    private int size;

    PollArrayWrapper(int var1) {
        int var2 = var1 * SIZE_POLLFD; // 8 * 8
        // 创建 AllocatedNativeObject
        this.pollArray = new AllocatedNativeObject(var2, true);
        this.pollArrayAddress = this.pollArray.address();
        this.size = var1;
    }

    ...

    // var1 为 source 文件描述符，var2 为 0
    void addWakeupSocket(int fdVal, int index) {

        // 将文件描述符保存到 native 中，下面会详细讲
        this.putDescriptor(index, fdVal);

        // 将 source 的 POLLIN 事件标志位感兴趣的，那么当 sink 端有数据写入时，
        // source 对应的文件描述符 wakeupSourceFd 就会处于就绪状态
        this.putEventOps(index, Net.POLLIN);
    }

    // 将 Descripter 放到 AllocatedNativeObject 中
    void putDescriptor(int index, int fdVal) {
        this.pollArray.putInt(SIZE_POLLFD * index + 0, fdVal);
    }

    void putEventOps(int index, int var2) {
        this.pollArray.putShort(SIZE_POLLFD * index + 4, (short)var2);
    }
}
```
可以看出它是将文件描述符放到 AllocatedNativeObject 中。
```java
class AllocatedNativeObject extends NativeObject {
    AllocatedNativeObject(int var1, boolean var2) {
        super(var1, var2);
    }
}

class NativeObject {
    protected static final Unsafe unsafe = Unsafe.getUnsafe();
    protected long allocationAddress;
    private final long address;
    private static ByteOrder byteOrder = null;
    private static int pageSize = -1;
    
    // size = 8; pageAligned = true
    protected NativeObject(int size, boolean pageAligned) {  
        if (!pageAligned) {  
            this.allocationAddress = unsafe.allocateMemory(size);  
            this.address = this.allocationAddress;  
        } else {  
            int ps = pageSize();  
            long a = unsafe.allocateMemory(size + ps);  
            this.allocationAddress = a;  
            this.address = a + ps - (a & (ps - 1));  
        }  
    }  
}
```
可以看出 pollArray 是通过 unsafe.allocateMemory((long)(size + ps)) 分配的一块系统内存。    

到这里，Selector.open() 就执行完成了。它主要是通过2个链接的 SocketChannel 建立了 Pipe，并将 pipe 的 wakeupSourceFd 放到 pollArray 中。 pollArray 是 Selector 的枢纽。最终返回 WindowsSelectorImpl 类型的 Selector。

## 2. serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT)
创建了 Selector 后，就要创建一个 ServerSocketChannel，然后将该 Channel 注册到 Selector 中：        
```java
ServerSocketChannel ssc = ServerSocketChannel.open();  
ssc.configureBlocking(false);  
ssc.socket().bind(new InetSocketAddress(port));  
  
ssc.register(selector, SelectionKey.OP_ACCEPT);  
```
### 2.1 ServerSocketChannel.open()
```java
public abstract class ServerSocketChannel extends AbstractSelectableChannel
    implements NetworkChannel {
    public static ServerSocketChannel open() throws IOException {
        // 由前面可知，它调用的是 WindowsSelectorProvider 的 openServerSocketChannel()
        return SelectorProvider.provider().openServerSocketChannel();
    }
}

// WindowsSelectorProvider 是 SelectorProviderImpl 的子类
public abstract class SelectorProviderImpl extends SelectorProvider {
    public ServerSocketChannel openServerSocketChannel() throws IOException {
        return new ServerSocketChannelImpl(this);
    }
}
```
可以看出，open 方法返回一个 ServerSocketChannelImpl 实例。

### 2.2 ssc.register(selector, SelectionKey.OP_ACCEPT)
由2.1看出，它调用的是 ServerSocketChannelImpl#register,具体的实现在父类：
```java
public abstract class SelectableChannel extends AbstractInterruptibleChannel implements Channel {
    public final SelectionKey register(Selector var1, int var2) throws ClosedChannelException {
        return this.register(var1, var2, (Object)null);
    }
}

// 最终调用到 SelectableChannel 的子类的 register 方法
public abstract class AbstractSelectableChannel extends SelectableChannel {

    // var1 为 WindowsSelectorImpl 对象；var2 为 SelectionKey.OP_ACCEPT；var3 为 null
    public final SelectionKey register(Selector var1, int var2, Object var3) throws ClosedChannelException {
        Object var4 = this.regLock;
        synchronized(this.regLock) {
            if (!this.isOpen()) {
                throw new ClosedChannelException();
            } else if ((var2 & ~this.validOps()) != 0) {
                throw new IllegalArgumentException();
            } else if (this.blocking) {
                throw new IllegalBlockingModeException();
            } else {
                SelectionKey var5 = this.findKey(var1); // 由于是首次注册，所以为空
                if (var5 != null) {
                    var5.interestOps(var2);
                    var5.attach(var3);
                }

                if (var5 == null) {
                    Object var6 = this.keyLock;
                    synchronized(this.keyLock) {
                        if (!this.isOpen()) {
                            throw new ClosedChannelException();
                        }
                        // 调用 WindowsSelectorImpl 的 register 方法
                        var5 = ((AbstractSelector)var1).register(this, var2, var3);
                        this.addKey(var5);
                    }
                }

                return var5;
            }
        }
    }
}
```
可以看出 Channel 调用 register 方法，实际是调用的 WindowSelectorImpl 的父类的 register 方法，将 Channel 和 option 传进去：   
```java
    // var2：SelectionKey.OP_ACCEPT；  var3： null
    protected final SelectionKey register(AbstractSelectableChannel var1, int var2, Object var3) {
        if (!(var1 instanceof SelChImpl)) {
            throw new IllegalSelectorException();
        } else {
            // 将 Selector 和 Channel 的实现类都封装到 SelectionKeyImpl 中
            SelectionKeyImpl var4 = new SelectionKeyImpl((SelChImpl)var1, this);
            var4.attach(var3);
            Set var5 = this.publicKeys;
            synchronized(this.publicKeys) {
                this.implRegister(var4); // 调用 implRegister(var4)，并传入 SelectionKeyImpl
            }

            var4.interestOps(var2);
            return var4;
        }
    }
``` 
register 方法会创建一个 SelectionKeyImpl，然后传给 WindowsSelectorImpl#implRegister 方法：    
```java
final class WindowsSelectorImpl extends SelectorImpl {
    private SelectionKeyImpl[] channelArray = new SelectionKeyImpl[8];
    private final WindowsSelectorImpl.FdMap fdMap = new WindowsSelectorImpl.FdMap();
    protected HashSet<SelectionKey> keys = new HashSet(); // 它是父类的

    protected void implRegister(SelectionKeyImpl var1) {
        Object var2 = this.closeLock;
        synchronized(this.closeLock) {
            if (this.pollWrapper == null) {
                throw new ClosedSelectorException();
            } else {
                this.growIfNeeded();
                this.channelArray[this.totalChannels] = var1;
                var1.setIndex(this.totalChannels);
                this.fdMap.put(var1);
                this.keys.add(var1);
                this.pollWrapper.addEntry(this.totalChannels, var1);
                ++this.totalChannels;
            }
        }
    }
}
```
将 SelectionKeyImpl 放到 channelArray、fdMap 和 pollWrapper 中。看下 pollWrapper:    
```java
class PollArrayWrapper {
    void addEntry(int index, SelectionKeyImpl ski) {
        this.putDescriptor(index, ski.channel.getFDVal());
    }
}
```
putDescriptor 上面讲过，就是将 socketChannel 的文件描述符放到 pollArray 中。    

### 总结
1. ServerSocketChannel.open() 会创建一个 ServerSocketChannelImpl 的 Channel 对象
2. ServerSocketChannel#register 方法会调用的 WindowSelectorImpl#register 方法，它会创建一个 SelectionKeyImpl 来存放 Channel（ServerSocketChannelImpl）和 Selector（WindowSelectorImpl）
3. 调用 implRegister(SelectionKeyImpl) 来将 SelectionKeyImpl 保存到各个列表中，并调用 pollWrapper.addEntry 来将 Channel 的文件描述符保存到 pollArray 中

## 3. selector.select()
这个 selector 就是 WindowSelectorImpl 的实例，调用的其实是他的父类的 select 方法：    
```java
public abstract class SelectorImpl extends AbstractSelector {

    public int select() throws IOException {
        return this.select(0L);
    }

    public int select(long timeout) throws IOException {
        return lockAndDoSelect((timeout == 0) ? -1 : timeout);  
    }  
  
    private int lockAndDoSelect(long timeout) throws IOException {  
        synchronized (this) {  
            ...
            synchronized (publicKeys) {  
                synchronized (publicSelectedKeys) {  
                    return doSelect(timeout);  
                }  
            }  
        }  
    }  
}
```
可以看出它又回到 WindowsSelectorImpl 中的 doSelect 方法：
```java
    private final WindowsSelectorImpl.SubSelector subSelector = new WindowsSelectorImpl.SubSelector();

    protected int doSelect(long timeout) throws IOException {
        if (this.channelArray == null) {
            throw new ClosedSelectorException();
        } else {
            this.timeout = timeout;
            this.processDeregisterQueue();
            if (this.interruptTriggered) {
                this.resetWakeupSocket();
                return 0;
            } else {
                this.adjustThreadsCount();
                this.finishLock.reset();
                this.startLock.startThreads();

                try {
                    this.begin();

                    try {
                        this.subSelector.poll();
                    } catch (IOException var7) {
                        this.finishLock.setException(var7);
                    }

                    if (this.threads.size() > 0) {
                        this.finishLock.waitForHelperThreads();
                    }
                } finally {
                    this.end();
                }

                this.finishLock.checkForException();
                this.processDeregisterQueue();
                int var3 = this.updateSelectedKeys();
                this.resetWakeupSocket();
                return var3;
            }
        }
    }

    private final class SubSelector {
        private final int pollArrayIndex;
        private final int[] readFds;
        private final int[] writeFds;
        private final int[] exceptFds;

        private SubSelector() {
            this.readFds = new int[1025];
            this.writeFds = new int[1025];
            this.exceptFds = new int[1025];
            this.pollArrayIndex = 0;
        }

        private int poll() throws IOException {
            return this.poll0(WindowsSelectorImpl.this.pollWrapper.pollArrayAddress, Math.min(WindowsSelectorImpl.this.totalChannels, 1024), this.readFds, this.writeFds, this.exceptFds, WindowsSelectorImpl.this.timeout);
        }

        ...

        private native int poll0(long var1, int var3, int[] var4, int[] var5, int[] var6, long var7);
    }
```
select 就是轮询 pollArray 中的 FD，看有没有事件发生，有的话就收集所有发生事件的 FD，然后退出阻塞。

## 4. selector.wakeup()
```java

WindowsSelectorImpl.java
----

public Selector wakeup() {
	synchronized (interruptLock) {
		if (!interruptTriggered) {
			setWakeupSocket();
			interruptTriggered = true;
		}
	}
	return this;
}
 
// Sets Windows wakeup socket to a signaled state.
private void setWakeupSocket() {
	setWakeupSocket0(wakeupSinkFd);
}
 
WindowsSelectorImpl.c
----
Java_sun_nio_ch_WindowsSelectorImpl_setWakeupSocket0(JNIEnv *env, jclass this,
                                                jint scoutFd)
{
    /* Write one byte into the pipe */
    send(scoutFd, (char*)&POLLIN, 1, 0);
}

```
可以看出 wakeup 调用了 native 的方法，用于往最开始建立的 pipe 的 sink 端写一个字节，那么 source 端的文件描述符就会处于就绪状态，然后 poll 方法就会返回，从而导致 select 方法返回。所以可以看出管道的意义就是让文件描述符处于就绪状态。





## 参考文献
+ [NIO selector原理浅析](https://blog.csdn.net/sj940611/article/details/72887074?utm_source=itdadao&utm_medium=referral)  
+ jdk 1.8 源码
+ [NIO源码阅读(3)-ServerSocketChannel](https://www.jianshu.com/p/5cadad72a2ec) 