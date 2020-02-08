## Android基础

### 基础知识

#### 四大组件

##### 1.Activity

- 定义

  1. 是用户操作的可视化界面；它为用户提供了一个完成操作指令的窗口。当我们创建完毕Activity之后，需要调用`setContentView()`方法来完成界面的显示；以此来为用户提供交互的入口。
  2. 一个Activity通常就是一个单独的屏幕（窗口）。
  3. Activity之间通过Intent进行通信。
  4. android应用中每一个Activity都必须要在AndroidManifest.xml配置文件中声明

- 生命周期

  ![activity生命周期](https://img-blog.csdn.net/20180521215653550?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3hjaGFoYQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

  1. onCreate

     该方法是在Activity被创建时回调，它是生命周期第一个调用的方法，我们在创建Activity时一般都需要重写该方法，然后在该方法中做一些初始化的操作，如通过setContentView设置界面布局的资源，初始化所需要的组件信息等。只会在创建时回调一次 不应当在此处理耗时处理会影响用户体验

  2. onStart

     此方法被回调时表示Activity正在启动，此时Activity已处于可见状态，只是还没有在前台显示，因此无法与用户进行交互。

  3. onResume

     当此方法回调时，则说明Activity已在前台可见，可与用户交互了，onResume方法与onStart的相同点是两者都表示Activity可见，只不过onStart回调时Activity还是后台无法与用户交互，而onResume则已显示在前台，可与用户交互。当然从流程图，我们也可以看出当Activity停止后（onPause方法和onStop方法被调用），重新回到前台时也会调用onResume方法，因此我们也可以在onResume方法中初始化一些资源，比如重新初始化在onPause或者onStop方法中释放的资源。 

     *此处会完成view绘制流程

  4. onPause

     此方法被回调时则表示Activity正在停止（Paused形态），一般情况下onStop方法会紧接着被回调。但通过流程图我们还可以看到一种情况是onPause方法执行后直接执行了onResume方法，这属于比较极端的现象了，这可能是用户操作使当前Activity退居后台后又迅速地再回到到当前的Activity，此时onResume方法就会被回调。当然，在onPause方法中我们可以做一些数据存储或者动画停止或者资源回收的操作，但是不能太耗时，因为这可能会影响到新的Activity的显示——onPause方法执行完成后，新Activity的onResume方法才会被执行。 

  5. onStop

     一般在onPause方法执行完成直接执行，表示Activity即将停止或者完全被覆盖（Stopped形态），此时Activity不可见，仅在后台运行。同样地，在onStop方法可以做一些资源释放的操作（不能太耗时）。

  6. onDestory

     此时Activity正在被销毁，也是生命周期最后一个执行的方法，一般我们可以在此方法中做一些回收工作和最终的资源释放。 

  7. onRestart

     表示Activity正在重新启动，当Activity由不可见变为可见状态时，该方法被回调。这种情况一般是用户打开了一个新的Activity时，当前的Activity就会被暂停（onPause和onStop被执行了），接着又回到当前Activity页面时，onRestart方法就会被回调。 

  *横竖屏切换等情况下activity异常销毁重建过程

  ![异常情况生命周期](https://img-blog.csdn.net/20160717152515688?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

  

- 四种启动模式

  1. standard（默认）

     在这个模式下，都会默认创建一个新的实例。因此，在这种模式下，可以有多个相同的实例，也允许多个相同Activity叠加。

  2. singleTask

     只有一个实例。在同一个应用程序中启动他的时候，若Activity不存在，则会在当前task创建一个新的实例，若存在，则会把task中在其之上的其它Activity destory掉并调用它的onNewIntent方法。
      如果是在别的应用程序中启动它，则会新建一个task，并在该task中启动这个Activity，singleTask允许别的Activity与其在一个task中共存，也就是说，如果我在这个singleTask的实例中再打开新的Activity，这个新的Activity还是会在singleTask的实例的task中。

  3. singleTop

     可以有多个实例，但是不允许多个相同Activity叠加。即，如果Activity在栈顶的时候，启动相同的Activity，不会创建新的实例，而会调用其onNewIntent方法。

  4. singleInstance

     只有一个实例，并且这个实例独立运行在一个task中，这个task只有这个实例，不允许有别的Activity存在。

     | 启动模式       | 使用场景                                       |
     | -------------- | :--------------------------------------------- |
     | singleTask     | 适合作为app的main页面                          |
     | singleTop      | 适合启动同一activity时又不想重复创建多个实例   |
     | singleInstance | 适合需要与程序分离开的页面，例如闹铃的响铃界面 |

     *ActivityRecord、TaskRecord、ActivityStack（AMS相关）

     每一个ActivityRecord都会有一个Activity与之对应，一个Activity可能会有多个ActivityRecord，因为Activity可以被多次实例化，取决于其launchmode。一系列相关的ActivityRecord组成了一个TaskRecord，TaskRecord是存在于ActivityStack中，ActivityStackSupervisor是用来管理这些ActivityStack的。

     ![img](https://upload-images.jianshu.io/upload_images/1439846-f4ba44585c2db823?imageMogr2/auto-orient/strip|imageView2/2/w/460/format/webp)

     *android:taskAffinity 

     taskAffinity，可以翻译为任务相关性。这个参数标识了一个 Activity 所需要的任务栈的名字，默认情况下，所有 Activity 所需的任务栈的名字为应用的包名，当 Activity 设置了 taskAffinity 属性，那么这个 Activity 在被创建时就会运行在和 taskAffinity 名字相同的任务栈中，如果没有，则新建 taskAffinity 指定的任务栈，并将 Activity 放入该栈中。另外，taskAffinity 属性主要和 singleTask 或者 allowTaskReparenting 属性配对使用，在其他情况下没有意义。

- 常见Flag

  1. FLAG_ACTIVITY_NEW_TASK

     相当于singleTask模式 使用application的context启动activity时必须制定该flag

  2. FLAG_ACTIVITY_SINGLE_TOP

     相当于singleTop模式

  3. FLAG_ACTIVITY_CLEAR_TOP

     - 与FLAG_ACTIVITY_NEW_TASK配合时相当于singleTask
     - 与singletop一起使用时相当于singletask的效果

  4. FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS

     被该flag标记的activity不会出现在历史列表中（即主流手机的回退选取列表中）

- 两个activity间跳转时的生命周期

  A跳B：

  1. A：onPause
  2. B：onCreate onStart onResume
  3. A：onStop(视情况而定 如果B完全覆盖了A则会调用) 

- App安全退出的几种方案

  1. 递归退出：使用startActivityForResult启动activity 在回调中递归调用setResult和finish
  2. 使用clearTop FLAG启动主页activity 之后再在主页activity的onNewIntent回调中finish掉主页activity
  3. 管理一个全局的activity栈 遍历调用finish

##### 2.Service

- 定义

  服务是Android中实现程序后台运行的解决方案，他非常适合是去执行那些不需要和用户交互而且还要长期运行的任务。服务的运行不依赖于任何用户界面，即使程序被切换到后台，或者用户打开了另一个应用程序，服务仍然能够保持独立运行。不过需要注意的是，服务并不是运行在一个独立的进程当中，而是依赖于创建服务时所在的应用程序进程。当某个应用程序被杀掉时，所有依赖该进程的服务也会停止运行。

- 服务的两种状态

  1. 启动状态

     当应用组件（如 Activity）**通过调用 startService() 启动服务时，**服务即处于“启动”状态。**一旦启动，服务即可在后台无限期运行，即使启动服务的组件已被销毁也不受影响，除非手动调用才能停止服务**， 已启动的服务通常是执行单一操作，而且不会将结果返回给调用方。

  2. 绑定状态

     当应用组件通过调用 bindService() 绑定到服务时，服务即处于“绑定”状态。绑定服务提供了一个客户端-服务器接口，允许组件与服务进行交互、发送请求、获取结果，甚至是利用进程间通信 (IPC) 跨进程执行这些操作。 仅当与另一个应用组件绑定时，绑定服务才会运行。 多个组件可以同时绑定到该服务，但全部取消绑定后，该服务即会被销毁。

- 服务在两种启动方式下的生命周期

  

  ![service的生命周期](https://img-blog.csdn.net/20161004164521384)

  1. startService

     - onCreate

       首次创建服务时，系统将调用此方法来执行一次性设置程序（在调用 onStartCommand() 或onBind() 之前）。如果服务已在运行，则不会调用此方法，该方法只调用一次

     - onStartCommand

       当另一个组件（如 Activity）通过调用 startService() 请求启动服务时，系统将调用此方法。一旦执行此方法，服务即会启动并可在后台无限期运行。 如果自己实现此方法，则需要在服务工作完成后，通过调用 stopSelf() 或 stopService() 来停止服务。（在绑定状态下，无需实现此方法。）

       *多次启动同一服务不会重复调用onCreate 而会直接调用onStartCommand

     - onDestory

       当服务不再使用且将被销毁时，系统将调用此方法。服务应该实现此方法来清理所有资源，如线程、注册的侦听器、接收器等，这是服务接收的最后一个调用。

  2. bindService

     - onCreate

       首次创建服务时，系统将调用此方法来执行一次性设置程序（在调用 onStartCommand() 或onBind() 之前）。如果服务已在运行，则不会调用此方法，该方法只调用一次

     - onBind

       当另一个组件想通过调用 bindService() 与服务绑定（例如执行 RPC）时，系统将调用此方法。在此方法的实现中，必须返回 一个IBinder 接口的实现类，供客户端用来与服务进行通信。无论是启动状态还是绑定状态，此方法必须重写，但在启动状态的情况下可以直接返回 null。

     - onUnbind

       当所有绑定了该服务的页面都调用unbindService完时回调该方法，可用于释放资源等处理

     - onDestory

       当服务不再使用且将被销毁时，系统将调用此方法。服务应该实现此方法来清理所有资源，如线程、注册的侦听器、接收器等，这是服务接收的最后一个调用。

- 服务保活

  1. 在服务的onStartCommand中返回START_STICKY
  2. 在manifest注册服务是提高服务优先级android:priority="1000"
  3. 在服务的onDestory方法中尝试重新启动服务
  
- IntentService

  1. 与普通service的差别
  
     - Service中的程序仍然运行于主线程中，而在IntentService中的程序运行在我们的异步后台线程中。
     - 在Service中当我的后台服务执行完毕之后需要在外部组件中调用stopService方法销毁服务，而IntentService并不需要，它会在工作执行完毕后自动销毁。
  
  2. 用法
  
     1. 编写自己的Service类继承IntentService，并重写其中的onHandleIntent(Intent)方法，该方法是IntentService的一个抽象方法，用来处理我们通过startService方法开启的服务，传入参数Intent就是开启服务的Intent。
     2. 注册我们的服务后在外部通过startservice启动服务即可
  
  3. 实现原理
  
     - 整体实现思路是通过handlerThread开启子线程 之后自定义一个handler来接受服务开始请求后在onHandleIntent交给用户去处理 处理完毕后调用stopSelf来自动销毁服务
  
       ```java
       public abstract class IntentService extends Service {
           private volatile Looper mServiceLooper;
           @UnsupportedAppUsage
           private volatile ServiceHandler mServiceHandler;
           private String mName;
           private boolean mRedelivery;
           
       	//自定义handler用于处理handlerThread线程对应的消息
           private final class ServiceHandler extends Handler {
               public ServiceHandler(Looper looper) {
                   super(looper);
               }
       
               @Override
               public void handleMessage(Message msg) {
                   //回调抽象方法给用户去处理任务
                   onHandleIntent((Intent)msg.obj);
                   //任务处理完毕关闭自身服务
                   stopSelf(msg.arg1);
               }
           }
           public IntentService(String name) {
               super();
               mName = name;
           }
       	//START_NOT_STICKY型服务会直接被关闭，而START_REDELIVER_INTENT 型服务会在可用资源不再吃紧的时候尝试再次启动服务。默认false 直接关闭
           public void setIntentRedelivery(boolean enabled) {
               mRedelivery = enabled;
           }
       
           @Override
           public void onCreate() {
            
               super.onCreate();
               //启动子线程
               HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
               thread.start();
       		//获取子线程的looper对象
               mServiceLooper = thread.getLooper();
               //使用子线程looper对象创建handler用于接收消息
               mServiceHandler = new ServiceHandler(mServiceLooper);
           }
       
           @Override
           public void onStart(@Nullable Intent intent, int startId) {
               Message msg = mServiceHandler.obtainMessage();
               msg.arg1 = startId;
               msg.obj = intent;
               //发送消息到子线程开始任务处理
               mServiceHandler.sendMessage(msg);
           }
           @Override
           public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
               //开始任务
               onStart(intent, startId);
               return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
           }
       
           @Override
           public void onDestroy() {
               //退出looper循环
               mServiceLooper.quit();
           }
       
           @Override
           @Nullable
           public IBinder onBind(Intent intent) {
               return null;
           }
       
           //用户在这个方法里写具体的任务逻辑
           @WorkerThread
           protected abstract void onHandleIntent(@Nullable Intent intent);
       }
       ```
  
- 

##### 3.广播BroadCastReceiver

- 定义

  用来响应系统范围内的广播组件，可以从Android系统和其它app发送或接收广播消息，类似于发布 - 订阅设计模式。其特点是异步的，广播发送者不会关心有无接收者接收。可应用于不同组件之间的通信、多线程通信和系统在特定情况下的通信。

- 整体流程

  1. 广播接收者BroadcastReceiver通过Binder机制向AMS(Activity Manager Service)进行注册；

     - 静态注册

       直接在Manifest.xml文件的节点中配置，使用< receiver >标签声明，并在标签内用 < intent-filter > 标签设置过滤器，该注册方式不管app是否处于活动状态，都会进行监听。

       *android P中禁止了静态注册 现在已经没法靠这个方法拉活app了

     - 动态注册

       直接在代码在代码中调用Context.registerReceiver()方法注册和调用unregisterReceiver取消注册

       *注意注册和解绑必须成对在生命周期中出现 否则可能存在内存泄漏、空指针问题

  2. 广播发送者通过Binder机制向AMS发送广播；(sendBroadcast)

  3. AMS查找符合相应条件（IntentFilter/Permission等）的BroadcastReceiver，将广播发送到BroadcastReceiver相应的消息循环队列中；(通常是应用主线程消息队列)

  4. 消息循环执行拿到此广播，回调BroadcastReceiver中的onReceive()方法。（主线程回调注意不要做太多耗时处理）

- 广播的类型

  1. 普通广播

     普通广播是完全异步的，通过Context的sendBroadcast()方法来发送，消息传递效率比较高，但所有receivers（接收器）的执行顺序不确定。缺点是接收器不能将处理结果传递给下一个接收器，并且无法在中途终止广播。

  2. 系统广播

     Android系统中内置了多个系统广播，只要涉及到手机的基本操作，基本上都会发出相应的系统广播。如：开机启动，充电与电量变化，网络状态改变，拍照，屏幕关闭与开启等。每个系统广播都具有特定的intent-filter，其中主要包括具体的action，系统广播发出后，将被相应的BroadcastReceiver接收。

  3. 有序广播

     “有序”是针对广播接收者而言的，指的是发送出去的广播被BroadcastReceiver按照先后循序接收，通过receiver的intent-filter中的android:priority属性来设置优先级，优先级从-1000～1000，数越大，优先级越高；priority属性相同者，动态注册的广播优先。其使用过程与普通广播非常类似，差异仅在于广播的发送方式通过Context.sendOrderedBroadcast()方法发送。

  4. 本地广播(应用内广播LocalBroadcastManager)

     Android中的广播可以跨App直接通信，可能会带来消耗性能和容易引起安全性的问题，为了解决这些问题，将全局广播设置成局部广播或者使用封装好的LocalBroadcastManager(只能动态注册)类。

     *实际上本地广播类似于eventbus等消息总线的实现思路 通过handler来实现消息的发送 在androidX的最新1.1版本中已被删除 不推荐使用了

##### 4.ContentProvider

- 定义

  ContentProvider的作用是**为不同的应用之间数据共享，提供统一的接口**，我们知道安卓系统中应用内部的数据是对外隔离的，要想让其它应用能使用自己的数据（例如通讯录）这个时候就用到了ContentProvider。

- 方法

  ContentProvider是一个抽象类，如果我们需要开发自己的内容提供者我们就需要继承这个类并复写其方法，需要实现的主要方法如下：

  1. onCreate

     在创建ContentProvider时调用 调用时机在application的attachBaseContext之后 oncreate之前 因此有些三方库用这一特征来用contentprovider做初始化操作

  2. query insert delete update 传统增删改查处理 需要注意同步问题

- uri详解

  其它应用可以通过ContentResolver来访问ContentProvider提供的数据，而ContentResolver通过uri来定位自己要访问的数据，所以我们要先了解uri。URI（Universal Resource Identifier）统一资源定位符，如果您使用过安卓的隐式启动就会发现，在隐式启动的过程中我们也是通过uri来定位我们需要打开的Activity并且可以在uri中传递参数。URI的格式如下：
   `[scheme:][//host:port][path][?query]`
   单单看这个可能我们并不知道是什么意思，下面来举个栗子就一目了然了
   URI:`http://www.baidu.com:8080/wenku/jiatiao.html?id=123456&name=jack`
   scheme：根据格式我们很容易看出来scheme为http **在contentprovider中scheme为content://**
   host：[www.baidu.com](http://www.baidu.com) **在contentprovider中即为在manifest中声明时定义的android:authorities**
   port：就是主机名后面path前面的部分为8080 **不需要**
   path：在port后面？的前面为wenku/jiatiao.html **不需要**
   query:?之后的都是query部分为 id=123456$name=jack **不需要**
   uri的各个部分在安卓中都是可以通过代码获取的，下面我们就以上面这个uri为例来说下获取各个部分的方法：
   getScheme() :获取Uri中的scheme字符串部分，在这里是http
   getHost():获取Authority中的Host字符串，即 [www.baidu.com](http://www.baidu.com)
   getPost():获取Authority中的Port字符串，即 8080
   getPath():获取Uri中path部分，即 wenku/jiatiao.html
   getQuery():获取Uri中的query部分，即 id=15&name=du

- *android 7.0以上版本应用内安装升级时流程

  1.安装APK方法

  ```java
  public static void installApk(int versionLevel) {
          //安装
          File file = new File(Environment.getExternalStorageDirectory().getAbsolutePath() + "/IEng-" + versionLevel + ".apk");
          Intent intent = new Intent(Intent.ACTION_VIEW);
      	//Application启动activity必须加该flag
          intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
          Uri data;
          if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {//判断版本大于等于7.0
              // "sven.com.fileprovider.fileprovider"即是在清单文件中配置的authorities
              // 通过FileProvider创建一个content类型的Uri
              String authority = Utils.getApp().getPackageName() + ".fileprovider";
              data = FileProvider.getUriForFile(Utils.getApp(), authority, file);
              intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);// 给目标应用一个临时授权
          } else {
              data = Uri.fromFile(file);
          }
          intent.setDataAndType(data, "application/vnd.android.package-archive");
          Utils.getApp().startActivity(intent);
          android.os.Process.killProcess(android.os.Process.myPid());
      }
  ```

  2.在AndroidManifest.xml 文件中添加provider 如下：

  ```xml
  <provider
              android:name="android.support.v4.content.FileProvider"
              android:authorities="${applicationId}.fileprovider"
              android:exported="false"
              android:grantUriPermissions="true">
              <meta-data
                  android:name="android.support.FILE_PROVIDER_PATHS"
                  android:resource="@xml/file_paths" />
          </provider>
  
  //file_paths
  <paths>
      <root-path
          name="root"
          path="."/>
      <files-path
          name="files"
          path="."/>
  
      <cache-path
          name="cache"
          path="."/>
  
      <external-path
          name="external"
          path="."/>
  
      <external-files-path
          name="external_file_path"
          path="."/>
  
      <external-cache-path
          name="external_cache_path"
          path="."/>
  </paths>
  ```

  3.android 8.0以上需要加入以下权限

  ```xml
  <uses-permission android:name="android.permission.REQUEST_INSTALL_PACKAGES"/>
  ```



#### View