okio 源码学习笔记

最近在学习okhttp的过程中，很多地方遇到了okio的功能，okio是square公司封装的IO框架，okhttp的网络IO都是通过okio来完成的。我们都知道Java原生的IO框架使用了装饰者模式，虽然体系结构很明确，但是有一个致命的缺点就是类太多，对数据进行IO操作时往往需要对IO流套多层。而okio则是对Java原生IO系统的一次封装，使得进行IO数据操作时更加方便灵活。
对于okio的介绍也有很多，这里推荐两篇个人认为挺好的文章，本文也是参考了其中的介绍，整理为学习笔记。

首先我们来看一下应用okio的一个简单的小例子，该方法的功能就是完成文件的拷贝

```
   public static void copyFile(String fromName, String toName) throws IOException{
        File from = new File(fromName);
        File to = new File(toName);

        BufferedSource source = Okio.buffer(Okio.source(from));
        BufferedSink sink = Okio.buffer(Okio.sink(to));

        String copyContent = "";
        while (!source.exhausted()){
            copyContent = source.readString(Charset.defaultCharset());
            sink.write(copyContent.getBytes());
        }

    }
```
首先我们需要明确在okio中，source代表输入流，sink代表输出流，我们可以看到构建输入输出流很简单，Okio这个类提供一系列的工具方法供我们使用，而在使用流的时候没有那么多类，包括BufferedXXX, DataXXX, FileXXX，以及XXXStream, Reader， Writer等区分， 在okio中source和sink代表基本的输入输出流的接口，对其封装的只有BufferedXXX一种类型，它提供了几乎所有 的我们常用的输入输出操作，不过在BufferedSource和BufferedSink中都包含了一个Buffer对象，这个类是我们需要重点学习的对象，okio都是通过这个类与InputeStream和OutputStream通信完成的IO操作，下面我们开始进入学习okio的代码结构。

## 1. 整体结构
在学习代码之前，我们首先看一下okio的整体结构图，该图是源自于推荐的第一篇文章中的图片，这里特别说明。

从图中我们可以看到结构很清晰，上面是输出流，下面是输入流，每个都只有BufferedXXX一个装饰器对象，并且两个装饰器对象都是依赖Buffered对象。这里需要说明的是Source, Sink, BufferedSource, BufferedSink都是接口，他们定义了需要的基本功能，而他们都有对应的实现类，RealBufferedSource和RealBufferedSink, 但是他们两个的功能实现几乎都是通过封装的Buffer对象来完成的，Buffer类同时实现了BufferedSource, BufferedSink接口，再次强调这个我们重点学习的对象。
在推荐的第一篇文章中是以Sink为例讲述的，这里就记录一下我学习Source的过程，避免重复，其实原理是完全一致的，了解其中之一，另外一个也就明白了。我们首先来看Source接口的定义
```
/**
 * Supplies a stream of bytes. Use this interface to read data from wherever
 * it's located: from the network, storage, or a buffer in memory. Sources may
 * be layered to transform supplied data, such as to decompress, decrypt, or
 * remove protocol framing.
 **/
public interface Source extends Closeable {
  /**
   * Removes at least 1, and up to {@code byteCount} bytes from this and appends
   * them to {@code sink}. Returns the number of bytes read, or -1 if this
   * source is exhausted.
   */
  long read(Buffer sink, long byteCount) throws IOException;

  /** Returns the timeout for this source. */
  Timeout timeout();

  /**
   * Closes this source and releases the resources held by this source. It is an
   * error to read a closed source. It is safe to close a source more than once.
   */
  @Override void close() throws IOException;
}
```
这里保留的Source源码中的第一段注释，其他部分可以自行查看，注释说明的很清楚，它作为一个输入流，可以为内存输入（提供）一个字节流，这个流可以是封装的任何的底层文件结构，比如网络通信的中的socket, 硬盘中保存的普通文件，或者内存中的缓存数据等等。代码中我们需要注意的时候read方法，它规定了应该将该流的内容读到内存中的Buffer对象中去。第二个方法为timeout返回一个timeout对象，关于IO的超时对象，我们留到本文的最后一部分介绍。下面再来看BufferedSource的代码：
```
/**
 * A source that keeps a buffer internally so that callers can do small reads without a performance
 * penalty. It also allows clients to read ahead, buffering as much as necessary before consuming
 * input.
 */
public interface BufferedSource extends Source {
  Buffer buffer();

  boolean exhausted() throws IOException;

  void require(long byteCount) throws IOException;

  boolean request(long byteCount) throws IOException;

  byte readXXX() throws IOException;

  void read(XXXX) throws IOException;

  void skip(long byteCount) throws IOException;

  int select(Options options) throws IOException;

  long indexOf(ByteString bytes, long fromIndex) throws IOException;
 
  boolean rangeEquals(long offset, ByteString bytes) throws IOException;
  /** Returns an input stream that reads from this source. */
  InputStream inputStream();
}
```
这里对代码做了最大的简化，首先来看它定义了一个buffer()方法用来获取该流对象中封装的Buffer对象，其次就是exhaust开始的三个方法，exhaust提供流是否结束的检查，另外两个则是用于检查流中是否还有若干字节，二者不同之处在于require是在不足时抛异常，而request则以false作为返回值。接下来就是最为重要的一系列的read方法，他们可以简单地分为两类，一个是返回值的形式，主要是返回一些类型的数据，有点类似于DataInputStream中定义的方法，还有一类是将流的内容读到传入的数组参数中，通常是字节数组。最后定义了skip方法用于跳过字节流中的一部分内容，select方法则用于匹配一定的选项规则，具体可以看源码中所举的例子，这里不再介绍，另外还有一系列的indexOf，rangeEqual等方法，通过名称也可以明白其中的意思，inputStream则可以将Source对象转为InputStream对象输出。

