本篇文章为okhttp源码学习笔记系列的第二篇文章，本篇文章的主要内容为okhttp中的连接与流的管理，因此需要重点介绍的就连接和流两个概念。客户端通过HTTP协议与服务器进行通信，首先需要建立连接，okhttp并没有使用URLConnection， 而是对socket直接进行封装，在socket之上建立了connection的概念，代表这物理连接。同时，一对请求与响应对应着输出和输入流， okhttp中同样使用流的逻辑概念，建立在connection的物理连接之上进行数据通信，（只不过不知道为什么okhttp中流的类名为HttpCodec）。在HTTP1.1以及之前的协议中，一个连接只能同时支持单个流，而SPDY和HTTP2.0协议则可以同时支持多个流，本文先考虑HTTP1.1协议的情况。connection负责与远程服务器建立连接， HttpCodec负责请求的写入与响应的读出，但是除此之外还存在在一个连接上建立新的流，取消一个请求对应的流，释放完成任务的流等操作，为了使HttpCodec不至于过于复杂，okhttp中引入了StreamAllocation负责管理一个连接上的流，同时在connection中也通过一个StreamAllocation的引用的列表来管理一个连接的流，从而使得连接与流之间解耦。另外，熟悉HTTP协议的同学肯定知道建立连接是昂贵的，因此在请求任务完成以后（即流结束以后）不会立即关闭连接，使得连接可以复用，okhttp中通过ConnecitonPool来完成连接的管理和复用。

因此本文会首先介绍StreamAllocation类，并说明连接与流的概念，以及管理机制，其次会依次介绍Connection和HttpCodec1(HttpCodec的一个实现类，代表HTTP1.1协议)，然后再通过分析ConnectionPool来分析连接的管理机制，最后会继续学习第一篇文章中最后介绍的整个网络请求流程（其实就是一系列的拦截器的调用），本文会介绍其中与连接和流有关的三个拦截器，分别时RetryInterceptor, ConnectionInterceptor和CallServerInterceptor。当然期间还有涉及Rout, RoutSelector等一些为建立连接或建立流有关的辅助类。

