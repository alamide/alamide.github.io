---
layout: post
title: Android Fragment
categories: android
tags: android fragment
date: 2023-07-10
---
Fragment 表示应用界面中可重复使用的一部分。可以定义和管理自己的布局，有自己的生命周期，可以处理自己的输入事件。Fragment 不能独立存在，需要依附于 Activity 或其它 Fragment。基于 Android 10 API 29

本次学习目标：
1. Fragment 的基本使用

2. Fragment 如何将布局添加到 Activity 中的 

3. Fragment 从源码的角度理解生命周期

4. Actvity 如何管理 Fragment 

5. Fragment 常见的 getActivity() 为 NULL 什么原因，如何解决

6. Fragment 的懒加载
<!--more-->
 
## 1.FragmentManager
```java
public class Activity extends ContextThemeWrapper
        implements LayoutInflater.Factory2,
        Window.Callback, KeyEvent.Callback,
        OnCreateContextMenuListener, ComponentCallbacks2,
        Window.OnWindowDismissedCallback,
        ContentCaptureManager.ContentCaptureClient {
    final FragmentController mFragments = FragmentController.createController(new HostCallbacks());

    final void performResume(boolean followedByPause, String reason) {
        mFragments.dispatchResume();
        mFragments.execPendingActions();
    }

    class HostCallbacks extends FragmentHostCallback<Activity> {
        
    }
}

public abstract class FragmentHostCallback<E> extends FragmentContainer {
    final FragmentManagerImpl mFragmentManager = new FragmentManagerImpl();


}

final class FragmentManagerImpl extends FragmentManager implements LayoutInflater.Factory2 {
    public void dispatchResume() {
        mStateSaved = false;
        dispatchMoveToState(Fragment.RESUMED);
    }

    private void dispatchMoveToState(int state) {
        if (mAllowOldReentrantBehavior) {
            moveToState(state, false);
        } else {
            try {
                mExecutingActions = true;
                moveToState(state, false);
            } finally {
                mExecutingActions = false;
            }
        }
        execPendingActions();
    }

    void moveToState(int newState, boolean always) {
        mCurState = newState;

        if (mActive != null) {
            boolean loadersRunning = false;

            // Must add them in the proper order. mActive fragments may be out of order
            final int numAdded = mAdded.size();
            for (int i = 0; i < numAdded; i++) {
                Fragment f = mAdded.get(i);
                moveFragmentToExpectedState(f);
                if (f.mLoaderManager != null) {
                    loadersRunning |= f.mLoaderManager.hasRunningLoaders();
                }
            }
        }
    }
}

public class FragmentController {
    private final FragmentHostCallback<?> mHost;

    public static final FragmentController createController(FragmentHostCallback<?> callbacks) {
        return new FragmentController(callbacks);
    }

    private FragmentController(FragmentHostCallback<?> callbacks) {
        mHost = callbacks;
    }

    public void dispatchResume() {
        mHost.mFragmentManager.dispatchResume();
    }
}
```

在 Activity 生命周期变化时，FragmentManager 的 mCurState 也会不断的变化，而 FragmentManager 所管理的 Fragment 也会做响应的变化

