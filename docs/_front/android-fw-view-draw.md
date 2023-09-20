---
layout: post
title: Android Activity 显示流程 基于 Android10, API 29
categories: android
tags: android viewrootimpl
date: 2023-07-17
---
Android Activity 是如何绘制的。
<!--more-->

从 Activity.handleLaunchActivity() 开始分析
```java
public final class ActivityThread extends ClientTransactionHandler {
    @Override
    public Activity handleLaunchActivity(ActivityClientRecord r,
        PendingTransactionActions pendingActions, Intent customIntent) {
        //
        final Activity a = performLaunchActivity(r, customIntent);

    }

    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        activity.attach(appContext, this, getInstrumentation(), r.token,
                r.ident, app, r.intent, r.activityInfo, title, r.parent,
                r.embeddedID, r.lastNonConfigurationInstances, config,
                r.referrer, r.voiceInteractor, window, r.configCallback,
                r.assistToken);
    }
}

public class Activity {
    private Window mWindow;

    final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window, ActivityConfigCallback activityConfigCallback, IBinder assistToken) {
        mWindow = new PhoneWindow(this, window, activityConfigCallback);

        mWindow.setWindowManager(
                (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                mToken, mComponent.flattenToString(),
                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);

        mWindowManager = mWindow.getWindowManager();
    }
}

public abstract class Window {
    public void setWindowManager(WindowManager wm, IBinder appToken, String appName,
            boolean hardwareAccelerated) {
        if (wm == null) {
            wm = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);
        }

        //WindowManager 与 Window 绑定
        mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
    }
}

//WINDOW_SERVICE 的注册
final class SystemServiceRegistry {
    static {
        registerService(Context.WINDOW_SERVICE, WindowManager.class,
                new CachedServiceFetcher<WindowManager>() {
                    @Override
                    public WindowManager createService(ContextImpl ctx) {
                        return new WindowManagerImpl(ctx);
                    }});
    
    }
}

public final class WindowManagerImpl implements WindowManager {
    private final Context mContext;
    private final Window mParentWindow;

    public WindowManagerImpl createLocalWindowManager(Window parentWindow) {
        return new WindowManagerImpl(mContext, parentWindow);
    }
}
```

setContentView() 做了什么
```java
public class Activity {
    public void setContentView(@LayoutRes int layoutResID) {
        getWindow().setContentView(layoutResID);
        initWindowDecorActionBar();
    }
}

public class PhoneWindow extends Window implements MenuBuilder.Callback {

    public static final int ID_ANDROID_CONTENT = com.android.internal.R.id.content;
    private DecorView mDecor;

    
    public void setContentView(int layoutResID) {
        installDecor();

        //将我们的布局加载到 DecorView 中
        mLayoutInflater.inflate(layoutResID, mContentParent);
    }

    private void installDecor() {
        if (mDecor == null) {
            mDecor = generateDecor(-1);
            mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
            mDecor.setIsRootNamespace(true);
            if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
                mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
            }
        } else {
            mDecor.setWindow(this);
        }

        if (mContentParent == null) {
            mContentParent = generateLayout(mDecor);
        }
    }

    protected ViewGroup generateLayout(DecorView decor) {
        int layoutResource;

        layoutResource = R.layout.screen_simple;

        mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);

        ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);

        return contentParent;
    }

    protected DecorView generateDecor(int featureId) {
        return new DecorView(context, featureId, this, getAttributes());
    }
}

public class DecorView extends FrameLayout implements RootViewSurfaceTaker, WindowCallbacks {

}
```

R.layout.screen_simple
```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true"
    android:orientation="vertical">
    <ViewStub android:id="@+id/action_mode_bar_stub"
              android:inflatedId="@+id/action_mode_bar"
              android:layout="@layout/action_mode_bar"
              android:layout_width="match_parent"
              android:layout_height="wrap_content"
              android:theme="?attr/actionBarTheme" />
    <FrameLayout
         android:id="@android:id/content"
         android:layout_width="match_parent"
         android:layout_height="match_parent"
         android:foregroundInsidePadding="false"
         android:foregroundGravity="fill_horizontal|top"
         android:foreground="?android:attr/windowContentOverlay" />
</LinearLayout>
```

```java
public final class ActivityThread extends ClientTransactionHandler {
    @Override
    public Activity handleLaunchActivity(ActivityClientRecord r,
        PendingTransactionActions pendingActions, Intent customIntent) {
        //
        final Activity a = performLaunchActivity(r, customIntent);

    }

    public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
            String reason) {
        ViewManager wm = a.getWindowManager();
        wm.addView(decor, l);


    }
}

public final class WindowManagerImpl implements WindowManager {
    private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();

    public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        //mParentWindow 就为 PhoneWindow
        mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
    }
}

public final class WindowManagerGlobal {
    public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
        ViewRootImpl root;
        root = new ViewRootImpl(view.getContext(), display);
        root.setView(view, wparams, panelParentView);
    }
}


public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks {
    public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
        //与 windowmanager 打交道
        res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                    getHostVisibility(), mDisplay.getDisplayId(), mTmpFrame,
                    mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                    mAttachInfo.mOutsets, mAttachInfo.mDisplayCutout, mInputChannel,
                    mTempInsets);
        mFragmentManagerState
        requestLayout();
    }

    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread();
            mLayoutRequested = true;
            scheduleTraversals();
        }
    }

    void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            if (!mUnbufferedInputDispatch) {
                scheduleConsumeBatchedInput();
            }
            notifyRendererOfFramePending();
            pokeDrawLockIfNeeded();
        }
    }
}
```

