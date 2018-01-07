摘要：之前的记录了一下 RxJava 最基本的使用方法，没有涉及到为什么这么多人使用它的具体原因，就是说没体现它的魅力所在嘛， 所以这一篇就记录一下我学习的RxJava 当中的变换操作符（Operator）和调度器（Scheduler），前者可以说是 RxJava 的核心功能之一，也是大多数人使用 RxJava 的主要原因，后者是可以让不同线程之间的代码在一条链路代码中书写，极大简化逻辑。

# 调度器（Scheduler）

开发过程中经常会碰到这样的需求：子线程中去请求服务器数据，拿到数据之后进行解析，然后回调给主线程中的接口去展示。

这种需求有很多种写法，比如用 AsyncTask ，在 doInBackground 中进行耗时操作，然后在 onPostExecute 当中接受结果，进行处理；或者是直接在主线程中切进行耗时操作，然后通过用 Looper.getMainLooper() 创建的 Handler 将结果发送至主线程去处理。

上面提到的两种方法，主要要解决的问题就是切换线程，因为 Android 中规定耗时操作不能在主线程当中进行，但是 UI 的更新操作又必须在主线程中进行，而 UI 的更新状态往往是需要耗时操作所得到的结果来做支撑的。所以为了解决这一矛盾，Google 官方给了 AsyncTask 和 Handler 两个工具。

RxJava 当然也可以解决上述问题，并且是在同一条链路中，不存在各种接口的回调，起到这个作用的就是 Scheduler，线程调度器。RxJava 通过它来指定每一行代码应该运行在什么样的线程环境，RxJava 当中已经内置了好几种 Scheduler：

## Scheduler 的种类

**1. Schedulers.newThread()** 

这个 Scheduler 会创建一个新的线程，并且用这个 Scheduler 指定的代码会在新创建的线程中去执行。

**2. Scheduler.io()**

这个 Scheduler 适用于一些执行阻塞式 IO 操作的，比如说：读写文件、读写数据库、访问网络等。它在内部是使用 CacheThreadPool 实现的。

```java

    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }

```
这个线程池没有核心线程数，它会根据需要去创建线程，并且有 60秒 的超时机制。**不要用这个 Scheduler 去指定计算操作的运行线程，这样可以避免创建多余的线程。**

**3. Schedulers.computation()**

这个 Scheduler 适合执行 CPU 密集型的操作，比如事件循环，处理回调和其他计算工作。它内部使用的是固定线程数的线程池，大小等于 CPU 核数。**不要用它去指定 IO 操作的代码运行线程环境，不然 IO 操作的等待会浪费 CPU。**

**4. AndroidScheculers.mainThread()**

这个 Scheduler 是 Android 独有的， 用它指定的代码会运行在主线程当中。

## Scheduler 的使用

有了上述 Scheduler 之后， 就可以使用 subscribeOn()  和 observerOn() 两个方法来指定代码的运行环境了。

这里的代码使用的是 RxJava2 的 API
```java

	   @Override
	    protected void onCreate(@Nullable Bundle savedInstanceState) {
	        super.onCreate(savedInstanceState);
	        setContentView(R.layout.activity_test);
	        tv = findViewById(R.id.id_tv);
	        Observable.create(new ObservableOnSubscribe<String>() {
	            @Override
	            public void subscribe(ObservableEmitter<String> e) throws Exception {
	
	                String result = getDataFromServer();
	
	                if (!TextUtils.isEmpty(result)) {
	                    e.onNext(result);
	                }
	
	            }
	        })
	                .subscribeOn(Schedulers.io())//指定subscribe（）发生在io线程
	                .observeOn(AndroidSchedulers.mainThread())//指定 Subscriber 的回调发在主线程
	                .subscribe(new Consumer<String>() {
	                    @Override
	                    public void accept(@NonNull String result) throws Exception {
	                        LogUtil.d(TAG, "[ accept ] " + "result=" + result);
	                        LogUtil.d(TAG, "[ accept ] " + "current thread is " + Thread.currentThread().getName());
	                        tv.setText(result);
	                    }
	                });
	    }
	
	    public String getDataFromServer() {
	        LogUtil.d(TAG, "[ getDataFromServer ] " + "current thread is " + Thread.currentThread().getName());
	        LogUtil.d(TAG, "[ getDataFromServer ] " + "get data from server");
	        return "Hello RxJava";
	    }

```

