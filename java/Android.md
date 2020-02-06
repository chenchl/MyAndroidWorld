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

#### butterknife

##### 1.核心原理

- 使用annotationProcessor （apt）编译器注解处理器来生成xxx_viewbinding类 该类的内容实际上就是根据编译期注解（RetentionPolicy.CLASS）来获取对应需要依赖注入的view控件 然后生成控件相应的findviewbyid、setonclicklistener等等系统原生方法来实现。最后需要用户手动在activity、fragment的oncreate、onviewcreated处调用bind方法来将生成的中间类绑定进来
- [Apt实战练习参考](https://github.com/chenchl/Annotation-Test)

#### EventBus

##### 1.核心原理流程

- Subscribe注解

  ```java
  @Documented
  @Retention(RetentionPolicy.RUNTIME)//运行期注解 因为历史原因还是存在使用反射查找的情况
  @Target({ElementType.METHOD})
  public @interface Subscribe {
      // 指定事件订阅方法的线程模式，即在那个线程执行事件订阅方法处理事件，默认为POSTING
      ThreadMode threadMode() default ThreadMode.POSTING;
      // 是否支持粘性事件，默认为false
      boolean sticky() default false;
      // 指定事件订阅方法的优先级，默认为0，如果多个事件订阅方法可以接收相同事件的，则优先级高的先接收到事件
      int priority() default 0;
  }
  ```

  其中threadMode属性有如下几个可选值：

  **ThreadMode.POSTING**，默认的线程模式，在那个线程发送事件就在对应线程处理事件，避免了线程切换，效率高。
  **ThreadMode.MAIN**，如在主线程（UI线程）发送事件，则直接在主线程处理事件；如果在子线程发送事件，则先将事件入队列，然后通过 Handler 切换到主线程，依次处理事件。
  **ThreadMode.MAIN_ORDERED**，无论在那个线程发送事件，都先将事件入队列，然后通过 Handler 切换到主线程，依次处理事件。
  **ThreadMode.BACKGROUND**，如果在主线程发送事件，则先将事件入队列，然后通过线程池依次处理事件；如果在子线程发送事件，则直接在发送事件的线程处理事件。
  **ThreadMode.ASYNC**，无论在那个线程发送事件，都将事件入队列，然后通过线程池处理。

- register

  

  ![register](https://upload-images.jianshu.io/upload_images/1633070-2cc84089d0fff8dc.png?imageMogr2/auto-orient/strip|imageView2/2/w/854/format/webp)

  大致流程：

  1. 首先通过getdefault方法获取eventbus的全局单例后调用register方法

     ```java
     public void register(Object subscriber) {
             // 得到当前要注册类的Class对象
             Class<?> subscriberClass = subscriber.getClass();
             // 根据Class查找当前类中订阅了事件的方法集合，即使用了Subscribe注解、有public修饰符、一个参数的方法
             // SubscriberMethod类主要封装了符合条件方法的相关信息：
             // Method对象、线程模式、事件类型、优先级、是否是粘性事等
             List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
             synchronized (this) {
                 // 循环遍历订阅了事件的方法集合，以完成注册
                 for (SubscriberMethod subscriberMethod : subscriberMethods) {
                     subscribe(subscriber, subscriberMethod);
                 }
             }
         }
     ```

  2. 根据subcribe注解获取当前类的订阅方法

     ```java
     List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
             // METHOD_CACHE是一个ConcurrentHashMap，直接保存了subscriberClass和对应SubscriberMethod的集合，以提高注册效率，赋值重复查找。
             List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
             if (subscriberMethods != null) {
                 return subscriberMethods;
             }
             // 由于使用了默认的EventBusBuilder，则ignoreGeneratedIndex属性默认为false，即是否忽略注解生成器
             if (ignoreGeneratedIndex) {
                 //使用反射查找
                 subscriberMethods = findUsingReflection(subscriberClass);
             } else {
                 //使用注解处理器生成的index类来提升效率
                 subscriberMethods = findUsingInfo(subscriberClass);
             }
             // 如果对应类中没有符合条件的方法，则抛出异常
             if (subscriberMethods.isEmpty()) {
                 throw new EventBusException("Subscriber " + subscriberClass
                         + " and its super classes have no public methods with the @Subscribe annotation");
             } else {
                 // 保存查找到的订阅事件的方法到内存中提升效率
                 METHOD_CACHE.put(subscriberClass, subscriberMethods);
                 return subscriberMethods;
             }
         }
     
     private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
             FindState findState = prepareFindState();
             findState.initForSubscriber(subscriberClass);
             // 初始状态下findState.clazz就是subscriberClass
             while (findState.clazz != null) {
                 findState.subscriberInfo = getSubscriberInfo(findState);
                 // 条件不成立
                 if (findState.subscriberInfo != null) {
                     SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
                     for (SubscriberMethod subscriberMethod : array) {
                         if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                             findState.subscriberMethods.add(subscriberMethod);
                         }
                     }
                 } else {
                     // 通过反射查找订阅事件的方法
                     findUsingReflectionInSingleClass(findState);
                 }
                 // 修改findState.clazz为subscriberClass的父类Class，即需要遍历父类
                 findState.moveToSuperclass();
             }
             // 查找到的方法保存在了FindState实例的subscriberMethods集合中。
             // 使用subscriberMethods构建一个新的List<SubscriberMethod>
             // 释放掉findState
             return getMethodsAndRelease(findState);
         }
     ```

  3. 注册当前类和订阅方法集合到subscriptionsByEventType（key为事件类型 value为订阅者的hashmap）和typesBySubscriber（key为订阅者 value为事件类型的hashmap），发送事件的时候要用到subscriptionsByEventType来根据发送的事件类型找到订阅了该事件的class集合 然后回调对应的方法即可

     ```java
     private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
             // 得到当前订阅了事件的方法的参数类型
             Class<?> eventType = subscriberMethod.eventType;
             // Subscription类保存了要注册的类对象以及当前的subscriberMethod
             Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
             // subscriptionsByEventType是一个HashMap，保存了以eventType为key,Subscription对象集合为value的键值对
             // 先查找subscriptionsByEventType是否存在以当前eventType为key的值
             CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
             // 如果不存在，则创建一个subscriptions，并保存到subscriptionsByEventType
             if (subscriptions == null) {
                 subscriptions = new CopyOnWriteArrayList<>();
                 subscriptionsByEventType.put(eventType, subscriptions);
             } else {
                 if (subscriptions.contains(newSubscription)) {
                     throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                             + eventType);
                 }
             }
             // 添加上边创建的newSubscription对象到subscriptions中
             int size = subscriptions.size();
             for (int i = 0; i <= size; i++) {
                 if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
                     subscriptions.add(i, newSubscription);
                     break;
                 }
             }
             // typesBySubscribere也是一个HashMap，保存了以当前要注册类的对象为key，注册类中订阅事件的方法的参数类型的集合为value的键值对
             // 查找是否存在对应的参数类型集合
             List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
             // 不存在则创建一个subscribedEvents，并保存到typesBySubscriber
             if (subscribedEvents == null) {
                 subscribedEvents = new ArrayList<>();
                 typesBySubscriber.put(subscriber, subscribedEvents);
             }
             // 保存当前订阅了事件的方法的参数类型
             subscribedEvents.add(eventType);
             // 粘性事件相关的 加入粘性事件的集合中
             if (subscriberMethod.sticky) {
                 if (eventInheritance) {
                     Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
                     for (Map.Entry<Class<?>, Object> entry : entries) {
                         Class<?> candidateEventType = entry.getKey();
                         if (eventType.isAssignableFrom(candidateEventType)) {
                             Object stickyEvent = entry.getValue();
                             checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                         }
                     }
                 } else {
                     Object stickyEvent = stickyEvents.get(eventType);
                     checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                 }
             }
         }
     ```

- post

  ![post](https://upload-images.jianshu.io/upload_images/1633070-3637c94ce7ff4b07.png?imageMogr2/auto-orient/strip|imageView2/2/w/688/format/webp)

  1. post方法首先判断是否发送的是粘性事件 如果是则将该事件保存到粘性事件集合中

  2. 将事件保存到当前线程的事件队列中（这里使用threadlocal实现线程隔离的作用）然后遍历事件队列并取出事件后交给postSingleEvent方法进行事件分发处理

  3. postSingleEvent通过subscriptionsByEventType获取当订阅了该事件的所有的订阅者集合 按照优先级顺序依次调用postToSubscription来反射处理对应回调

     *这里的postToSubscription会根据subcribe注解设置的线程模式来区分处理

     ```java
     private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
             // 判断订阅事件方法的线程模式
             switch (subscription.subscriberMethod.threadMode) {
                 // 默认的线程模式，在那个线程发送事件就在那个线程处理事件
                 case POSTING:
                     invokeSubscriber(subscription, event);
                     break;
                 // 在主线程处理事件
                 case MAIN:
                     // 如果在主线程发送事件，则直接在主线程通过反射处理事件
                     if (isMainThread) {
                         invokeSubscriber(subscription, event);
                     } else {
                          // 如果是在子线程发送事件，则将事件入队列，通过Handler切换到主线程执行处理事件
                         // mainThreadPoster 不为空
                         mainThreadPoster.enqueue(subscription, event);
                     }
                     break;
                 // 无论在那个线程发送事件，都先将事件入队列，然后通过 Handler 切换到主线程，依次处理事件。
                 // mainThreadPoster 不为空
                 case MAIN_ORDERED:
                     if (mainThreadPoster != null) {
                         mainThreadPoster.enqueue(subscription, event);
                     } else {
                         invokeSubscriber(subscription, event);
                     }
                     break;
                 case BACKGROUND:
                     // 如果在主线程发送事件，则先将事件入队列，然后通过线程池依次处理事件
                     if (isMainThread) {
                         backgroundPoster.enqueue(subscription, event);
                     } else {
                         // 如果在子线程发送事件，则直接在发送事件的线程通过反射处理事件
                         invokeSubscriber(subscription, event);
                     }
                     break;
                 // 无论在那个线程发送事件，都将事件入队列，然后通过线程池处理。
                 case ASYNC:
                     asyncPoster.enqueue(subscription, event);
                     break;
                 default:
                     throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
             }
         }
     ```

- unregister

  ![unregister](https://upload-images.jianshu.io/upload_images/1633070-b3b3a17a97d95d02.png?imageMogr2/auto-orient/strip|imageView2/2/w/609/format/webp)

  1. 取消注册很简单 核心就是先根据当前类对象获取当对应的订阅方法参数集合 然后遍历该集合 从subscriptionsByEventType中删除对应的value中对该类的引用 再从typesBySubscriber集合中直接删除以当前类对象为key的那条数据即可 至此取消注册完毕

     ```java
     public synchronized void unregister(Object subscriber) {
             // 得到当前注册类对象 对应的 订阅事件方法的参数类型 的集合
             List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);
             if (subscribedTypes != null) {
                 // 遍历参数类型集合，释放之前缓存的当前类中的Subscription
                 for (Class<?> eventType : subscribedTypes) {
                     unsubscribeByEventType(subscriber, eventType);
                 }
                 // 删除以当前类为key的键值对
                 typesBySubscriber.remove(subscriber);
             } else {
                 logger.log(Level.WARNING, "Subscriber to unregister was not registered before: " + subscriber.getClass());
             }
         }
     
     private void unsubscribeByEventType(Object subscriber, Class<?> eventType) {
             // 得到当前参数类型对应的Subscription集合
             List<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
             if (subscriptions != null) {
                 int size = subscriptions.size();
                 // 遍历Subscription集合
                 for (int i = 0; i < size; i++) {
                     Subscription subscription = subscriptions.get(i);
                     // 如果当前subscription对象对应的注册类对象 和 要取消注册的当前类对象相同，则删除当前subscription对象
                     if (subscription.subscriber == subscriber) {
                         subscription.active = false;
                         subscriptions.remove(i);
                         i--;//防止并发修改异常
                         size--;//防止并发修改异常
                     }
                 }
             }
         }
     ```

##### 2.注解处理器生成对应的Subscriber Index

​	*该方案采用apt注解实现 eventbus官方默认是采用反射实现 如果采用该方案实现的话组件化的情况下是个麻烦

- 类似于butterknife生成module对应的中间类来避免反射带来的内存消耗 对应的生成文件结构大致如下

  ```java
  public class MyEventBusIndex implements SubscriberInfoIndex {
      private static final Map<Class<?>, SubscriberInfo> SUBSCRIBER_INDEX;
  
      static {
          //对应整个module中的SubscriberInfo集合
          SUBSCRIBER_INDEX = new HashMap<Class<?>, SubscriberInfo>();
  		//结构是对应类的class 以及对应的订阅方法名和事件参数类型
          putIndex(new SimpleSubscriberInfo(MainActivity.class, true, new SubscriberMethodInfo[] {
              new SubscriberMethodInfo("changeText", String.class),
          }));
  
      }
  
      private static void putIndex(SubscriberInfo info) {
          SUBSCRIBER_INDEX.put(info.getSubscriberClass(), info);
      }
  
      @Override
      public SubscriberInfo getSubscriberInfo(Class<?> subscriberClass) {
          //根据对应的类class找到该类订阅的方法集合
          SubscriberInfo info = SUBSCRIBER_INDEX.get(subscriberClass);
          if (info != null) {
              return info;
          } else {
              return null;
          }
      }
  }
  ```

- 在项目的application中添加如下配置

  ```java
  EventBus.builder().addIndex(new MyEventBusIndex()).installDefaultEventBus();
  ```

#### ARouter

- ##### 同样是基于编译期注解@Route和apt注解处理器来生成对应class文件实现了IRouteGroup接口的对应接口集合类

  ```java
  @Target({ElementType.TYPE})//class文件注解
  @Retention(RetentionPolicy.CLASS)//编译期注解
  public @interface Route {
  
      /**
       * Path of route
       */
      String path();
  
      /**
       * Used to merger routes, the group name MUST BE USE THE COMMON WORDS !!!
       */
      String group() default "";
  
      /**
       * Name of route, used to generate javadoc.
       */
      String name() default "";
  
      /**
       * Extra data, can be set by user.
       * Ps. U should use the integer num sign the switch, by bits. 10001010101010
       */
      int extras() default Integer.MIN_VALUE;
  
      /**
       * The priority of route.
       */
      int priority() default -1;
  }
  
  //生成的类文件示例
  public class ARouter$$Group$$maincomponent implements IRouteGroup {
    @Override
    public void loadInto(Map<String, RouteMeta> atlas) {
      atlas.put("/maincomponent/accountinfo", RouteMeta.build(RouteType.ACTIVITY, AccountInfoActivity.class, "/maincomponent/accountinfo", "maincomponent", null, -1, -2147483648));
      atlas.put("/maincomponent/baseweb", RouteMeta.build(RouteType.ACTIVITY, BaseWebActivity.class, "/maincomponent/baseweb", "maincomponent", null, -1, -2147483648));
    }
  }
  //对应的是IRouteGroup集合
  public class ARouter$$Root$$maincomponent implements IRouteRoot {
    @Override
    public void loadInto(Map<String, Class<? extends IRouteGroup>> routes) {
      routes.put("maincomponent", ARouter$$Group$$maincomponent.class);
      routes.put("mainservice", ARouter$$Group$$mainservice.class);
    }
  }
  //Provider是服务相关集合
  public class ARouter$$Providers$$maincomponent implements IProviderGroup {
    @Override
    public void loadInto(Map<String, RouteMeta> providers) {
      providers.put("com.tope365.readapp.componentservice.main.MainService", RouteMeta.build(RouteType.PROVIDER, MainServiceImpl.class, "/mainservice/main", "mainservice", null, -1, -2147483648));
    }
  }
  
  ```

- Arouter init流程

  ```java
  //_ARouter类
  protected static synchronized boolean init(Application application) {
      mContext = application;
      //LogisticsCenter为核心类
      LogisticsCenter.init(mContext, executor);
      logger.info(Consts.TAG, "ARouter init success!");
      hasInit = true;
      //用于在主线程回调发送消息
      mHandler = new Handler(Looper.getMainLooper());
      return true;
  }
  //LogisticsCenter类
  public synchronized static void init(Context context, ThreadPoolExecutor tpe) throws HandlerException {
          mContext = context;
          executor = tpe;
  
          try {
              //使用autoservices自动注册 避免了运行期扫描dex的消耗和加固造成的无法扫描问题
              loadRouterMap();
              if (registerByPlugin) {
                  logger.info(TAG, "Load router map by arouter-auto-register plugin.");
              } else {
                  //传统扫描dex的方式加载路由表
                  Set<String> routerMap;
  
                  // It will rebuild router map every times when debuggable.
                  //debug模式或者是新版本（根据versioncode判断）需要重新扫描加载路由表
                  if (ARouter.debuggable() || PackageUtils.isNewVersion(context)) {
                      logger.info(TAG, "Run with debug mode or new install, rebuild router map.");
                      // These class was generated by arouter-compiler.
                      //扫描整个apk的dex文件 找到生成的路由表类文件集合
                      routerMap = ClassUtils.getFileNameByPackageName(mContext, ROUTE_ROOT_PAKCAGE);
                      //将集合存入sp 在下次启动时就可以不用扫描直接拿到对应的集合了 提升性能
                      if (!routerMap.isEmpty()) {
                          context.getSharedPreferences(AROUTER_SP_CACHE_KEY, Context.MODE_PRIVATE).edit().putStringSet(AROUTER_SP_KEY_MAP, routerMap).apply();
                      }
  
                      PackageUtils.updateVersion(context);    // Save new version name when router map update finishes.
                  } else {
                      //非新版本或debug，直接从sp里读取集合
                      logger.info(TAG, "Load router map from cache.");
                      routerMap = new HashSet<>(context.getSharedPreferences(AROUTER_SP_CACHE_KEY, Context.MODE_PRIVATE).getStringSet(AROUTER_SP_KEY_MAP, new HashSet<String>()));
                  }
  				//遍历路由表 
                  for (String className : routerMap) {
                      //找到实现了IRouteRoot接口的类并反射实例化 拿到路由表的信息存入Warehouse的groupsIndex map集合中
                      if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_ROOT)) {
                          ((IRouteRoot) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.groupsIndex);
                      } else if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_INTERCEPTORS)) {
                          //找到实现了IInterceptorGroup接口的类并反射实例化 拿到路由表的信息存入Warehouse的interceptorsIndex map集合中（拦截器）
                          ((IInterceptorGroup) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.interceptorsIndex);
                      } else if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_PROVIDERS)) {
                         //找到实现了IProviderGroup接口的类并反射实例化 拿到路由表的信息存入Warehouse的providersIndex map集合中（服务接口）
                          ((IProviderGroup) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.providersIndex);
                      }
                  }
              }
          } catch (Exception e) {
              throw new HandlerException(TAG + "ARouter init logistics center exception! [" + e.getMessage() + "]");
          }
      }
  ```

- navigation流程

  实例：ARouter.getInstance().build(ComponentPath.LOGIN_PATH).navigation();

  1. build()方法用于解析路由地址并封装postcard实体类 

     ```java
       protected Postcard build(String path) {
               if (TextUtils.isEmpty(path)) {
                   throw new HandlerException(Consts.TAG + "Parameter is invalid!");
               } else {
                   return build(path, extractGroup(path));
               }
           }
       
           /**
            * Build postcard by uri
            */
           protected Postcard build(Uri uri) {
               if (null == uri || TextUtils.isEmpty(uri.toString())) {
                   throw new HandlerException(Consts.TAG + "Parameter invalid!");
               } else {
                   return new Postcard(uri.getPath(), extractGroup(uri.getPath()), uri, null);
               }
           }
       
           /**
            * Build postcard by path and group
            */
           protected Postcard build(String path, String group) {
               if (TextUtils.isEmpty(path) || TextUtils.isEmpty(group)) {
                   throw new HandlerException(Consts.TAG + "Parameter is invalid!");
               } else {
                   //这里真正创建postcard对象 传参分别是路由地址和分组名
                   return new Postcard(path, group);
               }
           }
       
          	//用于解析group信息 实际很简单就是去第一个/前的string信息即为group名
           private String extractGroup(String path) {
               if (TextUtils.isEmpty(path) || !path.startsWith("/")) {
                   throw new HandlerException(Consts.TAG + "Extract the default group failed, the path must be start with '/' and contain more than 2 '/'!");
               }
       
               try {
                   String defaultGroup = path.substring(1, path.indexOf("/", 1));
                   if (TextUtils.isEmpty(defaultGroup)) {
                       throw new HandlerException(Consts.TAG + "Extract the default group failed! There's nothing between 2 '/'!");
                   } else {
                       return defaultGroup;
                   }
               } catch (Exception e) {
                   logger.warning(Consts.TAG, "Failed to extract default group! " + e.getMessage());
                   return null;
               }
           }
     ```

  2. 调用navigation方法来解析postcard中的path信息 在路由表中找到对应的路由routemeta信息后解析




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

- messagequene

  - enqueueMessage

    ```java
    boolean enqueueMessage(Message msg, long when) {
            if (msg.target == null) {//必须存在一个target对象 一般就是handler实体
                throw new IllegalArgumentException("Message must have a target.");
            }
        	//当前的msg对象是否正在使用中
            if (msg.isInUse()) {
                throw new IllegalStateException(msg + " This message is already in use.");
            }
    		
            synchronized (this) {
                //消息队列是否已经退出了 一般和线程同生共死 线程消亡后不应该继续处理
                if (mQuitting) {
                    IllegalStateException e = new IllegalStateException(
                            msg.target + " sending message to a Handler on a dead thread");
                    Log.w(TAG, e.getMessage(), e);
                    msg.recycle();
                    return false;
                }
    			//记录消息入队标志位
                msg.markInUse();
                //记录消息执行的绝对时间 
                msg.when = when;
                //获取当前的消息队列
                Message p = mMessages;
                //控制是否需要唤醒next方法中的nativepollonce阻塞
                boolean needWake;
                if (p == null || when == 0 || when < p.when) {
                    //当当前队列没有消息 或者消息的执行绝对时间为0或小于队列中第一个消息的时间时直接把当前消息插入到队列的头部准备执行
                    // New head, wake up the event queue if blocked.
                    msg.next = p;
                    mMessages = msg;
                    //mblocked的控制是根据idlehandler队列是否为空来设置true（执行时机在当前消息队列队首消息被处理完毕后）当messagequene的next方法成功取出消息时设置为false
                    needWake = mBlocked;
                } else {
                  	//这里根据消息的target是否为空和是否是异步消息来判断是否需要唤醒
                    needWake = mBlocked && p.target == null && msg.isAsynchronous();
                    //遍历消息队列 把消息插入到合适的位置
                    Message prev;
                    for (;;) {
                        prev = p;
                        p = p.next;
                       
                        if (p == null || when < p.when) {
                            //遍历到队尾或者消息的执行时间小于遍历到的消息时结束循环
                            break;
                        }
                        if (needWake && p.isAsynchronous()) {
                            needWake = false;
                        }
                    }
                    //将消息插入到队列中
                    msg.next = p; // invariant: p == prev.next
                    prev.next = msg;
                }
    
                // We can assume mPtr != 0 because mQuitting is false.
                if (needWake) {//需要唤醒则调用native方法进行唤醒 让next方法可以继续执行
                    nativeWake(mPtr);
                }
            }
            return true;
        }
    ```

  - next

    ```java
    Message next() {
       		
            int pendingIdleHandlerCount = -1; // -1 only during first iteration
            int nextPollTimeoutMillis = 0;
            for (;;) {
                if (nextPollTimeoutMillis != 0) {
                    Binder.flushPendingCommands();
                }
    			//阻塞 nextPollTimeoutMillis为最大超时时间
                nativePollOnce(ptr, nextPollTimeoutMillis);
    
                synchronized (this) {
                    // Try to retrieve the next message.  Return if found.
                    final long now = SystemClock.uptimeMillis();
                    Message prevMsg = null;
                    Message msg = mMessages;
                    //拿出队首的消息来
                    if (msg != null && msg.target == null) {
                        //招不到target的话就跳过继续遍历下一条消息
                        // Stalled by a barrier.  Find the next asynchronous message in the queue.
                        do {
                            prevMsg = msg;
                            msg = msg.next;
                        } while (msg != null && !msg.isAsynchronous());
                    }
                    if (msg != null) {
                        if (now < msg.when) {
                            //当前队首的消息执行时间还未到 则挂起到响应的时间后再执行
                            // Next message is not ready.  Set a timeout to wake up when it is ready.
                            nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                        } else {
                            //当前消息可以执行 将消息取出并从消息队列中移除（单链表移除）
                            // Got a message.
                            mBlocked = false;
                            if (prevMsg != null) {
                                prevMsg.next = msg.next;
                            } else {
                                mMessages = msg.next;
                            }
                            msg.next = null;
                            if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                            msg.markInUse();
                            return msg;
                        }
                    } else {
                        //消息队列为空
                        // No more messages.
                        nextPollTimeoutMillis = -1;
                    }
    
                    // Process the quit message now that all pending messages have been handled.
                    if (mQuitting) {
                        dispose();
                        return null;
                    }
    
                    // If first time idle, then get the number of idlers to run.
                    // Idle handles only run if the queue is empty or if the first message
                    // in the queue (possibly a barrier) is due to be handled in the future.	//获取当前是否存在IdleHandler
                    if (pendingIdleHandlerCount < 0
                            && (mMessages == null || now < mMessages.when)) {
                        pendingIdleHandlerCount = mIdleHandlers.size();
                    }
                    //没有的话继续执行循环取下一条消息
                    if (pendingIdleHandlerCount <= 0) {
                        // No idle handlers to run.  Loop and wait some more.
                        mBlocked = true;
                        continue;
                    }
    				//初始化mPendingIdleHandlers对象
                    if (mPendingIdleHandlers == null) {
                        mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                    }
                    mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
                }
    
                // Run the idle handlers.
                // We only ever reach this code block during the first iteration.
                //遍历mPendingIdleHandlers 取出IdleHandlers并调用它的queueIdle执行 执行完毕后从mIdleHandlers集合中删除
                for (int i = 0; i < pendingIdleHandlerCount; i++) {
                    final IdleHandler idler = mPendingIdleHandlers[i];
                    mPendingIdleHandlers[i] = null; // release the reference to the handler
    
                    boolean keep = false;
                    try {
                        keep = idler.queueIdle();
                    } catch (Throwable t) {
                        Log.wtf(TAG, "IdleHandler threw exception", t);
                    }
    
                    if (!keep) {
                        synchronized (this) {
                            mIdleHandlers.remove(idler);
                        }
                    }
                }
    
                // Reset the idle handler count to 0 so we do not run them again.
                pendingIdleHandlerCount = 0;
    
                // While calling an idle handler, a new message could have been delivered
                // so go back and look again for a pending message without waiting.
                nextPollTimeoutMillis = 0;
            }
        }
    ```

- handler 实际就是msg.target指向的对象 用于操作消息队列 对消息的回调进行处理

- handlerThread

  ```java
  public class HandlerThread extends Thread {
      int mPriority;//线程优先级
      int mTid = -1;
      Looper mLooper;//核心looper
      private @Nullable Handler mHandler;
  
      public HandlerThread(String name) {
          super(name);
          mPriority = Process.THREAD_PRIORITY_DEFAULT;
      }
      
      public HandlerThread(String name, int priority) {
          super(name);
          mPriority = priority;
      }
      //可重写用于执行一些初始化操作
      protected void onLooperPrepared() {
      }
  	//在thread的run方法里创建loop并开启loop死循环
      @Override
      public void run() {
          mTid = Process.myTid();
          Looper.prepare();
          synchronized (this) {
          	//这里很精髓 是用来looper初始化完毕后唤醒处理
              mLooper = Looper.myLooper();
              notifyAll();
          }
          Process.setThreadPriority(mPriority);
          onLooperPrepared();
          Looper.loop();
          mTid = -1;
      }
      
  	
      public Looper getLooper() {
          if (!isAlive()) {
              return null;
          }
          
          //当线程启动后不能马上使用 因为这时可能looper尚未初始化完毕 因此阻塞在这里直到looper初始化完毕后方可使用
          synchronized (this) {
              while (isAlive() && mLooper == null) {
                  try {
                      wait();
                  } catch (InterruptedException e) {
                  }
              }
          }
          return mLooper;
      }
  
      //获取当前线程的handler对象 使用的looper是当前线程的
      //也可以自己创建handler对象 但是必须使用getLooper方法进行创建
      @NonNull
      public Handler getThreadHandler() {
          if (mHandler == null) {
              mHandler = new Handler(getLooper());
          }
          return mHandler;
      }
       //退出当前线程 其实就是终止loop死循环
      public boolean quit() {
      	
          Looper looper = getLooper();
          if (looper != null) {
              looper.quit();
              return true;
          }
          return false;
      }
  
      //退出当前线程 其实就是终止loop死循环
      public boolean quitSafely() {
          Looper looper = getLooper();
          if (looper != null) {
              looper.quitSafely();
              return true;
          }
          return false;
      }
  
       //线程id
      public int getThreadId() {
          return mTid;
      }
  }
  
  ```

- IntentService

  结合了services和handlerThread 用来在后台线程执行services服务 就是典型的HandlerThread用法 内部实现了ServiceHandler 因此在使用时只需要重写onHandleIntent即可 执行完毕后会自动调用stopSelf来终止自身

####  android IPC机制（binder）

####  android app启动流程

##### 1.整体流程

![app启动流程](https://user-gold-cdn.xitu.io/2019/2/25/169222a25077ee68?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

1. 点击桌面App图标，Launcher进程采用Binder IPC(activityManagerProxy)向system_server进程AMS发起startActivity请求；
2. ams接受到请求后通过applicationThreadProxy通知launcher进程让其执行onpause，launcher执行完毕后再次通过AMP通知ams执行完毕
3. ams收到launcher的通知后通过socket的方式向zygote进程发送创建进程的请求；
4. Zygote进程fork出新的子进程，即App进程；并执行activitythread的main方法
5. App进程获取ams的代理amp，之后向sytem_server进程发起attachApplication请求；
6. system_server进程在收到请求后，进行一系列准备工作后，再通过ApplicationThreadProxy向App进程发送scheduleLaunchActivity请求以及BIND_APPLICATION_TRANSACTION请求；
7. app 进程中，收到 BIND_APPLICATION_TRANSACTION 命令后调用 ActivityThread.bindApplication()，通过handler向主线程发送BIND_APPLICATION 消息,之后通过Instrumentation类通过反射创建application实例对象并绑定context对象，之后调用application的oncreate方法
8. App进程的binder线程（ApplicationThread）在收到请求后，通过handler向主线程发送LAUNCH_ACTIVITY消息；
9. 主线程在收到Message后，调用activitythread.performLaunchActivity通过反射创建activity实例并调用生命周期等方法

