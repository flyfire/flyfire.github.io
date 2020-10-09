---
layout: post
title: "okhttp3源码分析"
date: 2018-04-23 16:43
comments: true
categories: 
- dev
- android
tags:
- android
---

周末在家断点调试了一下okhttp的执行流程，翻看了一下okhttp的源码，对okhttp有了一个大致的了解，在这记录一下。okhttp的精髓是``Dispatcher``和``Interceptor``，这里主要分析这两个。

# Dispatcher

``Dispatcher``是在``OkHttpClient.Builder``中实例化的，主要维护了``runningSyncCalls``，``runningAsyncCalls``,``readyAsyncCalls``三个队列。

同步的请求在执行``execute``方法时会将``RealCall``对象添加到``runningSyncCalls``队列中去，然后调用``getResponseWithInterceptorChain``去获取``Response``，不管成功失败，在``finally``块中都会执行``runningSyncCalls.remove(realcall)``将``RealCall``对象移除出队列。类似于一个生产者消费者系统。

异步的请求会将请求包装成一个``AsyncCall``，实际上就是一个Runnable，然后看``runningAsyncCalls``队列的长度是否小于``maxRequests``也即64并且同一个host的请求是否小于``maxRequestsPerHost``,如果满足则添加到``runningAsyncCalls``队列中去，并且调用线程池执行，否则添加到``readyAsyncCalls``队列中去。在``AsyncCall``这个``Runnable``的``run``方法中又会去调用``getResponseWithInterceptorChain``去获取``Response``，值得注意的是在``finally``块中，将``AsyncCall``移除出队列的时候会调用``promoteCalls``方法。

<!-- more -->

```java
synchronized void enqueue(AsyncCall call) {
    if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
      runningAsyncCalls.add(call);
      executorService().execute(call);
    } else {
      readyAsyncCalls.add(call);
    }
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
```

在``promoteCalls``方法中会将``AsyncCall``从``readyAsyncCalls``队列中移除，添加到``runningAsyncCalls``中并调用线程池执行。

下面就开始``getResponseWithInterceptorChain``的分析了。

# Interceptor

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

    Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
        originalRequest, this, eventListener, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());

    return chain.proceed(originalRequest);
}
```

可以看到在``getResponseWithInterceptorChain``方法中只是把各种``Interceptor``添加到集合中去，并且实例化了一个``RealInterceptorChain``,调用了一下``chain.proceed``方法。

## RealInterceptorChain

```java
    // Call the next interceptor in the chain.
    RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
        connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
        writeTimeout);
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);
```

在``proceed``方法中实例化一个新的``RealInterceptorChain``对象``next``,注意参数中``index+1``，取出``index``对应的``Interceptor``，调用``Interceptor``的``intercept``方法将``next``对象传入。实际上就是按照``Interceptor``添加的顺序去掉用各个``Interceptor``的``intercept``方法了。

先不管``OkHttpClient``Build的时候配置的``interceptors``和``networkInterceptors``，着重看看内置的几个``Interceptor``。

## RetryAndFollowUpInterceptor

```java
    StreamAllocation streamAllocation = new StreamAllocation(client.connectionPool(),
        createAddress(request.url()), call, eventListener, callStackTrace);
    this.streamAllocation = streamAllocation;

    int followUpCount = 0;
    Response priorResponse = null;
    while (true) {
      if (canceled) {
        streamAllocation.release();
        throw new IOException("Canceled");
      }

      Response response;
      boolean releaseConnection = true;
      try {
        response = realChain.proceed(request, streamAllocation, null, null);
        releaseConnection = false;
      } catch (RouteException e) {
        // The attempt to connect via a route failed. The request will not have been sent.
        if (!recover(e.getLastConnectException(), streamAllocation, false, request)) {
          throw e.getLastConnectException();
        }
        releaseConnection = false;
        continue;
      } catch (IOException e) {
        // An attempt to communicate with a server failed. The request may have been sent.
        boolean requestSendStarted = !(e instanceof ConnectionShutdownException);
        if (!recover(e, streamAllocation, requestSendStarted, request)) throw e;
        releaseConnection = false;
        continue;
      } finally {
        // We're throwing an unchecked exception. Release any resources.
        if (releaseConnection) {
          streamAllocation.streamFailed(null);
          streamAllocation.release();
        }
      }

      // Attach the prior response if it exists. Such responses never have a body.
      if (priorResponse != null) {
        response = response.newBuilder()
            .priorResponse(priorResponse.newBuilder()
                    .body(null)
                    .build())
            .build();
      }

      Request followUp = followUpRequest(response, streamAllocation.route());

      if (followUp == null) {
        if (!forWebSocket) {
          streamAllocation.release();
        }
        return response;
      }

      closeQuietly(response.body());

      if (++followUpCount > MAX_FOLLOW_UPS) {
        streamAllocation.release();
        throw new ProtocolException("Too many follow-up requests: " + followUpCount);
      }

      if (followUp.body() instanceof UnrepeatableRequestBody) {
        streamAllocation.release();
        throw new HttpRetryException("Cannot retry streamed HTTP body", response.code());
      }

      if (!sameConnection(response, followUp.url())) {
        streamAllocation.release();
        streamAllocation = new StreamAllocation(client.connectionPool(),
            createAddress(followUp.url()), call, eventListener, callStackTrace);
        this.streamAllocation = streamAllocation;
      } else if (streamAllocation.codec() != null) {
        throw new IllegalStateException("Closing the body of " + response
            + " didn't close its backing stream. Bad interceptor?");
      }

      request = followUp;
      priorResponse = response;
    }