## 1. 从两个拦截器开始
在上一篇文章中我们了解到okhttp中的网络请求过程其实就是一系列拦截器的处理过程，那么我们就可以以这些拦截器为主线看一下okhttp的实现中都做了哪些事情，首先我们再次贴出RealCall#getResponseWithInterceptor方法：
```
Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    interceptors.addAll(client.interceptors());
    interceptors.add(retryAndFollowUpInterceptor);
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));

    Interceptor.Chain chain = new RealInterceptorChain(
        interceptors, null, null, null, 0, originalRequest);
    return chain.proceed(originalRequest);
  }
```
从方法中可以看出okhttp内部定义了五个拦截器，分别负责重试（包括失败重连，添加认证头部信息，重定向等），request和response的转换（主要是将Request和Response对象转换成Http协议定义的报文形式）， 缓存以及连接处理，最后一个与其他稍有不同，它负责流的处理， 将请求数据写入到socket的输出流，并从输入流中获取响应数据，这个在第三篇文章中做分析。
本篇文章的重点是连接以及连接管理，可以看出与连接有关的时RetryAndFollowUpInterceptor和ConnectionInterceptor，那么我们就从这两个拦截器开始分析。
在前面一篇文章中我们了解到拦截器会对外提供一个方法，即intercept(chain)方法，在该方法中处理请求，并调用chain.proceed()方法获取响应，处理后返回响应结果，那么我们首先来看RetryAndFollowUpInterceptor#intercept()方法：
```
@Override public Response intercept(Chain chain) throws IOException {
    Request request = chain.request();

    streamAllocation = new StreamAllocation(
        client.connectionPool(), createAddress(request.url()), callStackTrace);

    int followUpCount = 0;
    Response priorResponse = null;
    while (true) {
      ...
      Response response = null;
      boolean releaseConnection = true;
      try {
        response = ((RealInterceptorChain) chain).proceed(request, streamAllocation, null, null);
        releaseConnection = false;
      } catch (... e) {
       	...
      } finally {
        // We're throwing an unchecked exception. Release any resources.
        if (releaseConnection) {
          streamAllocation.streamFailed(null);
          streamAllocation.release();
        }
      }
      ...
```
这里的代码只是前一半，并且做了部分简化，由于本篇文章分析的是连接与连接管理，因此这里我们只关注与连接相关的部分。这里我们看到在该方法中，首先创建了StreamAllocation对象，该对象是连接与流的桥梁，okhttp处理一个请求时，它负责为该请求找到一个合适的连接（找到是指从连接池中复用连接或者创建新连接）， 并在该连接上创建流对象，通过该流对象完成数据通信。这里的流对象负责数据通信，即向socket的输出流中写请求数据，并从socket的输出流中读取响应数据，而StreamAllocation则负责管理流，包括为请求查找连接，并在该连接上建立流对象，将连接封装的socket对象中的输入输出流传递到okhttp流对象中，使他去完成数据通信任务，这是一种功能或责任的分离，这一点有点像之前分析的Request对象和Call对象的关系，另外一个连接也是通过持有一个关于StreamAllocation的集合来管理一个连接上的多个流（只有在SPDY和HTTP2上存在一个连接上多个流，HTTP1.1及1.0则是在一个连接上同时只有一个流对象）。
从代码中我们看到，创建的streamAllocation对象传递到proceed()方法中，之前对于拦截器链的分析中我们知道，在proceed()方法中递归地创建新的chain对象，并添加proceed()方法中传递进来的对象，提供给后面的拦截器使用，这包括StreamAllocation, Connection, HttpCodec三个主要的类的对象，其中最后一个就是所谓的流对象（不太清楚okhttp为何如此命名）。
RetryAndFollowUpInterceptor#intercept()方法中其他的逻辑我们在之后的文章中再做分析，这里我们主要是需要了解StreamAllocation对象，以及继续熟悉okhttp中拦截器链的执行机制，下面我们再来分析Connection#intercept()方法：
```
@Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Request request = realChain.request();
    StreamAllocation streamAllocation = realChain.streamAllocation();

    // We need the network to satisfy this request. Possibly for validating a conditional GET.
    boolean doExtensiveHealthChecks = !request.method().equals("GET");
    HttpCodec httpCodec = streamAllocation.newStream(client, doExtensiveHealthChecks);
    RealConnection connection = streamAllocation.connection();

    return realChain.proceed(request, streamAllocation, httpCodec, connection);
  }
```
这里没有对代码做简化，可以看出ConnectionInterceptor的代码很简单，我们暂且忽略doExtensiveHealthChecks，这个跟具体的HTTP协议的规则有关， 这里最关键的是streamAllocation的方法，即newStream()方法，该方法负责寻找连接，并在该连接上建立新的流对象，并返回，而connection()方法只不过是分会找到的连接而已，这里看到三个对象在这时都已经分别创建出来，此时传递到InterceptorChain上，CallServerInterceptor就可以使用这些对象完成数据通信了，这一点在下一篇文章中分析流对象时再做分析，这里我们只是关注连接，那么我们就要StreamAllocation开始。

## 1. 连接与流的桥梁

首先我们明白HTTP通信执行网络请求需要在连接之上建立一个新的流执行该任务， 我们这里将StreamAllocation称之为连接与流的桥梁， 它负责为一次请求寻找连接并建立流，从而完成远程通信，所以StreamAllocation与请求，连接，流都相关，因此我们首先熟悉一下这三个概念，对于它们三个， StreamAllocation类之前的注释已经给出了解释，这里我们首先看一下它的注释：
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

