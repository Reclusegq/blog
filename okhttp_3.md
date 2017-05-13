# OkHttp 源码学习笔记 数据交换的流 HTTPCodec

在上一篇文章中介绍了okhttp中连接概念以及连接建立和管理，其中在拦截器链中的ConnectInterceptor负责建立连接，并在该连接上建立流，将其放置在拦截器链中，在拦截器链中的最后一个拦截器CallServerInterceptor，通过使用流的操作完成网络请求的数据交换。下面从该拦截器开始学习okhttp时如果通过流的操作完成网络通信的。

## 1. 最后一个拦截器CallServerInterceptor
CallServerInterceptor是okhttp中的最后一个拦截器，在拦截器链中，拦截器的顺序是用户定义的应用拦截器，RetryAndFollowUpInterceptor, BridgeInterceptor, CacheInterceptor, ConnectInterceptor, 网络拦截器，以及最后的CallServerInterceptor。前面的若干拦截器的作用主要包括两个方面，一是在递归调用链中对Request和返回的Response进行处理，这主要是针对用户自定义的拦截器，完成我们在应用中不同的需求；二是在拦截器链中完成一定功能，为最终的网络请求提供帮助，这主要是针对okhttp中定义的拦截器，比如ConnectInterceptor主要作用就是根据请求建立连接和流对象，从而帮助最终完成网络请求。虽然作用不同，但是这两类拦截器的代码结构都有共同的特点，即基本包括三个步骤，从chain对象中获取request对象以及其他对象，对请求做处理，然后调用chain.proceed()方法获取网络请求response， 最后在第三步中对响应做响应处理并返回处理之后的Response，当然部分拦截器可以没有第一步和第三步，但是基本结构都是一致的。然而，最后的一个拦截器CallServerInterceptor，其结构则与上述拦截器的结构有所不同，它的主要功能就是建立好的流对象，完成数据通信。
这里首先简单介绍一下流对象，在okhttp中，流对象对应着HttpCodec， 它有对应的两个子类， Http1Codec和Http2Codec， 分别对应Http1.1协议以及Http2.0协议，本文主要学习前者。在Http1Codec中主要包括两个重要的属性，即source和sink，它们分别封装了socket的输入和输出，CallServerInterceptor正是利用HttpCodec提供的I/O操作完成网络通信。下面来看CallServerInterceptor的源码。
对于拦截器，依然是学习它的intercept方法：
```
 @Override 
 public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    HttpCodec httpCodec = realChain.httpStream();
    StreamAllocation streamAllocation = realChain.streamAllocation();
    RealConnection connection = (RealConnection) realChain.connection();
    Request request = realChain.request();

    long sentRequestMillis = System.currentTimeMillis();
    //1. 向socket中写入请求header信息
    httpCodec.writeRequestHeaders(request);

    Response.Builder responseBuilder = null;
    if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
      //a. 省略部分
      ...

      if (responseBuilder == null) {
        //2. 向socket中写入请求body信息
        Sink requestBodyOut = httpCodec.createRequestBody(request, request.body().contentLength());
        BufferedSink bufferedRequestBody = Okio.buffer(requestBodyOut);
        request.body().writeTo(bufferedRequestBody);
        bufferedRequestBody.close();
      } else if (!connection.isMultiplexed()) {
        //b.省略部分
      }
    }
    //3. 完成网络请求的写入
    httpCodec.finishRequest();

    //4. 读取网络响应header信息
    if (responseBuilder == null) {
      responseBuilder = httpCodec.readResponseHeaders(false);
    }

    Response response = responseBuilder
        .request(request)
        .handshake(streamAllocation.connection().handshake())
        .sentRequestAtMillis(sentRequestMillis)
        .receivedResponseAtMillis(System.currentTimeMillis())
        .build();

    int code = response.code();
    if (forWebSocket && code == 101) {
        //d. 省略部分
    } else {
      //5. 读取网络响应的body信息
      response = response.newBuilder()
          .body(httpCodec.openResponseBody(response))
          .build();
    }
    // d. 省略部分 
    return response;
  }
```
本文对代码做部分的省略，突出主要的流程，省略已经做了a, b, c, b的标注，后面再分别对其学习分析，然后是最主要的流程做了注释，主要是分成5步，在执行网络请求之前，首先时从拦截器链中获取连接，流以及它们二者的管理者streamAllocation，还有原始的网络请求request，然后执行以下五个步骤
 - 1  向socket中写入请求header信息
 - 2  向socket中写入请求body信息
 - 3  完成网络请求的写入
 - 4 
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

   //不固定长度
    if ("chunked".equalsIgnoreCase(request.header("Transfer-Encoding"))) {
      // Stream a request body of unknown length.
      return newChunkedSink();
    }

    //固定长度
    if (contentLength != -1) {
      // Stream a request body of a known length.
      return newFixedLengthSink(contentLength);
    }

    throw new IllegalStateException(
        "Cannot stream a request body without chunked encoding or a known content length!");
  }
  //关于两个Sink, 其实就是输出流，它起到一个中介作用，在CallServierIntercepter中或通过Http1CodeC流对象获取到这两个Sink输出流，
  //而将请求体的内容写入到这两个输出流，进而会调用他们的write(source)方法，最终写入到Http1Codec中的Sink对象，也就是写入到socket

  //3. 写入和关闭方法，这一步的方法并不在Http1Codec中，而是如上所述，在CallServerIntercepter中，将写入请求体，并关闭创建的Sink对象

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

  //5. 创建响应的输入流
  private Source getTransferStream(Response response) throws IOException {
    if (!HttpHeaders.hasBody(response)) {
      return newFixedLengthSource(0);
    }

    if ("chunked".equalsIgnoreCase(response.header("Transfer-Encoding"))) {
      return newChunkedSource(response.request().url());
    }

    long contentLength = HttpHeaders.contentLength(response);
    if (contentLength != -1) {
      return newFixedLengthSource(contentLength);
    }
    ...
  }