来到编舞者
```java
public final class Choreographer {
    public void postCallback(int callbackType, Runnable action, Object token) {
        postCallbackDelayed(callbackType, action, token, 0);
    }

    public void postCallbackDelayed(int callbackType,
            Runnable action, Object token, long delayMillis) {
        postCallbackDelayedInternal(callbackType, action, token, delayMillis);
    }

    private void postCallbackDelayedInternal(int callbackType,
        Object action, Object token, long delayMillis) {
        scheduleFrameLocked(now);
    }

    private void scheduleFrameLocked(long now) {
        Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_VSYNC);
        msg.setAsynchronous(true);
        mHandler.sendMessageAtFrontOfQueue(msg);
    }

    private final class FrameHandler extends Handler {
        public FrameHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MSG_DO_FRAME:
                    doFrame(System.nanoTime(), 0);
                    break;
                case MSG_DO_SCHEDULE_VSYNC:
                    doScheduleVsync();
                    break;
                case MSG_DO_SCHEDULE_CALLBACK:
                    doScheduleCallback(msg.arg1);
                    break;
            }
        }
    }

    void doScheduleVsync() {
        synchronized (mLock) {
            if (mFrameScheduled) {
                scheduleVsyncLocked();
            }
        }
    }

    private void scheduleVsyncLocked() {
        mDisplayEventReceiver.scheduleVsync();
    }

    private final FrameDisplayEventReceiver mDisplayEventReceiver;

    private final class FrameDisplayEventReceiver extends DisplayEventReceiver
            implements Runnable {
        //接受垂直同步信号
        public void onVsync(long timestampNanos, long physicalDisplayId, int frame) {
            Message msg = Message.obtain(mHandler, this);
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
        }

        @Override
        public void run() {
            mHavePendingVsync = false;
            doFrame(mTimestampNanos, mFrame);
        }
    }

    void doFrame(long frameTimeNanos, int frame) {
        long intendedFrameTimeNanos = frameTimeNanos;
        startNanos = System.nanoTime();
        final long jitterNanos = startNanos - frameTimeNanos;

        //说明掉帧了
        if (jitterNanos >= mFrameIntervalNanos) {
            final long skippedFrames = jitterNanos / mFrameIntervalNanos;
            if (skippedFrames >= SKIPPED_FRAME_WARNING_LIMIT) {
                Log.i(TAG, "Skipped " + skippedFrames + " frames!  "
                        + "The application may be doing too much work on its main thread.");
            }
            final long lastFrameOffset = jitterNanos % mFrameIntervalNanos;
            if (DEBUG_JANK) {
                Log.d(TAG, "Missed vsync by " + (jitterNanos * 0.000001f) + " ms "
                        + "which is more than the frame interval of "
                        + (mFrameIntervalNanos * 0.000001f) + " ms!  "
                        + "Skipping " + skippedFrames + " frames and setting frame "
                        + "time to " + (lastFrameOffset * 0.000001f) + " ms in the past.");
            }
            frameTimeNanos = startNanos - lastFrameOffset;
        }

        if (frameTimeNanos < mLastFrameTimeNanos) {
            if (DEBUG_JANK) {
                Log.d(TAG, "Frame time appears to be going backwards.  May be due to a "
                        + "previously skipped frame.  Waiting for next vsync.");
            }
            scheduleVsyncLocked();
            return;
        }

        doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);
    }

    void doCallbacks(int callbackType, long frameTimeNanos) {
        for (CallbackRecord c = callbacks; c != null; c = c.next) {
            if (DEBUG_FRAMES) {
                Log.d(TAG, "RunCallback: type=" + callbackType
                        + ", action=" + c.action + ", token=" + c.token
                        + ", latencyMillis=" + (SystemClock.uptimeMillis() - c.dueTime));
            }
            c.run(frameTimeNanos);
        }
    }
}


```

```java
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks {
    final TraversalRunnable mTraversalRunnable = new TraversalRunnable();

    final class TraversalRunnable implements Runnable {
        @Override
        public void run() {
            doTraversal();
        }
    }

    void doTraversal() {
        if (mTraversalScheduled) {
            mTraversalScheduled = false;
            mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

            if (mProfile) {
                Debug.startMethodTracing("ViewAncestor");
            }

            performTraversals();

            if (mProfile) {
                Debug.stopMethodTracing();
                mProfile = false;
            }
        }
    }

    private void performTraversals() {
        performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);

        performLayout(lp, mWidth, mHeight);

        performDraw();
    }

    private void performDraw() {
        boolean canUseAsync = draw(fullRedrawNeeded);
    }

    private boolean draw(boolean fullRedrawNeeded) {
        drawSoftware(surface, mAttachInfo, xOffset, yOffset,
                        scalingRequired, dirty, surfaceInsets);
    }

    private boolean drawSoftware(Surface surface, AttachInfo attachInfo, int xoff, int yoff,
            boolean scalingRequired, Rect dirty, Rect surfaceInsets) {
        mView.draw(canvas);
    }
}
```

Activity 绘制 View 的流程
1. ActivityThread.handleLaunchActivity() 调用 activity.attach()，生成 PhoneWindow，并绑定到 WindowManager

2. Activity.onCreate() setContentView() ，生成 DocorView ，其是一个 FrameLayout，并将 R.layout.screen_simple （为 LinearLayout ）添加到布局中，我们自己的布局添加到 com.android.internal.R.id.content 中， 

3. handleResumeActivity() 开始将 docorview 添加到 wm 中

4. wm 为 WindowManagerImpl，最后调用 WindowManagerGlobal.addView() ，在这个方法中开始创建 ViewRootImpl ，使用 root.setView()

5. ViewRootImpl.setView() -> requestLayout()

6. requestLayout() 中调用 scheduleTraversals()，再后面开始使用编舞者，由编舞者来控制界面的刷新，最后回调回调  doTraversal() -> performTraversals() 

7. performMeasure() performLayout() performDraw()