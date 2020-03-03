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

#### apk打包流程

![android_build.png](https://user-gold-cdn.xitu.io/2017/3/2/35a4d886bc51ec6be29456eadd4b1fd2.png?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

- 通过aapt打包res资源文件，生成R.java、resources.arsc和res文件（二进制 & 非二进制如res/raw和pic保持原样）
- 处理.aidl文件，生成对应的Java接口文件
- 通过Java Compiler编译R.java、Java接口文件、Java源文件，生成.class文件
- 通过dex命令，将.class文件和第三方库中的.class文件处理生成classes.dex
- 通过apkbuilder工具，将aapt生成的resources.arsc和res文件、assets文件和classes.dex一起打包生成apk
- 通过Jarsigner工具，对上面的apk进行debug或release签名
- 通过zipalign工具，将签名后的apk进行对齐处理。

#### View

##### 1.Activity、PhoneWindow和DecorView

![Activity、PhoneWindow和DecorView](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8xODM0MjQzMC03MjY3YjY3Y2EwYmMwOWI1LnBuZw)

- PhoneWindow

  每一个Activity都包含一个Window对象，Window对象通常由PhoneWindow实现，内部包含了一个DecorView对象，该DectorView对象是所有应用窗口(Activity界面)的根View。 简而言之，PhoneWindow类是把一个FrameLayout类即DecorView对象进行一定的包装，将它作为应用窗口的根View，并提供一组通用的窗口操作接口。每个Activity 均会创建一个PhoneWindow对象，是Activity和整个View系统交互的接口。

- DecorView

  该类是一个FrameLayout的子类,实际上就是对普通FrameLayout的包装扩展，比如说添加TitleBar(标题栏)，以及TitleBar上的滚动条等。其核心功能如下：

  - Dispatch ViewRoot分发来的key、touch、trackball等外部事件

  - DecorView有一个直接的子View，我们称之为System Layout,这个View是从系统的Layout.xml中解析出的，它包含当前UI的风格，如是否带title、是否带process bar等。可以称这些属性为Window decorations

  - 作为PhoneWindow与ViewRoot之间的桥梁，ViewRoot通过DecorView设置窗口属性。

  - DecorView只有一个子元素为LinearLayout。代表整个Window界面，包含通知栏，标题栏，内容显示栏三块区域。

    TitleView：标题，可以设置requestWindowFeature(Window.FEATURE_NO_TITLE)取消

    ContentView：是一个id为content的FrameLayout。我们平常在Activity使用的setContentView就是设置在这里

##### 2.View事件分发机制

![view事件分发机制](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8xODM0MjQzMC01ZjVkOGU2ZTgzMjNjMGRiLnBuZw)

如图可分为三种情况 从上到下依次为activity viewgroup view

- Activity

  ```java
  public boolean dispatchTouchEvent(MotionEvent ev) {
          if (ev.getAction() == MotionEvent.ACTION_DOWN) {
              onUserInteraction();
          }
      	//获取到当前页面的phonewindow对象并交给decorview处理
          if (getWindow().superDispatchTouchEvent(ev)) {
              return true;
          }
          return onTouchEvent(ev);
      }
  ```

  activity实现了window的callback接口 因此当wms将view事件分发到app时会从activity的dispatchTouchEvent方法开始事件分发流程 activity会将事件传递到当前页面的window实现类phonewindow

  中 而phonewindow又将该事件传递给自己的内部变量decorview 本质上decorview是一个ViewGroup 因此后续的时间处理机制就和ViewGroup一致了

- ViewGroup

  1. 如果在viewgroup的dispatchTouchEvent中直接返回false则将事件交给父viewgroup的ontouchevent处理 
  2. 传递给viewgroup的onInterceptTouchEvent去判断是否拦截事件 如果onInterceptTouchEvent返回true则交给viewgroup自身的ontouchevent去处理
  3. onInterceptTouchEvent返回false的话则传递给子view或viewgroup的dispatchTouchEvent去处理

- View

  1. view的dispatchTouchEvent如果返回false则将事件回传给view的父viewgroup的ontouchevent处理
  2. view的dispatchTouchEvent如果返回true则执行view自身的ontouchevent处理

- View自身的事件分发流程

  dispatchTouchEvent -> onTouch(setOnTouchListener) -> onTouchEvent -> onClick

  onTouch和onTouchEvent的区别 两者都是在dispatchTouchEvent中调用的，onTouch优先于onTouchEvent，如果onTouch返回true，那么onTouchEvent则不执行，及onClick也不执行。

  ```java
  public boolean dispatchTouchEvent(MotionEvent event) {
        	.....
          final int actionMasked = event.getActionMasked();
          if (actionMasked == MotionEvent.ACTION_DOWN) {
              // Defensive cleanup for new gesture
              stopNestedScroll();
          }
  		......
          if (onFilterTouchEventForSecurity(event)) {
              if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
                  result = true;
              }
              //重点在这里 如果view设置了OnTouchListener优先处理onTouch
              //当OnTouchListener的onTouch方法返回true的情况下 则根据&&熔断特征跳过
              //onTouchEvent的执行
              ListenerInfo li = mListenerInfo;
              if (li != null && li.mOnTouchListener != null
                      && (mViewFlags & ENABLED_MASK) == ENABLED
                      && li.mOnTouchListener.onTouch(this, event)) {
                  result = true;
              }
  
              if (!result && onTouchEvent(event)) {
                  result = true;
              }
          }
  
          if (actionMasked == MotionEvent.ACTION_UP ||
                  actionMasked == MotionEvent.ACTION_CANCEL ||
                  (actionMasked == MotionEvent.ACTION_DOWN && !result)) {
              stopNestedScroll();
          }
  
          return result;
      }
  ```

- view的MotionEvent

  - ACTION_DOWN

    当屏幕检测到触点按下之后就会触发到这个事件(手指按下)

  - ACTION_MOVE

    当触点在屏幕上移动时触发(手指滑动)

  - ACTION_UP

    当触点松开时被触发(手指抬起)

  - ACTION_CANCEL

    不是由用户直接触发，由系统在需要的时候触发，例如当父view通过使函数onInterceptTouchEvent()返回true,从子view拿回处理事件的控制权时，就会给子view发一个ACTION_CANCEL事件，子view就再也不会收到后续事件了。

    *比如B viewgroup包含A view 假设在A处手指按下 然后滑动出A 这时当滑动出A的瞬间A就会收到B发来的 ACTION_CANCEL事件 那么滑出后的ACTION_MOVE和ACTION_UP便不会传递给A去处理而是由B直接去处理

###### View事件冲突解决方案

- 外部拦截法

  即父View根据需要对事件进行拦截。逻辑处理放在父View的onInterceptTouchEvent方法中。我们只需要重写父View的onInterceptTouchEvent方法，并根据逻辑需要做相应的拦截即可。

  - 根据业务逻辑需要，在ACTION_MOVE方法中进行判断，如果需要父View处理则返回true，否则返回false，事件分发给子View去处理。
  - ACTION_DOWN 一定返回false，不要拦截它，否则根据View事件分发机制，后续ACTION_MOVE 与 ACTION_UP事件都将默认交给父View去处理。
  - 原则上ACTION_UP也需要返回false，如果返回true，并且滑动事件交给子View处理，那么子View将接收不到ACTION_UP事件，子View的onClick事件也无法触发。而父View不一样，如果父View在ACTION_MOVE中开始拦截事件，那么后续ACTION_UP也将默认交给父View处理。

  ```java
  public boolean onInterceptTouchEvent(MotionEvent event) {
      boolean intercepted = false;
      int x = (int) event.getX();
      int y = (int) event.getY();
      switch (event.getAction()) {
          case MotionEvent.ACTION_DOWN: {
              intercepted = false;
              break;
          }
          case MotionEvent.ACTION_MOVE: {
              //比如对于viewpager嵌套listview来说
              //Math.abs(x-mLastXIntercept) >Math.abs(y-mLastYIntercept)
              //表示X轴滑动距离大于Y轴 这时事件由viewpager自己去处理 所以拦截了
              if (满足父容器的拦截要求) {
                  intercepted = true;
              } else {
                  intercepted = false;
              }
              break;
          }
          case MotionEvent.ACTION_UP: {
              intercepted = false;
              break;
          }
          default:
              break;
      }
      mLastXIntercept = x;
      mLastYIntercept = y;
      return intercepted;
  }
  
  ```

- 内部拦截法

  即父View不拦截任何事件，所有事件都传递给子View，子View根据需要决定是自己消费事件还是给父View处理。这需要子View使用requestDisallowInterceptTouchEvent方法才能正常工作。

  - 内部拦截法要求父View不能拦截ACTION_DOWN事件，由于ACTION_DOWN不受FLAG_DISALLOW_INTERCEPT标志位控制，一旦父容器拦截ACTION_DOWN那么所有的事件都不会传递给子View。
  - 滑动策略的逻辑放在子View的dispatchTouchEvent方法的ACTION_MOVE中，如果父容器需要获取点击事件则调用 parent.requestDisallowInterceptTouchEvent(false)方法，让父容器去拦截事件。

  ```java
  //子view
  public boolean dispatchTouchEvent(MotionEvent event) {
      int x = (int) event.getX();
      int y = (int) event.getY();
  
      switch (event.getAction()) {
          case MotionEvent.ACTION_DOWN: {
              //通知父view不要拦截后续事件
              parent.requestDisallowInterceptTouchEvent(true);
              break;
          }
          case MotionEvent.ACTION_MOVE: {
              int deltaX = x - mLastX;
              int deltaY = y - mLastY;
              //比如对于viewpager嵌套listview来说
              //Math.abs(deltaX) > Math.abs(deltaY)
              //表示X轴滑动距离大于Y轴 这时事件要由viewpager自己去处理 所以通知viewpager拦截
              if (父容器需要此类点击事件) {
                  parent.requestDisallowInterceptTouchEvent(false);
              }
              break;
          }
          case MotionEvent.ACTION_UP: {
              break;
          }
          default:
              break;
      }
      mLastX = x;
      mLastY = y;
      return super.dispatchTouchEvent(event);
  }
  
  //父view
  public boolean onInterceptTouchEvent(MotionEvent event) {
      int action = event.getAction();
      //不能拦截down事件 否则子view收不到任何事件分发
      if (action == MotionEvent.ACTION_DOWN) {
          return false;
      } else {
          return true;
      }
  }
  ```

##### 3.View绘制

- 整体绘制流程

  - 在**Activity**的`onResume`之后，当前**Activity**的**Window**对象中的View(DecorView)会被添加在**WindowManager**中。也就是在**ActivityThread**的`handleResumeActivity`方法中调用`wm.addView(decor, l);`将DecorView添加到**WindowManager**中；

    **WindowManager**继承**ViewManager**，它的实现类为**WindowManagerImpl**，该类中的方法的具体实现是由其代理类**WindowManagerGlobal**实现的；

    在它的`addView`方法中会创建**ViewRootImpl**的实例，然后将Window对应的View(DecorView)，ViewRootImpl，LayoutParams顺序添加在WindowManager中，最后将Window所对应的View设置给创建的ViewRootImpl，通过**ViewRootImpl**来更新界面并完成Window的添加过程；

    ```java
    public final class WindowManagerGlobal {
        /*******部分代码省略**********/
        public void addView(View view, ViewGroup.LayoutParams params,
                Display display, Window parentWindow) {
             /*******部分代码省略**********/
            final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams)params;
            //声明ViwRootImpl
            ViewRootImpl root;
            View panelParentView = null;
            synchronized (mLock) {
                // Start watching for system property changes.
                if (mSystemPropertyUpdater == null) {
                    mSystemPropertyUpdater = new Runnable() {
                        @Override public void run() {
                            synchronized (mLock) {
                                for (int i = mRoots.size() - 1; i >= 0; --i) {
                                    mRoots.get(i).loadSystemProperties();
                                }
                            }
                        }
                    };
                    SystemProperties.addChangeCallback(mSystemPropertyUpdater);
                }
                /*******部分代码省略**********/
                //创建ViwRootImpl
                root = new ViewRootImpl(view.getContext(), display);
                view.setLayoutParams(wparams);
                //将Window所对应的View、ViewRootImpl、LayoutParams顺序添加在WindowManager中
                mViews.add(view);
                mRoots.add(root);
                mParams.add(wparams);
            }
            try {
                //把将Window所对应的View设置给创建的ViewRootImpl
                //通过ViewRootImpl来更新界面并完成Window的添加过程。
                root.setView(view, wparams, panelParentView);
            } catch (RuntimeException e) {
                /*******部分代码省略**********/
            }
        }
    }
    ```

  - ##### requestLayout()方法请求view绘制 最终会逐层向上传递给ViewRootImpl调用其requestLayout方法

    ```java
    public final class ViewRootImpl implements ViewParent,
            View.AttachInfo.Callbacks, HardwareRenderer.HardwareDrawCallbacks {
        /*******部分代码省略**********/
        //请求对界面进行布局
        @Override
        public void requestLayout() {
            if (!mHandlingLayoutInLayoutRequest) {
                checkThread();
                mLayoutRequested = true;
                scheduleTraversals();
            }
        }
        /*******部分代码省略**********/
        //安排任务
        void scheduleTraversals() {
            if (!mTraversalScheduled) {
                mTraversalScheduled = true;
                mTraversalBarrier = mHandler.getLooper().postSyncBarrier();
                //异步回调执行
                mChoreographer.postCallback(
                        Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
                if (!mUnbufferedInputDispatch) {
                    scheduleConsumeBatchedInput();
                }
                notifyRendererOfFramePending();
            }
        }
    
        final TraversalRunnable mTraversalRunnable = new TraversalRunnable();
    
        final class TraversalRunnable implements Runnable {
            @Override
            public void run() {
                doTraversal();
            }
        }
        //做任务
        void doTraversal() {
            if (mTraversalScheduled) {
                mTraversalScheduled = false;
                mHandler.getLooper().removeSyncBarrier(mTraversalBarrier);
                if (mProfile) {
                    Debug.startMethodTracing("ViewAncestor");
                }
                Trace.traceBegin(Trace.TRACE_TAG_VIEW, "performTraversals");
                try {
                    //执行任务
                    performTraversals();
                } finally {
                    Trace.traceEnd(Trace.TRACE_TAG_VIEW);
                }
    
                if (mProfile) {
                    Debug.stopMethodTracing();
                    mProfile = false;
                }
            }
        }
    }
    
    //performTraversals
    private void performTraversals() {
            ......
            //最外层的根视图的widthMeasureSpec和heightMeasureSpec由来
            //lp.width和lp.height在创建ViewGroup实例时等于MATCH_PARENT
            //这里的根视图即为mdecorview
            int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
            int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
            ......
            performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
            ......
            performLayout(lp, desiredWindowWidth, desiredWindowHeight);
            ......
            performDraw();
            ......
     }
    ```

    - performMeasure

      ![performMeasure](https://upload-images.jianshu.io/upload_images/1797490-b959ad2d9c484d5e.png?imageMogr2/auto-orient/strip|imageView2/2/w/605/format/webp)

      循环递归遍历整个view树从顶层依次开始measure到底层最终确定各view尺寸

      执行顺序如下

      ViewRootImpl的`performMeasure`;

      DecorView(FrameLayout)的`measure`;

      DecorView(FrameLayout)的`onMeasure`;

      DecorView(FrameLayout)所有子View的`measure`；

      `measure`操作完成后得到的是对每个**View**经测量过的**measuredWidth和measuredHeight**

      `getMeasuredWidth()、getMeasuredHeight()`必须在`onMeasure`之后使用才有效；

    - performLayout

      执行顺序如下

      ViewRootImpl的performLayout。

      DecorView(FrameLayout)的layout方法。

      DecorView(FrameLayout)的onLayout方法。

      DecorView(FrameLayout)的layoutChildren方法。

      DecorView(FrameLayout)的所有子View的Layout。

      当**ViewRootImpl**的`performTraversals`中`performMeasure`执行完成以后会接着执行`performLayout`，**ViewRootImpl**调用`performLayout`执行Window对应的View的布局。

      `View.layout`方法可被重载，`ViewGroup.layout`为**final**的不可重载，`ViewGroup.onLayout`为**abstract**的，子类必须重载实现自己的位置逻辑。`View.onLayout`方法是一个空方法。

      layout操作完成之后得到的是对每个View进行位置分配后的mLeft、mTop、mRight、mBottom

      `getWidth()与getHeight()`方法必须在`layout(int l, int t, int r, int b)`执行之后才有效。

    - performDraw

      执行顺序如下

      ViewRootImpl的`performDraw`；

      ViewRootImpl的`draw`；

      ViewRootImpl的`drawSoftware`；

      DecorView(FrameLayout)的`draw`方法；

      DecorView(FrameLayout)的`dispatchDraw`方法；

      DecorView(FrameLayout)的`drawChild`方法；

      DecorView(FrameLayout)的所有子View的`draw`方法；

  - 

- Measure

  - MeasureSpec mode

    | 测量模式    | 描述                                                         |
    | ----------- | ------------------------------------------------------------ |
    | UNSPECIFIED | 父容器对当前view大小没有任何限制，当前view可取任意尺寸       |
    | AT_MOST     | 当前尺寸是view能取得最大尺寸（对应wrap_content，不大于父容器） |
    | EXACTLY     | 当前尺寸就是view的尺寸（对应match_parent、具体size数值）     |

  - onMeasure

    ```java
    private int getMySize(int defaultSize, int measureSpec) {
            int mySize = defaultSize;//默认大小
        	
    		//获取父viewgroup分发下来的测量模式和尺寸（实际上一般就是xml中定义的）
            int mode = MeasureSpec.getMode(measureSpec);
            int size = MeasureSpec.getSize(measureSpec);
    
            switch (mode) {
                case MeasureSpec.UNSPECIFIED: {//如果没有指定大小，就设置为默认大小 
                    mySize = defaultSize;
                    break;
                }
                case MeasureSpec.AT_MOST: {//如果测量模式是最大取值为size
                    //wrap_content
                    mySize = size;
                    break;
                }
                case MeasureSpec.EXACTLY: {//如果是固定的大小，那就不要去改变它
                    //match_parent、具体size数值
                    mySize = size;
                    break;
                }
            }
            return mySize;
    }
    
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
            super.onMeasure(widthMeasureSpec, heightMeasureSpec);
            int width = getMySize(100, widthMeasureSpec);
            int height = getMySize(100, heightMeasureSpec);
    		//调用该方法将实际view尺寸设定给view
            setMeasuredDimension(width, height);
    }
    ```

- layout

  viewgroup特有 在自定义viewgroup布局时会用到

  - onLayout

    ```java
    Override
        protected void onLayout(boolean changed, int l, int t, int r, int b) {
            int count = getChildCount();
            //记录当前的高度位置
            int curHeight = t;
            //将子View逐个摆放
            for (int i = 0; i < count; i++) {
                View child = getChildAt(i);
                //拿到每个子view的宽高
                int height = child.getMeasuredHeight();
                int width = child.getMeasuredWidth();
                //摆放子View，参数分别是子View矩形区域的左、上、右、下边
                child.layout(l, curHeight, l + width, curHeight + height);
                curHeight += height;
            }
        }
    ```

- draw

  篇幅过长 参考[gcssloop自定义view教程](https://www.gcssloop.com/customview/CustomViewIndex/)

###### requestLayout()、invalidate()、postInvalidate()的区别

![img](https://images2015.cnblogs.com/blog/554581/201706/554581-20170622000240538-1481075813.png)

- requestLayout

  requestLayout方法只会导致当前view的measure和layout，而draw不一定被执行，只有当view的位置发生改变才会执行draw方法，因此如果要使当前view重绘需要调用invalidate。

- invalidate

  当View的大小以及位置没有发生改变的时候,只会重新绘制该View而不会重新进行测量和布局。

- postInvalidate

  postInvalidate在非UI线程中调用 通过handler切换到主线程后 最终都会调用invalidateInternal

##### 4.View滑动

- scrollTo/scrollBy

  scrollTo（x，y） x y是相对于view中心将view中的内容滚动绝对距离 一次生效后不会再次改变位置

  scrollby（x，y） x y是相对于内容自身滚动相对距离 可多次生效

- scroller

  用于平滑滑动和惯性滑动

  1. 创建scroller实例
  2. 调用startScroll()方法来初始化滚动数据并刷新界面invalite
  3. 重写view的computeScroll()

  ```java
  public class ScrollerLayout extends ViewGroup {
  
      /**
       * 用于完成滚动操作的实例
       */
      private Scroller mScroller;
  
      /**
       * 判定为拖动的最小移动像素数
       */
      private int mTouchSlop;
  
      /**
       * 手机按下时的屏幕坐标
       */
      private float mXDown;
  
      /**
       * 手机当时所处的屏幕坐标
       */
      private float mXMove;
  
      /**
       * 上次触发ACTION_MOVE事件时的屏幕坐标
       */
      private float mXLastMove;
  
      /**
       * 界面可滚动的左边界
       */
      private int leftBorder;
  
      /**
       * 界面可滚动的右边界
       */
      private int rightBorder;
  
      public ScrollerLayout(Context context, AttributeSet attrs) {
          super(context, attrs);
          // 第一步，创建Scroller的实例
          mScroller = new Scroller(context);
          ViewConfiguration configuration = ViewConfiguration.get(context);
          // 获取TouchSlop值
          mTouchSlop = ViewConfigurationCompat.getScaledPagingTouchSlop(configuration);
      }
  
      @Override
      protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
          super.onMeasure(widthMeasureSpec, heightMeasureSpec);
          int childCount = getChildCount();
          for (int i = 0; i < childCount; i++) {
              View childView = getChildAt(i);
              // 为ScrollerLayout中的每一个子控件测量大小
              measureChild(childView, widthMeasureSpec, heightMeasureSpec);
          }
      }
  
      @Override
      protected void onLayout(boolean changed, int l, int t, int r, int b) {
          if (changed) {
              int childCount = getChildCount();
              for (int i = 0; i < childCount; i++) {
                  View childView = getChildAt(i);
                  // 为ScrollerLayout中的每一个子控件在水平方向上进行布局
                  childView.layout(i * childView.getMeasuredWidth(), 0, (i + 1) * childView.getMeasuredWidth(), childView.getMeasuredHeight());
              }
              // 初始化左右边界值
              leftBorder = getChildAt(0).getLeft();
              rightBorder = getChildAt(getChildCount() - 1).getRight();
          }
      }
  
      @Override
      public boolean onInterceptTouchEvent(MotionEvent ev) {
          switch (ev.getAction()) {
              case MotionEvent.ACTION_DOWN:
                  mXDown = ev.getRawX();
                  mXLastMove = mXDown;
                  break;
              case MotionEvent.ACTION_MOVE:
                  mXMove = ev.getRawX();
                  float diff = Math.abs(mXMove - mXDown);
                  mXLastMove = mXMove;
                  // 当手指拖动值大于TouchSlop值时，认为应该进行滚动，拦截子控件的事件
                  if (diff > mTouchSlop) {
                      return true;
                  }
                  break;
          }
          return super.onInterceptTouchEvent(ev);
      }
  
      @Override
      public boolean onTouchEvent(MotionEvent event) {
          switch (event.getAction()) {
              case MotionEvent.ACTION_MOVE://拦截事件后自身onTouchEvent处理
                  mXMove = event.getRawX();
                  int scrolledX = (int) (mXLastMove - mXMove);//手指滑动距离
                  if (getScrollX() + scrolledX < leftBorder) {//滑动到最左边
                      scrollTo(leftBorder, 0);
                      return true;
                  } else if (getScrollX() + getWidth() + scrolledX > rightBorder) {//滑动到最右边
                      scrollTo(rightBorder - getWidth(), 0);
                      return true;
                  }
                  //相对于当前内容位置滑动手指一动距离
                  scrollBy(scrolledX, 0);
                  mXLastMove = mXMove;
                  break;
              case MotionEvent.ACTION_UP:
                  // 当手指抬起时，根据当前的滚动值来判定应该滚动到哪个子控件的界面
                  int targetIndex = (getScrollX() + getWidth() / 2) / getWidth();
                  int dx = targetIndex * getWidth() - getScrollX();
                  // 第二步，调用startScroll()方法来初始化滚动数据并刷新界面
                  //记录当前位置和要滑动的距离
                  mScroller.startScroll(getScrollX(), 0, dx, 0);
                  //关键点必须调用invalidate刷新否则不会触发computeScroll
                  invalidate();
                  break;
          }
          return super.onTouchEvent(event);
      }
  
      @Override
      public void computeScroll() {
          // 第三步，重写computeScroll()方法，并在其内部完成平滑滚动的逻辑
          if (mScroller.computeScrollOffset()) {//需要进行滑动处理
              //滑动到指定的绝对位置点
              scrollTo(mScroller.getCurrX(), mScroller.getCurrY());
              //刷新
              invalidate();
          }
      }
  }
  ```

- 动画

  使用属性动画和补间动画均可 区别就是这两种动画本身的区别

- 修改布局参数

  修改布局参数后让父viewgroup在调用子viewlayout时修改位置

- 父布局使用layout修改view布局位置

  原理同修改布局参数 只是把逻辑挪到了父viewgroup去实现

- viewDragHelper

  android自带的view拖动辅助类 可实现复杂的view拖动 弹性回弹 组合滑动的效果

##### 5.View相关小知识点

- getX和getRawX区别

  getX表示触摸点距离自身左边界的距离
  getRawX表示触摸点距离屏幕左边界的距离

- 

#### 动画

##### 1.帧动画

##### 2.补间动画

- 基础特征
  - **功能：**可以实现移动、旋转、缩放、渐变四种效果以及这四种效果的组合形式。
  - **实现形式**：xml和代码。
  - **优点**：使用简单效果流畅。
  - **缺点**：
    1. 扩展性差，不支持自定义view
    2. 动画只改变控件在屏幕的位置，不改变控件的实际属性。典型例子：Button执行完动画移动到另外位置，点击事件还在原来的地方。

- 简单使用

- 原理概述

  1. 一般是调用View.startAnimation()开始动画

     ```java
     public void startAnimation(Animation animation) {
             animation.setStartTime(Animation.START_ON_FIRST_FRAME);
             setAnimation(animation);
             invalidateParentCaches();
             invalidate(true);
         }
     //首先使用setAnimation将传入的动画类设置给自身的内部变量
     //然后调用invalidate刷新draw
     
     //boolean draw(Canvas canvas, ViewGroup parent, long drawingTime)
     final Animation a = getAnimation();
     if (a != null) {
         more = applyLegacyAnimation(parent, drawingTime, a, scalingRequired);
         concatMatrix = a.willChangeTransformationMatrix();
         if (concatMatrix) {
             mPrivateFlags3 |= PFLAG3_VIEW_IS_ANIMATING_TRANSFORM;
         }
         transformToApply = parent.getChildTransformation();
     }
     ```

  2. 

- 

  

##### 3.属性动画

