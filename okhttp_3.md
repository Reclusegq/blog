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
        //c. 省略部分
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
 - 4  读取网络响应header信息
 - 5  读取网络响应的body信息
注意在一次网络请求中可能并不包括所有的这五个步骤，比如第二不，写入请求体，只有请求方法中有请求体的时候才会写入，而且有些情况只有在得到服务器允许的时候才会写入请求体，这一点后面会有提到；另外这里所使用的写入和读取两个词并不完全准确，比如第二步只是内存与socket的输出流建立关系，并没有真正写入，直到第三步刷新时才会将请求的信息写入到socket的输出流中，同样地，第五步中获取到响应的body信息，只是获取一个流对象，只有在应用代码中调用流对象的读方法或者response.string()方法等，才会从socket的输入流中读取信息到应用的内存中使用。
这五步逻辑很清晰，也很容易理解，下面中逐个学习省略的四个部分。首先看省略部分a的代码：
```
      // If there's a "Expect: 100-continue" header on the request, wait for a "HTTP/1.1 100
      // Continue" response before transmitting the request body. If we don't get that, return what
      // we did get (such as a 4xx response) without ever transmitting the request body.
      if ("100-continue".equalsIgnoreCase(request.header("Expect"))) {
        httpCodec.flushRequest();
        responseBuilder = httpCodec.readResponseHeaders(true);
      }
```
这里时处理一种特殊情况，即首先发送询问服务器是否可以发送带有请求体的请求，在该请求中请求的头部信息中添加Expect:100-continue字段，服务器如果可以接受请求体则可以返回一个100的响应码，客户端继续发送请求，具体的可以参考相关文章对100响应码的介绍。
这里我们看到如果请求中有该头部信息会跳过第二步，直接执行三四步，获取响应信息，我们继续往下看okhttp对该特殊情况的处理逻辑。这里我们有必要提前看一下HttpCodec的具体实现，当然我们这里是分析Http1Codec的readResponseHeaders(boolean)代码：
```
@Override public Response.Builder readResponseHeaders(boolean expectContinue) throws IOException {
    ...
    try {
      StatusLine statusLine = StatusLine.parse(source.readUtf8LineStrict());

      Response.Builder responseBuilder = new Response.Builder()
          .protocol(statusLine.protocol)
          .code(statusLine.code)
          .message(statusLine.message)
          .headers(readHeaders());

      if (expectContinue && statusLine.code == HTTP_CONTINUE) {
        return null;
      }
      ...
    } catch (EOFException e) {
      ...
    }
  }
```
这里因为时分析特殊情况，所以这里只看参数为true的情况，从代码中可以看到获取响应头部信息包括获取响应行和响应头两部分，具体代码可以自行查看，这里不再展开，当服务器同意接收请求体时回返回100的响应码，可见此时该方法返回空，其他情况会返回非空的响应对象。下面再回到CallServerInterceptor中的代码
当responseBuilder为空时继续执行正常逻辑，即从第二步开始执行。当responseBuilder不为空时，就不可以写如请求体信息，下面的else if()语句时针对Http2协议时关闭当前连接，这里我么暂时不考虑，Http1.1协议下，代码会跳过写请求体的步骤，继续执行，并且因为responseBuilder不为空也会跳过读取响应头的步骤，因为之前读过一次，但是响应码不是100而已，可见当响应码为100时会读取两次响应头，当然也执行了两次请求（注意httpCodec.finishRequest()方法的调用就是刷新输出流，也就相当于执行了一次请求）。
c省略部分是针对websocket所做的处理，由于对H5以及websocket不了解，这里就跳过该部分，有兴趣的同学可以自行查看。
d省略部分就是收尾工作，对一些特殊情况的处理，下面为代码：
```
if ("close".equalsIgnoreCase(response.request().header("Connection"))
        || "close".equalsIgnoreCase(response.header("Connection"))) {
      streamAllocation.noNewStreams();
    }

    if ((code == 204 || code == 205) && response.body().contentLength() > 0) {
      throw new ProtocolException(
          "HTTP " + code + " had non-zero Content-Length: " + response.body().contentLength());
    }
```
代码逻辑很简单，根据响应信息在必要时关闭连接，在上一篇关于连接的文章中我们介绍了streamAllocation的作用，对应关闭和回收资源的问题没有考虑清除，本文会在介绍完流的概念以后，再具体分析关于连接的关闭以及资源回收的问题。最后为检查响应码204和205两种响应，这两种响应没有响应体。
至此我们分析完了CallServerInterceptor的代码，可以看出由它实现的网络请求，而在完成这一功能的若干步骤中都是依赖HttpCodec提供的功能来完成的，我们只考虑Http1.1协议，所以下面开始分析Http1Codec的代码。
## 2. OkHttp中的流 HttpCodec
在分析以上所提到的五个步骤之前，需要说明在Http1Codec中使用了状态模式，其实就是对象维护它所处的状态，在不同的状态下执行对应的逻辑，并更新状态，在执行逻辑之前通过检查对象的状态避免网络请求的若干执行步骤发生错乱。首先来看状态的定义：