以上是BufferedSource中定义的接口方法，而该接口在okio中该接口有两个实现类，一个是RealBufferedSource，另外一个就是Buffer，这里我们先简单了解一下前者：

```
final class RealBufferedSource implements BufferedSource {
  public final Buffer buffer = new Buffer();
  public final Source source;
  boolean closed;

  RealBufferedSource(Source source) {
    if (source == null) throw new NullPointerException("source == null");
    this.source = source;
  }
...
```
从代码中可以看到它封装了一个Buffer对象以及一个Source对象， 其中Buffer对象提供缓存以及字节流与其他数据类型的转换的功能， 而source对象则是封装的底层输入字节流，包括文件，Socket等，这个在前面已经做了介绍，可见如此设计甚好。我们就可以预感到RealBufferedSource将多数在BufferedSource中定义的接口功能都代理给了Buffer。下面我们来看部分代码：

```
@Override public long read(Buffer sink, long byteCount) throws IOException {
    if (sink == null) throw new IllegalArgumentException("sink == null");
    if (byteCount < 0) throw new IllegalArgumentException("byteCount < 0: " + byteCount);
    if (closed) throw new IllegalStateException("closed");

    if (buffer.size == 0) {
      long read = source.read(buffer, Segment.SIZE);
      if (read == -1) return -1;
    }

    long toRead = Math.min(byteCount, buffer.size);
    return buffer.read(sink, toRead);
  }

@Override public byte readByte() throws IOException {
    require(1);
    return buffer.readByte();
  }
```
这里只是截取了其中具有代表性的两个方法，第一个是Source中定义的方法，我们可以看到该方法中，首先将source对象中的字节流内容读取到buffer对象中，然后通过Buffer对象提供的读功能完成将内容读取到指定缓存中的任务。
第二个方法是读取一个字节，同样也是首先将source对象中的内容读取到Buffer对象中，（这里的读取到Buffer对象的工作是在request方法中完成的， require方法是调用的request方法）然后将该方法的功能全权代理给Buffer对象来完成。有兴趣的同学可以查看其他方法的实现，起基本原理都是一致的，代理给Buffer对象。

以上内容我们就整体介绍了一下okio中的输入流的类的结构以及其中的关系，同样地，在输出流中，其结构与之类似，有兴趣的同学可以自行查看源码。在使用okio的功能时，不可或缺的是Okio这个工具类，该工具类中有两类静态方法， 一类是buffer()方法，它可以将任何source或sink对象转换为BufferedSource或BufferedSink对象，第二类方法就是sink()或source()方法，这一类方法可以将任何流对象，包括InputStream或OutputStream，以及流的底层结构文件，字符串，Socket等转换为source或者sink对象，这样以来就可以很方便地使用okio完成输入输出操作，而且也可以很容易与原生的JavaIO完成转换。


