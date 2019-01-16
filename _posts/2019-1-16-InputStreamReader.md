---
layout:     post
title:      InputStreamReader
subtitle:   看看字节流是如何转化为字符流的
header-img: img/android3.png
author:     Allen Vork
catalog: true
tags:
    - java
---

InputStreamReader 用于将字节流转换成字符流，看之前以为是读取2个字节，然后第一个字节往左移8位，然后与第二个字节相加，得到一个2字节的 int，然后强制转换成 char。但看到构造函数中传 Decoder 就知道没这么简单。所以这里来分析下。

## InputStreamReader
```java
public class InputStreamReader extends Reader {

    private final StreamDecoder sd;

    public InputStreamReader(InputStream in, String charsetName)
        throws UnsupportedEncodingException
    {
        super(in);
        ...
        sd = StreamDecoder.forInputStreamReader(in, this, charsetName);
    }

    public int read() throws IOException {
        return sd.read();
    }

    public int read(char cbuf[], int offset, int length) throws IOException {
        return sd.read(cbuf, offset, length);
    }
}
```
可以看出内部都是采用 StreamDecoder 来读取数据的。我们来具体看下这个类。

## StreamDecoder
### 构造函数
```java
    public static StreamDecoder forInputStreamReader(InputStream var0, Object var1, String var2) throws UnsupportedEncodingException {
        String var3 = var2;
        ...
        // var0: InputStream
        // var1：InputStreamReader
        // 通过 String 获得 Charset 对象
        return new StreamDecoder(var0, var1, Charset.forName(var3));
        ...
    }

    StreamDecoder(InputStream var1, Object var2, Charset var3) {
        // 设置读入数据是，出现错误字符或者不兼容字符时，替换这些字符
        this(var1, var2, var3.newDecoder().onMalformedInput(CodingErrorAction.REPLACE).onUnmappableCharacter(CodingErrorAction.REPLACE));
    }

    StreamDecoder(InputStream var1, Object var2, CharsetDecoder var3) {
        super(var2);
        this.isOpen = true; // 字节流是否打开
        this.haveLeftoverChar = false; // 是否有剩余的 char 没写，true 的话，leftoverChar 字段就是还没写的 char
        this.cs = var3.charset(); // 字符集
        this.decoder = var3; // 字符解码器，将字节序列按照字符集转换成16位的 Unicode 序列
        if (this.ch == null) {
            this.in = var1;
            this.ch = null;
            this.bb = ByteBuffer.allocate(8192); // 8M 的字节缓冲区
        }

        this.bb.flip();
    }
```
可以看出它主要就是：
1. 根据传入的编码名称创建一个字符编码器
2. 创建 8M 大小的字节缓冲区 

### Read()
从上面的 InputStreamReader 可知，调用 InputStreamReader#read 调用的是 sd.read()。
```java
    public int read() throws IOException {
        return this.read0();
    }

    private int read0() throws IOException {
        Object var1 = this.lock;
        synchronized(this.lock) {
            if (this.haveLeftoverChar) { // 之前读取了字节到 leftoverChar 中，直接返回该字节
                this.haveLeftoverChar = false;
                return this.leftoverChar;
            } else { // 从流中读取
                // 虽然 read0() 只返回1个字节，但这里每次是读取2个
                // 因为一个字符就是2个字节
                char[] var2 = new char[2];
                // 返回读取的字符个数
                int var3 = this.read(var2, 0, 2);
                switch(var3) {
                case -1: // 流已经读到末尾了
                    return -1;
                case 0:
                default:
                    assert false : var3;

                    return -1;
                case 2: // 读取了2个字符，由于本方法只返回一个，则把另一个存到 leftoverChar 中，下次读取直接返回这个
                    this.leftoverChar = var2[1];
                    this.haveLeftoverChar = true;
                case 1:
                    return var2[0]; // 返回读取到的第一个字符
                }
            }
        }
    }
```
可以看出它主要是：
1. 创建一个长度为 2 的数组，然后读取数据进来
2. 返回数组中的第一个 byte，第二个 byte 用 leftoverChar 存起来。当再次调用 read() 方法时，直接返回 leftoverChar。

看看 1 的 this.read(var2, 0, 2) 是怎么读取的：    
```java
    public int read(char[] var1, int var2, int var3) throws IOException {
        int var4 = var2; // 0
        int var5 = var3; // 2
        Object var6 = this.lock;
        synchronized(this.lock) {
            this.ensureOpen();
            // 对参数有效性进行判断
            if (var4 >= 0 && var4 <= var1.length && var5 >= 0 && var4 + var5 <= var1.length && var4 + var5 >= 0) {
                if (var5 == 0) {
                    return 0;
                } else { // 走这里
                    byte var7 = 0;
                    if (this.haveLeftoverChar) { // leftoverChar 有值则先放到数组中
                        var1[var4] = this.leftoverChar;
                        ++var4;
                        --var5;
                        this.haveLeftoverChar = false;
                        var7 = 1;
                        if (var5 == 0 || !this.implReady()) {
                            return var7;
                        }
                    }

                    if (var5 == 1) { // 说明只需要读取一个字节
                        int var8 = this.read0(); // 重复上面步骤读取一个字节到数组中
                        if (var8 == -1) {
                            return var7 == 0 ? -1 : var7;
                        } else {
                            var1[var4] = (char)var8; // 将读取的一个字节存进去，就可以返回了
                            return var7 + 1;
                        }
                    } else {
                        return var7 + this.implRead(var1, var4, var4 + var5); // 继续读取
                    }
                }
            } else {
                throw new IndexOutOfBoundsException();
            }
        }
    }
```
它做的事情就是读取字节到传进来的数组中：
1. 如果 haveLeftoverChar，则将 leftoverChar 写进数组，如果数组写满了则返回
2. 如果还需要读取一个 byte 的话，调用 read0() 来读一个字节，直接返回
3. 如果需要读取多个字节的话，调用 implRead(var1, var4, var4 + var5) 方法

