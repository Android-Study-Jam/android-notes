# Interceptors
For how to use Interceptors, read [Interceptors · square/okhttp Wiki · GitHub](https://github.com/square/okhttp/wiki/Interceptors)

In OkHttp-Calls we saw that in `RealCall`, before we get a Response, our request passes through a list of interceptors.
All custom Interceptors that user added when building the client get add to the list first:
`interceptors.addAll(client.interceptors());`
Then the request would also go through a couple of pre-defined Interceptors: `RetryAndFollowUpInterceptor`, `BridgeInterceptor`, `CacheInterceptor`, `ConnectInterceptor`
After that are user-added networkInterceptors:
```java
if (!forWebSocket) {
      interceptors.addAll(client.networkInterceptors());
    }
```

Notice that `forWebSocket` is defaulted to `false` when we use `client.newCall(Request)` to make a call:
```java
  @Override public Call newCall(Request request) {
    return RealCall.newRealCall(this, request, false /* for web socket */);
  }
```

And after the network Interceptors, finally `CallServerInterceptor`. Since it’s guaranteed to be the last one in the chain, `CallServerInterceptor` doesn’t call `chain.proceed()` in its `intercept()` method. 