```
  private static final int STATE_IDLE = 0; // Idle connections are ready to write request headers.
  private static final int STATE_OPEN_REQUEST_BODY = 1;
  private static final int STATE_WRITING_REQUEST_BODY = 2;
  private static final int STATE_READ_RESPONSE_HEADERS = 3;
  private static final int STATE_OPEN_RESPONSE_BODY = 4;
  private static final int STATE_READING_RESPONSE_BODY = 5;
  private static final int STATE_CLOSED = 6;
```
下面是Http1Codec的属性域：
```
  /** The client that configures this stream. May be null for HTTPS proxy tunnels. */
  final OkHttpClient client;
  /** The stream allocation that owns this stream. May be null for HTTPS proxy tunnels. */
  final StreamAllocation streamAllocation;

  final BufferedSource source;
  final BufferedSink sink;
  int state = STATE_IDLE;
```
这些属性域很容易理解，首先持有client，可以使用它所提供的功能，通常是获取一些用户所设置的属性，其次是streamAllocation，它是连接与流的桥梁，所以很容易理解需要它获取关于连接的功能。然后就是该流对象封装的输出流和输入流，两个流内部封装的自然就是socket的了。最后就是对象所处的状态。介绍完属性域以后我们就可以分步骤分析Http1Codec提供的功能了，在这些步骤中，逻辑很明确，首先检查状态，然后执行逻辑，最后更新状态，当然执行逻辑和更新状态是可以交换的，不会造成影响，这步骤分析中我们不再考虑状态的问题重点只是关系逻辑的执行。
-1. 首先第一步，写请求头：
```
  @Override public void writeRequestHeaders(Request request) throws IOException {
    String requestLine = RequestLine.get(
        request, streamAllocation.connection().route().proxy().type());
    writeRequest(request.headers(), requestLine);
  }

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
```
执行逻辑很清晰，可以分成两部分，对应Http协议，即写入请求行和请求头，至于请求行的获取有兴趣的同学可以自行查看源码。
-2. 接着，第二步，写请求体，这一步中Http1Codec提供一个包装了sink的输出流，也是一个Sink， 这里我们看是如何封装sink的：
```
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
```
属性Http协议的同学都知道其实请求体和响应体可以分成固定长度和非固定长度两种，其中非固定长度由头部信息中Transfer-Encoding=chunked来标识，固定长度则有对应的头部信息标识实体信息的对应长度。这里我们以非固定长度为例分析Http1Codec是如何封装sink的，对于固定长度的也是类似的逻辑。