下面我们来思考StreamAllocation所要解决的问题，简单来讲就是在合适的连接之上建立一个新的流，这个问题划分为两步就是寻找连接和新建流。那么，StreamAllocation的数据结构中应该包含Stream(okhttp中将接口的名字定义为HttpCodec)， Connction, 其次为了寻找合适的连接，应该包含一个URL地址， 连接池ConnectionPool, 而方法则应该包括我们前面提到的newStream()方法, 而findConnection则是为之服务的方法，其次在完成请求任务之后应该有finish方法用来关闭流对象，还有终止和取消等方法， 以及释放资源的方法。下面我们从StreamAllocation中寻找对应的属性和方法， 首先来看它的属性域：

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
这里我们看并没有使用URL代表一个地址，而是使用Address对象， 如果查看其代码可以看到它封装了一个URL, 其中还包括dns, 代理等信息，可以更好地满足HTTP协议中规定的细节。使用Address对象也可以直接ConnectionPool中查找对应满足条件的连接，同时Address对象可以用于在RoutSelector查询一个合适的路径Rout对象，该Route对象可以用于建立连接， 然后我们忽略标志位和统计信息，剩下的则是Connection和HttpCodec对象，这里我们先忽略调用栈callStackTrace， 不考虑。以上就是它所有的属性域，那么下面我们再来看它最重要的方法，即新建流的方法：

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

 这两个方法完成了寻找合适的连接的功能，这里我们可以将其分成两种类型：
   1. 从连接池中复用连接，这一点我们在连接管理部分中会有介绍，其实就是从维护的队列中找到合适的连接并返回，查找的依据就是Address对象；
   2. 新建连接，新建连接需要首先查找一个合适的路径，然后在该路径上实例化一个新的connection对象，在建立一个新的连接以后需要执行一系列的步骤，在代码中已经用注释的方式分四步标出。
   在或许到连接以后还需要执行检查过程，通常来说以上两种类型选择的连接都需要执行检查，只是这里新的连接一定可用，所以会跳过检查，其实检查的过程就是查看该连接的socket是否被关闭以及是否可以正常使用，有兴趣的可以自行查看代码。
 在建立新的连接以后需要处理一些事情，代码中的中文注释分了四步将其标出，其中需要说明第一步，acquire方法，connection中维护这一张在一个连接上的流的链表，该链表保存的是StreamAllocation的引用 Connction的该链表为空时说明该连接已经可以回收了，这部分在连接管理部分会有说明。其实在调用ConnectionPool.get()方法时，传入StreamAllocation对象也是为了调用StreamAllocation#acquire方法，将该StreamAllocation对象的引用添加到连接对应的链表中，用于管理一个连接上的流。

 此外还需要说明的两点，一是关于Internal, 这是一个抽象类，该类只有一个实现类，时HttpClient的一个匿名内部类，该类的一系列方法都是一个功能，就是将okhttp3中一些包访问权限的方法对外提供一个public的访问方法，至于为什么这么实现，目前还不太清楚，估计是在Okhttp的框架中方便使用包权限的方法， 如ConnectionPool的put和get方法，在Okhttp的框架代码中通过Internal代理访问， 而在外部使用时无法访问到这些方法，至于还有没有其他考虑有清楚的同学欢迎赐教。
 需要说明的第二点是RouteDatabase是一个黑名单，记录着不可用的路线，避免在同一个坑里栽倒两次，connect()方法就是将该路线移除黑名单，这里为了不影响StreamAllocation分析的连贯性，本文将RoutSelector和RouteDatabase等与路线选择相关的代码放到附录部分，这里我们暂且先了解它的功能而不关注它的实现。

至此就分析完了建立新流的过程，那么剩下的就是释放资源的逻辑，由于这一部分很多地方都与连接的复用管理，流的操作以及RetryAndFollowUpInterceptor中的逻辑有关，所以在这里暂且略过，后续部分再做分析，我们此处重点是了解如何查询连接，如何新建连接以及如何新建流对象。

