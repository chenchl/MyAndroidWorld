## Android

### 主流开源库源码

#### okhttp

- 整体流程图

  ![okhttp流程图](https://upload-images.jianshu.io/upload_images/4037105-dc97d1a43aed5334?imageMogr2/auto-orient/strip|imageView2/2/w/1000/format/webp)

- 基本使用方法

  ```java
  //同步请求
  String url = "http://wwww.baidu.com";
  OkHttpClient okHttpClient = new OkHttpClient();
  final Request request = new Request.Builder()
          .url(url)
          .build();
  final Call call = okHttpClient.newCall(request);
  //android P之后以及禁止直接在主线程请求网络
  new Thread(new Runnable() {
      @Override
      public void run() {
          try {
              Response response = call.execute();
              Log.d(TAG, "run: " + response.body().string());
          } catch (IOException e) {
              e.printStackTrace();
          }
      }
  }).start();
  
  //异步请求
  String url = "http://wwww.baidu.com";
  OkHttpClient okHttpClient = new OkHttpClient();
  final Request request = new Request.Builder()
          .url(url)
          .build();
  Call call = okHttpClient.newCall(request);
  call.enqueue(new Callback() {
      @Override
      public void onFailure(Call call, IOException e) {
          Log.d(TAG, "onFailure: ");
      }
  
      @Override
      public void onResponse(Call call, Response response) throws IOException {
          Log.d(TAG, "onResponse: " + response.body().string());
      }
  });
  ```

  #### Request

  每一个HTTP请求包含一个URL、一个方法（GET或POST或其他）、一些HTTP头。请求还可能包含一个特定内容类型的数据类的主体部分。

  #### Response

  响应是对请求的回复，包含状态码、HTTP头和主体部分。

  #### Call

  OkHttp使用Call抽象出一个满足请求的模型，尽管中间可能会有多个请求或响应。执行Call有两种方式，同步或异步（execute和enqueue）

- 流程详解

  1. 创建okhttpclient对象 通过源码分析这里使用了建造者模式构建builder对象来初始化各种配置参数

  2. 创建request对象 依旧采用建造者模式构建builder对象 

  3. 使用okhttpclient对象的newCall方法包装request对象，生成对应的realCall对象，该类对象包含两个方法execute及enquene(分别是同步和异步请求)

     同步方法直接将请求放到client对象的dispatcher的runningSyncCalls队列中，然后直接执行内部的getResponseWithInterceptorChain（）进入责任链拦截器中处理请求 

     ```java
     @Override public Response execute() throws IOException {
         synchronized (this) {
           if (executed) throw new IllegalStateException("Already Executed");
           executed = true;
         }
         transmitter.timeoutEnter();
         transmitter.callStart();
         try {
           client.dispatcher().executed(this);
           return getResponseWithInterceptorChain();
         } finally {
           client.dispatcher().finished(this);
         }
       }
     ```

     异步方法则是将请求加入到client对象的dispatcher的readyAsyncCalls（异步就绪）队列中等待执行，之后开始遍历readyAsyncCalls按顺序取出后放入runningAsyncCalls队列中真正开始执行请求（有个限定条件是当前runningAsyncCalls的最大同时请求数不能大于64且同一host地址的请求数不能大于5）最终都会和同步方法一样执行到realcall的getResponseWithInterceptorChain方法进入责任链拦截器中处理请求 

     ```
     @Override public void enqueue(Callback responseCallback) {
         synchronized (this) {
           if (executed) throw new IllegalStateException("Already Executed");
           executed = true;
         }
         transmitter.callStart();
         client.dispatcher().enqueue(new AsyncCall(responseCallback));
       }
     ```

  4. 责任链核心方法getResponseWithInterceptorChain

     ```java
     Response getResponseWithInterceptorChain() throws IOException {
         // Build a full stack of interceptors.
         List<Interceptor> interceptors = new ArrayList<>();
         interceptors.addAll(client.interceptors());
         interceptors.add(new RetryAndFollowUpInterceptor(client));
         interceptors.add(new BridgeInterceptor(client.cookieJar()));
         interceptors.add(new CacheInterceptor(client.internalCache()));
         interceptors.add(new ConnectInterceptor(client));
         if (!forWebSocket) {
           interceptors.addAll(client.networkInterceptors());
         }
         interceptors.add(new CallServerInterceptor(forWebSocket));
     
         Interceptor.Chain chain = new RealInterceptorChain(interceptors, transmitter, null, 0,
             originalRequest, this, client.connectTimeoutMillis(),
             client.readTimeoutMillis(), client.writeTimeoutMillis());
     
         boolean calledNoMoreExchanges = false;
         try {
           Response response = chain.proceed(originalRequest);
           if (transmitter.isCanceled()) {
             closeQuietly(response);
             throw new IOException("Canceled");
           }
           return response;
         } catch (IOException e) {
           calledNoMoreExchanges = true;
           throw transmitter.noMoreExchanges(e);
         } finally {
           if (!calledNoMoreExchanges) {
             transmitter.noMoreExchanges(null);
           }
         }
       }
     ```

     由源码可知责任链顺序依次为

     1. 用户自定义的interceptors（此处可用来定制log输出，缓存头定义 header头统一添加等处理）
     2. 负责失败重试以及重定向的RetryAndFollowUpInterceptor 
     3. 负责把用户构造的请求转换为发送到服务器的请求、把服务器返回的响应转换为用户友好的响应的 BridgeInterceptor；
     4. 负责读取缓存直接返回、更新缓存的 CacheInterceptor；
     5. 负责和服务器建立连接的 ConnectInterceptor；
     6. 用户自定义的networkInterceptors（此处一般用来自定义缓存方面的处理）
     7. 负责向服务器发送请求数据、从服务器读取响应数据的 CallServerInterceptor。

     整体执行核心方法是每个拦截器里的chain.proceed(originalRequest);用来传递给下一拦截器,调用链类似U字型模型 最后的response对象理论上应当是用户定义传递给okhttpclient的第一个拦截器里的response对象，至此整个网络请求完成 具体请求结果解析该response对象即可

     ```java
     public Response proceed(Request request, Transmitter transmitter, @Nullable Exchange exchange)
           throws IOException {
         if (index >= interceptors.size()) throw new AssertionError();
     
         calls++;
     
         // If we already have a stream, confirm that the incoming request will use it.
         if (this.exchange != null && !this.exchange.connection().supportsUrl(request.url())) {
           throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
               + " must retain the same host and port");
         }
     
         // If we already have a stream, confirm that this is the only call to chain.proceed().
         if (this.exchange != null && calls > 1) {
           throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
               + " must call proceed() exactly once");
         }
     
         // Call the next interceptor in the chain.
         RealInterceptorChain next = new RealInterceptorChain(interceptors, transmitter, exchange,
             index + 1, request, call, connectTimeout, readTimeout, writeTimeout);
         Interceptor interceptor = interceptors.get(index);
         Response response = interceptor.intercept(next);
     
         // Confirm that the next interceptor made its required call to chain.proceed().
         if (exchange != null && index + 1 < interceptors.size() && next.calls != 1) {
           throw new IllegalStateException("network interceptor " + interceptor
               + " must call proceed() exactly once");
         }
     
         // Confirm that the intercepted response isn't null.
         if (response == null) {
           throw new NullPointerException("interceptor " + interceptor + " returned null");
         }
     
         if (response.body() == null) {
           throw new IllegalStateException(
               "interceptor " + interceptor + " returned a response with no body");
         }
     
         return response;
       }
     ```



#### retrofit

- 原理技术点

  通过java接口以及注解来描述网络请求，并用动态代理的方式，在调用接口方法前后（before／after）注入自己的方法，before通过接口方法和注解生成网络请求的request，after通过client调用相应的网络框架（默认okhttp）去发起网络请求，并将返回的response通过converterFactorty转换成相应的数据model，最后通过calladapter转换成接口定义的数据方式返回（如rxjava Observable）

  ```java
  public <T> T create(final Class<T> service) {
      validateServiceInterface(service);
      return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
          new InvocationHandler() {
            private final Platform platform = Platform.get();
            private final Object[] emptyArgs = new Object[0];
  
            @Override public @Nullable Object invoke(Object proxy, Method method,
                @Nullable Object[] args) throws Throwable {
              // If the method is a method from Object then defer to normal invocation.
              if (method.getDeclaringClass() == Object.class) {
                return method.invoke(this, args);
              }
              if (platform.isDefaultMethod(method)) {
                return platform.invokeDefaultMethod(method, service, proxy, args);
              }
              return loadServiceMethod(method).invoke(args != null ? args : emptyArgs);
            }
          });
    }
  ```

- 重点类
  1.Retrofit类 创建接口api的动态代理对象（create()返回api service动态代理对象，调用代理对象上的方法时，会触发代理对象上的invoke方法，这里面会封装好OKHttpCall对象，OKHttpCall的数据返回根据calladapter转换为Observable）
  2.ServiceMethod 核心处理类，解析方法和注解，生成HttpRequest（toRequest方法；创建responseConverter（将response流转换为String或者实体类）； 创建callAdapter（转换为rxjava observable）
  3.OKHttpCall 封装okhttp3的调用

  4.Rxjava2CallAdapter 转换成Observable或Flowable、single、maybe等 （BodyObservable会对http code做检查，如果错误直接走onError流程）

  *如果callAdapter使用rxjava，则默认封装内部的走的是okhttp的同步请求方法，需要自行指定subscribeOn非主线程后使用 

- converter和callAdapter的作用

  1. converter用于将请求返回的responsebody转换为我们希望得到的数据格式 因此用了StringConverter

     gsonConverter、protobufConverter等等一系列实现了converter接口的实体类来进行中间的数据转换处理

  2. callAdapter用于将默认接口返回的Call<T>对象通过适配器模式转换成我们期望得到的返回值对象，典型应用如Rxjava2callAdapter、livedatacallAdapter等等，便于用户处理各种复杂场景的应用（如线程切换，生命周期绑定取消请求等等）

#### glide

##### 1.缓存机制

- 活动资源 (Active Resources) 如果当前对应的图片资源正在使用，则这个图片会被Glide放入活动缓存。

  活动资源中是一个”引用计数"的图片资源的弱引用集合。使用一个 Map<Key, WeakReference<EngineResource<?>>> 来存储的。

  *一个引用队列ReferenceQueue<EngineResource<?>> resourceReferenceQueue;每当向 activeResource 中添加一个 WeakReference 对象时都会将 resourceReferenceQueue 和这个 WeakReference 关联起来，用来跟踪这个 WeakReference 的 gc，一旦这个弱引用被 gc 掉，就会将它从 activeResource 中移除。(leakcancry原理类同)

- 内存缓存 (Memory Cache)如果图片最近被加载过，并且当前没有使用这个图片，则会被放入内存中

  内存缓存默认使用LRU(缓存淘汰算法/最近最少使用算法),当资源从活动资源移除的时候，会加入此缓存。使用图片的时候会主动从此缓存移除，加入活动资源。LRU在Android support-v4中提供了LruCache工具类。

  *LRUcache使用linkedhashmap实现 使用两个参数的构造方法创建 按照访问顺序排列

- 资源类型（Resource Disk Cache）被解码后的图片写入磁盘文件中，解码的过程可能修改了图片的参数(如:inSampleSize。inPreferredConfig)

  资源类型缓存的是经过解码后的图片，如果再使用就不需要再去进行解码配置(BitmapFactory.Options),加快获得图片速度。比如原图是一个100x100的ARGB_8888图片，在首次使用的时候需要的是50x50的RGB_565图片，那么Resource将50x50 RGB_565缓存下来，再次使用此图片的时候就可以从 Resource 获得。不需要去计算inSampleSize(缩放因子)。

- 原始数据 (Data Disk Cache)图片原始数据在磁盘中的缓存(从网络、文件中直接获得的原始数据)

  原始数据缓存是网络图像原始数据。

##### 2.Bitmap复用池

如果缓存都不存在，那么会从源地址获得图片(网络/文件)。而在解析图片的时候会需要可以获得BitmapPool(复用池)，达到复用的效果。复用并不能减少程序正在使用的内存大小。Bitmap复用，解决的是减少频繁申请内存带来的性能(抖动、碎片)问题。

BitmapPool是Glide中的Bitmap复用池,同样适用LRU来进行管理。在每次解析一张图片为Bitmap的时候(磁盘缓存、网络/文件)会从其BitmapPool中查找一个可被复用的Bitmap。当一个Bitmap从内存缓存 被动 的被移除(内存紧张、达到maxSize)的时候并不会被recycle。而是加入这个BitmapPool，只有从这个BitmapPool 被动的被移除的时候,Bitmap的内存才会真正被recycle释放。

*Bitmap复用方式为在解析的时候设置Options的inBitmap属性。

Bitmap的inMutable需要为true。
Android 4.4及以上只需要被复用的Bitmap的内存必须大于等于需要新获得Bitmap的内存，则允许复用此Bitmap。
4.4以下(3.0以上)则被复用的Bitmap与使用复用的Bitmap必须宽、高相等并且使用复用的Bitmap解码时设置的inSampleSize为1，才允许复用。

##### 3.生命周期管理

Glide在Glide.with(context)中就实现了生命周期管理，with根据传入的参数有不同的实现。

```java
//传入一个Context
  public static RequestManager with(@NonNull Context context)
  //传入一个activity
  public static RequestManager with(@NonNull Activity activity)
  //传入一个FragmentActivity
  public static RequestManager with(@NonNull FragmentActivity activity)
  //传入一个Fragment
  public static RequestManager with(@NonNull Fragment fragment)
  //传入一个View
  public static RequestManager with(@NonNull View view)
  
  public static RequestManager with(@NonNull Context context) {
    return getRetriever(context).get(context);
  }
  //由于传入参数是ApplicationContext，所以最终调用getApplicationManager方法。
  public RequestManager get(@NonNull Context context) {
    if (context == null) {
      throw new IllegalArgumentException("You cannot start a load on a null Context");
    } else if (Util.isOnMainThread() && !(context instanceof Application)) {
      if (context instanceof FragmentActivity) {
        //判断context类型是不是FragmentActivity
        return get((FragmentActivity) context);
      } else if (context instanceof Activity) {
        //判断context类型是不是Activity
        return get((Activity) context);
      } else if (context instanceof ContextWrapper) {
        //判断context类型是不是ContextWrapper
        return get(((ContextWrapper) context).getBaseContext());
      }
    }
    //context类型属于ApplicationContext
    return getApplicationManager(context);
  }
```

针对activity简单来说原理是通过往当前activity注入一个空fragment来通过lifecycle观察生命周期变化 实现请求的自动取消

#### leakcancry

##### 1.使用方法

gradle引入后在application中添加以下代码即可

```java
public class MyApplication extends Application {
    @Override
    public void onCreate() {
        super.onCreate();
        LeakCanary.install(this);
    }
}
```

*最新版本使用contentprovider来初始化则不需要任何操作了

contentprovider初始化原理是根据app启动生命周期来看 

顺序为makeApplication()（attachBaseContext）->installContentProviders(app)(contentprovider.oncreate)

->mInstrumentation.callApplicationOnCreate(app)(application.oncreate)

从上可以看出contentprovider的oncreate方法的执行时机在app.attachBaseContext和app.oncreate之间

既满足了初始化时可能需要取得上下文context的需要 同时不入侵用户的代码 直接在manifest文件配置一个contentprovider即可

##### 2.原理

- activity泄漏检测

  主要是通过app.registerActivityLifecycleCallbacks来监听生命周期 在ondestroy时调用RefWatcher.watch利用了Java的WeakReference和ReferenceQueue，通过将Activity包装到WeakReference中，被WeakReference包装过的Activity对象如果被回收，该WeakReference引用会被放到ReferenceQueue中，通过监测ReferenceQueue里面的内容就能检查到Activity是否能够被回收。

  1、 首先通过removeWeaklyReachablereference来移除已经被回收的Activity引用

  2、 通过gone(reference)判断当前弱引用对应的Activity是否已经被回收，如果已经回收说明activity能够被GC，直接返回即可。

  3、 如果Activity没有被回收，调用GcTigger.runGc方法运行GC，GC完成后在运行第1步，然后运行第2步判断Activity是否被回收了，如果这时候还没有被回收，那就说明Activity可能已经泄露。（进行这一步是容错性考虑，保证系统gc必须调用后才能确保真正的内存泄漏）

  4、 如果Activity泄露了，就抓取内存dump文件(Debug.dumpHprofData)

- fragment泄漏检测

  主要是通过app.registerActivityLifecycleCallbacks在onActivityCreated中拿到activity对象 向该activity对象的fragmentManager调用registerFragmentLifecycleCallbacks方法监听fragment生命周期变化，在onFragmentViewDestroyed时将fragment类似于activity一样通过WeakReference和ReferenceQueue来检测是否被回收

#### blockcancry

##### 1.使用方法

gradle引入后在application中添加以下代码即可

```java
public class DemoApplication extends Application {
 
    @Override
    public void onCreate() {
        super.onCreate();
        BlockCanary.install(this, new AppContext()).start();
    }
}
  
//参数设置
public class AppContext extends BlockCanaryContext {
    private static final String TAG = "AppContext";
 
    @Override
    public String provideQualifier() {
        String qualifier = "";
        try {
            PackageInfo info = DemoApplication.getAppContext().getPackageManager()
                    .getPackageInfo(DemoApplication.getAppContext().getPackageName(), 0);
            qualifier += info.versionCode + "_" + info.versionName + "_YYB";
        } catch (PackageManager.NameNotFoundException e) {
            Log.e(TAG, "provideQualifier exception", e);
        }
        return qualifier;
    }
 
    @Override
    public int provideBlockThreshold() {
        return 500;
    }
 
    @Override
    public boolean displayNotification() {
        return BuildConfig.DEBUG;
    }
 
    @Override
    public boolean stopWhenDebugging() {
        return false;
    }
}

```

##### 2.原理

1. 由于android UI线程是基于looper的消息队列机制，因此造成卡顿的原因主要就肯定发生在消息处理的过程中进行了耗时处理引发卡顿，又由于以下代码片段 因此检测卡顿的切入点就变成了再looper中插入一个合适的printer对象统计耗时即可

   ```java
   public static void loop() {
       ...
       for (;;) {
           ...
           // This must be in a local variable, in case a UI event sets the logger
           Printer logging = me.mLogging;
           if (logging != null) {
               logging.println(">>>>> Dispatching to " + msg.target + " " +
                       msg.callback + ": " + msg.what);
           }
           msg.target.dispatchMessage(msg);
           if (logging != null) {
               logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
           }
           ...
       }
   }
   //给looper设置printer方法如下
   Looper.getMainLooper().setMessageLogging(mBlockCanaryCore.monitor);
   
   //printer对象核心逻辑如下
   @Override
   public void println(String x) {
       if (!mStartedPrinting) {
           mStartTimeMillis = System.currentTimeMillis();
           mStartThreadTimeMillis = SystemClock.currentThreadTimeMillis();
           mStartedPrinting = true;
       } else {
           final long endTime = System.currentTimeMillis();
           mStartedPrinting = false;
           if (isBlock(endTime)) {
               notifyBlockEvent(endTime);
           }
       }
   }
    
   private boolean isBlock(long endTime) {
       return endTime - mStartTimeMillis > mBlockThresholdMillis;//大于一定的时间即判断为发生了卡顿
   }
   ```

   

### Android 系统相关源码

#### android消息队列相关

##### 1.handler looper messagequene

- Looper

  每个线程对应有且只有一个looper对象 存放到对应线程的threadlocal中 使用Looper.prepare方法创建并存放到当前线程的threadlocal中对应源码如下

  ```java
  private static void prepare(boolean quitAllowed) {
          if (sThreadLocal.get() != null) {
              throw new RuntimeException("Only one Looper may be created per thread");
          }
          sThreadLocal.set(new Looper(quitAllowed));
      }
  ```

  - threadlocal

  使用looper.loop方法开启死循环从messagequene中取出消息处理 大致伪代码如下

  ```java
  public static void loop() {
          final Looper me = myLooper();
          if (me == null) {
              throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
          }
          final MessageQueue queue = me.mQueue;
          for (;;) {
              //取出消息队列里下一条消息
              Message msg = queue.next(); // might block
              if (msg == null) {
                  // No message indicates that the message queue is quitting.
                  return;
              }
              final Printer logging = me.mLogging;
              if (logging != null) {
                  logging.println(">>>>> Dispatching to " + msg.target + " " +
                          msg.callback + ": " + msg.what);
              }
         
              try {
                  //真正执行消息处理的地方
                  msg.target.dispatchMessage(msg);
                  if (observer != null) {
                      observer.messageDispatched(token, msg);
                  }
                  dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;
              } catch (Exception exception) {
                
                  throw exception;
              } 
              if (logging != null) {
                  logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
              }
              msg.recycleUnchecked();
          }
      }
  ```

  

- handler

- 