```
  public Sink newChunkedSink() {
    if (state != STATE_OPEN_REQUEST_BODY) throw new IllegalStateException("state: " + state);
    state = STATE_WRITING_REQUEST_BODY;
    return new ChunkedSink();
  }

   private final class ChunkedSink implements Sink {
    ...

    @Override public void write(Buffer source, long byteCount) throws IOException {
      if (closed) throw new IllegalStateException("closed");
      if (byteCount == 0) return;

      sink.writeHexadecimalUnsignedLong(byteCount);
      sink.writeUtf8("\r\n");
      sink.write(source, byteCount);
      sink.writeUtf8("\r\n");
    }

    @Override public synchronized void flush() throws IOException {
      if (closed) return; // Don't throw; this stream might have been closed on the caller's behalf.
      sink.flush();
    }

    @Override public synchronized void close() throws IOException {
      if (closed) return;
      closed = true;
      sink.writeUtf8("0\r\n\r\n");
      detachTimeout(timeout);
      state = STATE_READ_RESPONSE_HEADERS;
    }
  }
```
这里使用一个内部类来封装sink， 这里我们只看其中的三个重要的方法，即write() flush() close()方法，逻辑都很清晰，非固定长度的请求体，都是在第一行写入一段数据的长度，然后在之后写入该段数据，从write()方法中可以看出是讲buffer中的数据写入到sink对象中，如果熟悉okio的执行逻辑，对此应该很容易理解。然后刷新和关闭逻辑则很简单，其中关闭时注意更新状态。
对于固定长度的请求体，其封装sink的逻辑是类似的，其中需要传入一个RemainingLength， 保证写数据结束时保证数据长度是正确的即可，有兴趣的可以查看代码。
-3. 第三步是完成请求的写入，其实这一步其实很简单，只有一行代码，就是执行流的刷新：
```
  @Override public void finishRequest() throws IOException {
    sink.flush();
  }
```
注意这一步是不需要检查状态的，因为此时的状态有可能是STATE_OPEN_REQUEST_BODY（没有请求体的情况）或者STATE_READ_RESPONSE_HEADERS(已经完成请求体写入的情况)。这一步只是刷新流，所以什么情况下都不会造成影响，所以没有必要检查状态，也没有更新状态，保持之前的状态即可。
-4. 第四步读取请求头，这一步的代码我们在前面是见到过的，这里再次贴出，方便查阅，并且没有省略：
```
@Override public Response.Builder readResponseHeaders(boolean expectContinue) throws IOException {
    if (state != STATE_OPEN_REQUEST_BODY && state != STATE_READ_RESPONSE_HEADERS) {
      throw new IllegalStateException("state: " + state);
    }
    try {
      StatusLine statusLine = StatusLine.parse(source.readUtf8LineStrict());
      Response.Builder responseBuilder = new Response.Builder()
          .protocol(statusLine.protocol)
          .code(statusLine.code)
          .message(statusLine.message)
          .headers(readHeaders());

      if (expectContinue && statusLine.code == HTTP_CONTINUE) {
        return null;
      }

      state = STATE_OPEN_RESPONSE_BODY;
      return responseBuilder;
    } catch (EOFException e) {
      // Provide more context if the server ends the stream before sending a response.
      IOException exception = new IOException("unexpected end of stream on " + streamAllocation);
      exception.initCause(e);
      throw exception;
    }
  }
```
可以此时所处的状态有可能为STATE_OPEN_REQUEST_BODY和STATE_READ_RESPONSE_HEADERS两种，然后读取请求行和请求头部信息，并返回响应的Builder。
-5. 第五步为获取响应体：
```
  @Override public ResponseBody openResponseBody(Response response) throws IOException {
    Source source = getTransferStream(response);
    return new RealResponseBody(response.headers(), Okio.buffer(source));
  }
```
在之前的介绍中，我们知道响应Response对象中是封装一个source对象，用于读取响应数据。所以ResponseBody的构建就是需要响应头和响应体两部分即可，响应头在上一部分中已经添加到response对象中了，headers()获取响应头即可。下面分析，如何封装source对象，获取一个对应的source对象，可能有些拗口，如果你熟悉装饰模式，以及okio的结构应该很容易明白。下面看getTransferStream的代码：
```
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

    // Wrap the input stream from the connection (rather than just returning
    // "socketIn" directly here), so that we can control its use after the
    // reference escapes.
    return newUnknownLengthSource();
  }
```
这里和写入请求体的地方十分类似，响应体也是分为固定长度和非固定长度两种，除此以外，为了代码的健壮性okhttp还定义了UnknownLengthSource，这里我们不对该意外情况分析，下面我们以固定长度为例分析source的封装。
```
  private class FixedLengthSource extends AbstractSource {
    private long bytesRemaining;

    public FixedLengthSource(long length) throws IOException {
      bytesRemaining = length;
      if (bytesRemaining == 0) {
        endOfInput(true);
      }
    }

    @Override public long read(Buffer sink, long byteCount) throws IOException {
      if (byteCount < 0) throw new IllegalArgumentException("byteCount < 0: " + byteCount);
      if (closed) throw new IllegalStateException("closed");
      if (bytesRemaining == 0) return -1;

      long read = source.read(sink, Math.min(bytesRemaining, byteCount));
      if (read == -1) {
        endOfInput(false); // The server didn't supply the promised content length.
        throw new ProtocolException("unexpected end of stream");
      }

      bytesRemaining -= read;
      if (bytesRemaining == 0) {
        endOfInput(true);
      }
      return read;
    }

    @Override public void close() throws IOException {
      if (closed) return;

      if (bytesRemaining != 0 && !Util.discard(this, DISCARD_STREAM_TIMEOUT_MILLIS, MILLISECONDS)) {
        endOfInput(false);
      }

      closed = true;
    }
  }
```
这里可以看到有一个成员变量bytesRemaining标识剩余的字节数，保证读取到的字节长度与头部信息中的长度保证一致。read()中的代码可以看到就是将该source对象的数据读取到封装的source中，用于构建ResponseBody。
这里需要注意在本篇文章中，我们不断地提到sink，source对象以及封装的sink和source对象，这里解释一下，前面两个代表http1Codec对象中的sink和source对象，即封装了socket的输出和输入流，而封装的sink和source对象则是构建的固定长度和非固定长度的输出输入流，其实它们只是对http1Codec成员变量中的sink和source的一种封装，其实就是装饰模式，封装以后的sink和source对象可以用于在外部写请求体和构建ResponseBody。
这里代码逻辑很清晰，不再详细介绍，相信通过阅读代码都可以明白，不过这里需要提及一下endOfInput()方法：
```
protected final void endOfInput(boolean reuseConnection) throws IOException {
      if (state == STATE_CLOSED) return;
      if (state != STATE_READING_RESPONSE_BODY) throw new IllegalStateException("state: " + state);

      detachTimeout(timeout);

      state = STATE_CLOSED;
      if (streamAllocation != null) {
        streamAllocation.streamFinished(!reuseConnection, Http1Codec.this);
      }
    }
```
这里的执行逻辑也不多，除去检查状态和更新状态之外，就是接触超时机制，最后需要注意就是调用streamAllocation的streamFinished()方法，该方法的参数包括连接是否可以继续使用，以及流对象本身，该方法用于连接和流的清理以及资源回收工作。在上一篇介绍连接的文章中，由于对流不熟悉，所以对这一部分介绍的不清楚，下一小节中则对该部分内容整体学习一下。
至此再去看Http1Codec的代码，基本上已经全部覆盖，没有分析到的其实就剩下几个关于Sink和Source的内部类，都分为固定长度和非固定长度，有兴趣的可自行查看源码。除此以外还有一个cancel()方法，如下：
```
  @Override public void cancel() {
    RealConnection connection = streamAllocation.connection();
    if (connection != null) connection.cancel();
  }
```
这里可以看出就是调用该流对应的cancel方法，关于这一点在下一小节中统一分析讲述。