```

主要实例化了一个``StreamAllocation``对象，并调用了``chain.proceed``方法。对连接的重定向也做了处理。``chain.proceed``方法会接着调用下一个``Interceptor``的``intercept``方法。

## BridgeIntercepto

```java
    @Override public Response intercept(Chain chain) throws IOException {
    Request userRequest = chain.request();
    Request.Builder requestBuilder = userRequest.newBuilder();

    RequestBody body = userRequest.body();
    if (body != null) {
      MediaType contentType = body.contentType();
      if (contentType != null) {
        requestBuilder.header("Content-Type", contentType.toString());
      }

      long contentLength = body.contentLength();
      if (contentLength != -1) {
        requestBuilder.header("Content-Length", Long.toString(contentLength));
        requestBuilder.removeHeader("Transfer-Encoding");
      } else {
        requestBuilder.header("Transfer-Encoding", "chunked");
        requestBuilder.removeHeader("Content-Length");
      }
    }

    if (userRequest.header("Host") == null) {
      requestBuilder.header("Host", hostHeader(userRequest.url(), false));
    }

    if (userRequest.header("Connection") == null) {
      requestBuilder.header("Connection", "Keep-Alive");
    }

    // If we add an "Accept-Encoding: gzip" header field we're responsible for also decompressing
    // the transfer stream.
    boolean transparentGzip = false;
    if (userRequest.header("Accept-Encoding") == null && userRequest.header("Range") == null) {
      transparentGzip = true;
      requestBuilder.header("Accept-Encoding", "gzip");
    }

    List<Cookie> cookies = cookieJar.loadForRequest(userRequest.url());
    if (!cookies.isEmpty()) {
      requestBuilder.header("Cookie", cookieHeader(cookies));
    }

    if (userRequest.header("User-Agent") == null) {
      requestBuilder.header("User-Agent", Version.userAgent());
    }

    Response networkResponse = chain.proceed(requestBuilder.build());

    HttpHeaders.receiveHeaders(cookieJar, userRequest.url(), networkResponse.headers());

    Response.Builder responseBuilder = networkResponse.newBuilder()
        .request(userRequest);

    if (transparentGzip
        && "gzip".equalsIgnoreCase(networkResponse.header("Content-Encoding"))
        && HttpHeaders.hasBody(networkResponse)) {
      GzipSource responseBody = new GzipSource(networkResponse.body().source());
      Headers strippedHeaders = networkResponse.headers().newBuilder()
          .removeAll("Content-Encoding")
          .removeAll("Content-Length")
          .build();
      responseBuilder.headers(strippedHeaders);
      String contentType = networkResponse.header("Content-Type");
      responseBuilder.body(new RealResponseBody(contentType, -1L, Okio.buffer(responseBody)));
    }

    return responseBuilder.build();
  }
```

``BridgeIntercepto``主要是给``Request``添加一些头部信息，如``connection:keep-alive``,``accept-encoding:gzip``等等，对返回回来的``Response``如果是经过``gzip``压缩的，会进行一个解压缩。

## CacheInterceptor

``CacheInterceptor``new的时候将``client.internalCache()``作为参数传递了过来。

```java
  InternalCache internalCache() {
    return cache != null ? cache.internalCache : internalCache;
  }
```

下面主要看下``Cache``类

### Cache

```java
  Cache(File directory, long maxSize, FileSystem fileSystem) {
    this.cache = DiskLruCache.create(fileSystem, directory, VERSION, ENTRY_COUNT, maxSize);
  }