## 2. 连接
下面开始分析连接对象，在okhttp中定义了Connection的接口，而该接口在okhttp只有一个实现类，即RealConnetion，下面我们重点分析该类。该类的主要功能就是封装Socket并对外提供输入输出流，那么它的内部结构也很容易联想到，它应该持有一个Socket， 并提供一个输入流一个输入流，在功能方法中对外提供connect()方法建立连接。这里由于okhttp是支持Https和HTTP2.0，如果不考虑这两种情况，RealConnection的代码将会比较简单，下面首先来看它的属性域：

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
RealConnection通过一个Route路线（或者说路由或路径）来建立连接，它封装一个Socket, 由于考虑到Https的情况，socket有可能时SSLSocket或者RawSocket， 所以这里有两个socket域， 一个在底层，一个在上层，由于我们不考虑Https的情况，那么两个就是等价的，我们只需要明白内部建立rawSocket， 对外提供socket就可以了（注释中也已经明白解释）。另外除了输入流和输出流之外还有连接所使用到的协议，在连接方法中会用到，最后剩下的就是跟连接管理部分相关的统计信息，allocationLimit是分配流的数量上限，对应HTTP1.1来说它就是1， allocations在StreamAllocation部分我们已经熟悉，它是用来统计在一个连接上建立了哪些流，通过StreamAllocation的acquire方法和release方法可以将一个allocation对象添加到链表或者移除链表，（不太清楚这两个方法放在connection中是不是更合理一些）， noNewStream之前说过可以简单理解为它标识该连接已经不可用，idleAtNanos记录该连接处于空闲状态的时间，这些将会在第三部分连接管理中做介绍，这里暂且略过不考虑。

下面就开始看它的连接方法， connect()方法

```
public void connect(int connectTimeout, int readTimeout, int writeTimeout,
      List<ConnectionSpec> connectionSpecs, boolean connectionRetryEnabled) {
    if (protocol != null) throw new IllegalStateException("already connected");
    ...
    while (protocol == null) {
      try {
          buildConnection(connectTimeout, readTimeout, writeTimeout, connectionSpecSelector);
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

## 3. 连接管理
对于连接的管理主要是分析ConnectionPool，以连接池的形式管理连接的复用， okhttp中尽可能对于相同地址的远程通信复用同一个连接，这样就节省了连接的代价。那么我们现在明白了ConnectionPool所要解决的问题就可以去思考它应当如何实现，它应当具备的功能可以简单分为三个方法，get, put, cleanup，即获取添加和清理的方法。其中最重要也是最复杂的是清理，即在连接池放满的情况下，如何选择需要淘汰的连接。这里需要考虑的指标包括连接的数量，连接的空闲时长等，那么下面我们来看ConnectionPool的代码：

```
/**
 * Manages reuse of HTTP and HTTP/2 connections for reduced network latency. HTTP requests that
 * share the same {@link Address} may share a {@link Connection}. This class implements the policy
 * of which connections to keep open for future use.
 */
