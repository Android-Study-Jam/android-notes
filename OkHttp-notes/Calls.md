# OkHttp
[The Last HttpURLConnection](https://publicobject.com/2016/07/03/the-last-httpurlconnection/)

## Request
```java
  final HttpUrl url;
  final String method;
  final Headers headers;
  final @Nullable RequestBody body;
```

## Response
```java
  final Request request;
  final Protocol protocol;
  final int code;
  final String message;
  final @Nullable Handshake handshake;
  final Headers headers;
  final @Nullable ResponseBody body;
  final @Nullable Response networkResponse;
  final @Nullable Response cacheResponse;
  final @Nullable Response priorResponse;
  final long sentRequestAtMillis;
  final long receivedResponseAtMillis;
```

## Call
```java
	 /** Returns the original request that initiated this call. */
  Request request();

  /**
   * Invokes the request immediately, and blocks until the response can be processed or is in error.
   */
  Response execute() throws IOException;

  /**
   * Schedules the request to be executed at some point in the future.
   *
   * <p>The {@link OkHttpClient#dispatcher dispatcher} defines when the request will run: usually
   * immediately unless there are several other requests currently being executed.
	 */
  void enqueue(Callback responseCallback);

  /** Cancels the request, if possible. Requests that are already complete cannot be canceled. */
  void cancel();
```

When we call `client.newCall(request).execute();`, this is what happens:
```java
  @Override public Call newCall(Request request) {
    return RealCall.newRealCall(this, request, false /* for web socket */);
  }
```

### RealCall
```java
  static RealCall newRealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    // Safely publish the Call instance to the EventListener.
    RealCall call = new RealCall(client, originalRequest, forWebSocket);
    call.eventListener = client.eventListenerFactory().create(call);
    return call;
  }

  private RealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    // ...
    this.retryAndFollowUpInterceptor = new RetryAndFollowUpInterceptor(client, forWebSocket);
  }
```

Implementation of `execute()` and `enqueue()`:
```java
  @Override public Response execute() throws IOException {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    try {
      client.dispatcher().executed(this);
      Response result = getResponseWithInterceptorChain();
      if (result == null) throw new IOException("Canceled");
      return result;
    } finally {
      client.dispatcher().finished(this);
    }
  }

  @Override public void enqueue(Callback responseCallback) {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
  }
```
Both of them checks if the call has been executed yet, if not, then give the call to `Dispatcher`
When calling `execute()`, `Dispatcher` would execute the call immediately, but before we get a `Response` our original request would need to go through an `Intercepter.Chain`:

```java
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
        interceptors, null, null, null, 0, originalRequest, this, eventListener);
    return chain.proceed(originalRequest);
  }
```

The dispatcher would then add the call to a queue, and every time dispatcher.finished() is called, it would check if there’s any more calls in the queue and execute them.