```

可以看到``Cache``底层实际上是使用了``DiskLruCache``。

主要看下``get``和``put``方法

```java
    @Nullable Response get(Request request) {
    String key = key(request.url());
    DiskLruCache.Snapshot snapshot;
    Entry entry;
    try {
      snapshot = cache.get(key);
      if (snapshot == null) {
        return null;
      }
    } catch (IOException e) {
      // Give up because the cache cannot be read.
      return null;
    }

    try {
      entry = new Entry(snapshot.getSource(ENTRY_METADATA));
    } catch (IOException e) {
      Util.closeQuietly(snapshot);
      return null;
    }

    Response response = entry.response(snapshot);

    if (!entry.matches(request, response)) {
      Util.closeQuietly(response.body());
      return null;
    }

    return response;
  }
```

```java
    @Nullable CacheRequest put(Response response) {
    String requestMethod = response.request().method();

    if (HttpMethod.invalidatesCache(response.request().method())) {
      try {
        remove(response.request());
      } catch (IOException ignored) {
        // The cache cannot be written.
      }
      return null;
    }
    if (!requestMethod.equals("GET")) {
      // Don't cache non-GET responses. We're technically allowed to cache
      // HEAD requests and some POST requests, but the complexity of doing
      // so is high and the benefit is low.
      return null;
    }

    if (HttpHeaders.hasVaryAll(response)) {
      return null;
    }
    // 注意这里
    Entry entry = new Entry(response);
    DiskLruCache.Editor editor = null;
    try {
      editor = cache.edit(key(response.request().url()));
      if (editor == null) {
        return null;
      }
      entry.writeTo(editor);
      //注意这里
      return new CacheRequestImpl(editor);
    } catch (IOException e) {
      abortQuietly(editor);
      return null;
    }
  }
```

```java
    Entry(Response response) {
      this.url = response.request().url().toString();
      this.varyHeaders = HttpHeaders.varyHeaders(response);
      this.requestMethod = response.request().method();
      this.protocol = response.protocol();
      this.code = response.code();
      this.message = response.message();
      this.responseHeaders = response.headers();
      this.handshake = response.handshake();
      this.sentRequestMillis = response.sentRequestAtMillis();
      this.receivedResponseMillis = response.receivedResponseAtMillis();
    }

    public void writeTo(DiskLruCache.Editor editor) throws IOException {
      BufferedSink sink = Okio.buffer(editor.newSink(ENTRY_METADATA));

      sink.writeUtf8(url)
          .writeByte('\n');
      sink.writeUtf8(requestMethod)
          .writeByte('\n');
      sink.writeDecimalLong(varyHeaders.size())
          .writeByte('\n');
      for (int i = 0, size = varyHeaders.size(); i < size; i++) {
        sink.writeUtf8(varyHeaders.name(i))
            .writeUtf8(": ")
            .writeUtf8(varyHeaders.value(i))
            .writeByte('\n');
      }

      sink.writeUtf8(new StatusLine(protocol, code, message).toString())
          .writeByte('\n');
      sink.writeDecimalLong(responseHeaders.size() + 2)
          .writeByte('\n');
      for (int i = 0, size = responseHeaders.size(); i < size; i++) {
        sink.writeUtf8(responseHeaders.name(i))
            .writeUtf8(": ")
            .writeUtf8(responseHeaders.value(i))
            .writeByte('\n');
      }
      sink.writeUtf8(SENT_MILLIS)
          .writeUtf8(": ")
          .writeDecimalLong(sentRequestMillis)
          .writeByte('\n');
      sink.writeUtf8(RECEIVED_MILLIS)
          .writeUtf8(": ")
          .writeDecimalLong(receivedResponseMillis)
          .writeByte('\n');

      if (isHttps()) {
        sink.writeByte('\n');
        sink.writeUtf8(handshake.cipherSuite().javaName())
            .writeByte('\n');
        writeCertList(sink, handshake.peerCertificates());
        writeCertList(sink, handshake.localCertificates());
        sink.writeUtf8(handshake.tlsVersion().javaName()).writeByte('\n');
      }
      sink.close();
    }