okio中将所有的功能方法都定义在了BufferedXXX接口中，而两个接口分别有两个实现类，即RealBufferedXXX 和Buffer， Buffer对象实现了上述两个接口，而RealBufferedXXX中的功能都是代理给了Buffer对象，因此在下一部分我们需要重点学习Buffer对象。

## 2 Buffer对象
关于Buffer对象，其注释明确说明了，它是内存中字节数据的集合。在前面我们知道在okio中几乎所有的IO操作都是经由Buffer中转实现的，而Buffer中的内部数据结构就是一组字节数组，通过在内存中的中转缓存实现一系列的IO操作。所以在了解Buffer如何实现IO操作功能之前，我们先来了解一下Buffer的底层数据结构，这就涉及Segment的概念。

首先我们需要明确，在Buffer的底层是有Segment的链表实现的，而且是一个环形的双向链表，即普通的双向链表收尾连接起来。这里放一张本文开头中推荐的第二篇文章中的一张示意图以示意，这里特此说明。

所以这里我们首先来学习一下Segment的结构，其代码为：

```
/**
 * A segment of a buffer.
 */
final class Segment {
	...

  final byte[] data;

  /** The next byte of application data byte to read in this segment. */
  int pos;

  /** The first byte of available data ready to be written to. */
  int limit;

  /** True if other segments or byte strings use the same byte array. */
  boolean shared;

  /** True if this segment owns the byte array and can append to it, extending {@code limit}. */
  boolean owner;

  /** Next segment in a linked or circularly-linked list. */
  Segment next;

  /** Previous segment in a circularly-linked list. */
  Segment prev;

  ...
```
我们首先来看一下Segment的结构，这里由于不擅长画图，又没有在网上找到合适的图片，所以暂且以文字描述，如果是熟悉Java的nio框架的同学可以借鉴ByteBuffer的结构，Segment与之类似。data是底层的字节数组，pos表示当前读到的位置，lim表示当前写到的位置，读操作的结束点为lim, 写操作的结束位置为data的末尾，也就是SIZE-1，这两个属性与ByteBuffer的结构完全一致。而在okio中为了避免过多的数据拷贝，它使用了共享的方式，我们在后面还会介绍到。这里share属性表示该Segment对象的数据即data数组是与其他Segment对象共享的， 而owner属性表示该Segment对data数组是否拥有所有权。举个例子来说，当我们需要拷贝一份数据，刚好处于一个Segment中，为了避免拷贝，我们可以新建一个Segment对象，但是新的Segment对象与之前的Segment对象共享data数组，此时两个Segment对象的share属性都置为true， 而原有的Segment的ower属性为true，新建的Segment对象ower属性则为false， 此时原Segment对于数组中lim到数组结尾的空间具有写权限，而新建的Segment则没有。对于最后两个属性pre, next则应该很容易明白他们是用于构建链表的。下面我们来看它的构造器， 注意其中的ower和shareed属性：

```
Segment() {
    this.data = new byte[SIZE];
    this.owner = true;
    this.shared = false;
  }

  Segment(Segment shareFrom) {
    this(shareFrom.data, shareFrom.pos, shareFrom.limit);
    shareFrom.shared = true;
  }

  Segment(byte[] data, int pos, int limit) {
    this.data = data;
    this.pos = pos;
    this.limit = limit;
    this.owner = false;
    this.shared = true;
  }
```
第一个构造器用于新建一个Segment对象，并且新建了data数组，而后两个构造器则是用于新建Segment对象，并且与别的Segment共享data数组，需要注意理解其中的shareed和ower属性的设置，后面我们还会提到如何利用这两个属性。下面我们来看Segment的方法：
```
  public Segment pop() {
    Segment result = next != this ? next : null;
    prev.next = next;
    next.prev = prev;
    next = null;
    prev = null;
    return result;
  }

  public Segment push(Segment segment) {
    segment.prev = this;
    segment.next = next;
    next.prev = segment;
    next = segment;
    return segment;
  }
```
这里是两个最为常用但是也很简单的方法，就是简单的pop和push方法，pop方法用于将自己移除链表，并返回下一个节点的引用，即下一个Segment对象，如果这是最后一个节点，则返回null; push方法则是将一个新的Segment对象添加到当前节点的后面。

