本篇文章为okhttp源码学习笔记系列的第二篇文章，本篇文章的主要内容为okhttp中的连接与流的管理，因此需要重点介绍的就连接和流两个概念。客户端通过HTTP协议与服务器进行通信，首先需要建立连接，okhttp并没有使用URLConnection， 而是对socket直接进行封装，在socket之上建立了connection的概念，代表这物理连接。同时，一对请求与响应对应着输出和输入流， okhttp中同样使用流的逻辑概念，建立在connection的物理连接之上进行数据通信，（只不过不知道为什么okhttp中流的类名为HttpCodec）。在HTTP1.1以及之前的协议中，一个连接只能同时支持单个流，而SPDY和HTTP2.0协议则可以同时支持多个流，本文先考虑HTTP1.1协议的情况。connection负责与远程服务器建立连接， HttpCodec负责请求的写入与响应的读出，但是除此之外还存在在一个连接上建立新的流，取消一个请求对应的流，释放完成任务的流等操作，为了使HttpCodec不至于过于复杂，okhttp中引入了StreamAllocation负责管理一个连接上的流，同时在connection中也通过一个StreamAllocation的引用的列表来管理一个连接的流，从而使得连接与流之间解耦。另外，熟悉HTTP协议的同学肯定知道建立连接是昂贵的，因此在请求任务完成以后（即流结束以后）不会立即关闭连接，使得连接可以复用，okhttp中通过ConnecitonPool来完成连接的管理和复用。

因此本文会首先介绍StreamAllocation类，并说明连接与流的概念，以及管理机制，其次会依次介绍Connection和HttpCodec1(HttpCodec的一个实现类，代表HTTP1.1协议)，然后再通过分析ConnectionPool来分析连接的管理机制，最后会继续学习第一篇文章中最后介绍的整个网络请求流程（其实就是一系列的拦截器的调用），本文会介绍其中与连接和流有关的三个拦截器，分别时RetryInterceptor, ConnectionInterceptor和CallServerInterceptor。当然期间还有涉及Rout, RoutSelector等一些为建立连接或建立流有关的辅助类。

## 1. 连接与流的桥梁

首先我们明白HTTP通信执行网络请求需要在连接之上建立一个新的流执行该任务， 我们这里将StreamAllocation称之为连接与流的桥梁， 它负责为一次请求寻找连接并建立流，从而完成远程通信，这里说寻找是包括连接可能会从连接池中复用也可能是新建连接。在介绍StreamAllocation类之前，我们先了解一下连接和流的概念，对于二者， StreamAllocation类之前的注释已经给出了解释，这里我们首先看一下它的注释：
```
/**
 * This class coordinates the relationship between three entities:
 *
 * <ul>
 *     <li><strong>Connections:</strong> physical socket connections to remote servers. These are
 *         potentially slow to establish so it is necessary to be able to cancel a connection
 *         currently being connected.
 *     <li><strong>Streams:</strong> logical HTTP request/response pairs that are layered on
 *         connections. Each connection has its own allocation limit, which defines how many
 *         concurrent streams that connection can carry. HTTP/1.x connections can carry 1 stream
 *         at a time, HTTP/2 typically carry multiple.
 *     <li><strong>Calls:</strong> a logical sequence of streams, typically an initial request and
 *         its follow up requests. We prefer to keep all streams of a single call on the same
 *         connection for better behavior and locality.
 * </ul>
 ...
 **/
```
注释说明的很清楚，Connection时建立在Socket之上的物理通信信道，而Stream则是代表逻辑的HTTP请求/响应对， 至于Call，是对一次请求任务或是说请求过程的封装，在第一篇文章中我们已经做出介绍，而一个Call可能会涉及多个流（如请求重定向，auth认证等情况）， 而Okhttp使用同一个连接完成这一系列的流上的请求任务，这一点的实现我们将在介绍RetryInterceptor的部分中说明。

