*摘要:近日看公司直播项目,其中有一个功能就是退出某房间之后,直播界面会以悬浮窗的形式出现,并且可以拖动悬浮窗到界面中任意位置,点击悬浮框之后,又可以回到房间中继续观看直播。现在这个功能在主流的直播或者视频类软件中都可以看到，比如：某鱼、某猫、某珠、某牙、某tube。当然了，某tobe当中的悬浮窗效果更佳炫酷，可以炫酷地从悬浮框中将视频主界面慢慢拖动出来，具体效果下载某tube就能看到。这篇文章就记录一下传统悬浮窗播放视频的原理，以及悬浮框涉及到的 Window 和 WindowManager 的相关知识。*
## Window 和 WindowManager 概述
Window 表示一个窗口的概念，在日常开发中直接接触到 Window 的机会并不多，但是在某些特殊的时候，我们需要在桌面上显示一个类似悬浮框的东西（360的小火箭、360手机助手最新版当中桌面上显示的枫叶），那么这种效果就需要用 Window 来实现。Window 是一个抽象类，它的具体实现类是 PhoneWindow，创建一个 Window 跟简单，只需通过 WindowManager 即可完成。
WindowManager 是完结访问 Window 的入口，Window 的具体实现位于 WindowManagerService 中，WindowManager 和 Window 打交道是一个 IPC 过程。
Android 中的**所有**视图都是通过 Window 来呈现的，不管是 Activity 、 Dialog 还是 Toast，他们的实际视图都是附加在 Window 上的，因此，**Window 实际是 View 的直接管理者**。比如说，在事件分发的过程中，点击事件首先是由 Window 传递给 DecorView，然后再由 DecorView 往子 View 分发，最终分发到能够消耗这个点击事件的 View 当中；并且 Activity 生命周期方法 onCreate 中经常调用的 setContentView 方法底层也是通过 Window 来完成的。

## WindowManager
上面概述中提到，要想创建一个 Window ，只需通过 WindowManager 即可实现。

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

上面的代码就将一个 Button 添加到屏幕上（600,600）的位置，对 WindowMnager.Layoutparams做说明：

- Flags 参数表示 Window 的属性，它可以有很多选项，通过这些选项可以控制 Window 的显示特性，比较常用的有：
	**1. FLAG_NOT_FOCUSABLE**<p>
			表示 Window 不需要获取焦点，也不需要接收各种输入事件，此标记会同时启用 FLAG_NOT_TOUCH_MODE，事件会直接传递给下层的具有焦点的 Window。<p>
	**2. FLAG_NOT_TOUCH_MODE**<p>
			