## 3. Okhttp中连接与流的资源回收与清理
经过前面的学习，我们知道在okhttp中关于连接和流，有三个重要的类，即RealConnection, StreamAllocation和Http1Codec，这里我们只是分析Http1.1协议，所以这三个类是一一对应的，即在一个连接上分配一个流对象，通过该流对象完成最终的网络请求。而完成一次网络请求可能会有成功和失败两种可能，同时也还会有在中途中断的可能，针对以上情况，当我们建立连接获取到socket对象，并在该连接之上建立请求之后，无论那一种情况发生，都需要资源的回收和清理工作，而StreamAllocation作为连接和流的桥梁，自然承担该项工作的主要责任。
我们知道一次请求包括成功，失败和中断等几种情况，在StreamAllocation中对资源回收和清理时，最终都会调用deallocate()方法，这里我们首先看该方法，然后自底向上分析：

```
private Socket deallocate(boolean noNewStreams, boolean released, boolean streamFinished) {
    assert (Thread.holdsLock(connectionPool));

    //1. 修改属性，置状态
    if (streamFinished) {
      this.codec = null;
    }
    if (released) {
      this.released = true;
    }
    Socket socket = null;
    if (connection != null) {
      if (noNewStreams) {
        connection.noNewStreams = true;
      }
      
      if (this.codec == null && (this.released || connection.noNewStreams)) {
        //2. 清理该StreamAllocation对象在Connection中的引用
        release(connection);
        //3. 执行清理逻辑
        if (connection.allocations.isEmpty()) {
          connection.idleAtNanos = System.nanoTime();
          if (Internal.instance.connectionBecameIdle(connectionPool, connection)) {
            socket = connection.socket();
          }
        }
        connection = null;
      }
    }
    //4. 返回需要关闭的socket对象
    return socket;
  }
```
这里首先介绍三个参数的含义，noNewStreams代表该对象对应的连接对象不再可用了，released表示可以将该streamAllocation对象置为可以释放的状态了，streamFinished表示该对象对应的流已经完成了任务，或成功或失败，就是可以将codec置为空了。
下面开始分析它的逻辑，分成四步，已在注释中说明。首先是根据三个参数分别置不同的属性状态，其中released是代表streamAllocation的状态，noNewStreams代表连接对象的状态。然后是根据不同的状态执行清理工作，这里需要注意在判断条件时是使用的成员变量而不是参数，也就是各对象目前所处的状态，如果流对象被置空(说明完成网络请求或者请求失败)，并且该对象处于可以释放的状态或者连接不可用了（可能是因为中断时设置的状态，流成功或失败结束时也会设置状态，具体后面分析），此时就将连接置为空闲状态，并在必要时返回可以关闭的socket对象（这里所说的必要时，是有ConnectionPool对连接维护的逻辑决定的，可以参考上一篇对连接描述的文章）。在熟悉了这个流程以后，我们就可以分成四种情况（即成功，失败，中断，取消），分别分析它们对应的过程。至于第三步清理引用，即release()方法，可以自行查看代码，较为简单，其实就是将自己从connection中的streamAllocation列表中移除。

