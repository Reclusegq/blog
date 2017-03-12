# okhttp 学习笔记（综述）
关于okhttp是一款优秀的网络请求框架，关于它的源码分析文章有很多，这里分享我在学习过程中读到的感觉比较好的文章可以做参考，本系列的文章是在学习okhttp源码过程中的笔记，记录一是为了总结知识，二是为了分享学习过程，其中有错误和欠缺之处，还请不吝批评指正。

1. [拆轮子系列：拆 OkHttp](https://blog.piasy.com/2016/07/11/Understand-OkHttp/)
2. [带你学开源项目：OkHttp--自己动手实现okhttp](http://wingjay.com/2016/07/21/%E5%B8%A6%E4%BD%A0%E5%AD%A6%E5%BC%80%E6%BA%90%E9%A1%B9%E7%9B%AE%EF%BC%9AOkHttp-%E8%87%AA%E5%B7%B1%E5%8A%A8%E6%89%8B%E5%AE%9E%E7%8E%B0okhttp/)
3. [OkHttp3源码分析](http://www.jianshu.com/p/aad5aacd79bf)

okhttp是一个网络请求框架，不仅仅可以用于Android应用中。在okhttp之前，Android中有不少的优秀网络请求框架，比如HttpClient，Volley等，而okhttp虽然与这些框架完成相同的事情，但是与之存在本质的不同，前者都是对Java中的UrlConnection进行封装，而okhttp则是直接对socket进行封装，也就是说他们所处的层次不同。除此以外，okhttp在各种功能属性和性能方面都有很大的优势，关于这些可以参考以上几篇文章，这里直接进入正题介绍okhttp的实现。

网络请求所要完成的基本问题无非就是实现客户端请求获取服务器的资源，完成客户端与服务器的通信，简单来说，okhttp就是在特定环境下(这里可以说移动客户端与服务器的通信）实现Http协议。那么这里首要问题就是解决如何构建请求Request，如何获取相应Response，然后当我们拥有了request，response， 以及系统为我们提供的socket通信机制以后，就需要想办法解决如何发送request和接收response，也就是实现http协议所规定的细节流程。在request到response的转换过程中，http协议涉及到 了建立连接，维护连接（连接复用），缓存处理以及cookie处理等问题。在请求的过程中，存在重定向等问题，一次请求可能涉及到多次的请求操作，同时还会有多次请求同一个服务器上的资源的情况，这里就涉及到连接复用的问题。另外也更多地存在多次请求相同资源的情况，这也就涉及到缓存处理的情况。此外还有cookie持久化的问题。除此以外，okhttp还支持https协议，同时也支持SPDY和http2协议，因此计划将分为请求和响应的构建与转换，连接和流管理，缓存管理，cookie持久化几部分介绍我对okhttp框架的学习过程。在这个过程中完全忽略对https以及SPDY和Http2的考虑，在后续的学习中将会补充这两部分的描述。

本篇文章主要介绍网络请求的整个过程，而不过于深究一些细节，首先会简单介绍request和response的构建， 以及Okhttp引入的Call类，该类是对request的封装，okhttp之所以对请求再次封装，（仅仅是个人见解）一是为了分离职责，将封装请求与执行请求过程的操作（如同步与异步，终止或者取消等操作）分离，二是因为，http请求经常会遇到重定向等问题，因此一次请求过程会涉及多次请求，而okhttp则倾向于将这多次请求使用同一个连接完成，这也就引入了将会在第二篇文章中重点介绍的call, streamAllocation, connection三个概念，具体原因则在第二篇文章中详细分析。同时由于网络请求通常都会是异步操作，请求的过程将会在不同线程中执行，因此在第二部分中会重点介绍okhttp中的请求任务队列以及调度机制。最后在第三部分则会介绍okhttp中网络请求的执行流程，这里主要涉及到intercepter与chain之间的递归调用，从而使得不同的拦截器完成对请求以及响应的处理，最后实现后续文章中重点介绍的连接，缓存管理等机制。通过这三部分就可以从整体上把握okhttp时如何完成请求与响应的构建，以及他们的转换操作。

## 1. 请求与响应的封装

### Request
如果熟悉Http协议就会知道，请求报文由请求头和请求体组成，那么Request则是请求报文在Java世界中的存在形式，对于其代码，我们只需要了解其内部结果就够了，如下：

```
/**
 * An HTTP request. Instances of this class are immutable if their {@link #body} is null or itself
 * immutable.
 */
public final class Request {
  final HttpUrl url;
  final String method;
  final Headers headers;
  final RequestBody body;
  final Object tag;

  ...
 }
```
这里的tag是一个请求的标识，可以在后续的过程中终止或取消这个请求，除此以外，url和method都明白为请求url和请求方法，而其他的头部信息则以headers的形式统一保存，而Headers类是一个工具类，统一管理请求和响应的头部信息，其内部结构就是一个nameAndValue的字符串数组保存头部信息，保存形式为一个键值紧跟一个value值， 熟悉Android中的ArrayMap数据结构的同学应该对这种实现方式比较了解，其优势可以参考ArrayMap的介绍文章，Headers类是一个工具类，维护一个nameAndValue并对外提供各种方法（与HashMap类似，只是占用内存更少，而查询效率略低），具体代码有兴趣的同学可以自行查看。最后就是请求体，下面为RequestBody的代码：

```
public abstract class RequestBody {
  /** Returns the Content-Type header for this body. */
  public abstract MediaType contentType();

  /**
   * Returns the number of bytes that will be written to {@code out} in a call to {@link #writeTo},
   * or -1 if that count is unknown.
   */
  public long contentLength() throws IOException {
    return -1;
  }

  /** Writes the content of this request to {@code out}. */
  public abstract void writeTo(BufferedSink sink) throws IOException;
...
}
```
RequestBody为一个抽象类，实现类中必须指定contentType， 即请求体的报文类型，至于具体类型有哪些，可以查看http协议的相关内容，然后需要指定请求体需要写入的内容，即在writeTo方法中实现，这里的Sink和后面遇到的Source是属于Okio的内容，也就是square公司封装的一个IO框架，暂时可以将他们分别理解为OutputStream和InputStream。该方法会在建立连接以后调用，如request.requsetBody().writeTo(Okio.sink(socket.outputStream()))， 类似与这种形式，从而完成将请求体内容写入到socket中。同时为了发送字符串和文件内容方便，RequestBody还提供了静态工具方法，create的多种重载形式，具体可以自行参考源码，其实就是复写上述的三个方法而已。

最后，请求的构建过程参数角度，其实自然也就能想到这里会使用建造者模式，通过Builder设置不同参数，熟悉Okhttp使用的同学，对此应该较为熟悉，这里不再介绍。Response的封装与之类似，但是获取响应必然与缓存有关，这里先省略与缓存相关的部分（将在第三篇文章中介绍）， 只是简单介绍一些响应的获取构建以及响应体的内容提取。

### Response

```
/**
 * An HTTP response. Instances of this class are not immutable: the response body is a one-shot
 * value that may be consumed only once and then closed. All other properties are immutable.
 *
 * <p>This class implements {@link Closeable}. Closing it simply closes its response body. See
 * {@link ResponseBody} for an explanation and examples.
 */
public final class Response implements Closeable {
  final Request request;
  final int code;
  final String message;
  final Handshake handshake;
  final Headers headers;
  final ResponseBody body;

  Response(Builder builder) {
  ...
  }
  ...
}
```
这里省略了所有与缓存相关的代码之后，结构与请求几乎完全一致，统一它也是由Builder构造，包括头部信息和响应体，另外Http协议中规定了响应码以及响应信息都有对应的字段属性，另外需要注意的是注释中说明的响应体只能使用一次，并且使用完毕以后需要关闭，Response实现的Closeable接口，其内部也是调用的ResponseBody的close方法。关于Response在缓存管理中将会重点介绍，这里只是与request对应做一下简单了解即可。下面看ResponseBody的代码：
```
public abstract class ResponseBody implements Closeable {
  /** Multiple calls to {@link #charStream()} must return the same instance. */
  private Reader reader;

  public abstract MediaType contentType();

  /**
   * Returns the number of bytes in that will returned by {@link #bytes}, or {@link #byteStream}, or
   * -1 if unknown.
   */
  public abstract long contentLength();

  ...

  public abstract BufferedSource source();
...
}
```
同样一个抽象类，内部包括三个方法，只不过与RequestBody中对应的writeTo()不同的是，这里为source()方法，即返回一个输入流，预计在建立连接中会有response.body().source(Okio.source(socket.inputStream()))之类的调用，完成响应体的获取，从而输入一个BufferedSource，可以将其简单理解为一个输入流，通过操作它，我们进而可以获取inputStream, Reader, String, byte[]等， ResponseBody都有提供对应的方法，其实现方式可以自行查看源码，这里不再详细介绍。不过需要仔细阅读的是这个类的注释，如下：
```
/**
 * A one-shot stream from the origin server to the client application with the raw bytes of the
 * response body. Each response body is supported by an active connection to the webserver. This
 * imposes both obligations and limits on the client application.
 *
 * <h3>The response body must be closed.</h3>
 *
 * Each response body is backed by a limited resource like a socket (live network responses) or
 * an open file (for cached responses). Failing to close the response body will leak resources and
 * may ultimately cause the application to slow down or crash.
 *
 * <p>Both this class and {@link Response} implement {@link Closeable}. Closing a response simply
 * closes its response body. If you invoke {@link Call#execute()} or implement {@link
 * Callback#onResponse} you must close this body by calling any of the following methods:
 *
 * <ul>
 *   <li>Response.close()</li>
 *   <li>Response.body().close()</li>
 *   <li>Response.body().source().close()</li>
 *   <li>Response.body().charStream().close()</li>
 *   <li>Response.body().byteString().close()</li>
 *   <li>Response.body().bytes()</li>
 *   <li>Response.body().string()</li>
 * </ul>
 *
 * <p>There is no benefit to invoking multiple {@code close()} methods for the same response body.
 *
 * <p>For synchronous calls, the easiest way to make sure a response body is closed is with a {@code
 * try} block. With this structure the compiler inserts an implicit {@code finally} clause that
 * calls {@code close()} for you.
 *
 * <pre>   {@code
 *
 *   Call call = client.newCall(request);
 *   try (Response response = call.execute()) {
 *     ... // Use the response.
 *   }
 * }</pre>
 *
 * You can use a similar block for asynchronous calls: <pre>   {@code
 *
 *   Call call = client.newCall(request);
 *   call.enqueue(new Callback() {
 *     public void onResponse(Call call, Response response) throws IOException {
 *       try (ResponseBody responseBody = response.body()) {
 *         ... // Use the response.
 *       }
 *     }
 *
 *     public void onFailure(Call call, IOException e) {
 *       ... // Handle the failure.
 *     }
 *   });
 * }</pre>
 *
 * These examples will not work if you're consuming the response body on another thread. In such
 * cases the consuming thread must call {@link #close} when it has finished reading the response
 * body.
 *
 * <h3>The response body can be consumed only once.</h3>
 *
 * <p>This class may be used to stream very large responses. For example, it is possible to use this
 * class to read a response that is larger than the entire memory allocated to the current process.
 * It can even stream a response larger than the total storage on the current device, which is a
 * common requirement for video streaming applications.
 *
 * <p>Because this class does not buffer the full response in memory, the application may not
 * re-read the bytes of the response. Use this one shot to read the entire response into memory with
 * {@link #bytes()} or {@link #string()}. Or stream the response with either {@link #source()},
 * {@link #byteStream()}, or {@link #charStream()}.
 */
 ```
 虽然有点长，但是只是说明了两个问题，一是每个ResponseBody的背后都是一个socket或者文件资源作为数据后备，因此在使用完以后需要关闭资源，不然会造成浪费，二是，ResponseBody应该只能使用一次， 因为不能将响应完全缓存到内存中，因此需要我们通过byte()或者string一次性将响应体加载到内存使用，或者source()和byteStream()方法以流的方式使用，使用完成以后需要关闭，不能第二次使用。除此以外我们还需要注意的是，网络通信都是字节流，那么就会涉及到字符编解码的问题，即charset问题， 这个问题这里不再介绍，Java的I/O库对此有处理，Okio同样也有处理，有兴趣的可以自行查看。

### Call
在简单介绍完请求与响应之后，就需要介绍在使用Okhttp过程中经常遇到的Call类了，它是对reques的封装，也是对一次请求过程的封装，因为从call中我们可以获取响应，因此可以将其看做是一次请求过程的封装，期间涉及到的重定向等问题对于用户即使用okhttp的程序员来说是透明的，这也是我们喜欢它的原因。同时在第二篇文章，连接和流的管理中，流的概念还会涉及到call, 到时还会详细介绍，这里首先看Call的接口代码：
```
/**
 * A call is a request that has been prepared for execution. A call can be canceled. As this object
 * represents a single request/response pair (stream), it cannot be executed twice.
 */
public interface Call extends Cloneable {

  Request request();

  void enqueue(Callback responseCallback);

  void cancel();

  boolean isExecuted();

  boolean isCanceled();

  /**
   * Create a new, identical call to this one which can be enqueued or executed even if this call
   * has already been.
   */
  Call clone();

  interface Factory {
    Call newCall(Request request);
  }
}
```
这里首先看接口，省去了注释时为了代码长度不会过长，不过还是建议阅读注释，从接口方法中可以清楚地看到它是封装的请求，可以获取它所封装的请求，同时提供同步和异步的执行方法， 最后是可以取消或复制，以及判断它的状态的方法，最后有一个工厂类，可以实例化Call, 实现工厂接口的类中我们最常用的就是OkHttpClient， 在Okhttp的使用介绍文章中通常都会重点介绍该类，可以通过它完成参数配置，后续我们也会不断接触到它，这里它实现工厂接口，就可以实例化Call对象。下面我们再分析一下在okhttp中Call的唯一实现类，RealCall， 其代码如下:

```
final class RealCall implements Call {
  final OkHttpClient client;
  final RetryAndFollowUpInterceptor retryAndFollowUpInterceptor;

  /** The application's original request unadulterated by redirects or auth headers. */
  final Request originalRequest;
  final boolean forWebSocket;

  // Guarded by this.
  private boolean executed;

  RealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    this.client = client;
    this.originalRequest = originalRequest;
    this.forWebSocket = forWebSocket;
    this.retryAndFollowUpInterceptor = new RetryAndFollowUpInterceptor(client, forWebSocket);
  }

  @Override public Request request() {
    return originalRequest;
  }

  //同步执行方法
  @Override public Response execute() throws IOException {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    try {
     //仅仅是标识该call对象已经开始执行
      client.dispatcher().executed(this);
      //真正地去执行网络请求
      Response result = getResponseWithInterceptorChain();
      if (result == null) throw new IOException("Canceled");
      return result;
    } finally {
      //标识该call对象的请求过程结束
      client.dispatcher().finished(this);
    }
  }

  ...

 //异步执行方法
  @Override public void enqueue(Callback responseCallback) {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
  }
  ...
 ```
这里对Call的代码做了简化，我们暂且忽略forWebsocket字段，以及callStackTrace，重点关注同步以及异步执行方法，需要我们注意的是在代码中添加的注释部分，即同同步方法网络请求是通过getResponseWithInterceptorChain方法获取的响应， 涉及到dispatcher的部分只是设置标志位而已（为了与异步执行统一处理，我们将在本篇文章的第二部分介绍dispatcher部分中介绍），同样异步执行也是将执行的getResponseWithInterceptorChain方法获取的网络请求，该方法将在本篇文章的第三部分重点介绍。Call接口的取消操作都交由retryAndFollowUpInterceptor来处理，关于该拦截器将会在第二篇文章连接和流管理中介绍。
下面需要介绍的是在异步方法中的可以加入任务队列的Runnable, AsyncCall, 该类时RealCall的一个内部类，虽然叫AsyncCall， 但是是一个异步执行的Runnable，与Call并没有关系， 虽然它也有一个execute方法，但是它其实是Runnable的run方法，与Call的execute方法无关。我在刚开始看代码的时候就被此迷惑，因此还是有必要首先看一下NamedRunnable的代码， AsyncCall实现了该接口。其代码为：

```
/**
 * Runnable implementation which always sets its thread name.
 */
public abstract class NamedRunnable implements Runnable {
  protected final String name;

  public NamedRunnable(String format, Object... args) {
    this.name = Util.format(format, args);
  }

  @Override public final void run() {
    String oldName = Thread.currentThread().getName();
    Thread.currentThread().setName(name);
    try {
      execute();
    } finally {
      Thread.currentThread().setName(oldName);
    }
  }

  protected abstract void execute();
}
```
这里可以看出来其实我们这里的类就是一个Runnable， 只不过它可以为执行的线程命名，同时将run方法的名称修改为了execute()， 就是在run方法中调用execute()， 在Java编程思想一书中，作者也提到run这个名称不好，应该叫做execute()， square公司的工程师应该也有相同见解，不过为了修改线程名称，需要在run方法中操作，而将具体的执行代码留给子类来实现，也只能需要另外起一个方法名称吧。
下面来看AsyncCall的代码如下：

```
final class AsyncCall extends NamedRunnable {
    private final Callback responseCallback;

    AsyncCall(Callback responseCallback) {
      super("OkHttp %s", redactedUrl());
      this.responseCallback = responseCallback;
    }

    ...

    @Override protected void execute() {
      boolean signalledCallback = false;
      try {
        Response response = getResponseWithInterceptorChain();
        if (retryAndFollowUpInterceptor.isCanceled()) {
          signalledCallback = true;
          responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
        } else {
          signalledCallback = true;
          responseCallback.onResponse(RealCall.this, response);
        }
      } catch (IOException e) {
        if (signalledCallback) {
          // Do not signal the callback twice!
          Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
        } else {
          responseCallback.onFailure(RealCall.this, e);
        }
      } finally {
        client.dispatcher().finished(this);
      }
    }
  }

```
从execute方法中可以看出，网络请求同样也是通过getResponseWithInterceptorChain方法获取的， 在获取到响应以后分别处理成功和失败的情况，分别调用回调的响应方法。

通过分析Call的代码，我们也就熟悉了okhttp时如果封装请求来获取响应的，而在大部分情况下网络请求都是异步执行的，对于异步执行，任务就需要管理和调度，因此我们还需要学习一下okhttp中时如何管理请求任务队列以及如何调度网络请求任务的。

## 2. 任务队列的管理与调度

首先明确这一部分是针对异步执行方法来讲的。其实在okhttp的任务管理和调度都是通过Dispatcher类来完成的， 那么我们来想，任务的管理和调度问题，其实应该将AyncCall对象放在一个队列的数据结构中，由于在多线程的程序中可以同时执行多个请求，那么我们需要维护两个数据结构，一个排队等候的队列，一个正在执行的任务集合，另外我们需要一个线程尺执行这些任务。在思考完数据结构以后，再去考虑该类需要设计的方法，首先我们需要一个入队方法，在入队时要么直接去执行要么加入排队等候的队列，其次我们需要一个唤醒排队等候队列的方法，即一个任务在执行结束后需要调用的方法，最后就是在任务执行结束时需要设置标识，方便我们的唤醒方法被调用，那么这个方法我们在上一部分已经见到了，即dispatcher.finishde()方法。在思考完以后，我们来看Dispatcher的代码：
### Dispatcher

```
/**
 * Policy on when async requests are executed.
 *
 * <p>Each dispatcher uses an {@link ExecutorService} to run calls internally. If you supply your
 * own executor, it should be able to run {@linkplain #getMaxRequests the configured maximum} number
 * of calls concurrently.
 */
public final class Dispatcher {
  private int maxRequests = 64;
  private int maxRequestsPerHost = 5;
  private Runnable idleCallback;

  /** Executes calls. Created lazily. */
  private ExecutorService executorService;

  /** Ready async calls in the order they'll be run. */
  private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();

  /** Running asynchronous calls. Includes canceled calls that haven't finished yet. */
  private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();

  /** Running synchronous calls. Includes canceled calls that haven't finished yet. */
  private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();

  public Dispatcher(ExecutorService executorService) {
    this.executorService = executorService;
  }

  public Dispatcher() {
  }

  ...

  synchronized void enqueue(AsyncCall call) {
    if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
      runningAsyncCalls.add(call);
      executorService().execute(call);
    } else {
      readyAsyncCalls.add(call);
    }
  }

  private void promoteCalls() {
    if (runningAsyncCalls.size() >= maxRequests) return; // Already running max capacity.
    if (readyAsyncCalls.isEmpty()) return; // No ready calls to promote.

    for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
      AsyncCall call = i.next();

      if (runningCallsForHost(call) < maxRequestsPerHost) {
        i.remove();
        runningAsyncCalls.add(call);
        executorService().execute(call);
      }

      if (runningAsyncCalls.size() >= maxRequests) return; // Reached max capacity.
    }
  }

  ...

  /** Used by {@code AsyncCall#run} to signal completion. */
  void finished(AsyncCall call) {
    finished(runningAsyncCalls, call, true);
  }

  /** Used by {@code Call#execute} to signal completion. */
  void finished(RealCall call) {
    finished(runningSyncCalls, call, false);
  }

  private <T> void finished(Deque<T> calls, T call, boolean promoteCalls) {
    int runningCallsCount;
    Runnable idleCallback;
    synchronized (this) {
      if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
      if (promoteCalls) promoteCalls();
      runningCallsCount = runningCallsCount();
      idleCallback = this.idleCallback;
    }

    if (runningCallsCount == 0 && idleCallback != null) {
      idleCallback.run();
    }
  }
  ...
}
```
这里对代码做了简化，只留下我们要找的三个方法（当然不只是三个，因为有同类型的重载）。首先来看数据结构，这里我们需要了解的是，okhttp对想同一个主机地址的网络请求也做了上限，因此前两个字段分别是请求个数的上限阈值，然后dispatcher也负责维护同步请求的状态，因此这里也有一个同步请求的队列runningSyncCalls，因为同步请求也是需要时间的，在一个时刻也会有一个队列。下面就是我们想要找的两个队列，一直正在执行的异步请求队列runningAsyncCalls，一个是排队等候队列readyAsyncCalls，最后是真正工作的线程池executorService。
下面我们来看我们想要看到的三个方法， 第一个入队方法， enqueue， 逻辑很简单，如我们所想，如果可以执行则直接执行，否则排队等候，只不过这里的需要考虑的上限不仅有总的请求数目上限还有对于同一个主机的请求上限，那么我们就可以猜测runningCallsForHost方法就是遍历runningSyncCalls队列，找出这个请求对应的主机上的其他请求上限数量，有兴趣的可以自行查看代码。
然后我们再来看第三个方法，即finished()方法，标识一个请求任务的结束，当然这里三种重载形式，很容易看出promoteCalls字段是标识这是否需要唤醒等候队列任务的，同步任务的结束不需要，而异步任务的结束则需要，这一点显而易见，接着就是从队列中移除任务，并调用唤醒方法。不过这里需要注意的是如果正在执行的任务数为0以后会执行空闲任务，在空闲任务中我们可以自行定义当队列空闲的时候需要做的事情，如发出通知，输出log等。
接着就看第二个方法，即唤醒方法， promoteCalls()， 逻辑也很简单，就是遍历排队等候队列，然后执行最靠前的任务，一直达到最大的上限。只不过还是需要判断两个上限。

其实到这里Dispatcher的代码基本上已经分析结束了，剩余的方法要么是工具方法，要么是getter和setter方法。最近尝试阅读一些优秀的开源代码，有一点点的心得体会，不敢说方法论，算是自己的一点总结想在这里插入一段与大家分享讨论，有不对的地方还希望批评指正。其实要说也没有多少深入的道理，只是在想阅读代码的时候，遇到一个类，它必然有自己的功能或是说自己的使命，而且根据单一职责的原则，一个类只能有一个原因引起它的改变，那么这里断章取义，它也应该只有一个功能或者说相近的一类功能。 那么我们就需要先考虑它所要解决的问题，在问题明确以后就需要考虑它需要在内存中维护的数据结构，因为实现功能必然需要维护信息，信息就需要一定的数据结构在内存中保存，在考虑清楚数据结构以后再去思考该类实现它的功能所需要的步骤，就是将问题分解，要么分情况要么分步骤（高中数学中所谓的加法原理和乘法原理），在这些子问题中我们需要设计那些方法来解决子问题。在做出上述思考以后再去阅读优秀框架的源码就会发现简单很多，当我们抽出重点的字段属性以及方法，剩下的属性中基本上都是标志位，剩下的方法中要么是工具方法（我么暂且这么称呼，通常是private的， 为重点的方法提供功能服务）要么就是对外提供的getter和setter方法， 以及对外提供的工具方法。当然虽然我这么说，本人在阅读代码的时候更多的是事后诸葛，在看完代码以后再去思考，不过我认为这样也有助于我们更好地理解开源框架设计的原理，所谓的知其然知其所以然。固然如此，我还是建议并且以后也会尝试去事先思考，其实思考的过程有两个难点，一是明确问题，很多时候我们不明确问题，造成迷失在代码里，同样写代码也是变成代码堆积，没有很好的设计，因此明确问题才是成败的关键，其次时问题的分割，这一点我们可以很好地从源码中学习到经验，他们是如何分解问题，通过几个方法从而实现对问题的处理，相信多次练习思考以及学习以后自己也可以很好地把握分解问题的尺度，很好地通过设计类的结构以及方法拆解来解决我们的问题。最后给一点小小地建议就是大胆地猜测方法的功能（特别是private的功能方法，它通常是提供一种功能服务），自顶向下看待代码，而不要纠结于代码细节。

好了，言归正传，任务队列的管理与调度其实就是Dispatcher一个类来完成，虽然我们只是针对异步任务来讲解，但是它也负责同步任务的维护，如 executed()方法的调用标识任务的开始，finished()方法的调用标识任务结束，具体代码可以自行查阅。在任务被调度执行以后，任务就需要去执行了，也就是请求流程的执行过程。本篇文章的最后一部分对此做介绍，这一部分也是很多文章都会重点介绍的interceptor的调用流程，在okhttp中所有的功能几乎都是通过定义interceptor， 对request和response做操作来实现的，其实就也就是请求流程的执行过程。

## 3. 请求的执行流程
Http协议中，从request转换为我们所想要的respose会涉及到失败重试，重定向，缓存处理，cookie处理等，所以okhttp设计了一系列的interceptor来分别做这些事情，同时设计了一个Chain的类来串联所有的拦截器，最终完成请求的执行流程。本文的第三部分重点介绍此过程。

在第三部分中，我们知道无论是同步请求还是异步请求最后都是通过getResponseWithIntercepters()方法获取响应，那么我们首先来看RealCall#getResponseWithInterceptorChain()方法的代码：
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
 这里还是省略forWebSocket字段，（关于websocket， 我还没有了解过，有兴趣的可以自行学习，这里暂且略过）。代码中很明显就是构建一个拦截器的链表，其中networkIntercepters就是在OkHttpClient中配置的我们自定的拦截器，可以完成我们需要的功能。通过拦截器链表构建一个Chain， 即一个链，完成最终的请求过程，下面我们首先来看接口Chain和Interceptor的代码如下：

 ```
 /**
 * Observes, modifies, and potentially short-circuits requests going out and the corresponding
 * responses coming back in. Typically interceptors add, remove, or transform headers on the request
 * or response.
 */
public interface Interceptor {
  Response intercept(Chain chain) throws IOException;

  interface Chain {
    Request request();

    Response proceed(Request request) throws IOException;

    Connection connection();
  }
}
```
Interceptor只有一个方法，就是接收一个chain, 然后返回响应， 当然从这个chain中我们可以获取请求，对请求可以做处理， 然后调用chain.proceed()方法获取响应，然后再对响应做处理就可以将该响应返回了。而Chain的主要方法时proceed()方法，它可以理解为一个接力传递，为了描述涉及到两个类的递归过程，我们首先来看下面这张图。

![网络请求流程](http://upload-images.jianshu.io/upload_images/1074740-b0211c76a8745bf5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Chain_1为getResponseWithInterceptorChain方法中构建的Chain对象，我们调用它的proceed()方法即可以将request转换为response， 那么它内部做了什么事情呢，在proceed(request)方法中会封装请求，流，连接等构建Chain_2对象，并将该对象传递给Interceptor_A对象，然后调用Interceptor_A的intercept(Chain Chain_2)方法获取响应， 而在Interceptor_A的intecept()方法中可以获取到Chain_1装在Chian_2中的request对象，对请求做操作，然后调用Chain_2的proceed(request)方法，将它处理完的请求传递进去供应Chain_2构建Chain_3，这样Interceptor_B就可以从Chain_3中获取的请求就是Interceptor_A处理过的请求，与此同时Interceptor_A从Chain_2的proceed(request)方法中获取响应， 至于Chain_B中的操作则与Chain_A相同，依次递归调用。但注意到最后Chain_3会构建一个Chain_4(图中未画出， 它含有的请求应该就是Interceptor_B处理后的请求)， 这样Interceptor_C就可以从Chain_4中获取请求，然后从中获取连接上的流，进而执行对socket的读写操作， 而不会再调用Chain_4的proceed()方法， 返回的响应则会逐级的向前传递。
这一段可能有些绕，如果看不懂可以先去查看其他文章中 的介绍，不过有些文章中将Chain看做一个链，然后每个拦截器都是从链上获取请求，然后加工以后再将响应放置到上面，逐级的操作，从Chain这个名字上看确实如此，可能这是从更高的抽象角度可以如此看待，不过在看源码的过程中，Chain的proceed()方法中确实是不断地创建新的chain对象并传递到拦截器的intercept()方法中。由于水平有限，只能这么讲，图画的也比较粗糙，下面从源码的角度再去理解这个过程，然后回过头来再来理解这个过程或许会好一些。
首先来看Chain的唯一实现类RealInterceptorChain的代码。同样地我们来分析它的数据结构，它是在递归的过程中传递请求的，而响应则是在递归方法中则逐级已经返回，所以该类应该有一个Request, Chain的第二个功能时串联各个拦截器，那么它也 应该有一个拦截器的链表，然后分析它的方法应该很简单，我们所能想到的就是proceed(request)方法，在该方法中应该会使用传递进来的请求对象封装新的Chain对象，并调用下一个拦截器的intercept(Chain)方法， 这里说下一个拦截器，那么就需要一个标志位，标识下一个拦截器在拦截器链表中的位置。下面来看代码：

```
/**
 * A concrete interceptor chain that carries the entire interceptor chain: all application
 * interceptors, the OkHttp core, all network interceptors, and finally the network caller.
 */
public final class RealInterceptorChain implements Interceptor.Chain {
  private final List<Interceptor> interceptors;
  private final StreamAllocation streamAllocation;
  private final HttpCodec httpCodec;
  private final Connection connection;
  private final int index;
  private final Request request;
  private int calls;

  public RealInterceptorChain(List<Interceptor> interceptors, StreamAllocation streamAllocation,
      HttpCodec httpCodec, Connection connection, int index, Request request) {
    this.interceptors = interceptors;
    this.connection = connection;
    this.streamAllocation = streamAllocation;
    this.httpCodec = httpCodec;
    this.index = index;
    this.request = request;
  }

  ...
  @Override public Response proceed(Request request) throws IOException {
    return proceed(request, streamAllocation, httpCodec, connection);
  }

  public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
      Connection connection) throws IOException {
    if (index >= interceptors.size()) throw new AssertionError();

    calls++;

    // If we already have a stream, confirm that the incoming request will use it.
    if (this.httpCodec != null && !sameConnection(request.url())) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must retain the same host and port");
    }

    // If we already have a stream, confirm that this is the only call to chain.proceed().
    if (this.httpCodec != null && calls > 1) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must call proceed() exactly once");
    }

    // Call the next interceptor in the chain.
    RealInterceptorChain next = new RealInterceptorChain(
        interceptors, streamAllocation, httpCodec, connection, index + 1, request);
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);

    // Confirm that the next interceptor made its required call to chain.proceed().
    if (httpCodec != null && index + 1 < interceptors.size() && next.calls != 1) {
      throw new IllegalStateException("network interceptor " + interceptor
          + " must call proceed() exactly once");
    }

    // Confirm that the intercepted response isn't null.
    if (response == null) {
      throw new NullPointerException("interceptor " + interceptor + " returned null");
    }

    return response;
  }
...
}
```
代码中省略了部分的方法， 这里字段属性中我们先不去关注StreamAllocation, HttpCodec和Connection, 在下一篇文章中将会重点介绍这三个类， 他们分别代表一次请求的流的分配， 流， 连接， 这里我们只需要清楚这些属性是有各个拦截器逐渐设置到Chain上的， 最后一个拦截器可以获取到他们然后进而执行网络请求，即对socket执行读写操作就可以了， 然后数值call是用来监督一个拦截器中的intercept()方法中执行chain.proceed()方法只能执行一次，当然有个例外就是重试拦截器，至于具体逻辑，我们后面来看。

接着就是Chain的重要方法， proceed(request)方法， 我们先忽略前面的条件判断，直接来看下面两句
```
// Call the next interceptor in the chain.
    RealInterceptorChain next = new RealInterceptorChain(
        interceptors, streamAllocation, httpCodec, connection, index + 1, request);
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);
 ```
就如之前的流程中所述， 首先使用传递进来的处理过的request（其实不只是请求，还包括处理过的StreamAllocation, HttpCodec, 就是流， Connection）构建一个新的Chain对象， 调用下一个拦截器的intercept(Chain)方法。 然后此方法中三个判断条件，这里暂不分析，它们跟流的分配有关，下一篇文章中在学习完StreamAllocation, HttpCodec以及Connection三个概念以后在来看这三个判断条件的具体含义。对于Interceptor的intercept()方法，由于有诸多的拦截器，完成不同任务，在后续的分析中将逐一介绍，这里先说明它们通用的流程，当然最后一个CallServerInterceptor除外，我们自定义的NetworkInterceptor也应该遵守这样的流程，即从chain中获取request, 有的还可以获取StreamAllocation, HttpCodec以及Connection等， 然后对他们做操作，然后调用chain(request)方法获取响应，然后对响应做处理后返回。
RealInterceptorChain的代码中就只剩下工具方法和getter方法了， 此时再尝试回过头看刚才的流程，是否可以更明晰一些。

至此，我们就了解了okhttp中整个网络请求过程包括请求的封装，Call对请求的封装， 任务的队列管理和调度，任务的执行过程，拦截器和拦截器链的配合递归调用，最后一个过程有点绕之外，整个过程很明晰也比较容易理解，可以在代码之上学习优秀的开源框架在过程和功能的划分和分解上是如何做出处理的。后面则是以getResponseWithInterceptors()方法中加入的拦截器为主线，依次介绍连接和流管理， 缓存管理以及Cookie的持久化等模块，后续有时间还会再学习okhttp对https， SPDY， Http2的支持，现在的分析中先忽略到对它们支持的部分。