看下 implRead(char[], 0, 2)是如何读取2个字符到 char[] 中：
```java
    int implRead(char[] var1, int var2, int var3) throws IOException {
        assert var3 - var2 > 1;
        // 将这个字符数组包装到缓冲区
        CharBuffer var4 = CharBuffer.wrap(var1, var2, var3 - var2);
        if (var4.position() != 0) {
            var4 = var4.slice();
        }

        boolean var5 = false;

        while(true) {
            // 解码字节缓冲区 bb 到字符缓冲区 CharBuffer var4 中
            CoderResult var6 = this.decoder.decode(this.bb, var4, var5);
            if (var6.isUnderflow()) { // 已全部解码
                // hasRemaining 表示是否有空间剩余
                // var4.position() > 0 && !this.inReady() 表示字符缓冲区有内容并且输入流已经读完了
                if (var5 || !var4.hasRemaining() || var4.position() > 0 && !this.inReady()) {
                    break;
                }
                // 读取字节
                int var7 = this.readBytes();
                if (var7 < 0) { // 读到末尾了
                    var5 = true;
                    if (var4.position() == 0 && !this.bb.hasRemaining()) {
                        break;
                    }

                    this.decoder.reset(); // 清除解码器的所有状态
                }
            } else {
                if (var6.isOverflow()) { // 字符缓冲区已满，应用未满的字符缓冲区再次调用该方法
                    assert var4.position() > 0;
                    break;
                }

                var6.throwException();
            }
        }

        if (var5) {
            this.decoder.reset();
        }

        if (var4.position() == 0) {
            if (var5) {
                return -1;
            }

            assert false;
        }

        return var4.position(); // 返回读取到的个数
    }
```
1. 将要保存读出的数据的数组包装到 CharBuffer var4 字符缓冲区
2. 调用 readBytes() 方法来读取数据，然后解码字节缓冲区 bb 的数据到字符缓冲区 var4 中

可以看出就是读取数据并解码到字符缓冲区中。我们来看下这个 readBytes() 方法，猜测就是读数据到字节缓冲区 bb 中：
```java
    private int readBytes() throws IOException {
        //压缩缓冲区,当缓冲区中当前位置已到界限时，则时当前位置归0，界限位置到容量位置
        this.bb.compact();

        int var1;
        try {
            int var2;
            if (this.ch != null) {
                var1 = this.ch.read(this.bb);
                if (var1 < 0) {
                    var2 = var1;
                    return var2;
                }
            } else {
                var1 = this.bb.limit(); // 即 allocate 的时候分配的 8M
                var2 = this.bb.position(); // 首次为0

                assert var2 <= var1;

                int var3 = var2 <= var1 ? var1 - var2 : 0; // 8M

                assert var3 > 0;
                // 从底层输入流中读取，首次是最多读取 8M 的数据
                int var4 = this.in.read(this.bb.array(), this.bb.arrayOffset() + var2, var3);
                if (var4 < 0) {
                    int var5 = var4;
                    return var5;
                }

                if (var4 == 0) {
                    throw new IOException("Underlying input stream returned zero bytes");
                }

                assert var4 <= var3 : "n = " + var4 + ", rem = " + var3;

                this.bb.position(var2 + var4);
            }
        } finally {
            this.bb.flip();
        }

        var1 = this.bb.remaining();

        assert var1 != 0 : var1;

        return var1;
    }
```
可以看出它实质是从底层输入流中一次性读取 8M 的数据到字节缓冲数组中。

##总结
+ InputStreamReader 组合了一个 StreamDecoder，流的读取等操作都是通过它来完成的
+ 构造 InputStreamReader 时就会创建一个 StreamDecoder 对象，根据传入的编码名称创建一个字符编码器，然后创建一个大小为 8M 的字节缓冲数组 bb 用来存储从底层输入流读取的字节
+ 读取数据：
 	+ 将传进来的字符数组包装到 CharBuffer 中
	+ 从底层输入流中一次性读取字节到缓冲数组 bb 中，读取的大小为 bb 剩余空间的大小（首次为 8M）
	+ 将读取的字节数组通过 CharsetDecoder 解码到 CharBuffer 中


## 参考文献
+ [StreamDecoder和StreamEncoder](https://blog.csdn.net/u011863951/article/details/80004958)   
+ [源码分析: PipedInputStream和PipedOutputStream](https://blog.csdn.net/ai_bao_zi/article/details/81205286)