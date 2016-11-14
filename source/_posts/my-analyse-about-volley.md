title: my analyse about volley
date: 2014-12-21 22:01:02
tags: android
---
# volley使用简介
volley,是谷歌官方出的http请求框架，方便开发者处理相关的网络请求处理。volley目前仅仅适合简单快速的http请求（原因稍后会提到）。
## 使用例子
第一步，获得请求队列RequestQueue，这个直接调用Volley.newRequestQueue(context)方法即可获得。
第二步，实现自己的Request，然后把这个Request添加到刚刚获得的请求队列RequestQueue中去。
## 源码跟踪解析
### 请求队列RequestQueue的获得

```java
    public static RequestQueue newRequestQueue(Context context, HttpStack stack) {
            File cacheDir = new File(context.getCacheDir(), DEFAULT_CACHE_DIR);

            String userAgent = "volley/0";
            try {
                String packageName = context.getPackageName();
                PackageInfo info = context.getPackageManager().getPackageInfo(packageName, 0);
                userAgent = packageName + "/" + info.versionCode;
            } catch (NameNotFoundException e) {
            }

            if (stack == null) {
                if (Build.VERSION.SDK_INT >= 9) {
                    stack = new HurlStack();
                } else {
                    // Prior to Gingerbread, HttpUrlConnection was unreliable.
                    // See: http://android-developers.blogspot.com/2011/09/androids-http-clients.html
                    stack = new HttpClientStack(AndroidHttpClient.newInstance(userAgent));
                }
            }
            //http操作的工具类
            Network network = new BasicNetwork(stack);

            RequestQueue queue = new RequestQueue(new DiskBasedCache(cacheDir), network);
            queue.start();

            return queue;
        }
```

HurlStack是一个通过HttpURLConnection，而HttpClientStack则是HttpClient来通信的。跟到代码里面，发现这2个类都是进行一些参数的设置，这里暂时不贴代码出来了。
他们2个都是继承HttpStack的，都实现了performRequest方法，这个performRequest方法签名如下

```java
    public HttpResponse performRequest(Request<?> request, Map<String, String> additionalHeaders)
```

这个就是根据网络请求request 和 特殊的请求头来进行网络请求，得到响应并且返回 HttpResponse。这样可以在HttpURLConnection和HttpClient，统一了代码。

第34行，BasicNetwork是http请求的工具类，并且在构造函数中传入了上一步构造的stack。

    在第36行，这才是真正的构造了请求队列RequestQueue，我们来看看构造方法中的2个参数。

```java
    public RequestQueue(Cache cache, Network network) {
            this(cache, network, DEFAULT_NETWORK_THREAD_POOL_SIZE);
        }
```

cache看名字就知道了是缓存相关的操作类了，再看看上面代码中的第14行和36行，new DiskBasedCache(cacheDir)，传入一个file去，作为缓存。
我们也跟代码去

```java
     public DiskBasedCache(File rootDirectory, int maxCacheSizeInBytes) {
            mRootDirectory = rootDirectory;
            mMaxCacheSizeInBytes = maxCacheSizeInBytes;
        }

        /**
         * Constructs an instance of the DiskBasedCache at the specified directory using
         * the default maximum cache size of 5MB.
         * @param rootDirectory The root directory of the cache.
         */
        public DiskBasedCache(File rootDirectory) {
            this(rootDirectory, DEFAULT_DISK_USAGE_BYTES);
        }
```