而在 Activity resume 之后，新添加的 Fragment 的生命周期如下
```java
getSupportFragmentManager()
                        .beginTransaction()
                        .add(R.id.fl_container, fragments[0], "Tab1")
                        .commit();

public abstract class FragmentManager implements FragmentResultOwner {
    private final ArrayList<OpGenerator> mPendingActions = new ArrayList<>();

    public FragmentTransaction beginTransaction() {
        return new BackStackRecord(this);
    }

    void enqueueAction(@NonNull OpGenerator action, boolean allowStateLoss) {
        mPendingActions.add(action);
        scheduleCommit();
    }

    void scheduleCommit() {
        synchronized (mPendingActions) {
            boolean pendingReady = mPendingActions.size() == 1;
            if (pendingReady) {
                mHost.getHandler().removeCallbacks(mExecCommit);
                mHost.getHandler().post(mExecCommit);
                updateOnBackPressedCallbackEnabled();
            }
        }
    }

    private Runnable mExecCommit = new Runnable() {
        @Override
        public void run() {
            execPendingActions(true);
        }
    };

    boolean execPendingActions(boolean allowStateLoss) {
        while (generateOpsForPendingActions(mTmpRecords, mTmpIsPop)) {
            mExecutingActions = true;
            try {
                removeRedundantOperationsAndExecute(mTmpRecords, mTmpIsPop);
            } finally {
                cleanupExec();
            }
            didSomething = true;
        }
    }

    private void removeRedundantOperationsAndExecute(@NonNull ArrayList<BackStackRecord> records,
            @NonNull ArrayList<Boolean> isRecordPop) {
        executeOpsTogether(records, isRecordPop, startIndex, numRecords);
    }

    private void executeOpsTogether(@NonNull ArrayList<BackStackRecord> records,
            @NonNull ArrayList<Boolean> isRecordPop, int startIndex, int endIndex) {
        executeOps(records, isRecordPop, startIndex, endIndex);

        moveToState(mCurState, true);
    }

    void moveToState(int newState, boolean always) {
        mFragmentStore.moveToExpectedState();
    }

    private static void executeOps(@NonNull ArrayList<BackStackRecord> records,
            @NonNull ArrayList<Boolean> isRecordPop, int startIndex, int endIndex) {
        for (int i = startIndex; i < endIndex; i++) {
            final BackStackRecord record = records.get(i);
            final boolean isPop = isRecordPop.get(i);
            if (isPop) {
                record.bumpBackStackNesting(-1);
                record.executePopOps();
            } else {
                record.bumpBackStackNesting(1);
                record.executeOps();
            }
        }
    }
}

class FragmentStore {
    void moveToState(int newState, boolean always) {
        mFragmentStore.moveToExpectedState();
    }
    void moveToExpectedState() {
        // Must add them in the proper order. mActive fragments may be out of order
        for (Fragment f : mAdded) {
            FragmentStateManager fragmentStateManager = mActive.get(f.mWho);
            if (fragmentStateManager != null) {
                fragmentStateManager.moveToExpectedState();
            }
        }
    }
}

class FragmentStateManager {
    void moveToExpectedState() {
        //不断循环，直到转换到合适状态
        while ((newState = computeExpectedState()) != mFragment.mState) {
            stateWasChanged = true;
                if (newState > mFragment.mState) {
                    // Moving upward
                    int nextStep = mFragment.mState + 1;
                    switch (nextStep) {
                        case Fragment.ATTACHED:
                            attach();
                            break;
                        case Fragment.CREATED:
                            create();
                            break;
                        case Fragment.VIEW_CREATED:
                            ensureInflatedView();
                            createView();
                            break;
                        case Fragment.AWAITING_EXIT_EFFECTS:
                            activityCreated();
                            break;
                        case Fragment.ACTIVITY_CREATED:
                            if (mFragment.mView != null && mFragment.mContainer != null) {
                                SpecialEffectsController controller = SpecialEffectsController
                                        .getOrCreateController(mFragment.mContainer,
                                                mFragment.getParentFragmentManager());
                                int visibility = mFragment.mView.getVisibility();
                                SpecialEffectsController.Operation.State finalState =
                                        SpecialEffectsController.Operation.State.from(visibility);
                                controller.enqueueAdd(finalState, this);
                            }
                            mFragment.mState = Fragment.ACTIVITY_CREATED;
                            break;
                        case Fragment.STARTED:
                            start();
                            break;
                        case Fragment.AWAITING_ENTER_EFFECTS:
                            mFragment.mState = Fragment.AWAITING_ENTER_EFFECTS;
                            break;
                        case Fragment.RESUMED:
                            resume();
                            break;
                    }
                } 
        }
    }
}

final class BackStackRecord extends FragmentTransaction implements
        FragmentManager.BackStackEntry, FragmentManager.OpGenerator {

    static final class Op {
        int mCmd;
        Fragment mFragment;

    }

    BackStackRecord(@NonNull FragmentManager manager) {
        mManager = manager;
    }

    public FragmentTransaction add(@IdRes int containerViewId, @NonNull Fragment fragment,
            @Nullable String tag) {
        doAddOp(containerViewId, fragment, tag, OP_ADD);
        return this;
    }

    void doAddOp(int containerViewId, Fragment fragment, @Nullable String tag, int opcmd) {
        //保存一系列的操作
        addOp(new Op(opcmd, fragment));
    }

    int commitInternal(boolean allowStateLoss) {
        mManager.enqueueAction(this, allowStateLoss);
    }

    void executeOps() {
        final int numOps = mOps.size();
        for (int opNum = 0; opNum < numOps; opNum++) {
            switch (op.mCmd) {
                case OP_ADD:
                    f.setAnimations(op.mEnterAnim, op.mExitAnim, op.mPopEnterAnim, op.mPopExitAnim);
                    mManager.setExitAnimationOrder(f, false);
                    mManager.addFragment(f);
                    break;
            }
        }
    }
}
```