```

在``Cache``的``put``方法中``Entry``写入到``DiskLruCache``的只是请求头，url，响应头之类的信息，同时返回了一个``CacheRequestImpl(editor)``对象，Find Usage之后发现是在``CacheInterceptor``的``intercept``中被调用的。

```java
    @Override public Response intercept(Chain chain) throws IOException {
    Response cacheCandidate = cache != null
        ? cache.get(chain.request())
        : null;

    long now = System.currentTimeMillis();

    CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
    Request networkRequest = strategy.networkRequest;
    Response cacheResponse = strategy.cacheResponse;

    if (cache != null) {
      cache.trackResponse(strategy);
    }

    if (cacheCandidate != null && cacheResponse == null) {
      closeQuietly(cacheCandidate.body()); // The cache candidate wasn't applicable. Close it.
    }

    // If we're forbidden from using the network and the cache is insufficient, fail.
    if (networkRequest == null && cacheResponse == null) {
      return new Response.Builder()
          .request(chain.request())
          .protocol(Protocol.HTTP_1_1)
          .code(504)
          .message("Unsatisfiable Request (only-if-cached)")
          .body(Util.EMPTY_RESPONSE)
          .sentRequestAtMillis(-1L)
          .receivedResponseAtMillis(System.currentTimeMillis())
          .build();
    }

    // If we don't need the network, we're done.
    if (networkRequest == null) {
      return cacheResponse.newBuilder()
          .cacheResponse(stripBody(cacheResponse))
          .build();
    }

    Response networkResponse = null;
    try {
      networkResponse = chain.proceed(networkRequest);
    } finally {
      // If we're crashing on I/O or otherwise, don't leak the cache body.
      if (networkResponse == null && cacheCandidate != null) {
        closeQuietly(cacheCandidate.body());
      }
    }

    // If we have a cache response too, then we're doing a conditional get.
    if (cacheResponse != null) {
      if (networkResponse.code() == HTTP_NOT_MODIFIED) {
        Response response = cacheResponse.newBuilder()
            .headers(combine(cacheResponse.headers(), networkResponse.headers()))
            .sentRequestAtMillis(networkResponse.sentRequestAtMillis())
            .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis())
            .cacheResponse(stripBody(cacheResponse))
            .networkResponse(stripBody(networkResponse))
            .build();
        networkResponse.body().close();

        // Update the cache after combining headers but before stripping the
        // Content-Encoding header (as performed by initContentStream()).
        cache.trackConditionalCacheHit();
        cache.update(cacheResponse, response);
        return response;
      } else {
        closeQuietly(cacheResponse.body());
      }
    }

    Response response = networkResponse.newBuilder()
        .cacheResponse(stripBody(cacheResponse))
        .networkResponse(stripBody(networkResponse))
        .build();

    if (cache != null) {
      if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {
        // Offer this request to the cache.
        // *****注意这里*****
        CacheRequest cacheRequest = cache.put(response);
        return cacheWritingResponse(cacheRequest, response);
      }

      if (HttpMethod.invalidatesCache(networkRequest.method())) {
        try {
          cache.remove(networkRequest);
        } catch (IOException ignored) {
          // The cache cannot be written.
        }
      }
    }

    return response;
  }
```

可以看到主要是对``cacheResponse``和``networkResponse``进行的一些处理，比如返回304 not modified之类的处理结果。然后将``Response``写入缓存中。

```java
    /**
   * Returns a new source that writes bytes to {@code cacheRequest} as they are read by the source
   * consumer. This is careful to discard bytes left over when the stream is closed; otherwise we
   * may never exhaust the source stream and therefore not complete the cached response.
   */
  private Response cacheWritingResponse(final CacheRequest cacheRequest, Response response)
      throws IOException {
    // Some apps return a null body; for compatibility we treat that like a null cache request.
    if (cacheRequest == null) return response;
    Sink cacheBodyUnbuffered = cacheRequest.body();
    if (cacheBodyUnbuffered == null) return response;

    final BufferedSource source = response.body().source();
    final BufferedSink cacheBody = Okio.buffer(cacheBodyUnbuffered);

    Source cacheWritingSource = new Source() {
      boolean cacheRequestClosed;

      @Override public long read(Buffer sink, long byteCount) throws IOException {
        long bytesRead;
        try {
          bytesRead = source.read(sink, byteCount);
        } catch (IOException e) {
          if (!cacheRequestClosed) {
            cacheRequestClosed = true;
            cacheRequest.abort(); // Failed to write a complete cache response.
          }
          throw e;
        }

        if (bytesRead == -1) {
          if (!cacheRequestClosed) {
            cacheRequestClosed = true;
            cacheBody.close(); // The cache response is complete!
          }
          return -1;
        }

        sink.copyTo(cacheBody.buffer(), sink.size() - bytesRead, bytesRead);
        cacheBody.emitCompleteSegments();
        return bytesRead;
      }

      @Override public Timeout timeout() {
        return source.timeout();
      }

      @Override public void close() throws IOException {
        if (!cacheRequestClosed
            && !discard(this, HttpCodec.DISCARD_STREAM_TIMEOUT_MILLIS, MILLISECONDS)) {
          cacheRequestClosed = true;
          cacheRequest.abort();
        }
        source.close();
      }
    };

    String contentType = response.header("Content-Type");
    long contentLength = response.body().contentLength();
    return response.newBuilder()
        .body(new RealResponseBody(contentType, contentLength, Okio.buffer(cacheWritingSource)))
        .build();
  }