下面我们来思考StreamAllocation所要解决的问题，简单来讲就是在合适的连接之上建立一个新的流，这个问题划分为两步就是寻找连接和新建流。那么，StreamAllocation的数据结构中应该包含Stream(okhttp中将接口的名字定义为HttpCodec)， Connction, 其次为了寻找合适的连接，应该包含一个URL地址， 连接池ConnectionPool, 而方法则应该包括newStream()方法, 而findConnection则是为之服务的方法，其次在完成请求任务之后应该有finish方法，还有终止的取消方法， 以及释放资源的方法。下面我们从StreamAllocation中寻找对应的属性和方法， 其代码为：

```
public final class StreamAllocation {
  public final Address address;
  private Route route;
  private final ConnectionPool connectionPool;
  private final Object callStackTrace;

  // State guarded by connectionPool.
  private final RouteSelector routeSelector;
  private int refusedStreamCount;
  private RealConnection connection;
  private boolean released;
  private boolean canceled;
  private HttpCodec codec;

  public StreamAllocation(ConnectionPool connectionPool, Address address, Object callStackTrace) {
    this.connectionPool = connectionPool;
    this.address = address;
    this.routeSelector = new RouteSelector(address, routeDatabase());
    this.callStackTrace = callStackTrace;
  }
...
```
这里我们看并没有使用URL代表一个地址，而是使用Address对象， 如果查看其代码可以看到它封装了一个URL, 其中还包括dns, 代理等信息，可以更好地满足HTTP协议中规定的细节。然后有一个Rout对象用于建立连接，以及RoutSelector, 用于选择合适的路线去建立连接， 然后就是两个标志位以及一个拒绝流数量的统计信息，剩下的则是Connection和HttpCodec对象，这里我们先忽略调用栈callStackTrace， 不考虑。那么下面我们再来看新建流的方法：

