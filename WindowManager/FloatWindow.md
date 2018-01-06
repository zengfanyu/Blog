title: Android中的Window、WindowManager以及悬浮框视频播放的实现
date: 2018-1-2 21:27:15
tags:
- Android
categories: [Android]
---

![](https://i.imgur.com/LmSCVR5.jpg)

*摘要:近日看公司直播项目,其中有一个功能就是退出某房间之后,直播界面会以悬浮窗的形式出现,并且可以拖动悬浮窗到界面中任意位置,点击悬浮框之后,又可以回到房间中继续观看直播。现在这个功能在主流的直播或者视频类软件中都可以看到，比如：某鱼、某猫、某珠、某牙、某tube。当然了，某tobe当中的悬浮窗效果更佳炫酷，可以炫酷地从悬浮框中将视频主界面慢慢拖动出来，具体效果下载某tube就能看到。这篇文章就记录一下传统悬浮窗播放视频的原理，以及悬浮框涉及到的 Window 和  WindowManager 的相关知识。*

<!-- more -->

<font face=黑体>

## Window 和 WindowManager 概述
`Window` 表示一个窗口的概念，在日常开发中直接接触到 `Window` 的机会并不多，但是在某些特殊的时候，我们需要在桌面上显示一个类似悬浮框的东西（360的小火箭、360手机助手最新版当中桌面上显示的枫叶），那么这种效果就需要用 `Window` 来实现。`Window` 是一个抽象类，它的具体实现类是 `PhoneWindow`，创建一个 `Window` 跟简单，只需通过 `WindowManager` 即可完成。
`WindowManager` 是完结访问 `Window` 的入口，`Window` 的具体实现位于 `WindowManagerService` 中`，`WindowManager 和 `Window` 打交道是一个 `IPC` 过程。
`Android` 中的**所有**视图都是通过 `Window` 来呈现的，不管是 `Activity` 、 `Dialog` 还是 `Toast`，他们的实际视图都是附加在 `Window` 上的，因此，**Window 实际是 View 的直接管理者**。比如说，在事件分发的过程中，点击事件首先是由 `Window` 传递给 `DecorView`，然后再由 `DecorView` 往子 `View` 分发，最终分发到能够消耗这个点击事件的 `View` 当中；并且 `Activity` 生命周期方法 `onCreate` 中经常调用的 `setContentView` 方法底层也是通过 `Window` 来完成的。

## 创建一个 Window
上面概述中提到，要想创建一个 `Window` ，只需通过 `WindowManager` 即可实现。

```java

	public void addWindow(){
	        Button button = new Button(getApplicationContext());
	        button.setText("动态添加");
	        WindowManager.LayoutParams layoutParams = new WindowManager.LayoutParams(WindowManager.LayoutParams.WRAP_CONTENT, WindowManager.LayoutParams.WRAP_CONTENT, 0, 0, PixelFormat.TRANSPARENT);
	        layoutParams.flags = WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL
	                | WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
	                | WindowManager.LayoutParams.FLAG_SHOW_WHEN_LOCKED;
	        layoutParams.gravity = Gravity.LEFT | Gravity.TOP;
	        layoutParams.x= 600;
	        layoutParams.y= 600;
	        getWindowManager().addView(button, layoutParams);
	    }

```

上面的代码就将一个 `Button` 添加到屏幕上 `（600,600）` 的位置，对 `WindowMnager.Layoutparams` 常用参数做说明：

```java

	public LayoutParams(int w, int h, int xpos, int ypos, int _type,int _flags, int _format) {
                
```
- `w ,h` 表示 `Window` 的宽高，可以通过构造方法传入，也可以在创建好 `WindowManager.Layoutparams` 之后，直接给其 `width` ，`height` 成员变量赋值。<p>

- `xpos，ypos` 表示 `Window` 在手机屏幕上的绝对位置，与 `w，h` 一样，这两个值也可以在实例化 `WindowManager.Layoutparams` 之后给 `x，y` 成员变量属性赋值，要向更改悬浮窗的位置，就是改变的这两个参数<p>

- `_type` 表示的是 `Window` 的类型，`Window` 有三种类型：<p>
	1. 应用 `Window` ，这个 `Window` 对应着一个 `Activity`，层级范围（1~99）
	2. 子 `Window` ， 不能单独存在，它需要附属在一个特定的父 `Window` 中，比如说 `Dialog` ，层级范围（1000~1999）
	3. 系统 `Window` ，这是需要申明权限才能够创建的 `Window`， 比如说常用的 Toast ,层级范围（2000~2999）。

`TYPE_SYSTEM_OVERLAY（2006），TYPE_TOAST（2005），TYPE_PHONE（2002）`<p>

- `Flags` 参数表示 `Window` 的属性，它可以有很多选项，通过这些选项可以控制 `Window` 的显示特性，比较常用的有：<p>
	**1. FLAG_NOT_FOCUSABLE**
			表示 `Window` 不需要获取焦点，也不需要接收各种输入事件，此标记会同时启用 `FLAG_NOT_TOUCH_MODE`，事件会直接传递给下层的具有焦点的 `Window`。
	**2. FLAG_NOT_TOUCH_MODE**
	此模式下,系统会将当前 `Window` 区域以外的点击事件传递给底层的 `Window` ，当前 `Window` 区域以内的会自己处理，一般来说这个标记都需要开启，不然其他的 `Window` 接收不到单击事件。
	**3. FLAG_SHOW_WHEN_LOCKED**
	让 `Window` 显示在锁屏界面上。
	
	
`WindowManager` 常用的方法就三个：添加 `View`，删除 `View`，更新 `View` 。这三个方法定义在 `ViewManager` 中，`WindowManager` 继承了 `ViewManager`。想做悬浮窗播放视频，就需要用到这三个方法，其中悬浮框随手指拖拽而移动就是在 `onTouchEvent` 回调中调用 `updateView` 的方法。

```java

	public interface ViewManager{
	
		public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params);
		 
		
		public void updateViewLayout(@NonNull View view, @NonNull ViewGroup.LayoutParams params);
		 
	
		public void removeView(View view);
	
	}

```

		
### 实现要点

1. `WindowManager` 和 `Window` 相关，用于展示悬浮框。
2. 要实现悬浮框，那么就会涉及到权限问题，从 `Andtoid 6.0` 开始，需要在运行时去获取悬浮窗的权限。
3. 启动悬浮窗的组件（`Activity` 或者 `Fragment or else`）在启动了悬浮窗之后，自己本身肯定是要关闭的，所以这里悬浮框就很适合在 `Service` 中管理。
4. 悬浮窗一般是可以与用户交互的，那么这里就会涉及到触摸反馈。

### 后续代码前提
- 播放器播放需要一个 `m3u8` url,公司自研播放器代码不贴出。
- 当前 `WatchVideoActivity` 正在全屏播放，此时点击了“悬浮窗播放”按钮。 
- 这里的悬浮窗播放指的是点播,非直播情况

### 清单文件中的权限

```xml

 	<uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW" />

```

### 检查权限并启动Service

```java

		//悬浮窗播放按钮
        final Button button_litter_player = (Button) findViewById(R.id.button_litter_player);
        button_litter_player.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
				//此处为检查用户是否已经授权我们的应用悬浮窗权限
                boolean check = ConstInfo.hasPermissionFloatWin(getApplicationContext());
                if (!check) {
                    Toast.makeText(getApplication(), "悬浮窗权限未打开，请去打开应用悬浮窗权限", Toast.LENGTH_SHORT).show();
                } else {
	                //FloatWindowService 就是用于管理悬浮窗的 Service
                    Intent intent = new Intent(WatchVideoActivity.this, FloatWindowService.class);
                    Bundle bundle = new Bundle();
                    //当前播放视频的m3u8地址
                    bundle.putString("m3u8Url", getCurrentUrl());                              
                   //主要是记录当前播放的位置,这样在悬浮窗出现后,可以接着之前全屏播放的点继续播放
                    bundle.putInt(EXTRA_VIDEO_CURRENT_POSITION, mVideoView.getCurrentPosition());
                    finish();
                }

            }
        });


```

其中检查权限的方法是发射调用:

```java

    /**
     * 判断是否开启浮窗权限,api未公开，使用反射调用
     * @return
     */
    public static boolean hasPermissionFloatWin(Context context) {

        Log.d(ConstInfo.TAG, "hasAuthorFloatWin android.os.Build.VERSION.SDK_INT="+android.os.Build.VERSION.SDK_INT);
        if (android.os.Build.VERSION.SDK_INT < 19) {
            return true;
        }
        try {
            AppOpsManager appOps = (AppOpsManager) context.getSystemService(Context.APP_OPS_SERVICE);
            Class c = appOps.getClass();
            Class[] cArg = new Class[3];
            cArg[0] = int.class;
            cArg[1] = int.class;
            cArg[2] = String.class;
            Method lMethod = c.getDeclaredMethod("checkOp", cArg);
            //24是浮窗权限的标记
            return (AppOpsManager.MODE_ALLOWED == (Integer) lMethod.invoke(appOps, 24, Binder.getCallingUid(), context.getPackageName()));

        }catch(Exception e){
            return false;
        }
    }

```

### FloatWindowService

>Service 详情戳 [Android Developer # Service Guide](https://developer.android.com/guide/components/services.html?hl=zh-cn)

1. `Service` 的 `onCreate` 方法只会在 `Service` 首次创建的时候调用一次,所以在这个方法中创建悬浮框的实例比较适合,因为只支持一个悬浮窗
2. `onStartCommond` 方法在每次调用 `startService` 方法时都会调用,所以在这个方法中适合检查悬浮窗的状态,比如:是否需要退出悬浮窗,还是直接开始在悬浮窗中继续播放等等.
3. `onDestroy` 方法中就直接销毁悬浮窗实例即可.

```java

	public class FloatWindowService extends Service {
	    private static final String TAG = "==FloatWindowService==";
	    public static final String ACTION_PLAY = "com.xxxx.testxxxsdk.FloatWindowService.ACTION_PLAY";
	    public static final String ACTION_EXIT = "com.xxxx.testxxxsdk.FloatWindowService.ACTION_EXIT";
	    public static final String PLAY_TYPE = "com.xxxx.testxxxsdk.FloatWindowService.PLAY_TYPE";
	
	    public static final String EXTRA_VIDEO_LIST = "list";
		//用于标记当前悬浮窗时候已经显示
	    public static boolean mIsFloatWindowShown = false;
	    //悬浮窗实例
	    private FloatWindow mFloatWindow;
	
	    @Override
	    public void onCreate() {
	        super.onCreate();
	        LogUtil.d(TAG,"[onCreate] " + "FloatWindowService onCreate");
	        //这里将Service本身传入悬浮窗,是为了实现点击悬浮窗重新进入WatchVideoActivity 全屏播放,且提供 Context,
	        mFloatWindow = new FloatWindow(this);
	        mFloatWindow.createFloatView();
	        mIsFloatWindowShown = true;
	    }
	
	    @Override
	    public int onStartCommand(Intent intent, int flags, int startId) {
	        LogUtil.d(TAG, "[onStartCommand] " + "FloatWindowService onStart");
	        //此处为特殊逻辑处理,和项目需求相关,不做解释
	        if (intent.hasExtra(ACTION_EXIT)) {
	            stopSelf();
	        } else {
		        //在这里就拿到之前点击悬浮窗按钮时传递过来的数据,包括播放m3u8地址和当前播放位置等
	            Bundle bundle = intent.getBundleExtra(ACTION_PLAY);
	            if (bundle != null && mFloatWindow != null) {
	                LogUtil.d(TAG,"[onStartCommand] " + "FloatWindowService onStart play bundle");
	                //将bundle数据交给悬浮窗控件本身去处理
	                mFloatWindow.play(bundle);
	            }
	        }
	        return START_STICKY;
	    }
	
	    @Override
	    public void onDestroy() {
	        super.onDestroy();
	        LogUtil.d(TAG,"[onDestroy] " +"FloatWindowService onDestroy" );
	        if (mFloatWindow != null) {
	            mFloatWindow.destroy();
	        }
	        mIsFloatWindowShown = false;
	    }
	
	    @Nullable
	    @Override
	    public IBinder onBind(Intent intent) {
	        return null;
	    }
	
	}

```

上述代码可以看到,**Service 在这里就是管理了悬浮窗的生命周期,以及传递数据的作用**.


### FloatWindow

这是**悬浮窗的实现类**,之前的代码在"悬浮播放"这一功能来说,都是铺垫. 参照文章前面对 `WindowManager` 的描述,这里肯定也会涉及到悬浮窗参数和悬浮窗布局,以及悬浮窗的交互.


首先是布局,这列悬浮窗比较简单 `top_window_player`:

```xml

	<?xml version="1.0" encoding="utf-8"?>
	<RelativeLayout android:id="@+id/root_view"
	                xmlns:android="http://schemas.android.com/apk/res/android"
	                android:layout_width="match_parent"
	                android:layout_height="match_parent"
	                android:background="@drawable/top_window_player_bg">
	  
	    <ImageView
	        android:layout_width="wrap_content"
	        android:layout_height="wrap_content"
	        android:layout_centerInParent="true"
	        android:padding="10dp"
	        android:src="@drawable/logo"
	        />
	
	    <ProgressBar
	        android:id="@+id/progressbar_loading"
	        android:layout_width="wrap_content"
	        android:layout_height="wrap_content"
	        android:layout_centerInParent="true"
	        android:indeterminateDrawable="@anim/loading_anim"
	        android:visibility="gone"
	        />
	
	    <com.xxxxxx.xxxsdk.XXXVideoView
	        android:id="@+id/live_player_videoview"
	        android:layout_width="match_parent"
	        android:layout_height="match_parent"
	        android:layout_centerInParent="true"
	        android:visibility="gone"/>
	
	    <ImageButton
	        android:id="@+id/lsq_closeButton"
	        android:layout_width="20dp"
	        android:layout_height="20dp"
	        android:layout_alignParentRight="true"
	        android:background="@drawable/close_small"
	        android:paddingRight="5dp"
	        android:paddingTop="5dp"/>
	</RelativeLayout>

```

 `ImageView` 是用于展示默认状态图,`ImageButton` 为右上角叉叉,`XXXVideoView` 为自研的播放器,这里就不贴出代码了.

构造方法

- 这里将 `Service` 本身传入悬浮窗,是为了实现点击悬浮窗重新进入 `WatchVideoActivity`  全屏播放,
- 提供 `Context`
- 绑定两者生命周期,即悬浮窗销毁时,服务就要停止

```java

    public FloatWindow(Service hostService)
    {
        mHostService = hostService;
        mAppContext = mHostService.getApplication();
    }

```

 `createFloatView()` 真正创建 `Window` 的方法

这个方法中做 3 件事 : 

- 使用 `WindowManager` 创建 `Window` 
- 布局控件初始化
- 触摸反馈

```java

    public void createFloatView() {
        wmParams = new WindowManager.LayoutParams();
        mWindowManager = (WindowManager) mAppContext.getSystemService(mAppContext.WINDOW_SERVICE);
        //即使应用退出,悬浮窗也可以可以再桌面当中显示
        wmParams.type = WindowManager.LayoutParams.TYPE_PHONE;
        wmParams.format = PixelFormat.RGBA_8888;
        //悬浮窗需要自己处理点击事件
        wmParams.flags = WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL | WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE;
        //初始位置在屏幕左边的中间
        wmParams.gravity = Gravity.LEFT | Gravity.CENTER_VERTICAL;
		// 悬浮窗的宽为手机屏幕宽度的三分之一, 4:3 高宽比
        wmParams.width = TestApplication.SCREEN_WIDTH / 3;
        wmParams.height = (wmParams.width / 3) * 4;    
		//Service中的Context
        LayoutInflater inflater = LayoutInflater.from(mAppContext);
        mFloatLayout = (RelativeLayout) inflater.inflate(R.layout.top_window_player, null);
        mWindowManager.addView(mFloatLayout, wmParams);
        
        progressbar_loading = (ProgressBar) mFloatLayout.findViewById(R.id.progressbar_loading);
        ImageButton closebutton = (ImageButton) mFloatLayout.findViewById(R.id.lsq_closeButton);
        closebutton.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                mHostService.stopSelf();
            }
        });


        // 设置悬浮窗的Touch监听
        mFloatLayout.setOnTouchListener(new View.OnTouchListener() {
            int lastX, lastY;
            int paramX, paramY;
            
            public boolean onTouch(View v, MotionEvent event) {
                switch (event.getAction()) {
                    case MotionEvent.ACTION_DOWN:
		                //手指按下的位置
                        lastX = (int) event.getRawX();
                        lastY = (int) event.getRawY();
                        //记录手指按下时,悬浮窗的位置
                        paramX = wmParams.x;
                        paramY = wmParams.y;
                        break;
                    case MotionEvent.ACTION_MOVE:
                        int dx = (int) event.getRawX() - lastX;
                        int dy = (int) event.getRawY() - lastY;
                        wmParams.x = paramX + dx;
                        wmParams.y = paramY + dy;
                        // 更新悬浮窗位置
                        mWindowManager.updateViewLayout(mFloatLayout, wmParams);
                        break;
                    case MotionEvent.ACTION_UP:
                    //当手指按下的位置和手指抬起来的位置距离小于5像素时,将此次触摸归结为点击事件,
                        if (Math.abs(event.getRawX() - lastX) < 5 && Math.abs(event.getRawY() - lastY) < 5)
                            mFloatLayout.callOnClick();
                        break;
                }
                return true;
            }
        });
        
		//设置悬浮窗的点击监听
        mFloatLayout.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {                
					//点击悬浮窗重新跳转回 WatchVideoActivity 全屏播放
                    Intent intent = new Intent(mHostService.getApplicationContext(), WatchVideoActivity.class);
					//同理播放器在WatchVideoActivity 中全屏播放也是需要播放地址和 悬浮窗已经播放到的无照顾
                    intent.putExtra("m3u8Url", mUrl);                 
                    intent.putExtra(EXTRA_VIDEO_CURRENT_POSITION, mVideoView.getCurrentPosition());
	                 //Service 中启动 Activity
                    intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                    mHostService.startActivity(intent);
                    //销毁服务
                    mHostService.stopSelf();
            }
        });

        mVideoView = (XXXVideoView) mFloatLayout.findViewById(R.id.live_player_videoview);
        //初始化播放器
        mVideoView.initialize();
        //监听播放器 播放器相关不贴出代码
        mVideoView.setListener(mVideoListener);
		
    }

```

开始播放的方法

```java

    public void play(Bundle param) {
        mBundleParam = param;
        if (mBundleParam == null)
            return;
            
        //拿到从 WatchVideoActivity 中传递过来的 播放地址
        mUrl = mBundleParam.getString("m3u8Url");
        //拿到从 WatchVideoActivity 中传递过来的当前播放位置,以便继续播放
        mCurrPositionFromWatchVod = mBundleParam.getInt(WatchVideoActivity.EXTRA_VIDEO_CURRENT_POSITION, -1);
        //播放器相关,省略部分代码
        stop_play();
        start_play();
    }

```


## 效果

![](https://i.imgur.com/dVuUslE.gif)

## 相关知识点学习资料

1. [Android 悬浮窗参数权限的小结](https://www.liaohuqiu.net/cn/posts/android-windows-manager/),这篇文章写得时间较早,其中有点内容在我测试机 红米Note 4X 当中并没有办法验证,索性,还是需要向用户申请悬浮窗权限.
2. [Service 官方文档](https://developer.android.com/guide/components/services.html?hl=zh-cn) 
3. [运行时权限官方文档](https://developer.android.com/training/permissions/requesting.html?hl=zh-cn) ,  [鸿洋大神------ Android 6.0 运行时权限处理完全解析](http://blog.csdn.net/lmj623565791/article/details/50709663)
4. [ Android 带你彻底理解 Window 和 WindowManager](http://blog.csdn.net/yhaolpz/article/details/68936932) ,  [《Android 开发艺术探索》 08-理解Window和WindowManager
抄书系列](http://szysky.com/2016/08/15/%E3%80%8AAndroid-%E5%BC%80%E5%8F%91%E8%89%BA%E6%9C%AF%E6%8E%A2%E7%B4%A2%E3%80%8B-08-%E7%90%86%E8%A7%A3Window%E5%92%8CWindowManager/)