```

```java
    private final class CacheRequestImpl implements CacheRequest {
    private final DiskLruCache.Editor editor;
    private Sink cacheOut;
    private Sink body;
    boolean done;

    CacheRequestImpl(final DiskLruCache.Editor editor) {
      this.editor = editor;
      this.cacheOut = editor.newSink(ENTRY_BODY);
      this.body = new ForwardingSink(cacheOut) {
        @Override public void close() throws IOException {
          synchronized (Cache.this) {
            if (done) {
              return;
            }
            done = true;
            writeSuccessCount++;
          }
          super.close();
          editor.commit();
        }
      };
    }

    @Override public void abort() {
      synchronized (Cache.this) {
        if (done) {
          return;
        }
        done = true;
        writeAbortCount++;
      }
      Util.closeQuietly(cacheOut);
      try {
        editor.abort();
      } catch (IOException ignored) {
      }
    }

    @Override public Sink body() {
      return body;
    }
  }
```

可以看到调用了okio的``Sink``和``Source``进行了copy。

## ConnectInterceptor

```java
    // We need the network to satisfy this request. Possibly for validating a conditional GET.
    boolean doExtensiveHealthChecks = !request.method().equals("GET");
    HttpCodec httpCodec = streamAllocation.newStream(client, chain, doExtensiveHealthChecks);
    RealConnection connection = streamAllocation.connection();