```
public HttpCodec newStream(OkHttpClient client, boolean doExtensiveHealthChecks) {
    int connectTimeout = client.connectTimeoutMillis();
    int readTimeout = client.readTimeoutMillis();
    int writeTimeout = client.writeTimeoutMillis();
    boolean connectionRetryEnabled = client.retryOnConnectionFailure();

    try {
      RealConnection resultConnection = findHealthyConnection(connectTimeout, readTimeout,
          writeTimeout, connectionRetryEnabled, doExtensiveHealthChecks);

      HttpCodec resultCodec;
      if (resultConnection.http2Connection != null) {
        resultCodec = new Http2Codec(client, this, resultConnection.http2Connection);
      } else {
        resultConnection.socket().setSoTimeout(readTimeout);
        resultConnection.source.timeout().timeout(readTimeout, MILLISECONDS);
        resultConnection.sink.timeout().timeout(writeTimeout, MILLISECONDS);
        resultCodec = new Http1Codec(
            client, this, resultConnection.source, resultConnection.sink);
      }

      synchronized (connectionPool) {
        codec = resultCodec;
        return resultCodec;
      }
    } catch (IOException e) {
      throw new RouteException(e);
    }
  }
 ```
 其流程很明确，我们先不考虑HTTP2.0的情况， 流程就是找到合适的良好连接，然后实例化HttpCodec对象，并设置对应的属性，返回该流对象即可，该流对象依赖连接的输入流和输出流，从而可以在流之上进行请求的写入和响应的读出。下面来开findConnection的代码：

 ```
  /**
   * Finds a connection and returns it if it is healthy. If it is unhealthy the process is repeated
   * until a healthy connection is found.
   */
  private RealConnection findHealthyConnection(int connectTimeout, int readTimeout,
      int writeTimeout, boolean connectionRetryEnabled, boolean doExtensiveHealthChecks)
      throws IOException {
    while (true) {
      RealConnection candidate = findConnection(connectTimeout, readTimeout, writeTimeout,
          connectionRetryEnabled);

      // If this is a brand new connection, we can skip the extensive health checks.
      synchronized (connectionPool) {
        //successCount记录该连接上执行流任务的次数，为零说明是新建立的连接
        if (candidate.successCount == 0) {
          return candidate;
        }
      }

      // Do a (potentially slow) check to confirm that the pooled connection is still good. If it
      // isn't, take it out of the pool and start again.
      if (!candidate.isHealthy(doExtensiveHealthChecks)) {  // 判断连接是否可用
        noNewStreams(); //在该方法中会设置该Allocation对应的Connection对象的noNewStream标志位，标识这在该连接不再使用，在回收的线程中会将其回收
        continue;
      }

      return candidate;
    }
  }

  /**
   * Returns a connection to host a new stream. This prefers the existing connection if it exists,
   * then the pool, finally building a new connection.
   */
  private RealConnection findConnection(int connectTimeout, int readTimeout, int writeTimeout,
      boolean connectionRetryEnabled) throws IOException {
    Route selectedRoute;
    synchronized (connectionPool) {

    //一系列条件判断
    ...

      RealConnection allocatedConnection = this.connection;
      if (allocatedConnection != null && !allocatedConnection.noNewStreams) {//noNewStream是一个标识为，标识该连接不可用
        return allocatedConnection;
      }

      // Attempt to get a connection from the pool.
      //可以在OkhttpClient中查看到该方法， 其实就是调用connnctionPool.get(address, streamAllocation);

      RealConnection pooledConnection = Internal.instance.get(connectionPool, address, this);  
      if (pooledConnection != null) {
        this.connection = pooledConnection;
        return pooledConnection;
      }

      selectedRoute = route;
    }

    if (selectedRoute == null) {
      selectedRoute = routeSelector.next(); //选择下一个路线Rout
      synchronized (connectionPool) {
        route = selectedRoute;
        refusedStreamCount = 0;
      }
    }
    RealConnection newConnection = new RealConnection(selectedRoute);

    synchronized (connectionPool) {
      acquire(newConnection);   //1. 将该StreamAllocation对象，即this 添加到Connection对象的StreamAllocation引用列表中，标识在建立新的流使用到了该连接
      Internal.instance.put(connectionPool, newConnection);  //2. 将新建的连接加入到连接池， 与get方法类型，也是在OkHttpClient调用的pool.put()方法
      this.connection = newConnection;
      if (canceled) throw new IOException("Canceled");
    }

    newConnection.connect(connectTimeout, readTimeout, writeTimeout, address.connectionSpecs(),
        connectionRetryEnabled);   //3. 连接方法，将在介绍Connection的部分介绍
    routeDatabase().connected(newConnection.route());  //4.新建的连接一定可用，所以将该连接移除黑名单

    return newConnection;
  }

  ...
  /**
   * Use this allocation to hold {@code connection}. Each call to this must be paired with a call to
   * {@link #release} on the same connection.
   */
  public void acquire(RealConnection connection) {
    assert (Thread.holdsLock(connectionPool));
    connection.allocations.add(new StreamAllocationReference(this, callStackTrace));
  }

  ```

 这两个方法完成了寻找合适的连接的功能，流程也很明确，分成三步， 一是从连接池中查找，其过程在本文的第三部分会有介绍，有则返回执行第三步没有则继续，二是选择一个合适的线路，建立新的连接，并在建立新连接之后执行一系列的步骤，在代码中已经用注释的方式分四步标出。第三步则是对选择的连接执行检查，这里如果经历了第二步即建立新的连接则会跳过检查，其实检查的过程就是查看该连接的socket是否被关闭以及是否可以正常使用，有兴趣的可以自行查看代码。
 在建立新的连接以后需要处理一些事情，代码中的中文注释分了四步将其标出，其中需要说明第一步，acquire方法，connection中维护这一张在一个连接上的流的链表，该链表保存的是StreamAllocation的引用，只有新建的连接才会在该链表中添加该StreamAllocation对象，那么应该是已经存在的Connection或者从ConnectionPool中取出的Connection链表中已经持有对该StreamAllocation的引用，只有StreamAllocation对象调用release()方法时才释放该引用， Connction的该链表为空时说明该连接已经可以回收了，这部分在连接管理部分会有说明。
 此外还需要说明的两点，一是关于Internal, 这是一个抽象类，该类只有一个实现类，时HttpClient的一个匿名内部类，该类的一系列方法都是一个功能，就是将okhttp3中一些包访问权限的方法对外提供一个public的访问方法，至于为什么这么实现，目前还不太清楚，估计是在Okhttp的框架中方便使用包权限的方法， 如ConnectionPool的put和get方法，在Okhttp的框架代码中通过Internal代理访问， 而在外部使用时无法访问到这些方法，至于还有没有其他考虑有清楚的同学欢迎赐教。
 需要说明的第二点是RouteDatabase是一个黑名单，记录着不可用的路线，避免在同一个坑里栽倒两次，connect()方法就是将该路线移除黑名单，这里为了不影响StreamAllocation分析的连贯性，本文将RoutSelector和RouteDatabase等与路线选择相关的代码放到附录部分，这里我们暂且先了解它的功能而不关注它的实现。