看看 Log情况：

```java

	 D/===RxJavaSample==: TestActivity [ getDataFromServer ] current thread is RxCachedThreadScheduler-1
	 D/===RxJavaSample==: TestActivity [ getDataFromServer ] get data from server
	 D/===RxJavaSample==: TestActivity [ accept ] result=Hello RxJava
	 D/===RxJavaSample==: TestActivity [ accept ] current thread is main

```

很明显了， Observable 是在子线程中发送事件， 而 Obserber 接收并处理事件是在 主线程中进行的。

总结一下，

- `subscribeOn()` 用于指定 Observable 发送事件的线程。
- `obserberOn()` 用于指定 Observer 接受并处理事件的线程。

关于两者多次使用的情况，做一下总结：

-`subscribeOn()`：多次调用，只有第一次有效。
-`observerOn()`：每调用一次，下面的代码就会切换一次。

举个例子(使用了 lambda 表达式)：

```java

	  @Override
	    protected void onCreate(@Nullable Bundle savedInstanceState) {
	        super.onCreate(savedInstanceState);
	        setContentView(R.layout.activity_test);
	        tv = findViewById(R.id.id_tv);
	
	        Observable.create((ObservableOnSubscribe<String>) e -> {
	            LogUtil.d(TAG, "[ subscribe ] " + "current thread is " + Thread.currentThread().getName());
	            String token = getTokenFromServer();
	            LogUtil.d(TAG, "[ subscribe ] " + "token:" + token);
	            e.onNext(token);
	        })
	                .subscribeOn(Schedulers.io())//第一次指定subscribe()在io线程
	                .subscribeOn(AndroidSchedulers.mainThread())//第二次指定subscribe()的线程在主线程
	                .observeOn(Schedulers.io())//指定下面map操作发生在io线程
	                .map(s -> {
	                    LogUtil.d(TAG, "[ apply ] " + "current thread is " + Thread.currentThread().getName());
	                    String playUrl = getPlayUrl(s);
	                    LogUtil.d(TAG, "[ apply ] " + "play url is " + playUrl);
	                    return playUrl;
	
	                })
	                .observeOn(AndroidSchedulers.mainThread())//指定Observer接受事件是在主线程
	                .subscribe(s -> {
	                    LogUtil.d(TAG, "[ accept ] " + "current thread is " + Thread.currentThread().getName());
	                    LogUtil.d(TAG, "[ accept ] " + "s=" + s);
	                    tv.setText(s);
	                });
	    }
	
	
	    public String getTokenFromServer() {
	        return "token-123456";
	    }
	
	    public String getPlayUrl(String token) {
	        return "api.xxxx.com?a=12&b=34&token=" + token;
	    }

```

输出结果：

```java

	01-07 18:45:51.191 12827-12853/com.rengwuxian.rxjavasamples D/===RxJavaSample==: TestActivity [ subscribe ] current thread is RxCachedThreadScheduler-1
	01-07 18:45:51.191 12827-12853/com.rengwuxian.rxjavasamples D/===RxJavaSample==: TestActivity [ subscribe ] token:token-123456
	01-07 18:45:51.191 12827-12854/com.rengwuxian.rxjavasamples D/===RxJavaSample==: TestActivity [ apply ] current thread is RxCachedThreadScheduler-2
	01-07 18:45:51.191 12827-12854/com.rengwuxian.rxjavasamples D/===RxJavaSample==: TestActivity [ apply ] play url is api.xxxx.com?a=12&b=34&token=token-123456
	01-07 18:45:51.351 12827-12827/com.rengwuxian.rxjavasamples D/===RxJavaSample==: TestActivity [ accept ] current thread is main
	01-07 18:45:51.351 12827-12827/com.rengwuxian.rxjavasamples D/===RxJavaSample==: TestActivity [ accept ] s=api.xxxx.com?a=12&b=34&token=token-123456

```

可以看到在第二次调用 subscribeOn(AndroidSchedulers.mainThread()) 并没有起作用，拿 token 的操作仍然是在 io 线程执行的。

 而第一次调用 observeOn(Schedulers.io()) 自后，后面的 map 操作用 token 去拿 url 地址这个过程是在 io 线程执行的。

在第二次调用 observeOn(AndroidSchedulers.mainThread()) 之后，将 url 地址显示在 TextView 这个过程是在 主线程中执行的。


## 变换操作符 ##