这2个构造方法，下面个就是给了个默认值，大小是5M。
我们来看看这个类的所有方法
![DiskBasedCache](https://raw.githubusercontent.com/aheadlcx/learngit/master/DiskBasedCache.jpg)
根据方法名称，我们就可以猜到这是写入和读取 之类的操作了，细节之处，敬请慢慢读源码了。


回到我们刚刚的研究源码主线上  queue.start();就这么一个start方法，到底干了些什么，我们再跟进去看看。

```java
    public void start() {
            stop(); // Make sure any currently running dispatchers are stopped.
            // Create the cache dispatcher and start it.
            // 下面几个参数分别是缓存线程队列、网络请求队列、cache缓存的玩意（具体未明）和分发器
            mCacheDispatcher = new CacheDispatcher(mCacheQueue, mNetworkQueue,
                    mCache, mDelivery);
            mCacheDispatcher.start();

            // Create network dispatchers (and corresponding threads) up to the pool
            // size.
            //mDispatchers的长度，默认是4，所以下面的for循环一共是5次，0-4
            for (int i = 0; i < mDispatchers.length; i++) {
                NetworkDispatcher networkDispatcher = new NetworkDispatcher(
                        mNetworkQueue, mNetwork, mCache, mDelivery);
                mDispatchers[i] = networkDispatcher;
                networkDispatcher.start();
            }
        }
```

他首先开了一个用于处理缓存相关的线程CacheDispatcher，这个代码我们再跟进去看。

```java
    public void run() {
            if (DEBUG) VolleyLog.v("start new dispatcher");
            Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);

            // Make a blocking call to initialize the cache.
            mCache.initialize();

            while (true) {
                try {
                    // Get a request from the cache triage queue, blocking until
                    // at least one is available.
                    final Request<?> request = mCacheQueue.take();
                    request.addMarker("cache-queue-take");

                    // If the request has been canceled, don't bother dispatching it.
                    //如果已经被cancele，就不应该进行耗时操作了，直接finish掉
                    if (request.isCanceled()) {
                        request.finish("cache-discard-canceled");
                        continue;
                    }

                    // Attempt to retrieve this item from cache.
                    //从缓存中取出数据
                    Cache.Entry entry = mCache.get(request.getCacheKey());
                    //如果缓存数据被删除了，就得把这个属于缓存中的请求，放到网络请求中去了
                    if (entry == null) {
                        request.addMarker("cache-miss");
                        // Cache miss; send off to the network dispatcher.
                        mNetworkQueue.put(request);
                        continue;
                    }

                    // If it is completely expired, just send it to the network.
                    //如果请求过期了，就得重新进行网络请求了啊
                    //这个判断是否过期的判断，暂时不知道当时缓存时，怎么写入的。现在检查是否超时仅仅是，缓存的实体的时间毫秒数 < 当前系统的毫秒数
                    if (entry.isExpired()) {
                        request.addMarker("cache-hit-expired");
                        request.setCacheEntry(entry);
                        mNetworkQueue.put(request);
                        continue;
                    }

                    // We have a cache hit; parse its data for delivery back to the request.
                    request.addMarker("cache-hit");
                    //这里是请求返回的结果
                    Response<?> response = request.parseNetworkResponse(
                            new NetworkResponse(entry.data, entry.responseHeaders));
                    request.addMarker("cache-hit-parsed");

                    //是否需要刷新
                    if (!entry.refreshNeeded()) {
                        // Completely unexpired cache hit. Just deliver the response.
                        mDelivery.postResponse(request, response);
                    } else {
                        // Soft-expired cache hit. We can deliver the cached response,
                        // but we need to also send the request to the network for
                        // refreshing.
                        request.addMarker("cache-hit-refresh-needed");
                        request.setCacheEntry(entry);

                        // Mark the response as intermediate.
                        response.intermediate = true;

                        // Post the intermediate response back to the user and have
                        // the delivery then forward the request along to the network.
                        mDelivery.postResponse(request, response, new Runnable() {
                            @Override
                            public void run() {
                                try {
                                    mNetworkQueue.put(request);
                                } catch (InterruptedException e) {
                                    // Not much we can do about this.
                                }
                            }
                        });
                    }

                } catch (InterruptedException e) {
                    // We may have been interrupted because it was time to quit.
                    if (mQuit) {
                        return;
                    }
                    continue;
                }
            }
        }
```

代码有点长，上面我都注释了一点，大概意思就是这个请求是否应该被cancel了，如果是就不要给我做任何操作了。说到这个cancel，就想起了官方的AsyncTask的cancel方法了
，这个AsyncTask的cancel方法很坑爹，及时你cancel了，线程还是会执行相关的操作（万一哪天哪位帅哥在进行长期耗时或者查询数据库的时候，上了个同步锁，就kengdie了。），仅仅是cancel了就不回调给UI线程。回过来，这个run方法里面，还检查了是否过期应该刷新之类的，如果是就这个request放到网络请求队列中去了。


下面我们继续来看看RequestQueue的start方法中。

```java
        //mDispatchers的长度，默认是4，所以下面的for循环一共是5次，0-4
            for (int i = 0; i < mDispatchers.length; i++) {
                NetworkDispatcher networkDispatcher = new NetworkDispatcher(
                        mNetworkQueue, mNetwork, mCache, mDelivery);
                mDispatchers[i] = networkDispatcher;
                networkDispatcher.start();
            }
```

默认开启了5个网络线程，我们来看看网络线程的run方法

```java
    public void run() {
            Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
            while (true) {
                long startTimeMs = SystemClock.elapsedRealtime();
                Request<?> request;
                try {
                    // Take a request from the queue.
                    request = mQueue.take();
                } catch (InterruptedException e) {
                    // We may have been interrupted because it was time to quit.
                    if (mQuit) {
                        return;
                    }
                    continue;
                }

                try {
                    request.addMarker("network-queue-take");

                    // If the request was cancelled already, do not perform the
                    // network request.
                    if (request.isCanceled()) {
                        request.finish("network-discard-cancelled");
                        continue;
                    }

                    addTrafficStatsTag(request);

                    // Perform the network request.
                    //这里才是真正的进行网络请求
                    NetworkResponse networkResponse = mNetwork.performRequest(request);
                    request.addMarker("network-http-complete");

                    // If the server returned 304 AND we delivered a response already,
                    // we're done -- don't deliver a second identical response.
                    //如果网络请求返回304，并且我们已经有一个相同的response了，我们就不再展现相同的数据了
                    if (networkResponse.notModified && request.hasHadResponseDelivered()) {
                        request.finish("not-modified");
                        continue;
                    }

                    // Parse the response here on the worker thread.
                    //这里很关键，这里是网络返回的数据，解析为我们自己的实体。request的parseNetworkResponse
                    //需要开发者自己实现。这里如果有有效时间的限制需求的话，服务器应该返回相关的信息给我们
                    //Cache.Entry，这个内部类就是存储相关的信息，在Response中，含有一个变量Cache.Entry，解析出来，再写到缓存中去，这个切记，
                    //系统对比是否在有效期内，就是，对比Cache.Entry的   ttl < System.currentTimeMillis();，所以服务器需要给个截止时间时间戳回来
                    Response<?> response = request.parseNetworkResponse(networkResponse);
                    request.addMarker("network-parse-complete");

                    // Write to cache if applicable.
                    // TODO: Only update cache metadata instead of entire record for 304s.
                    //写入缓存中去
                    if (request.shouldCache() && response.cacheEntry != null) {
                        mCache.put(request.getCacheKey(), response.cacheEntry);
                        request.addMarker("network-cache-written");
                    }

                    // Post the response back.
                    request.markDelivered();
                    mDelivery.postResponse(request, response);
                } catch (VolleyError volleyError) {
                    volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
                    parseAndDeliverNetworkError(request, volleyError);
                } catch (Exception e) {
                    VolleyLog.e(e, "Unhandled exception %s", e.toString());
                    VolleyError volleyError = new VolleyError(e);
                    volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
                    mDelivery.postError(request, volleyError);
                }
            }
        }
```

这个run方法也是一个死循环，和缓存线程一样。疑惑来了，我这个网络线程默认是5个，这个不怕安全问题么？那我们再来看看构造的时候，传入的参数，看看用的是什么容器。
PriorityBlockingQueue<Request<?>> mNetworkQueue，传入的这个是PriorityBlockingQueue容器，看名字就猜到了个大概了吧。。这个类代码有点多，暂时不上代码了。有空再去研究下这个容器是怎么实现的。
我们的第一步获得一个请求队列，目前研究完毕了。

## 第二步，add一个request
我们再回头来看RequestQueue这个类的add方法

```java
    public <T> Request<T> add(Request<T> request) {
        // Tag the request as belonging to this queue and add it to the set of
        // current requests.
        request.setRequestQueue(this);
        synchronized (mCurrentRequests) {
            mCurrentRequests.add(request);
        }

        // Process requests in the order they are added.
        //设置一个自增的int类型标记
        request.setSequence(getSequenceNumber());
        request.addMarker("add-to-queue");

        // If the request is uncacheable, skip the cache queue and go straight
        // to the network.
        //如果这个request是不能用缓存的数据的话，就直接去网络请求,默认是可以取缓存数据的
        if (!request.shouldCache()) {
            mNetworkQueue.add(request);
            return request;
        }

        // Insert request into stage if there's already a request with the same
        // cache key in flight.
        synchronized (mWaitingRequests) {
            //request 的key 默认是url，其实这是不完全很科学的，有待改进
            String cacheKey = request.getCacheKey();
            //如果等待线程队列中有这个key了，
            if (mWaitingRequests.containsKey(cacheKey)) {
                // There is already a request in flight. Queue up.
                Queue<Request<?>> stagedRequests = mWaitingRequests
                        .get(cacheKey);
                if (stagedRequests == null) {
                    stagedRequests = new LinkedList<Request<?>>();
                }
                stagedRequests.add(request);
                mWaitingRequests.put(cacheKey, stagedRequests);
                if (VolleyLog.DEBUG) {
                    VolleyLog
                            .v("Request for cacheKey=%s is in flight, putting on hold.",
                                    cacheKey);
                }
            } else {

                // Insert 'null' queue for this cacheKey, indicating there is
                // now a request in
                // flight.
                //放一个null的queue到这个集结待命区map中，对应这个cachekey
                mWaitingRequests.put(cacheKey, null);
                //并且在缓存队列中添加这个request
                mCacheQueue.add(request);
            }
            return request;
        }
    }
```

大概就这样了。本人看完volley的代码，感觉到我工作中的项目代码和他一比，质量上被秒了。。总感觉到volley用到了很多设计模式，写的很好。
不过目前还没吸收到肚子中去，有空得再研究一些第三方的http框架，对比下，才能更好的吸收volley的设计之美。如有研究volley设计模式的朋友，敬请教育~~