至此就分析完了建立新流的过程，那么下面就是释放资源的逻辑，流的结束包括成功结束，终止取消，失败等情况，我们分别看他们的代码：

```
public void streamFinished(boolean noNewStreams, HttpCodec codec) {
    synchronized (connectionPool) {
      if (codec == null || codec != this.codec) {
        throw new IllegalStateException("expected " + this.codec + " but was " + codec);
      }
      if (!noNewStreams) {
        connection.successCount++;
      }
    }
    deallocate(noNewStreams, false, true);
  }

  ...
  public void release() {
    deallocate(false, true, false);
  }

  /** Forbid new streams from being created on the connection that hosts this allocation. */
  public void noNewStreams() {
    deallocate(true, false, false);
  }
  ...
   /**
   * Releases resources held by this allocation. If sufficient resources are allocated, the
   * connection will be detached or closed.
   */
  private void deallocate(boolean noNewStreams, boolean released, boolean streamFinished) {
    RealConnection connectionToClose = null;
    synchronized (connectionPool) {
      if (streamFinished) {
        this.codec = null;
      }
      if (released) {
        this.released = true;
      }
      if (connection != null) {
        if (noNewStreams) {
          connection.noNewStreams = true;
        }
        if (this.codec == null && (this.released || connection.noNewStreams)) {
          release(connection);
          if (connection.allocations.isEmpty()) {
            connection.idleAtNanos = System.nanoTime();
            if (Internal.instance.connectionBecameIdle(connectionPool, connection)) {
              connectionToClose = connection;
            }
          }
          connection = null;
        }
      }
    }
    if (connectionToClose != null) {
      Util.closeQuietly(connectionToClose.socket());
    }
  }
```
这里前三个方法都会调用deallocate方法来释放资源，该方法有三个参数，分别标识对应的连接是否可以使用，是否将该StreamAllocation对象从Connection的引用流链表里移除，流是否完成任务。该方法主要作用是设置对应标志位，释放流，并在必要时（即一个连接上的StreamAllocation应用链表为空）释放连接，在连接管理部分还会介绍连接的释放。

至此StreamAllocation就介绍完了，后面还有cancel和streamFailed方法，前者比较简单可自行查看，后者还不太清楚一些异常的含义，后续继续学习过程中会做补充，但已经不影响正常分析StreamAllocation的功能，此时我们已经明白该类的功能就是在连接之上新建一个流，完成通信任务，很好地起到了桥梁作用，可能看代码有些突兀，起初考虑先介绍Connection和HttpCodec， 但是没有StreamAllocation又很难说清楚，而StreamAllocation有依赖Connection的一些内容，所以这里关于acquire, release等与资源有关的部分有些不明白的可以在看完介绍连接管理部分以后再回过头再看一次StreamAllocation的代码，相结合理解或许会更清晰，这里我们首先明白他是如何完成新建流的就可以了。

## 2. 连接和流

### RealConnection
首先okhttp中Connection的唯一实现类是RealConnection， 该类的主要功能就是封装Socket并对外提供输入输出流，那么它的内部结构也很容易联想到，它应该持有一个Socket， 并提供一个输入流一个输入流，在功能方法中对外提供connect()方法建立连接。这里由于okhttp是支持Https和HTTP2.0，如果不考虑这两种情况，RealConnection的代码将会比较简单，下面首先来看它的属性域：