```

``ConnectInterceptor``主要就是调用了``streamAllocation``的``newStream``方法。

```java
    public HttpCodec newStream(
      OkHttpClient client, Interceptor.Chain chain, boolean doExtensiveHealthChecks) {
    int connectTimeout = chain.connectTimeoutMillis();
    int readTimeout = chain.readTimeoutMillis();
    int writeTimeout = chain.writeTimeoutMillis();
    int pingIntervalMillis = client.pingIntervalMillis();
    boolean connectionRetryEnabled = client.retryOnConnectionFailure();

    try {
      RealConnection resultConnection = findHealthyConnection(connectTimeout, readTimeout,
          writeTimeout, pingIntervalMillis, connectionRetryEnabled, doExtensiveHealthChecks);
      HttpCodec resultCodec = resultConnection.newCodec(client, chain, this);

      synchronized (connectionPool) {
        codec = resultCodec;
        return resultCodec;
      }
    } catch (IOException e) {
      throw new RouteException(e);
    }
  }

  /**
   * Finds a connection and returns it if it is healthy. If it is unhealthy the process is repeated
   * until a healthy connection is found.
   */
  private RealConnection findHealthyConnection(int connectTimeout, int readTimeout,
      int writeTimeout, int pingIntervalMillis, boolean connectionRetryEnabled,
      boolean doExtensiveHealthChecks) throws IOException {
    while (true) {
      RealConnection candidate = findConnection(connectTimeout, readTimeout, writeTimeout,
          pingIntervalMillis, connectionRetryEnabled);

      // If this is a brand new connection, we can skip the extensive health checks.
      synchronized (connectionPool) {
        if (candidate.successCount == 0) {
          return candidate;
        }
      }

      // Do a (potentially slow) check to confirm that the pooled connection is still good. If it
      // isn't, take it out of the pool and start again.
      if (!candidate.isHealthy(doExtensiveHealthChecks)) {
        noNewStreams();
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
      int pingIntervalMillis, boolean connectionRetryEnabled) throws IOException {
    boolean foundPooledConnection = false;
    RealConnection result = null;
    Route selectedRoute = null;
    Connection releasedConnection;
    Socket toClose;
    synchronized (connectionPool) {
      if (released) throw new IllegalStateException("released");
      if (codec != null) throw new IllegalStateException("codec != null");
      if (canceled) throw new IOException("Canceled");

      // Attempt to use an already-allocated connection. We need to be careful here because our
      // already-allocated connection may have been restricted from creating new streams.
      releasedConnection = this.connection;
      toClose = releaseIfNoNewStreams();
      if (this.connection != null) {
        // We had an already-allocated connection and it's good.
        result = this.connection;
        releasedConnection = null;
      }
      if (!reportedAcquired) {
        // If the connection was never reported acquired, don't report it as released!
        releasedConnection = null;
      }

      if (result == null) {
        // Attempt to get a connection from the pool.
        // 注意这里，从ConnectionPool中找Connection
        Internal.instance.get(connectionPool, address, this, null);
        if (connection != null) {
          foundPooledConnection = true;
          result = connection;
        } else {
          selectedRoute = route;
        }
      }
    }
    closeQuietly(toClose);

    if (releasedConnection != null) {
      eventListener.connectionReleased(call, releasedConnection);
    }
    if (foundPooledConnection) {
      eventListener.connectionAcquired(call, result);
    }
    if (result != null) {
      // If we found an already-allocated or pooled connection, we're done.
      return result;
    }

    // If we need a route selection, make one. This is a blocking operation.
    boolean newRouteSelection = false;
    if (selectedRoute == null && (routeSelection == null || !routeSelection.hasNext())) {
      newRouteSelection = true;
      routeSelection = routeSelector.next();
    }

    synchronized (connectionPool) {
      if (canceled) throw new IOException("Canceled");

      if (newRouteSelection) {
        // Now that we have a set of IP addresses, make another attempt at getting a connection from
        // the pool. This could match due to connection coalescing.
        List<Route> routes = routeSelection.getAll();
        for (int i = 0, size = routes.size(); i < size; i++) {
          Route route = routes.get(i);
          Internal.instance.get(connectionPool, address, this, route);
          if (connection != null) {
            foundPooledConnection = true;
            result = connection;
            this.route = route;
            break;
          }
        }
      }

      if (!foundPooledConnection) {
        if (selectedRoute == null) {
          selectedRoute = routeSelection.next();
        }

        // Create a connection and assign it to this allocation immediately. This makes it possible
        // for an asynchronous cancel() to interrupt the handshake we're about to do.
        route = selectedRoute;
        refusedStreamCount = 0;
        result = new RealConnection(connectionPool, selectedRoute);
        acquire(result, false);
      }
    }

    // If we found a pooled connection on the 2nd time around, we're done.
    if (foundPooledConnection) {
      eventListener.connectionAcquired(call, result);
      return result;
    }

    // Do TCP + TLS handshakes. This is a blocking operation.
    // 进行tcp握手
    result.connect(connectTimeout, readTimeout, writeTimeout, pingIntervalMillis,
        connectionRetryEnabled, call, eventListener);
    routeDatabase().connected(result.route());

    Socket socket = null;
    synchronized (connectionPool) {
      reportedAcquired = true;

      // Pool the connection.
      // 把connection放到ConnectionPool中
      Internal.instance.put(connectionPool, result);

      // If another multiplexed connection to the same address was created concurrently, then
      // release this connection and acquire that one.
      if (result.isMultiplexed()) {
        socket = Internal.instance.deduplicate(connectionPool, address, this);
        result = connection;
      }
    }
    closeQuietly(socket);

    eventListener.connectionAcquired(call, result);
    return result;
  }
```

```java
    @Override public RealConnection get(ConnectionPool pool, Address address,
          StreamAllocation streamAllocation, Route route) {
        return pool.get(address, streamAllocation, route);
      }
```

```java
    /**
   * Returns a recycled connection to {@code address}, or null if no such connection exists. The
   * route is null if the address has not yet been routed.
   */
  @Nullable RealConnection get(Address address, StreamAllocation streamAllocation, Route route) {
    assert (Thread.holdsLock(this));
    for (RealConnection connection : connections) {
      if (connection.isEligible(address, route)) {
        streamAllocation.acquire(connection, true);
        return connection;
      }
    }
    return null;
  }
```

```java
    /**
   * Returns true if this connection can carry a stream allocation to {@code address}. If non-null
   * {@code route} is the resolved route for a connection.
   */
  public boolean isEligible(Address address, @Nullable Route route) {
    // If this connection is not accepting new streams, we're done.
    if (allocations.size() >= allocationLimit || noNewStreams) return false;

    // If the non-host fields of the address don't overlap, we're done.
    if (!Internal.instance.equalsNonHost(this.route.address(), address)) return false;

    // If the host exactly matches, we're done: this connection can carry the address.
    if (address.url().host().equals(this.route().address().url().host())) {
      return true; // This connection is a perfect match.
    }

    // At this point we don't have a hostname match. But we still be able to carry the request if
    // our connection coalescing requirements are met. See also:
    // https://hpbn.co/optimizing-application-delivery/#eliminate-domain-sharding
    // https://daniel.haxx.se/blog/2016/08/18/http2-connection-coalescing/

    // 1. This connection must be HTTP/2.
    if (http2Connection == null) return false;

    // 2. The routes must share an IP address. This requires us to have a DNS address for both
    // hosts, which only happens after route planning. We can't coalesce connections that use a
    // proxy, since proxies don't tell us the origin server's IP address.
    if (route == null) return false;
    if (route.proxy().type() != Proxy.Type.DIRECT) return false;
    if (this.route.proxy().type() != Proxy.Type.DIRECT) return false;
    if (!this.route.socketAddress().equals(route.socketAddress())) return false;

    // 3. This connection's server certificate's must cover the new host.
    if (route.address().hostnameVerifier() != OkHostnameVerifier.INSTANCE) return false;
    if (!supportsUrl(address.url())) return false;

    // 4. Certificate pinning must match the host.
    try {
      address.certificatePinner().check(address.url().host(), handshake().peerCertificates());
    } catch (SSLPeerUnverifiedException e) {
      return false;
    }

    return true; // The caller's address can be carried by this connection.
  }
```

```java
/**
   * Use this allocation to hold {@code connection}. Each call to this must be paired with a call to
   * {@link #release} on the same connection.
   */
  public void acquire(RealConnection connection, boolean reportedAcquired) {
    assert (Thread.holdsLock(connectionPool));
    if (this.connection != null) throw new IllegalStateException();

    this.connection = connection;
    this.reportedAcquired = reportedAcquired;
    connection.allocations.add(new StreamAllocationReference(this, callStackTrace));
  }
```

``StreamAllocation.acquire(connection, true)``的时候将``StreamAllocaion``的弱引用加入到``connection.allocations``集合中，这将在释放``Connection``的时候用到。

在将``Connection``放入``ConnectionPool``中时，调用了``ConnectionPool``的``put``方法。

```java
void put(RealConnection connection) {
    assert (Thread.holdsLock(this));
    if (!cleanupRunning) {
      cleanupRunning = true;
      executor.execute(cleanupRunnable);
    }
    connections.add(connection);
  }
```

线程池执行``cleanupRunnable``

```java
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

        idleConnectionCount++;

        // If the connection is ready to be evicted, we're done.
        long idleDurationNs = now - connection.idleAtNanos;
        if (idleDurationNs > longestIdleDurationNs) {
          longestIdleDurationNs = idleDurationNs;
          longestIdleConnection = connection;
        }
      }

      if (longestIdleDurationNs >= this.keepAliveDurationNs
          || idleConnectionCount > this.maxIdleConnections) {
        // We've found a connection to evict. Remove it from the list, then close it below (outside
        // of the synchronized block).
        connections.remove(longestIdleConnection);
      } else if (idleConnectionCount > 0) {
        // A connection will be ready to evict soon.
        return keepAliveDurationNs - longestIdleDurationNs;
      } else if (inUseConnectionCount > 0) {
        // All connections are in use. It'll be at least the keep alive duration 'til we run again.
        return keepAliveDurationNs;
      } else {
        // No connections, idle or in use.
        cleanupRunning = false;
        return -1;
      }
    }

    closeQuietly(longestIdleConnection.socket());

    // Cleanup again immediately.
    return 0;
  }

  /**
   * Prunes any leaked allocations and then returns the number of remaining live allocations on
   * {@code connection}. Allocations are leaked if the connection is tracking them but the
   * application code has abandoned them. Leak detection is imprecise and relies on garbage
   * collection.
   */
  private int pruneAndGetAllocationCount(RealConnection connection, long now) {
    List<Reference<StreamAllocation>> references = connection.allocations;
    for (int i = 0; i < references.size(); ) {
      Reference<StreamAllocation> reference = references.get(i);

      if (reference.get() != null) {
        i++;
        continue;
      }

      // We've discovered a leaked allocation. This is an application bug.
      StreamAllocation.StreamAllocationReference streamAllocRef =
          (StreamAllocation.StreamAllocationReference) reference;
      String message = "A connection to " + connection.route().address().url()
          + " was leaked. Did you forget to close a response body?";
      Platform.get().logCloseableLeak(message, streamAllocRef.callStackTrace);

      references.remove(i);
      connection.noNewStreams = true;

      // If this was the last allocation, the connection is eligible for immediate eviction.
      if (references.isEmpty()) {
        connection.idleAtNanos = now - keepAliveDurationNs;
        return 0;
      }
    }

    return references.size();
  }

  public static final class StreamAllocationReference extends WeakReference<StreamAllocation> {
    /**
     * Captures the stack trace at the time the Call is executed or enqueued. This is helpful for
     * identifying the origin of connection leaks.
     */
    public final Object callStackTrace;

    StreamAllocationReference(StreamAllocation referent, Object callStackTrace) {
      super(referent);
      this.callStackTrace = callStackTrace;
    }
  }