```
 /**
   * Splits this head of a circularly-linked list into two segments. The first
   * segment contains the data in {@code [pos..pos+byteCount)}. The second
   * segment contains the data in {@code [pos+byteCount..limit)}. This can be
   * useful when moving partial segments from one buffer to another.
   *
   * <p>Returns the new head of the circularly-linked list.
   */
  public Segment split(int byteCount) {
    if (byteCount <= 0 || byteCount > limit - pos) throw new IllegalArgumentException();
    Segment prefix;

    // We have two competing performance goals:
    //  - Avoid copying data. We accomplish this by sharing segments.
    //  - Avoid short shared segments. These are bad for performance because they are readonly and
    //    may lead to long chains of short segments.
    // To balance these goals we only share segments when the copy will be large.
    if (byteCount >= SHARE_MINIMUM) {
      prefix = new Segment(this);
    } else {
      prefix = SegmentPool.take();
      System.arraycopy(data, pos, prefix.data, 0, byteCount);
    }

    prefix.limit = prefix.pos + byteCount;
    pos += byteCount;
    prev.push(prefix);
    return prefix;
  }
```
这是一个将Segment分割的方法，这里保留了源码中的注释，可以仔细阅读一下，明确说明了是 为了避免过多的数据拷贝，使用数组共享的方式，不过为了避免将Segment分割的过小造成链表太长，这里设置了共享的最小的大小。该方法常用与在数据拷贝之前，首先将需要拷贝的字节数分割为一个新的Segment对象，便于拷贝。
这里的SegmentPool顾名思义就是一个Segment的共享池，避免创建对象和回收引起的内存抖动，该对象提供两个静态方法就是take()和recycle()很容易猜到什么含义，这里不再介绍。
这里需要注意的是，创建新的Segment对象，不管是不是共享，新建的Segment对象都会添加当前Segment之前的位置，并且返回新建的Segment对象，而原Segment对象的pos会向后移动byteCount字节，lim不变，并且具有写权限，而新建的Segment如果是与原Segment共享data数组，则新Segment对象不能对lim之后的空间进行写操作，这一点在前面也做过介绍。既然有分割那么也应该有合并的方法，那么下面来看compact()方法：
```
  /**
   * Call this when the tail and its predecessor may both be less than half
   * full. This will copy data so that segments can be recycled.
   */
  public void compact() {
    if (prev == this) throw new IllegalStateException();
    if (!prev.owner) return; // Cannot compact: prev isn't writable.
    int byteCount = limit - pos;
    int availableByteCount = SIZE - prev.limit + (prev.shared ? 0 : prev.pos);
    if (byteCount > availableByteCount) return; // Cannot compact: not enough writable space.
    writeTo(prev, byteCount);
    pop();
    SegmentPool.recycle(this);
  }

  /** Moves {@code byteCount} bytes from this segment to {@code sink}. */
  public void writeTo(Segment sink, int byteCount) {
    if (!sink.owner) throw new IllegalArgumentException();
    if (sink.limit + byteCount > SIZE) {
      // We can't fit byteCount bytes at the sink's current position. Shift sink first.
      if (sink.shared) throw new IllegalArgumentException();
      if (sink.limit + byteCount - sink.pos > SIZE) throw new IllegalArgumentException();
      System.arraycopy(sink.data, sink.pos, sink.data, 0, sink.limit - sink.pos);
      sink.limit -= sink.pos;
      sink.pos = 0;
    }

    System.arraycopy(data, pos, sink.data, sink.limit, byteCount);
    sink.limit += byteCount;
    pos += byteCount;
  }
```
compact()方法用于将当前Segment节点与之前的节点合并，将当前节点的数据写入到之前的节点中，代码逻辑很清晰，如pre.owner为false则不能写入，然后判断之前的节点是否具有足够的空间，这里注意availableByteCount的计算方法，如果之前的节点不是被共享的，那么pos到lim之间的数据可以向前移动，移动到0～lim-pos的位置，所以availableByteCount就多出了pos大小的空间，而如果之前的节点是被共享的，那么数据是不能被移动的。最后如果可用空间足够大就执行数据移动，并删除当前Segment对象，并且回收到SegmentPool中。
下面来看数据移动的方法， 数据移动的方法逻辑也比较清晰，与compact()类似，如果数据空间不足，而数据可移动（即不是共享状态）移动后充足则移动数据，否则会抛出异常。然后执行数据拷贝并设置sink的lim属性以及原Segment的pos属性即可完成任务。