```
public final class RealConnection extends Http2Connection.Listener implements Connection {
  private final Route route;

  /** The low-level TCP socket. */
  private Socket rawSocket;

  /**
   * The application layer socket. Either an {@link SSLSocket} layered over {@link #rawSocket}, or
   * {@link #rawSocket} itself if this connection does not use SSL.
   */
  public Socket socket;
...
  private Protocol protocol;
...
  public int successCount;
  public BufferedSource source;
  public BufferedSink sink;
  public int allocationLimit;
  public final List<Reference<StreamAllocation>> allocations = new ArrayList<>();
  public boolean noNewStreams;
  public long idleAtNanos = Long.MAX_VALUE;

  public RealConnection(Route route) {
    this.route = route;
  }
...
```
RealConnection通过一个Route路线来建立连接，它封装一个Socket, 由于考虑到Https的情况，socket有可能时SSLSocket或者RawSocket， 所以这里有两个socket域， 一个在底层，一个在上层，由于我们不考虑Https的情况，那么两个就是等价的，我们只需要明白内部建立rawSocket， 对外提供socket就可以了（注释中也已经明白解释）。另外除了输入流和输出流之外还有连接所使用到的协议，在连接方法中会用到，最后剩下的就是跟连接管理部分相关的统计信息，这些将会在第三部分连接管理中做介绍，这里暂且略过不考虑。

下面就开始看它的连接方法， connect()

```
public void connect(int connectTimeout, int readTimeout, int writeTimeout,
      List<ConnectionSpec> connectionSpecs, boolean connectionRetryEnabled) {
    if (protocol != null) throw new IllegalStateException("already connected");

    ...

    while (protocol == null) {
      try {
        if (route.requiresTunnel()) {
          ...
        } else {
          buildConnection(connectTimeout, readTimeout, writeTimeout, connectionSpecSelector);
        }
      } catch (IOException e) {
        ...
    }
  }
```

这里对代码做了最大的简化，主要去掉了异常处理的部分以及Https需要考虑的部分，从代码中可以看出，建立连接是通过判断protocol是否为空来确定是否已经建立 连接的， 下面就继续看buildeConnection()方法：

```
 private void buildConnection(int connectTimeout, int readTimeout, int writeTimeout,
      ConnectionSpecSelector connectionSpecSelector) throws IOException {
    connectSocket(connectTimeout, readTimeout);
    establishProtocol(readTimeout, writeTimeout, connectionSpecSelector);
  }

  private void connectSocket(int connectTimeout, int readTimeout) throws IOException {
    Proxy proxy = route.proxy();
    Address address = route.address();

    //根据是否需要代理建立不同的Socket
    rawSocket = proxy.type() == Proxy.Type.DIRECT || proxy.type() == Proxy.Type.HTTP
        ? address.socketFactory().createSocket()
        : new Socket(proxy);

    rawSocket.setSoTimeout(readTimeout);
    try {
    	//内部就是调用socket.connect(InetAddress, timeout)方法建立连接
      Platform.get().connectSocket(rawSocket, route.socketAddress(), connectTimeout);
    } catch (ConnectException e) {
     	...
    }

    //获取输入输出流
    source = Okio.buffer(Okio.source(rawSocket));
    sink = Okio.buffer(Okio.sink(rawSocket));
  }

  private void establishProtocol(int readTimeout, int writeTimeout,
      ConnectionSpecSelector connectionSpecSelector) throws IOException {
    if (route.address().sslSocketFactory() != null) {
      ...
    } else {
      protocol = Protocol.HTTP_1_1;
      socket = rawSocket;
    }

    if (protocol == Protocol.HTTP_2) {
    	...
    } else {
      //HTTP1.1及以下版本中， 每个连接只允许有一个流
      this.allocationLimit = 1;
    }
  }
```
这里同样对代码做了简化，并在重要的地方做了注释， 流程也很清楚，就是建立socket连接，并获取输入输出流，然后设置正确的协议，此时connect()方法就可以跳出while循环，完成连接。
至此，就完成了连接过程，所以如果除去HTTPs和HTTP2的部分，RealConnection的代码很简单，不过对于HTTPS和HTTP2的处理还是挺多，后续还会继续学习。下面再看另外一个概念，流。

