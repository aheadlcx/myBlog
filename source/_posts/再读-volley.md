title: 再读 volley
date: 2016-04-17 12:21:44
tags: [htpp框架]
categories": [android]
---
之前读过 Volley 的源码，可是仅仅是从使用方面在看，一些实现的细节，有不少疑点。volley 在  
设计模式上，也是很好的学习素材，每次读，都有不少收获。
<!--more  -->

# RequestQueue
cancelAll 方法有2个变种方法，入参分别是 RequestFilter filter 和 final Object tag 。  
其中后者，最后都是调用的前者.这样的设计，其实很好，他把怎么过滤，放开交给 API 调用者，到底需要  
cancel 什么，交给调用者去区分。

```java
public interface RequestFilter {
        public boolean apply(Request<?> request);
    }
```


```java
public void cancelAll(RequestFilter filter) {
        synchronized (mCurrentRequests) {
            for (Request<?> request : mCurrentRequests) {
                if (filter.apply(request)) {
                    request.cancel();
                }
            }
        }
    }
```


```java
public void cancelAll(final Object tag) {
        if (tag == null) {
            throw new IllegalArgumentException("Cannot cancelAll with a null tag");
        }
        cancelAll(new RequestFilter() {
            @Override
            public boolean apply(Request<?> request) {
                return request.getTag() == tag;
            }
        });
    }
```

例如，我们可以把所有的 GET reuqest 给 cancel 了。我们可以这样做。

```java
public void cancelAll(){
        cancelAll(new RequestFilter() {
            @Override
            public boolean apply(Request<?> request) {
                return request.getMethod() == Request.Method.GET;
            }
        });
    }
```

个人觉得可以进一步，筛选条件放开给外部，后面的操作甚至也可以放开给外部。改成下面这样就可以了，

```java
public interface RequestFilter {
        boolean apply(Request<?> request);
        void handlerRequest(Request<?> request);
    }
```


```java
public void cancelAll(RequestFilter filter) {
        synchronized (mCurrentRequests) {
            for (Request<?> request : mCurrentRequests) {
                if (filter.apply(request)) {
//                    request.cancel();
                    filter.handlerRequest(request);
                }
            }
        }
    }
```

## 疑惑
### mWaitingRequests 的作用机理
让所有相同的 getCacheKey 的 request (这里，应该理解为，调用者，认可的，意义相同的 request)  
进入 mWaitingRequests (当 mWaitingRequests 不存在这个 getCacheKey Key 的时候，会同事进入到 mCacheQueue)，  
先让第一个 request 先进入到 mCacheQueue ，找到 cache 或者找不到，从网络线程中取得，  
再回调 request.finish 方法，接着回调，RequestQueue 的 finish 方法，再让其他意义相同的   
request 进入到 mCacheQueue 中。这样，可以避免多个意义相同的 request 进入到 mCacheQueue  
接着，找不到 cache，再进入到 mNetworkQueue ，浪费用户的内存。谷歌这设计，真 nice 。

```java
<T> void finish(Request<T> request) {
        // Remove from the set of requests currently being processed.
        synchronized (mCurrentRequests) {
            mCurrentRequests.remove(request);
        }
        synchronized (mFinishedListeners) {
          for (RequestFinishedListener<T> listener : mFinishedListeners) {
            listener.onRequestFinished(request);
          }
        }

        if (request.shouldCache()) {
            synchronized (mWaitingRequests) {
                String cacheKey = request.getCacheKey();
                Queue<Request<?>> waitingRequests = mWaitingRequests.remove(cacheKey);
                if (waitingRequests != null) {
                    if (VolleyLog.DEBUG) {
                        VolleyLog.v("Releasing %d waiting requests for cacheKey=%s.",
                                waitingRequests.size(), cacheKey);
                    }
                    // Process all queued up requests. They won't be considered as in flight, but
                    // that's not a problem as the cache has been primed by 'request'.
                    mCacheQueue.addAll(waitingRequests);
                }
            }
        }
    }
```

### 在缓存和网络线程中，一个防止泄露的代码注释

```java
Request<?> request;
        while (true) {
            long startTimeMs = SystemClock.elapsedRealtime();
            // release previous request object to avoid leaking request object when mQueue is drained.
            request = null;
            try {
                // Take a request from the queue.
                request = mQueue.take();
            }
```