至此我们就是介绍完了Segment的方法，它的代码很少，功能也很简单，包括pop, push, slit, compact和writeTo五个方法，而Buffer的底层结构就是关于Segment的环形双向链表，那么Buffer的IO操作都是通过Segment的操作来完成的。下面我们来简单学习一下Buffer的部分功能。

```
/**
 * A collection of bytes in memory.
 */
public final class Buffer implements BufferedSource, BufferedSink, Cloneable {
  
	...
  Segment head;
  long size;

  public Buffer() {
  }
...
```
可以看到Buffer的属性域很简单，就是一个关于Segment的双向链表，head属性指向该链表的头部，size表示该Buffer的大小，下面我们来看Buffer的部分功能方法：

```

  @Override public void write(Buffer source, long byteCount) {
    // Move bytes from the head of the source buffer to the tail of this buffer
    // while balancing two conflicting goals: don't waste CPU and don't waste
    // memory.
    //
    //
    // Don't waste CPU (ie. don't copy data around).
    //
    // Copying large amounts of data is expensive. Instead, we prefer to
    // reassign entire segments from one buffer to the other.
    //
    //
    // Don't waste memory.
    //
    // As an invariant, adjacent pairs of segments in a buffer should be at
    // least 50% full, except for the head segment and the tail segment.
    //
    // The head segment cannot maintain the invariant because the application is
    // consuming bytes from this segment, decreasing its level.
    //
    // The tail segment cannot maintain the invariant because the application is
    // producing bytes, which may require new nearly-empty tail segments to be
    // appended.
    //
    //
    // Moving segments between buffers
    //
    // When writing one buffer to another, we prefer to reassign entire segments
    // over copying bytes into their most compact form. Suppose we have a buffer
    // with these segment levels [91%, 61%]. If we append a buffer with a
    // single [72%] segment, that yields [91%, 61%, 72%]. No bytes are copied.
    //
    // Or suppose we have a buffer with these segment levels: [100%, 2%], and we
    // want to append it to a buffer with these segment levels [99%, 3%]. This
    // operation will yield the following segments: [100%, 2%, 99%, 3%]. That
    // is, we do not spend time copying bytes around to achieve more efficient
    // memory use like [100%, 100%, 4%].
    //
    // When combining buffers, we will compact adjacent buffers when their
    // combined level doesn't exceed 100%. For example, when we start with
    // [100%, 40%] and append [30%, 80%], the result is [100%, 70%, 80%].
    //
    //
    // Splitting segments
    //
    // Occasionally we write only part of a source buffer to a sink buffer. For
    // example, given a sink [51%, 91%], we may want to write the first 30% of
    // a source [92%, 82%] to it. To simplify, we first transform the source to
    // an equivalent buffer [30%, 62%, 82%] and then move the head segment,
    // yielding sink [51%, 91%, 30%] and source [62%, 82%].

    if (source == null) throw new IllegalArgumentException("source == null");
    if (source == this) throw new IllegalArgumentException("source == this");
    checkOffsetAndCount(source.size, 0, byteCount);

    while (byteCount > 0) {
      // Is a prefix of the source's head segment all that we need to move?
      if (byteCount < (source.head.limit - source.head.pos)) {
        Segment tail = head != null ? head.prev : null;
        if (tail != null && tail.owner
            && (byteCount + tail.limit - (tail.shared ? 0 : tail.pos) <= Segment.SIZE)) {
          // Our existing segments are sufficient. Move bytes from source's head to our tail.
          source.head.writeTo(tail, (int) byteCount);
          source.size -= byteCount;
          size += byteCount;
          return;
        } else {
          // We're going to need another segment. Split the source's head
          // segment in two, then move the first of those two to this buffer.
          source.head = source.head.split((int) byteCount);
        }
      }

      // Remove the source's head segment and append it to our tail.
      Segment segmentToMove = source.head;
      long movedByteCount = segmentToMove.limit - segmentToMove.pos;
      source.head = segmentToMove.pop();
      if (head == null) {
        head = segmentToMove;
        head.next = head.prev = head;
      } else {
        Segment tail = head.prev;
        tail = tail.push(segmentToMove);
        tail.compact();
      }
      source.size -= movedByteCount;
      size += movedByteCount;
      byteCount -= movedByteCount;
    }
  }

  @Override public long read(Buffer sink, long byteCount) {
    if (sink == null) throw new IllegalArgumentException("sink == null");
    if (byteCount < 0) throw new IllegalArgumentException("byteCount < 0: " + byteCount);
    if (size == 0) return -1L;
    if (byteCount > size) byteCount = size;
    sink.write(this, byteCount);
    return byteCount;
  }
```
首先我们来看两个比较重要的方法，即读和写方法，其中最主要的逻辑就在write()方法中，这里保留了所有的注释，个人认为还是要去读一下注释的，这里不再对注释解释，可以自行查看，通过理解其中的例子，就可以理解write()方法的执行逻辑了。这里只是解释其中的代码逻辑。首先是在一个循环中执行写操作，直到byteCount为零。然后看需要写的字节数量是否超出了source的第一个Segment的范围，如果没有超出，则看sink的最后一个Segment是否可以写入并且具有充足的空间写入，如果可以就直接拷贝过去，这里调用了Segment.writeTo()方法，前面也做过介绍，其实就是System.arrayCopy()将数组拷贝过去。否则的话就将source的head节点拆分，保证source的head节点是需要全部写入到sink中去的。后面的逻辑就比较简单了，就是将source的head节点挂到sink的尾部，然后执行一次compact()操作，并更新各个属性就完成了一个Segment的写入，这里不会更新Segment节点时不需要数组拷贝，所以节约了CPU，而在compact()时执行少量的数据拷贝，提高内存的利用率。如此循环，完成数据的写入操作。