### HTTPCodec

在okttp3中，将流定义成HttpCodec， 也没有去查为什么叫这个名字，有知道的同学欢迎在评论中告知，不胜感激。还是同样的原因，我们只考虑Http1.1及以下版本，我们只需要考虑Http1Codec代码。对于Http1Codec的学习，我们还是很有必要先看一下它的类注释，描述的很清楚
```
/**
 * A socket connection that can be used to send HTTP/1.1 messages. This class strictly enforces the
 * following lifecycle:
 *
 * <ol>
 *     <li>{@linkplain #writeRequest Send request headers}.
 *     <li>Open a sink to write the request body. Either {@linkplain #newFixedLengthSink
 *         fixed-length} or {@link #newChunkedSink chunked}.
 *     <li>Write to and then close that sink.
 *     <li>{@linkplain #readResponse Read response headers}.
 *     <li>Open a source to read the response body. Either {@linkplain #newFixedLengthSource
 *         fixed-length}, {@linkplain #newChunkedSource chunked} or {@linkplain
 *         #newUnknownLengthSource unknown length}.
 *     <li>Read from and close that source.
 * </ol>
 *
 * <p>Exchanges that do not have a request body may skip creating and closing the request body.
 * Exchanges that do not have a response body can call {@link #newFixedLengthSource(long)
 * newFixedLengthSource(0)} and may skip reading and closing that source.
 */
```
流的功能也很清楚就是完成最终的HTTP通信，在socket提供的输出流中写出请求，并在socket的输出流中读取响应。注释也将的很清楚，Http1Codec要求用户整个请求过程需要遵守下列的执行过程，即依次调用如下方法完成网络请求。顺序依次为：写请求（发送请求头部信息）， 打开一个开写入请求体的输出流， 写入请求体并关闭流，读取响应头部信息，打开一个输入响应体的输入流， 读取响应体并关闭流。过程对称而且很容易理解，那么我们就直接来看这些代码实现中都做了哪些事情:

```
public final class Http1Codec implements HttpCodec {
  private static final int STATE_IDLE = 0; // Idle connections are ready to write request headers.
  private static final int STATE_OPEN_REQUEST_BODY = 1;
  private static final int STATE_WRITING_REQUEST_BODY = 2;
  private static final int STATE_READ_RESPONSE_HEADERS = 3;
  private static final int STATE_OPEN_RESPONSE_BODY = 4;
  private static final int STATE_READING_RESPONSE_BODY = 5;
  private static final int STATE_CLOSED = 6;

  /** The client that configures this stream. May be null for HTTPS proxy tunnels. */
  final OkHttpClient client;
  /** The stream allocation that owns this stream. May be null for HTTPS proxy tunnels. */
  final StreamAllocation streamAllocation;

  final BufferedSource source;
  final BufferedSink sink;
  int state = STATE_IDLE;

  public Http1Codec(OkHttpClient client, StreamAllocation streamAllocation, BufferedSource source,
      BufferedSink sink) {
    this.client = client;
    this.streamAllocation = streamAllocation;
    this.source = source;
    this.sink = sink;
  }
```
这里使用六个常量标识HttpCodec对象目前所处的状态， 初始化为IDLE状态， sink和source分别为从connection中获取的输入输出流，其实也就是socket的输入输出流。部分步骤可以省略