这里因为是一个 死循环，request 有可能是上一个 request ，如果现在 队列里面是空的，那么就  
会阻塞了，导致上一个 request 回收不了。这也不是最好的方式，最新代码中已经更改为，

```java
while(true) {
  Request<?> request = mQueue.take();
}
```

### DiskBasedCache 相关
1. CacheHeader.readHeader(InputStream is) 方法
构造 CacheHeader 有一堆 readXX 方法，全都看不懂 ~ ~  关键是方法有较详细注解，如下

```java
/*
 * Homebrewed simple serialization system used for reading and writing cache
 * headers on disk. Once upon a time, this used the standard Java
 * Object{Input,Output}Stream, but the default implementation relies heavily
 * on reflection (even for standard types) and generates a ton of garbage.
 */
 /**
 * Simple wrapper around {@link InputStream#read()} that throws EOFException
 * instead of returning -1.
 */
private static int read(InputStream is) throws IOException {
    int b = is.read();
    if (b == -1) {
        throw new EOFException();
    }
    return b;
}

```

2. CacheHeader 和 Entry 的关系和区别


# Cache
cache 过期，有2个概念，isExpired 和 refreshNeeded
控制条件分别如下。**undo** 暂时未明白具体啥意思。如果是 isExpired 那么整个  cache 就不再采用了。

```java
/** TTL for this record. */
        public long ttl;
/** True if the entry is expired. */
        public boolean isExpired() {
            return this.ttl < System.currentTimeMillis();
        }
```


```java
/** Soft TTL for this record. */
        public long softTtl;
/** True if a refresh is needed from the original data source. */
public boolean refreshNeeded() {
    return this.softTtl < System.currentTimeMillis();
}

```

# ImageRequest
返回 response 转换 bitmap 时，需要找到合适的 decodeOptions.inSampleSize 值。
getResizedDimension 方法，有点难以理解，记录如下。
关键代码如下

```java
double ratio = (double) actualSecondary / (double) actualPrimary;
int resized = maxPrimary;

// If ScaleType.CENTER_CROP fill the whole rectangle, preserve aspect ratio.
if (scaleType == ScaleType.CENTER_CROP) {
    if ((resized * ratio) < maxSecondary) {
        //等同于下面句式，
//                (  maxPrimary / actualPrimary )  < maxSecondary / actualSecondary;
//                (actualSecondary / actualPrimary ) * maxPrimary < maxSecondary;
//                (actualSecondary / actualPrimary )  < maxSecondary / maxPrimary
//                (actualSecondary /  maxSecondary)  <  actualPrimary / maxPrimary
        resized = (int) (maxSecondary / ratio);
    }

    return resized;
}

if ((resized * ratio) > maxSecondary) {
  //  if 判断逻辑，可以转换为，下面不等式。意思就是， Secondary 对比 Primary ，实际值比 最大值，更加偏大。
  //                (actualSecondary /  maxSecondary)  >  actualPrimary / maxPrimary

    // Secondary 实际值比最大值，更加偏大，那么为了不让 Primary 和 Secondary 都不能超过 最大值。
    //  
    // actualSecondary / actualPrimary = maxSecondary / resized ;
    resized = (int) (maxSecondary / ratio);

}
```

拿到 desiredWidth 和  desiredHeight 之后，再计算 decodeOptions.inSampleSize 的值

```java
// Visible for testing.
    static int findBestSampleSize(
            int actualWidth, int actualHeight, int desiredWidth, int desiredHeight) {
        double wr = (double) actualWidth / desiredWidth;
        double hr = (double) actualHeight / desiredHeight;
        double ratio = Math.min(wr, hr);
        float n = 1.0f;
        while ((n * 2) <= ratio) {
            n *= 2;
        }

        return (int) n;
    }
```

# request 缓存和 image 缓存
每个 request 的 response 缓存到 dish 的时候，key 用的是 request 的 getCacheKey ，
默认是

```java
/**
     * Returns the cache key for this request.  By default, this is the URL.
     */
    public String getCacheKey() {
        return mMethod + ":" + mUrl;
    }
```

而， ImageLoader 中的 ImageCache ，key 用的是