```

通过判断``connection.allocations``的集合元素的弱引用指向是否为空来判断``connection``是否应该被释放。

## CallServerInterceptor

```java
    @Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    HttpCodec httpCodec = realChain.httpStream();
    StreamAllocation streamAllocation = realChain.streamAllocation();
    RealConnection connection = (RealConnection) realChain.connection();
    Request request = realChain.request();

    long sentRequestMillis = System.currentTimeMillis();

    realChain.eventListener().requestHeadersStart(realChain.call());
    httpCodec.writeRequestHeaders(request);
    realChain.eventListener().requestHeadersEnd(realChain.call(), request);

    Response.Builder responseBuilder = null;
    if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
      // If there's a "Expect: 100-continue" header on the request, wait for a "HTTP/1.1 100
      // Continue" response before transmitting the request body. If we don't get that, return
      // what we did get (such as a 4xx response) without ever transmitting the request body.
      if ("100-continue".equalsIgnoreCase(request.header("Expect"))) {
        httpCodec.flushRequest();
        realChain.eventListener().responseHeadersStart(realChain.call());
        responseBuilder = httpCodec.readResponseHeaders(true);
      }

      if (responseBuilder == null) {
        // Write the request body if the "Expect: 100-continue" expectation was met.
        realChain.eventListener().requestBodyStart(realChain.call());
        long contentLength = request.body().contentLength();
        CountingSink requestBodyOut =
            new CountingSink(httpCodec.createRequestBody(request, contentLength));
        BufferedSink bufferedRequestBody = Okio.buffer(requestBodyOut);

        request.body().writeTo(bufferedRequestBody);
        bufferedRequestBody.close();
        realChain.eventListener()
            .requestBodyEnd(realChain.call(), requestBodyOut.successfulCount);
      } else if (!connection.isMultiplexed()) {
        // If the "Expect: 100-continue" expectation wasn't met, prevent the HTTP/1 connection
        // from being reused. Otherwise we're still obligated to transmit the request body to
        // leave the connection in a consistent state.
        streamAllocation.noNewStreams();
      }
    }

    httpCodec.finishRequest();

    if (responseBuilder == null) {
      realChain.eventListener().responseHeadersStart(realChain.call());
      responseBuilder = httpCodec.readResponseHeaders(false);
    }

    Response response = responseBuilder
        .request(request)
        .handshake(streamAllocation.connection().handshake())
        .sentRequestAtMillis(sentRequestMillis)
        .receivedResponseAtMillis(System.currentTimeMillis())
        .build();

    int code = response.code();
    if (code == 100) {
      // server sent a 100-continue even though we did not request one.
      // try again to read the actual response
      responseBuilder = httpCodec.readResponseHeaders(false);

      response = responseBuilder
              .request(request)
              .handshake(streamAllocation.connection().handshake())
              .sentRequestAtMillis(sentRequestMillis)
              .receivedResponseAtMillis(System.currentTimeMillis())
              .build();

      code = response.code();
    }

    realChain.eventListener()
            .responseHeadersEnd(realChain.call(), response);

    if (forWebSocket && code == 101) {
      // Connection is upgrading, but we need to ensure interceptors see a non-null response body.
      response = response.newBuilder()
          .body(Util.EMPTY_RESPONSE)
          .build();
    } else {
      response = response.newBuilder()
          .body(httpCodec.openResponseBody(response))
          .build();
    }

    if ("close".equalsIgnoreCase(response.request().header("Connection"))
        || "close".equalsIgnoreCase(response.header("Connection"))) {
      streamAllocation.noNewStreams();
    }

    if ((code == 204 || code == 205) && response.body().contentLength() > 0) {
      throw new ProtocolException(
          "HTTP " + code + " had non-zero Content-Length: " + response.body().contentLength());
    }

    return response;
  }
```

分别调用``httpCodec.writeRequestHeaders(request);``写请求头部，``request.body().writeTo(bufferedRequestBody);``写请求body，``httpCodec.readResponseHeaders(false);``读响应头部，``httpCodec.openResponseBody(response)``读响应body。

至此，所有``interceptor``分析完毕。

接下来会写一下关于okhttp的扩展，比如如何监听下载进度，appliation interceptor与network interceptor的区别，如何设置断网情况下的缓存等等。