下面来看read()方法就比较简单了，它其实就是调用了sink.write()方法，其实也就是借助上面的write方法，实现数据读取，反方向写入就是数据读取，将数据读取到sink中。
下面我们再来看Buffer中三个比较典型的方法作为实例：

```
  /** Copy {@code byteCount} bytes from this, starting at {@code offset}, to {@code out}. */
  public Buffer copyTo(Buffer out, long offset, long byteCount) {
    if (out == null) throw new IllegalArgumentException("out == null");
    checkOffsetAndCount(size, offset, byteCount);
    if (byteCount == 0) return this;

    out.size += byteCount;

    // Skip segments that we aren't copying from.
    Segment s = head;
    for (; offset >= (s.limit - s.pos); s = s.next) {
      offset -= (s.limit - s.pos);
    }

    // Copy one segment at a time.
    for (; byteCount > 0; s = s.next) {
      Segment copy = new Segment(s);
      copy.pos += offset;
      copy.limit = Math.min(copy.pos + (int) byteCount, copy.limit);
      if (out.head == null) {
        out.head = copy.next = copy.prev = copy;
      } else {
        out.head.prev.push(copy);
      }
      byteCount -= copy.limit - copy.pos;
      offset = 0;
    }

    return this;
  }
```
第一个是copyTo()方法，这个方法有两个重载形式，一个是CopyTo outputStream, 一个是copyTo Buffer, 这里只是介绍第二个作为例子，其实二者形式很相近。拷贝的步骤包括，跳过一定的字节数，然后逐个Segment进行拷贝，这里Segment地方data数组会进行共享。在创建完新的Segment对象以后添加到双向列表中，就可以完成了数据拷贝任务。