```java
/**
 * Creates a cache key for use with the L1 cache.
 * @param url The URL of the request.
 * @param maxWidth The max-width of the output.
 * @param maxHeight The max-height of the output.
 * @param scaleType The scaleType of the imageView.
 */
private static String getCacheKey(String url, int maxWidth, int maxHeight, ScaleType scaleType) {
    return new StringBuilder(url.length() + 12).append("#W").append(maxWidth)
            .append("#H").append(maxHeight).append("#S").append(scaleType.ordinal()).append(url)
            .toString();
}
```

这样，如果 2个请求 的 imageUrl 是一样的情况下，而 maxWidth 和 maxHeight 不一样的话，后面那个  
请求，就不会取内存缓存了，而是去 disk cache 中取，再返回给内存缓存。

# Volley 缓存处理
Cache 对应的是 DiskBasedCache 。  
所有的请求，是先放到 CacheDispatcher 处理，因此我们先从开始。  

## CacheDispatcher 源码中缓存相关

```java
// Attempt to retrieve this item from cache.
               Cache.Entry entry = mCache.get(request.getCacheKey());
               if (entry == null) {
                   request.addMarker("cache-miss");
                   // Cache miss; send off to the network dispatcher.
                   mNetworkQueue.put(request);
                   continue;
               }

               // If it is completely expired, just send it to the network.
               if (entry.isExpired()) {
                   request.addMarker("cache-hit-expired");
                   request.setCacheEntry(entry);
                   mNetworkQueue.put(request);
                   continue;
               }

                .....

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
                    final Request<?> finalRequest = request;
                    mDelivery.postResponse(request, response, new Runnable() {
                        @Override
                        public void run() {
                            try {
                                mNetworkQueue.put(finalRequest);
                            } catch (InterruptedException e) {
                                // Not much we can do about this.
                            }
                        }
                    });
                }
```

从上面代码，可以看到，先从 mCache 中取出缓存。
1. 如果没有，马上就把 request 放到 mNetworkQueue 。
2.1. 如果有，但是已经完全过期，处理同上。
2.2.1. 如果有，并且没有过期和不需要刷新，则返回。
2.2.2. 如果有，并且没有过期，但是需要刷新，则先返回缓存，并且把这个请求，放到 mNetworkQueue  
中，这里意味着，会先后返回缓存和网络的 data 。


 **undo** 这里值得思考的一个点是，假设一个情景，如果一个新闻类 app ，他的新闻列表请求是可以缓存的，  
如果一个下拉刷新操作，我们是先取得了 缓存，并且清空了之前 adapter 中的数据，取而代之的是 缓存中  
的数据，再然后，我们在网络请求中，网络异常了，那么我们的界面到底该如何处理。  
  首先，有一点，我们的 request listener 的 onResponse 方法中，是获取不到原始的 response   
的，也就是说， listener 是获取不到属性 response.intermediate = true; 的。  
先取到缓存，我们是需要先展示出来，如果网络请求也获取到了，就展示网络的 data 。这没问题。关键  
在于，如果网络请求异常了，我们正常情况下，列表第一次请求异常时，是会展示一个 异常 UI 界面的，  
但是如果之前 已经 get 到一个 缓存数据了，网络异常的时候还去展示 异常 UI 界面，那就有点奇怪了。  
因此，这需要 每个页面，需要自己处理，检查到异常时，先看看页面当前是否有数据，没数据再展示   
异常 UI 界面 ，有数据，就来个 toast 提示好了。那这就坑爹了，每个列表界面都需要做这个逻辑判断。



从上面看到，我们需要关注下，mCache 的实现，他的缓存，到底是持久化缓存，还是内存缓存。  
先提前说下，这是内存缓存 + 持久化缓存 结合的。

## NetworkDispatcher 源码中缓存相关

```java
// Write to cache if applicable.
                // TODO: Only update cache metadata instead of entire record for 304s.
                if (request.shouldCache() && response.cacheEntry != null) {
                    mCache.put(request.getCacheKey(), response.cacheEntry);
                    request.addMarker("network-cache-written");
                }
```

从上面代码可以看到，基本上，缓存相关的，都是直接交给 mCache 来处理了。接下来，看看 mCache 源码。