#### 1. 成功的网络请求
在上一小节的最后我们介绍了endOfInput()方法，即在读取响应体结束以后调用，此时是完成了网络请求，执行清理工作，此时调用了StreamAllocation的finishStreamed()方法，代码如下：
```
  public void streamFinished(boolean noNewStreams, HttpCodec codec) {
    Socket socket;
    synchronized (connectionPool) {
      if (codec == null || codec != this.codec) {
        throw new IllegalStateException("expected " + this.codec + " but was " + codec);
      }
      if (!noNewStreams) {
        connection.successCount++;
      }
      socket = deallocate(noNewStreams, false, true);
    }
    closeQuietly(socket);
  }
```
这里的noNewStreams依然标识对应的该连接是否可以继续使用，然后调用上面的deallocate()方法，表示网络请求过程已经完成，会置空codec对象，但是并没有设置released状态，如果连接依然可用，并且此时streamAllocation并不是released的状态，此时并不会将连接置为空闲状态，还可以在该连接上继续分配新的流对象，完成新的网络请求，只不过请求的地址需要是完全相同的。如果是读取响应实体时发生的错误，此时endOfInput方法传入false，也就是对应noNewStreams参数，此时就会将连接置为空闲状态，并在必要时关闭socket（这里将请求成功，但是读取失败归到了成功完成网络请求的类别中）。除此以外，网络请求成功时，还有可能在响应头部信息中connection设置为close属性，此时需要关闭连接，此时也是调用的该方法，noNewStreams参数也为true，在CallServerInterceptor分析的最后提到过该逻辑。
#### 2. 失败的网络请求
在网络请求失败后，即在RetryAndFollowUpInterceptor中会调用StreamAllocation的streamFailed()方法，该方法的代码如下：
```
  public void streamFailed(IOException e) {
    Socket socket;
    boolean noNewStreams = false;

    synchronized (connectionPool) {
      if (e instanceof StreamResetException) {
        ...
      } else if (connection != null
          && (!connection.isMultiplexed() || e instanceof ConnectionShutdownException)) {
        noNewStreams = true;

        // If this route hasn't completed a call, avoid it for new connections.
        if (connection.successCount == 0) {
          if (route != null && e != null) {
            routeSelector.connectFailed(route, e);
          }
          route = null;
        }
      }
      socket = deallocate(noNewStreams, false, true);
    }

    closeQuietly(socket);
  }
```
在RetryAndFollowUpInterceptor中调用该方法时，参数为null， 所以这里我们先看参数为null的情况，此时连接不可用，所以noNewStreams参数为true，这里还有一个逻辑就是此时检查一下该连接的成功次数，这个值在完成一次网络请求时并且该链接不关闭时会加一，如果这个连接从来都没有成功过，那么就要将它加入到黑名单中，这个黑名单在ConnectionPool的维护连接的逻辑中会用到，具体可以参考上一篇文章。接着调用deallocation()方法，并在必要时关闭socket。对于该方法的参数不为空时，并且异常是StreamResetException的子类时，这里暂且不分析，不明白其中的原理，也没有找到在哪里调用的该方法，有清楚的可以在评论中告知，不胜感激。