```
 //1. 写请求头部信息，格式与Http请求报文相同
  /** Returns bytes of a request header for sending on an HTTP transport. */
  public void writeRequest(Headers headers, String requestLine) throws IOException {
    if (state != STATE_IDLE) throw new IllegalStateException("state: " + state);
    sink.writeUtf8(requestLine).writeUtf8("\r\n");
    for (int i = 0, size = headers.size(); i < size; i++) {
      sink.writeUtf8(headers.name(i))
          .writeUtf8(": ")
          .writeUtf8(headers.value(i))
          .writeUtf8("\r\n");
    }
    sink.writeUtf8("\r\n");
    state = STATE_OPEN_REQUEST_BODY;
  }

  //2. 写请求体
   @Override public Sink createRequestBody(Request request, long contentLength) {
    if ("chunked".equalsIgnoreCase(request.header("Transfer-Encoding"))) {
      // Stream a request body of unknown length.
      return newChunkedSink();
    }

    if (contentLength != -1) {
      // Stream a request body of a known length.
      return newFixedLengthSink(contentLength);
    }

    throw new IllegalStateException(
        "Cannot stream a request body without chunked encoding or a known content length!");
  }
 //不固定长度
  public Sink newChunkedSink() {
    if (state != STATE_OPEN_REQUEST_BODY) throw new IllegalStateException("state: " + state);
    state = STATE_WRITING_REQUEST_BODY;
    return new ChunkedSink();
  }
//固定长度
  public Sink newFixedLengthSink(long contentLength) {
    if (state != STATE_OPEN_REQUEST_BODY) throw new IllegalStateException("state: " + state);
    state = STATE_WRITING_REQUEST_BODY;
    return new FixedLengthSink(contentLength);
  }
  //关于两个Sink, 其实就是输出流，在他们的write(source)方法中，就是将请求体的输入流写入到Http1Codec的sink对象中，最终写入到socket

  //3. 刷新和关闭方法，这一步的方法并不在Http1Codec中，而是在两个内部类中，有兴趣的可以自行查看代码，这里不再贴出

  //4. 读取响应头
   /** Parses bytes of a response header from an HTTP transport. */
  public Response.Builder readResponse() throws IOException {
    if (state != STATE_OPEN_REQUEST_BODY && state != STATE_READ_RESPONSE_HEADERS) {
      throw new IllegalStateException("state: " + state);
    }

    try {
      while (true) {
        StatusLine statusLine = StatusLine.parse(source.readUtf8LineStrict());

//读取响应行以及头部信息，构建响应构建者并设置对应参数
        Response.Builder responseBuilder = new Response.Builder()
            .protocol(statusLine.protocol)
            .code(statusLine.code)
            .message(statusLine.message)
            .headers(readHeaders());

        if (statusLine.code != HTTP_CONTINUE) {
          state = STATE_OPEN_RESPONSE_BODY;
          return responseBuilder;
        }
      }
    } catch (EOFException e) {
    	...
    }
  }

  /** Reads headers or trailers. */
  public Headers readHeaders() throws IOException {
    Headers.Builder headers = new Headers.Builder();
    // parse the result headers until the first blank line
    for (String line; (line = source.readUtf8LineStrict()).length() != 0; ) {
      Internal.instance.addLenient(headers, line);
    }
    return headers.build();
  }

//6. 创建可以读取响应体的输入流，在他们的read()方法中将Http1Codec的source的内容即响应体内容读取到某个输出流中
    public Source newFixedLengthSource(long length) throws IOException {
    if (state != STATE_OPEN_RESPONSE_BODY) throw new IllegalStateException("state: " + state);
    state = STATE_READING_RESPONSE_BODY;
    return new FixedLengthSource(length);
  }

  public Source newChunkedSource(HttpUrl url) throws IOException {
    if (state != STATE_OPEN_RESPONSE_BODY) throw new IllegalStateException("state: " + state);
    state = STATE_READING_RESPONSE_BODY;
    return new ChunkedSource(url);
  }

  public Source newUnknownLengthSource() throws IOException {
    if (state != STATE_OPEN_RESPONSE_BODY) throw new IllegalStateException("state: " + state);
    if (streamAllocation == null) throw new IllegalStateException("streamAllocation == null");
    state = STATE_READING_RESPONSE_BODY;
    streamAllocation.noNewStreams();
    return new UnknownLengthSource();
```