## DiskBasedCache 源码
put（set） 和 get 方法，肯定是值得关注的，先从这入手。  
DiskBasedCache 的 initialize 方法会把持久化保存的数据，读取出关键部分  
（储存对象为 CacheHeader ），放到内存中。供后面 CacheDispatcher 使用。
先看 put 方法

```java
public synchronized void put(String key, Entry entry) {
        pruneIfNeeded(entry.data.length);
        File file = getFileForKey(key);
        try {
            BufferedOutputStream fos = new BufferedOutputStream(new FileOutputStream(file));
            CacheHeader e = new CacheHeader(key, entry);
            boolean success = e.writeHeader(fos);
            if (!success) {
                fos.close();
                VolleyLog.d("Failed to write header for %s", file.getAbsolutePath());
                throw new IOException();
            }
            fos.write(entry.data);
            fos.close();
            putEntry(key, e);
            return;
        } catch (IOException e) {
        }
        boolean deleted = file.delete();
        if (!deleted) {
            VolleyLog.d("Could not clean up file %s", file.getAbsolutePath());
        }
    }
```

pruneIfNeeded 方法是一个检查本地缓存控件大小的方法，超过了最大值，就删除一些。后面再  
看这个方法。 getFileForKey 这个方法，会根据 key 生成一个 file 返回，这个 file 的文件名  
很值得研究，2年前，一直没看懂，最近看到 [张涛]("http://kymjs.com/code/2016/03/08/01")  
博客得知，是为了减少文件名重名的，字符串的 hasCode 不是唯一的。  


再看 get 方法,很明显，get 方法，先从 mEntries 中取出来。mEntries 是这个类实例的成员变量，  
每次启动，都会清空，也就是说，即使我们本地已经存有一个缓存了，但是一个全新的 app 启动，还是不会  
去取缓存数据的。

 ```java
 @Override
     public synchronized Entry get(String key) {
         CacheHeader entry = mEntries.get(key);
         // if the entry does not exist, return.
         if (entry == null) {
             return null;
         }

         File file = getFileForKey(key);
         CountingInputStream cis = null;
         try {
             cis = new CountingInputStream(new BufferedInputStream(new FileInputStream(file)));
             CacheHeader.readHeader(cis); // eat header
             byte[] data = streamToBytes(cis, (int) (file.length() - cis.bytesRead));
             return entry.toCacheEntry(data);
         } catch (IOException e) {
             VolleyLog.d("%s: %s", file.getAbsolutePath(), e.toString());
             remove(key);
             return null;
         }  catch (NegativeArraySizeException e) {
             VolleyLog.d("%s: %s", file.getAbsolutePath(), e.toString());
             remove(key);
             return null;
         } finally {
             if (cis != null) {
                 try {
                     cis.close();
                 } catch (IOException ioe) {
                     return null;
                 }
             }
         }
     }
 ```

 回过头来，我们看看 pruneIfNeeded 方法， 当空间不够的时候，会迭代 mEntries ，删除首先  
 记录的，不够再删除下一个，直至足够空间为止。

 ```java
 private void pruneIfNeeded(int neededSpace) {
         if ((mTotalSize + neededSpace) < mMaxCacheSizeInBytes) {
             return;
         }
         if (VolleyLog.DEBUG) {
             VolleyLog.v("Pruning old cache entries.");
         }

         long before = mTotalSize;
         int prunedFiles = 0;
         long startTime = SystemClock.elapsedRealtime();

         Iterator<Map.Entry<String, CacheHeader>> iterator = mEntries.entrySet().iterator();
         while (iterator.hasNext()) {
             Map.Entry<String, CacheHeader> entry = iterator.next();
             CacheHeader e = entry.getValue();
             boolean deleted = getFileForKey(e.key).delete();
             if (deleted) {
                 mTotalSize -= e.size;
             } else {
                VolleyLog.d("Could not delete cache entry for key=%s, filename=%s",
                        e.key, getFilenameForKey(e.key));
             }
             iterator.remove();
             prunedFiles++;

             if ((mTotalSize + neededSpace) < mMaxCacheSizeInBytes * HYSTERESIS_FACTOR) {
                 break;
             }
         }

         if (VolleyLog.DEBUG) {
             VolleyLog.v("pruned %d files, %d bytes, %d ms",
                     prunedFiles, (mTotalSize - before), SystemClock.elapsedRealtime() - startTime);
         }
     }
 ```