```
流的主要功能完成数据的通信，所以它的主要流程就是针对封装Socket的输入输出，okhttp中使用Connecton封装Socket, StreamAllocation作为桥梁使得在连接上建立一个流，流就可以获取连接封装socket的输入流和输出流，然后HttpCodec对象针对输出流和输入流完成数据通信，并对外提供写入读出方法。这里主要流程分成五步在代码中说明，其中输入输出都是对称的， 其中writeResquest，和readResponse对应，前者将传入的请求头写入到sink对象，后者则是读取source对象中的数据构建一个Response.Builder，提供给调用者来构建Response， 
createResquestBody(request, contentLength)和openResponseBody(Response)对应，其中前者可以看作是对外提供一个输出流，该输出流包裹着sink，这样调用者就可以将请求体写入到构建的输出流中，后者与之对应，就是封装source，并构建一个ResponseBody提供给调用者，其中openResponseBody方法就是调用的上述getTransformStream方法来封装source的。最后是第三步，刷新并关闭，外部调用者可以调用如下方法：

```
@Override public void finishRequest() throws IOException {
    sink.flush();
  }
```
而与之对应的source的关闭则由用户也就是使用者在处理完响应数据以后关闭ResponseBody，这也就是为什么ResponseBody必须要关闭。


流的结束包括成功结束，终止取消，失败等多种情况，这里我们首先来看他们最终都会调用的释放资源的方法：
```
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
释放资源的方法包括三个参数，第一个noNewStream可以简单地理解为该allocation对应的connection对象已经不可用，可以回收了，可以从代码中看出connection.noNewstream标志位被设置，在连接管理部分我们会看到该连接会从连接池中回收。第二个参数可以理解为将allocation对象的引用可以从connection的管理流的链表中移除，但是这也只是标识可以，还需要等到流对应任务执行结束以后才可以移除，标识流任务执行完成就是第三个参数了。下面我们来分析代码的逻辑， finish以后流对象会被置空，但是释放该allocation对象，即从链表中移除该allocation对应的引用需要等到连接不可用或者调用了该allocation对用的release方法以后才可以。在释放了allocation对象以后需要检查对应的连接对象流的分配是否已经为空，没有流分配以后则调用connectionPool.becameIdle(connection)方法，这里由于该方法时包权限，所以也借助了Internal, 该方法试图从连接池中移除该连接，如果移除则返回true， 如果只是唤醒连接池的清理任务则返回false, 关于连接池我们在连接管理部分会做分析。我们可以看release(connection)方法所执行的释放任务：
```
/** Remove this allocation from the connection's list of allocations. */
private void release(RealConnection connection) {
  for (int i = 0, size = connection.allocations.size(); i < size; i++) {
    Reference<StreamAllocation> reference = connection.allocations.get(i);
    if (reference.get() == this) {
      connection.allocations.remove(i);
      return;
    }
  }
  throw new IllegalStateException();
}

/**
   * Use this allocation to hold {@code connection}. Each call to this must be paired with a call to
   * {@link #release} on the same connection.
   */
  public void acquire(RealConnection connection) {
    assert (Thread.holdsLock(connectionPool));
    connection.allocations.add(new StreamAllocationReference(this, callStackTrace));
  }
```


这里前三个方法都会调用deallocate方法来释放资源，该方法有三个参数，分别标识对应的连接是否可以使用，是否将该StreamAllocation对象从Connection的引用流链表里移除，流是否完成任务。该方法主要作用是设置对应标志位，释放流，并在必要时（即一个连接上的StreamAllocation应用链表为空）释放连接，在连接管理部分还会介绍连接的释放。

至此StreamAllocation就介绍完了，后面还有cancel和streamFailed方法，前者比较简单可自行查看，后者还不太清楚一些异常的含义，后续继续学习过程中会做补充，但已经不影响正常分析StreamAllocation的功能，此时我们已经明白该类的功能就是在连接之上新建一个流，完成通信任务，很好地起到了桥梁作用，可能看代码有些突兀，起初考虑先介绍Connection和HttpCodec， 但是没有StreamAllocation又很难说清楚，而StreamAllocation有依赖Connection的一些内容，所以这里关于acquire, release等与资源有关的部分有些不明白的可以在看完介绍连接管理部分以后再回过头再看一次StreamAllocation的代码，相结合理解或许会更清晰，这里我们首先明白他是如何完成新建流的就可以了。