#### 3. 中断的网络请求
在RetryAndFollowUpInterceptor中，网络请求不断重试或重定向，如果不可重试或重定向，此时都需要将StreamAllocation置为可以释放的状态，此时会调用StreamAllocation的方法release()方法，该方法的代码如下：
```
public void release() {
    Socket socket;
    synchronized (connectionPool) {
      socket = deallocate(false, true, false);
    }
    closeQuietly(socket);
  }
```
可以看到该方法很简单，只是调用deallocate方法，release为true，它设置StreamAllocation为可释放状态，如果没有流对象或者连接不可用时，就开始回收资源，必要时关闭socket。

#### 4. 取消的网络请求
我们都知道在okhttp中，对网络请求的封装是使用的Callu对象，该对象有cancel()方法，也就是可以在任意时候取消一次网络请求，下面我们就从Call的cancel()方法开始学习如何取消一次网络请求
Call.cancel():
```
 @Override public void cancel() {
    retryAndFollowUpInterceptor.cancel();
  }
```
可见是调用了RetryAndFollowUpInterceptor的cancel()方法，该方法的代码为：
```
  public void cancel() {
    canceled = true;
    StreamAllocation streamAllocation = this.streamAllocation;
    if (streamAllocation != null) streamAllocation.cancel();
  }
```
修改成员变量，并调用StreamAllocation()的cancel()方法(关于这里的赋值，使用局部变量的意图还不太明白），这里终于调用到了StreamAllocationd 取消方法，下面看其cancel()方法：
```
public void cancel() {
    HttpCodec codecToCancel;
    RealConnection connectionToCancel;
    synchronized (connectionPool) {
      canceled = true;
      codecToCancel = codec;
      connectionToCancel = connection;
    }
    if (codecToCancel != null) {
      codecToCancel.cancel();
    } else if (connectionToCancel != null) {
      connectionToCancel.cancel();
    }
  }
```
逻辑可以总结为如果有可以取消的流则取消流，没有则取消连接，其实我们在前面介绍Http1Codec的cancel()方法中也看到了它也是调用的对应的连接对象的cancel()方法，下面我们来看connection的cancel()方法：
```
  public void cancel() {
    // Close the raw socket so we don't end up doing synchronous I/O.
    closeQuietly(rawSocket);
  }
```
关闭原始的socket对象，简单直接。关闭socket对象以后，所有的I/O逻辑都会抛出一场，因此取消了一次网络请求，此时需要手动关闭一些建立的流对象。

到这里我们就分析完了StreamAllocation在回收和清理资源方面所做的工作。总结起来有四个可以调用的方法，其中前三个最后对会调用deallocate方法去处理连接和流对象，而cancel()方法则是直接关闭了socket对象。如果在这里还不太明白，比较难理解的是中间的两个，其实可以结合RetryAndFollowUpIntercepter理解，中间的两个方法中在重试拦截器中被多次调用，这与具体的网络请求逻辑相关，关于RetryAndFollowUpInterceptor在前一篇文章中分析较少，因此下一小节会对其做一次全面的分析学习，从而也可以更好地辅助理解StreamAllocation对清理和回收所做的工作。

## 4. 再谈RetryAndFollowUpInterceptor
对于拦截器， 我们还是看其intercept()方法：
```
 @Override public Response intercept(Chain chain) throws IOException {
    Request request = chain.request();

    streamAllocation = new StreamAllocation(
        client.connectionPool(), createAddress(request.url()), callStackTrace);

    int followUpCount = 0;
    Response priorResponse = null;
    while (true) {
      //1. 判断是否为取消状态
      if (canceled) {
        streamAllocation.release();
        throw new IOException("Canceled");
      }

      Response response = null;
      boolean releaseConnection = true;
      try {
        response = ((RealInterceptorChain) chain).proceed(request, streamAllocation, null, null);
        releaseConnection = false;
      } catch (RouteException e) {
        //2. 出现异常状况
        // The attempt to connect via a route failed. The request will not have been sent.
        if (!recover(e.getLastConnectException(), false, request)) {
          throw e.getLastConnectException();
        }
        releaseConnection = false;
        continue;
      } catch (IOException e) {
        // An attempt to communicate with a server failed. The request may have been sent.
        boolean requestSendStarted = !(e instanceof ConnectionShutdownException);
        if (!recover(e, requestSendStarted, request)) throw e;
        releaseConnection = false;
        continue;
      } finally {
        //3. 最后判断是否需要执行清理
        // We're throwing an unchecked exception. Release any resources.
        if (releaseConnection) {
          streamAllocation.streamFailed(null);
          streamAllocation.release();
        }
      }
      ...
    }
  }
```
这里由于该方法较长，我将他们分成了两个部分，前半部分是针对重试请求的处理，后半部分是针对重定向请求的处理。我们还是首先看该方法的前半部分，这里我将前半部分划分成了三步，首先是判断该请求是否被取消了，如果被取消了，此时有两种可能，一是还没有流对象，此时调用release()方法，置为可释放状态，终止网络请求，并将在条件满足时将连接设置为空闲状态。第二种可能是已经有流对象，此次网络请求就没有办法终止，此时只是设置可释放状态，等网络请求结束以后将流对象置空，此时可以立刻在条件满足时将连接置为空闲状态。
如果没有取消，则开始执行网络请求，这里也分成两种可能，一是网络请求出现意外状态，即抛出异常，此时需要判断是否可以重试，判断条件可以自行查看，这里不详细说明，但是在判断方法中，调用了streamAllocation的streamFailed()方法，并传递了Exception。如果可以重试，此时的releaseConnection为false, finnally中不执行任何逻辑，此时会执行下一次循环，如果不可以重试，则执行第三步，及finnally中的逻辑，即调用StreamAllocation的失败方法，此时会将流对象置空，同时调用release()方法，由于codec已经为空，release()方法调用完以后则会在条件满足时将connection置为空闲状态，然后由于抛出异常，此时会跳出while循环。
上面说如果没有取消，还会有另外一种情况，即正确完成网络请求，此时处理重定向情况，直接去执行该方法的第二部分，下面是第二部分代码，注意这里都是在while()循环之中：
```
...
      Request followUp = followUpRequest(response);

      //没有重定向或重试情况
      if (followUp == null) {
        if (!forWebSocket) {
          streamAllocation.release();
        }
        return response;
      }

      //有重定向情况
      closeQuietly(response.body());

      //两种不可以重定向的情况
      if (++followUpCount > MAX_FOLLOW_UPS) {
        streamAllocation.release();
        throw new ProtocolException("Too many follow-up requests: " + followUpCount);
      }

      if (followUp.body() instanceof UnrepeatableRequestBody) {
        streamAllocation.release();
        throw new HttpRetryException("Cannot retry streamed HTTP body", response.code());
      }

      //对于不同的连接，释放原来的streamAllocation, 建立新的streamAllocation
      if (!sameConnection(response, followUp.url())) {
        streamAllocation.release();
        streamAllocation = new StreamAllocation(
            client.connectionPool(), createAddress(followUp.url()), callStackTrace);
      } else if (streamAllocation.codec() != null) {
        throw new IllegalStateException("Closing the body of " + response
            + " didn't close its backing stream. Bad interceptor?");
      }

      //设置属性，开始重试，执行下一次循环
      request = followUp;
      priorResponse = response;
    }
  }
```
首先获取重定向请求对象，对于不需要重定向的请求，直接返回相应对象，结束循环。如果需要重定向则需要关闭上一个响应对象的响应体，此时调用body的close()方法，最终对调用streamAllocation的streamFinished()方法，将其上的codec对象置空。然后判断两种不可以重定向的情况，抛出异常结束循环。如果可以执行重定向的话，对于不同连接则需要释放原来的streamAllocation，并建立新的streamAllocation，而对于相同连接直接使用即可，即在相同连接上建立新的流，这里判断了原来的codec对象一定置为空了。最后设置新的请求对象，再次执行循环，完成重定向的请求。

至此较为完整地分析了RetryAndFollowUpInterceptor的代码，这里需要注意重试和重定向没有任何关系，这里只是写到一个拦截器中，放在一个无限循环中而已。通过这一段代码的分析也可以更好地了解到StreamAllocation在资源管理中，回收和清理工作的几个函数的作用。不过由于水平有限，对于recover()函数，以及抛出的各种异常类型没有深入分析，StreamAllocation的streamFailed()方法也没有分析透彻，有了解的欢迎一起探讨。不过这里已经不影响对整体流程的分析。

## 5. 总结
本篇文章主要是介绍了Okhttp流的概念，以及CallServerInterceptor是如果使用HttpCodec提供的功能完成网络请求的数据通信。在介绍完流的概念以后，结合之前的连接，重新对StreamAllocation中资源的回收和清理做了学习和分析，主要介绍了四个可以调用的方法，分别处理不同的情况，最后重新介绍了RetryAndFollowUpInterceptor的执行逻辑，以及在执行重试或重定向时是如何调用StreamAllocation的清理方法的。
通过三篇文章，我们已经大概分析了okhttp中一次网络请求的大致过程。从Call对象对请求的封装，到使用dispatcher对请求的分发，再到执行请求时调用getResponseWithInterceptors()方法获取请求，最后说明这个拦截器链的递归调用结构。在这个拦截器链中，RetryAndFollowUpInterceptor负责重试和重定向，ConnectionInterceptor负责建立连接和流对象，CallServerInterceptor负责完成最终的网络请求，以上就是几乎整个的网络请求过程。在拦截器链中还有两个拦截器没有介绍，其中较为简单的是BridgeInterceptor，它主要负责okhttp中请求和响应对象与实际Http协议中定义的请求和响应之间的转换，以及处理cookie相关的内容，另一个是个CacheInterceptor，顾名思义也就是用来处理缓存的拦截器，接下来将会分成两篇文章分别介绍它们的功能。