public final class ConnectionPool {
  /**
   * Background threads are used to cleanup expired connections. There will be at most a single
   * thread running per connection pool. The thread pool executor permits the pool itself to be
   * garbage collected.
   */
  private static final Executor executor = new ThreadPoolExecutor(0 /* corePoolSize */,
      Integer.MAX_VALUE /* maximumPoolSize */, 60L /* keepAliveTime */, TimeUnit.SECONDS,
      new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp ConnectionPool", true));

  boolean cleanupRunning;
  private final Runnable cleanupRunnable = new Runnable() {
    ...
  };

  /** The maximum number of idle connections for each address. */
  private final int maxIdleConnections;
  private final long keepAliveDurationNs;

  private final Deque<RealConnection> connections = new ArrayDeque<>();
  final RouteDatabase routeDatabase = new RouteDatabase();


  /**
   * Create a new connection pool with tuning parameters appropriate for a single-user application.
   * The tuning parameters in this pool are subject to change in future OkHttp releases. Currently
   * this pool holds up to 5 idle connections which will be evicted after 5 minutes of inactivity.
   */
  public ConnectionPool() {
    this(5, 5, TimeUnit.MINUTES);
  }

  public ConnectionPool(int maxIdleConnections, long keepAliveDuration, TimeUnit timeUnit) {
    this.maxIdleConnections = maxIdleConnections;
    this.keepAliveDurationNs = timeUnit.toNanos(keepAliveDuration);

    // Put a floor on the keep alive duration, otherwise cleanup will spin loop.
    if (keepAliveDuration <= 0) {
      throw new IllegalArgumentException("keepAliveDuration <= 0: " + keepAliveDuration);
    }
  }

 ```
首先来看它的属性域，最主要的就是connections，可见在ConnectionPool内部以队列的方式存储连接，而routeDatabase是一个黑名单，用来记录不可用的route，但是看代码目测ConnectionPool中并没有使用它，可能后续还有其他用处，此处不做分析。剩下的就是与清理相关的，从名字则可以看出他们的用途，最开始的三个分别是执行清理任务的线程池，标志位以及清理任务，而maxIdleConnections和keepAliveDurationNs则是清理中淘汰连接的指标，这里需要说明的是maxIdleConnections是值每个地址上最大的空闲连接数（如注释所说），看来okhttp只是限制了与同一个远程服务器的空闲连接数量，对整体的空闲连接数量并没有做限制，但是从代码来看并不是如此，该值时标识该连接池中整体的空闲连接的最大数量，当一个连接数量超过这个值时则会触发清理，这里留有疑问，不知注释是何意， 另外还有一个疑问是okhttp并没有限制一个连接池中连接的最大数量，而只是限制了连接池中的最大空闲连接数量，由于不懂HTTP协议，水平也有限，不太清楚这里需不需要限制，有了解的欢迎在评论中告知，不胜感激。
最后我们可以从默认的构造器中看出okhttp允许每个地址同时可以有五个连接，每个连接空闲时间最多为五分钟。

下面我们首先来看较为简单一些的get和put方法
```
/** Returns a recycled connection to {@code address}, or null if no such connection exists. */
  RealConnection get(Address address, StreamAllocation streamAllocation) {
    assert (Thread.holdsLock(this));
    for (RealConnection connection : connections) {
      if (connection.allocations.size() < connection.allocationLimit
          && address.equals(connection.route().address)
          && !connection.noNewStreams) {
        streamAllocation.acquire(connection);
        return connection;
      }
    }
    return null;
  }
```
获取一个connection时按照Address来匹配，而建立连接也是通过Address查询一个合适的Route来建立的，当匹配到连接时，会将新建立的流对应的StreamAllocation添加到connection.allocations中， 如果这里调用connection.allocations.add(StreamAllocation)或者Connection自定义的add方法会不会更清晰一些，不太明白为什么将该任务放在了StreamAllocation的acquire方法中，此处的streamAllocation的acquire方法其实也就是做了这件事情，用来管理一个连接上的流。

put方法
```
void put(RealConnection connection) {
    assert (Thread.holdsLock(this));
    if (!cleanupRunning) {
      cleanupRunning = true;
      executor.execute(cleanupRunnable);
    }
    connections.add(connection);
  }
```
put方法更为简单，就是异步触发清理任务，然后将连接添加到队列中。那么下面开始重点分析它的清理过程，首先来看清理任务的定义：
```
private final Runnable cleanupRunnable = new Runnable() {
    @Override public void run() {
      while (true) {
        long waitNanos = cleanup(System.nanoTime());
        if (waitNanos == -1) return;
        if (waitNanos > 0) {
          long waitMillis = waitNanos / 1000000L;
          waitNanos -= (waitMillis * 1000000L);
          synchronized (ConnectionPool.this) {
            try {
              ConnectionPool.this.wait(waitMillis, (int) waitNanos);
            } catch (InterruptedException ignored) {
            }
          }
        }
      }
    }
  };
 ```
 逻辑也很简单，就是调用cleanup方法执行清理，并等待一段时间，持续清理，而等待的时间长度时有cleanup函数返回值指定的，那么我们继续来看cleanup函数

 ```
 /**
   * Performs maintenance on this pool, evicting the connection that has been idle the longest if
   * either it has exceeded the keep alive limit or the idle connections limit.
   *
   * <p>Returns the duration in nanos to sleep until the next scheduled call to this method. Returns
   * -1 if no further cleanups are required.
   */
  long cleanup(long now) {
    int inUseConnectionCount = 0;
    int idleConnectionCount = 0;
    RealConnection longestIdleConnection = null;
    long longestIdleDurationNs = Long.MIN_VALUE;

    // Find either a connection to evict, or the time that the next eviction is due.
    synchronized (this) {
      for (Iterator<RealConnection> i = connections.iterator(); i.hasNext(); ) {
        RealConnection connection = i.next();

        // If the connection is in use, keep searching.
        if (pruneAndGetAllocationCount(connection, now) > 0) {
          inUseConnectionCount++;
          continue;
        }
        //统计空闲连接的数量
        idleConnectionCount++;

        // If the connection is ready to be evicted, we're done.
        long idleDurationNs = now - connection.idleAtNanos;
        if (idleDurationNs > longestIdleDurationNs) {//找出空闲时间最长的连接以及对应的空闲时间
          longestIdleDurationNs = idleDurationNs;
          longestIdleConnection = connection;
        }
      }


      if (longestIdleDurationNs >= this.keepAliveDurationNs
          || idleConnectionCount > this.maxIdleConnections) {
        // We've found a connection to evict. Remove it from the list, then close it below (outside
        // of the synchronized block).
        connections.remove(longestIdleConnection);  //在符合清理条件下，清理空闲时间最长的连接
      } else if (idleConnectionCount > 0) {
        // A connection will be ready to evict soon.
        return keepAliveDurationNs - longestIdleDurationNs;  //不符合清理条件，则返回下次需要执行清理的等待时间
      } else if (inUseConnectionCount > 0) {
        // All connections are in use. It'll be at least the keep alive duration 'til we run again.
        return keepAliveDurationNs;  //没有空闲的连接，则隔keepAliveDuration之后再次执行
      } else {
        // No connections, idle or in use.
        cleanupRunning = false;  //清理结束
        return -1;
      }
    }

    closeQuietly(longestIdleConnection.socket());  //关闭socket资源

    // Cleanup again immediately.
    return 0;  //这里是在清理一个空闲时间最长的连接以后会执行到这里，需要立即再次执行清理
  }
 ```
这里的首先统计空闲连接数量，然后查找最长空闲时间的连接以及对应空闲时长，然后判断是否超出最大空闲连接数量或者超过最大空闲时长，满足其一则执行清理最长空闲时长的连接，然后立即再次执行清理，否则会返回对应的等待时间，代码中中文注释已做说明。方法中用到了一个方法来查看一个连接上分配流的数量，这里不再贴出，有兴趣的可以自行查看。
接下来我们梳理一下清理的任务，清理任务是异步执行的，遵循两个指标，最大空闲连接数量和最大空闲时长，满足其一则清理空闲时长最大的那个连接，然后循环执行，要么等待一段时间，要么继续清理下一个连接，直到清理所有连接，清理任务才可以结束，下一次put方法调用时，如果已经停止的清理任务则会被再次触发开始。
ConnectionPool的主要方法就是这三个，其余的则是工具私有方法或者getter, setter方法，另外还有一个需要介绍一下我们在StreamAllocation中遇到的方法 connectionBecameIdle标识一个连接处于了空闲状态，即没有流任务，那么就需要调用该方法，有ConnectionPool来决定是否需要清理该连接：
```
 /**
   * Notify this pool that {@code connection} has become idle. Returns true if the connection has
   * been removed from the pool and should be closed.
   */
  boolean connectionBecameIdle(RealConnection connection) {
    assert (Thread.holdsLock(this));
    if (connection.noNewStreams || maxIdleConnections == 0) {
      connections.remove(connection);
      return true;
    } else {
      notifyAll(); // Awake the cleanup thread: we may have exceeded the idle connection limit.
      return false;
    }
  }
```
这里noNewStream标志位之前说过，它可以理解为该连接已经不可用，所以可以直接清理，而maxIdleConnections==0则标识不允许有空闲连接，也是可以直接清理的，否则唤醒清理任务的线程，执行清理方法。

至此则分析完了连接管理的逻辑，其实就是连接复用，主要包括get, put , cleanup三个方法，重点时清理任务的执行，我们可以在OkHttpClient中配置

## 后记

关于okhttp的连接和连接管理，逻辑还是比较容易理解，但是StreamAllocation的概念在刚接触时还是比较令人费解，但是为了逻辑顺序本篇文章还是从StreamAllocation开始分析，进而引入了连接和连接复用管理部分，读者可在理解了连接和连接复用管理部分的代码以后在回头再去读StreamAllocation的代码或许更容易理解一些。在最初的计划中是将连接与流一起分析，正是因为StreamAlloc