```
  /** Write {@code byteCount} bytes from this to {@code out}. */
  public Buffer writeTo(OutputStream out, long byteCount) throws IOException {
    if (out == null) throw new IllegalArgumentException("out == null");
    checkOffsetAndCount(size, 0, byteCount);

    Segment s = head;
    while (byteCount > 0) {
      int toCopy = (int) Math.min(byteCount, s.limit - s.pos);
      out.write(s.data, s.pos, toCopy);

      s.pos += toCopy;
      size -= toCopy;
      byteCount -= toCopy;

      if (s.pos == s.limit) {
        Segment toRecycle = s;
        head = s = toRecycle.pop();
        SegmentPool.recycle(toRecycle);
      }
    }

    return this;
  }
```
要介绍的第二个方法是写入方法，基本的逻辑就是逐个Segment进行数据写入，首先计算需要写入的字节数量，然后写入到输出流中，最后更新每个段的pos, lim 属性，并且回收可以回收的Segment对象。

```
  private void readFrom(InputStream in, long byteCount, boolean forever) throws IOException {
    if (in == null) throw new IllegalArgumentException("in == null");
    while (byteCount > 0 || forever) {
      Segment tail = writableSegment(1);
      int maxToCopy = (int) Math.min(byteCount, Segment.SIZE - tail.limit);
      int bytesRead = in.read(tail.data, tail.limit, maxToCopy);
      if (bytesRead == -1) {
        if (forever) return;
        throw new EOFException();
      }
      tail.limit += bytesRead;
      size += bytesRead;
      byteCount -= bytesRead;
    }
  }
```
介绍的第三个方法是读取方法，首先是获取一个可以写入的Segment对象，然后计算读取的字节数量，然后执行数据读取，将数据读取到Segment的data中，最后是更新lim属性，完成了读取任务。
Buffer对象同时实现了BufferedSource和BufferedSink两个接口，在这两个接口定义的方法中，Buffer都是通过类似的逻辑通过操作双向链表中的Segment对象完成数据的IO任务，有兴趣的同学可以自行查看Buffer的源码。

至此就是完成了对Buffer的分析，当我们了解了Buffer的功能实现，也就明白BufferedSource和BufferedSink的实现方式，这里虽然在使用过程中不会涉及到Buffer的直接操作，更不会涉及Segment的操作，不过这里阅读其中的源码可以借鉴的东西还是很多，而且这里只要理解了Segment的操作，就可以较为容易的看懂Buffer的代码。

除此之外，这里简单提一下okio中另外一个重要的类， ByteString，从这个类的名字中可以看出它是一个字节串，此外它的实例是不可变的对象，这一点类似于String对象，它底层有一个data[]数组对象，维护数据，延迟初始化uft-8的字符串，两份数据不会干扰，用空间换取了效率，同时由于它是不可变的对象，在多线程中就具备了安全和效率两方面的优势，此外它提供了一系列的api可以完成它与流之间的数据交换，与Buffer之间的数据交换，以及与string等类型之间的转换，有兴趣的同学可以阅读其源码，较为简单可以通读其代码，这里不再介绍。

## 3. Okio的超时机制 	
okio中使用timeout对象控制I/O的操作的超时。该超时机制使用了时间段（Timeout)和绝对时间点(Deadline)两种计算超时的方式，可以选择使用其中一种。下面我们看其源码，首先看它的属性：

```
  private boolean hasDeadline;
  private long deadlineNanoTime;
  private long timeoutNanos;

```
可以看到Timeout中，使用deadlineNanoTime记录过期的绝对时间点，使用timeoutNanos记录过期的一段时间，在Timeout类中的前半部分都是针对这三个属性的设置和返回方法，可以理解过简单的getter和setter方法，只不过setter方法返回Timeout对象本身，代码比较简单，读者可以自行查看。
下面是针对超时的处理，第一种是超出deadline时，抛异常：
```
  public void throwIfReached() throws IOException {
    if (Thread.interrupted()) {
      throw new InterruptedIOException("thread interrupted");
    }

    if (hasDeadline && deadlineNanoTime - System.nanoTime() <= 0) {
      throw new InterruptedIOException("deadline reached");
    }
  }
```
代码逻辑很简单，到达截止日期时就跑出异常，该方法用在一次I/O操作之后调用，通过调用一次该方法检查是否超时。+