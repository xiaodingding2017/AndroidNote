# Activity启动过程分析

Activity启动过程涉及到的类：ActivityManagerService，ActivityStack，ApplicationThread，ActivityThread。

ActivityManagerService和ActivityStack位于同一个进程中，而ApplicationThread和ActivityThread位于另一个进程中。其中，ActivityManagerService是负责管理Activity的生命周期的，ActivityManagerService还借助ActivityStack是来把所有的Activity按照后进先出的顺序放在一个堆栈中；对于每一个应用程序来说，都有一个ActivityThread来表示应用程序的主进程，而每一个ActivityThread都包含有一个ApplicationThread实例，它是一个Binder对象，负责和其它进程进行通信。

Step 1. 无论是通过Launcher来启动Activity，还是通过Activity内部调用startActivity接口来启动新的Activity，都通过Binder进程间通信进入到ActivityManagerService进程中，并且调用ActivityManagerService.startActivity接口；

 Step 2. ActivityManagerService调用ActivityStack.startActivityMayWait来做准备要启动的Activity的相关信息；

Step 3. ActivityStack通知ApplicationThread要进行Activity启动调度了，这里的ApplicationThread代表的是调用ActivityManagerService.startActivity接口的进程，对于通过点击应用程序图标的情景来说，这个进程就是Launcher了，而对于通过在Activity内部调用startActivity的情景来说，这个进程就是这个Activity所在的进程了；

Step 4. ApplicationThread不执行真正的启动操作，它通过调用ActivityManagerService.activityPaused接口进入到ActivityManagerService进程中，看看是否需要创建新的进程来启动Activity；

Step 5. 对于通过点击应用程序图标来启动Activity的情景来说，ActivityManagerService在这一步中，会调用startProcessLocked来创建一个新的进程，而对于通过在Activity内部调用startActivity来启动新的Activity来说，这一步是不需要执行的，因为新的Activity就在原来的Activity所在的进程中进行启动；

Step 6. ActivityManagerServic调用ApplicationThread.scheduleLaunchActivity接口，通知相应的进程执行启动Activity的操作；

Step 7. ApplicationThread把这个启动Activity的操作转发给ActivityThread，ActivityThread通过ClassLoader导入相应的Activity类，然后把它启动起来。



## 点击应用程序图标引发Android应用程序中的默认Activity的启动

**Step 1. Launcher.startActivitySafely**

public final class Launcher extends Activity

public void onClick(View v) {
		Object tag = v.getTag();
		if (tag instanceof ShortcutInfo) {
			// Open shortcut
			final Intent intent = ((ShortcutInfo) tag).intent;
			int[] pos = new int[2];
			v.getLocationOnScreen(pos);
			intent.setSourceBounds(new Rect(pos[0], pos[1],
				pos[0] + v.getWidth(), pos[1] + v.getHeight()));
			startActivitySafely(intent, tag);
		} else if (tag instanceof FolderInfo) {
			......
		} else if (v == mHandleView) {
			......
		}
	}

void startActivitySafely(Intent intent, Object tag) {
		intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
		try {
			startActivity(intent);
		} catch (ActivityNotFoundException e) {
			......
		} catch (SecurityException e) {
			......
		}
	}

Intent.FLAG_ACTIVITY_NEW_TASK表示要在一个新的Task中启动这个Activity。一个Task是一系列Activity的集合，这个集合是以堆栈的形式来组织的，遵循后进先出的原则。

**Step 2. Activity.startActivity**

public class Activity extends ContextThemeWrapper
		implements LayoutInflater.Factory,
		Window.Callback, KeyEvent.Callback,
		OnCreateContextMenuListener, ComponentCallbacks 

@Override
public void startActivity(Intent intent) {
	startActivityForResult(intent, -1);
}

调用startActivityForResult来进一步处理，第二个参数传入-1表示不需要这个Actvity结束后的返回结果。

**Step 3. Activity.startActivityForResult**

```java
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
        @Nullable Bundle options) {
    if (mParent == null) {
        options = transferSpringboardActivityOptions(options);
        Instrumentation.ActivityResult ar =
            mInstrumentation.execStartActivity(
                this, mMainThread.getApplicationThread(), mToken, this,
                intent, requestCode, options);
        if (ar != null) {
            mMainThread.sendActivityResult(
                mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                ar.getResultData());
        }
```

这里的mInstrumentation是Activity类的成员变量，它的类型是Intrumentation，定义在frameworks/base/core/java/android/app/Instrumentation.java文件中，它用来监控应用程序和系统的交互。  这里的mMainThread也是Activity类的成员变量，它的类型是ActivityThread，它代表的是应用程序的主线程。这里通过mMainThread.getApplicationThread获得它里面的ApplicationThread成员变量，它是一个Binder对象，ActivityManagerService会使用它来和ActivityThread来进行进程间通信。这里我们需注意的是，这里的mMainThread代表的是Launcher应用程序运行的进程。这里的mToken也是Activity类的成员变量，它是一个Binder对象的远程接口。options： Additional options for how the Activity should be started.

**Step 4. Instrumentation.execStartActivity**

```java
public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target,
        Intent intent, int requestCode, Bundle options) {
    IApplicationThread whoThread = (IApplicationThread) contextThread;
    Uri referrer = target != null ? target.onProvideReferrer() : null;
    if (referrer != null) {
        intent.putExtra(Intent.EXTRA_REFERRER, referrer);
    }
    if (mActivityMonitors != null) {
        synchronized (mSync) {
            final int N = mActivityMonitors.size();
            for (int i=0; i<N; i++) {
                final ActivityMonitor am = mActivityMonitors.get(i);
                ActivityResult result = null;
                if (am.ignoreMatchingSpecificIntents()) {
                    result = am.onStartActivity(intent);
                }
                if (result != null) {
                    am.mHits++;
                    return result;
                } else if (am.match(who, null, intent)) {
                    am.mHits++;
                    if (am.isBlocking()) {
                        return requestCode >= 0 ? am.getResult() : null;
                    }
                    break;
                }
            }
        }
    }
    try {
        intent.migrateExtraStreamToClipData();
        intent.prepareToLeaveProcess(who);
        int result = ActivityTaskManager.getService()
            .startActivity(whoThread, who.getBasePackageName(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()),
                    token, target != null ? target.mEmbeddedID : null,
                    requestCode, 0, null, options);
        checkStartActivityResult(result, intent);
    } catch (RemoteException e) {
        throw new RuntimeException("Failure from system", e);
    }
    return null;
}
```

ActivityTaskManager.getService().startActivity方法是调用的ActivityTaskManagerService类的startActivity方法；

**Step 5. ActivityTaskManagerService.startActivity方法**

```java
public class ActivityTaskManagerService extends IActivityTaskManager.Stub {
    public final int startActivity(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
        return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
                resultWho, requestCode, startFlags, profilerInfo, bOptions,
                UserHandle.getCallingUserId());
    }
```

里面会调用startActivityAsUser方法。



**Step 6. ActivityTaskManagerService.startActivityAsUser方法**

```java
int startActivityAsUser(IApplicationThread caller, String callingPackage,
        Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId,
        boolean validateIncomingUser) {
    enforceNotIsolatedCaller("startActivityAsUser");

    userId = getActivityStartController().checkTargetUser(userId, validateIncomingUser,
            Binder.getCallingPid(), Binder.getCallingUid(), "startActivityAsUser");

    // TODO: Switch to user app stacks here.
    return getActivityStartController().obtainStarter(intent, "startActivityAsUser")
            .setCaller(caller)
            .setCallingPackage(callingPackage)
            .setResolvedType(resolvedType)
            .setResultTo(resultTo)
            .setResultWho(resultWho)
            .setRequestCode(requestCode)
            .setStartFlags(startFlags)
            .setProfilerInfo(profilerInfo)
            .setActivityOptions(bOptions)
            .setMayWait(userId)
            .execute();

}
```

getActivityStartController()方法会得到mActivityStartController，mActivityStartController是ActivityTaskManagerService类的成员变量，是一个ActivityStartController实例。obtainStarter方法里调用了mFactory.obtain().setIntent(intent).setReason(reason)；mFactory是DefaultFactory实例，obtain方法返回一个ActivityStarter类的实例。execute方法调用startActivityMayWait方法。



**Step7. ActivityStarter.startActivityMayWait方法**

```java
private int startActivityMayWait(IApplicationThread caller, int callingUid,
        String callingPackage, int requestRealCallingPid, int requestRealCallingUid,
        Intent intent, String resolvedType, IVoiceInteractionSession voiceSession,
        IVoiceInteractor voiceInteractor, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, WaitResult outResult,
        Configuration globalConfig, SafeActivityOptions options, boolean ignoreTargetSecurity,
        int userId, TaskRecord inTask, String reason,
        boolean allowPendingRemoteAnimationRegistryLookup,
        PendingIntentRecord originatingPendingIntent, boolean allowBackgroundActivityStart) {
        
        ......
        
        ResolveInfo rInfo = mSupervisor.resolveIntent(intent, resolvedType, userId,
                0 /* matchFlags */,
                        computeResolveFilterUid(
                                callingUid, realCallingUid, mRequest.filterCallingUid));
     ......
      // Collect information about the target of the Intent.
        ActivityInfo aInfo = mSupervisor.resolveActivity(intent, rInfo, startFlags, profilerInfo);
    ......
     int res = startActivity(caller, intent, ephemeralIntent, resolvedType, aInfo, rInfo, voiceSession, voiceInteractor, resultTo, resultWho, requestCode, callingPid, callingUid, callingPackage, realCallingPid, realCallingUid, startFlags, options, ignoreTargetSecurity, componentSpecified, outRecord, inTask, reason, allowPendingRemoteAnimationRegistryLookup, originatingPendingIntent, allowBackgroundActivityStart);
   ......
   
```

获取activity info之后，最后调用startActivity方法。

**Step8.ActivityStarter.startActivity方法**

```java
private int startActivity(IApplicationThread caller, Intent intent, Intent ephemeralIntent, String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid, String callingPackage, int realCallingPid, int realCallingUid, int startFlags, SafeActivityOptions options, boolean ignoreTargetSecurity, boolean componentSpecified, ActivityRecord[] outActivity, TaskRecord inTask, boolean allowPendingRemoteAnimationRegistryLookup, PendingIntentRecord originatingPendingIntent, boolean allowBackgroundActivityStart) {
    ......
 ActivityRecord r = new ActivityRecord(mService, callerApp, callingPid, callingUid, callingPackage, intent, resolvedType, aInfo, mService.getGlobalConfiguration(), resultRecord, resultWho, requestCode, componentSpecified, voiceSession != null, mSupervisor, checkedOptions, sourceRecord);
    ......
final int res = startActivity(r, sourceRecord, voiceSession, voiceInteractor, startFlags, true /* doResume */, checkedOptions, inTask, outActivity, restrictedBgActivity);
    mSupervisor.getActivityMetricsLogger().notifyActivityLaunched(res, outActivity[0]);
  return res;
```

参数resultTo是Launcher这个Activity里面的一个Binder对象，通过它可以获得Launcher这个Activity的相关信息，保存在sourceRecord变量中。创建即将要启动的Activity的相关信息，并保存在r变量中。接着调用startActivity方法。



**Step9.ActivityStarter.startActivity方法**

```java
private int startActivity(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
            ActivityRecord[] outActivity, boolean restrictedBgActivity) {
    int result = START_CANCELED;
    final ActivityStack startedActivityStack;
    try {
        mService.mWindowManager.deferSurfaceLayout();
        result = startActivityUnchecked(r, sourceRecord, voiceSession, voiceInteractor,
                startFlags, doResume, options, inTask, outActivity, restrictedBgActivity);
    } finally {
   ......
   
```

里面调用startActivityUnchecked方法。

**Step10.ActivityStarter.startActivityUnchecked方法**

```java
private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,ActivityRecord[] outActivity, boolean restrictedBgActivity) {
  ......
  if ((mLaunchFlags & FLAG_ACTIVITY_CLEAR_TOP) != 0
      || isDocumentLaunchesIntoExisting(mLaunchFlags)
      || isLaunchModeOneOf(LAUNCH_SINGLE_INSTANCE, LAUNCH_SINGLE_TASK)) {
     final TaskRecord task = reusedActivity.getTaskRecord();
     // In this situation we want to remove all activities from the task up to the one
    // being started. In most cases this means we are resetting the task to its initial
   // state.
   final ActivityRecord top = task.performClearTaskForReuseLocked(mStartActivity,
                        mLaunchFlags);

                // The above code can remove {@code reusedActivity} from the task, leading to the
                // the {@code ActivityRecord} removing its reference to the {@code TaskRecord}. The
                // task reference is needed in the call below to
                // {@link setTargetStackAndMoveToFrontIfNeeded}.
                if (reusedActivity.getTaskRecord() == null) {
                    reusedActivity.setTask(task);
                }

                if (top != null) {
                    if (top.frontOfTask) {
                        // Activity aliases may mean we use different intents for the top activity,
                        // so make sure the task now has the identity of the new intent.
                        top.getTaskRecord().setIntent(mStartActivity);
                    }
                    deliverNewIntent(top);
                }
......
   
    // If the activity being launched is the same as the one currently at the top, then
        // we need to check if it should only be launched once.
        final ActivityStack topStack = mRootActivityContainer.getTopDisplayFocusedStack();
        final ActivityRecord topFocused = topStack.getTopActivity();
        final ActivityRecord top = topStack.topRunningNonDelayedActivityLocked(mNotTop);
        final boolean dontStart = top != null && mStartActivity.resultTo == null
                && top.mActivityComponent.equals(mStartActivity.mActivityComponent)
                && top.mUserId == mStartActivity.mUserId
                && top.attachedToProcess()
                && ((mLaunchFlags & FLAG_ACTIVITY_SINGLE_TOP) != 0
                || isLaunchModeOneOf(LAUNCH_SINGLE_TOP, LAUNCH_SINGLE_TASK))
                // This allows home activity to automatically launch on secondary display when
                // display added, if home was the top activity on default display, instead of
                // sending new intent to the home activity on default display.
                && (!top.isActivityTypeHome() || top.getDisplayId() == mPreferredDisplayId);
 ......
     
    mTargetStack.startActivityLocked(mStartActivity, topFocused, newTask, mKeepCurTransition, mOptions);
  if (mDoResume) {
            final ActivityRecord topTaskActivity =
                    mStartActivity.getTaskRecord().topRunningActivityLocked();
            if (!mTargetStack.isFocusable()
                    || (topTaskActivity != null && topTaskActivity.mTaskOverlay
                    && mStartActivity != topTaskActivity)) {
                // If the activity is not focusable, we can't resume it, but still would like to
                // make sure it becomes visible as it starts (this will also trigger entry
                // animation). An example of this are PIP activities.
                // Also, we don't want to resume activities in a task that currently has an overlay
                // as the starting activity just needs to be in the visible paused state until the
                // over is removed.
                mTargetStack.ensureActivitiesVisibleLocked(mStartActivity, 0, !PRESERVE_WINDOWS);
                // Go ahead and tell window manager to execute app transition for this activity
                // since the app transition will not be triggered through the resume channel.
                mTargetStack.getDisplay().mDisplayContent.executeAppTransition();
            } else {
                // If the target stack was not previously focusable (previous top running activity
                // on that stack was not visible) then any prior calls to move the stack to the
                // will not update the focused stack.  If starting the new activity now allows the
                // task stack to be focusable, then ensure that we now update the focused stack
                // accordingly.
                if (mTargetStack.isFocusable()
                        && !mRootActivityContainer.isTopDisplayFocusedStack(mTargetStack)) {
                    mTargetStack.moveToFront("startActivityUnchecked");
                }
                mRootActivityContainer.resumeFocusedStacksTopActivities(
                        mTargetStack, mStartActivity, mOptions);
            }
        }    
......
```

查找是否有task可以启动activity，如果没有就创建一个新的task启动activity。

第二部分代码是判断当前需要启动的activity和栈顶的activity是否为同一个activity，如果是则需要判断启动模式以便确定是否需要启动一个新的activity实例。由于此处栈顶是Launcher，和MainActivity不是同一个activity，所以会直接跳过。后面调用ActivityStack.startActivityLocked方法，该方法在这个函数定义在frameworks/base/services/java/com/android/server/am/ActivityStack.java文件中，主要是task的一些处理。最后调用的mRootActivityContainer.resumeFocusedStacksTopActivities(mTargetStack, mStartActivity, mOptions)方法，resumeFocusedStacksTopActivities方法里会调用ActivityStack.resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options)方法，resumeTopActivityUncheckedLocked方法会调用resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options)方法。



**Step11.ActivityStack.resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options)方法**

```java
private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
        if (!mService.isBooting() && !mService.isBooted()) {
            // Not ready yet!
            return false;
        }

        // Find the next top-most activity to resume in this stack that is not finishing and is
        // focusable. If it is not focusable, we will fall into the case below to resume the
        // top activity in the next focusable task.
        ActivityRecord next = topRunningActivityLocked(true /* focusableOnly */);
    ......
 // Remember how we'll process this pause/resume situation, and ensure
        // that the state is reset however we wind up proceeding.
        boolean userLeaving = mStackSupervisor.mUserLeaving;
        mStackSupervisor.mUserLeaving = false;
    ......
   
```

函数先通过调用topRunningActivityLocked函数获得堆栈顶端的Activity，这里就是MainActivity了.接下来把mUserLeaving的保存在本地变量userLeaving中，然后重新设置为false.

。。。。。。（未完待续）



整个应用程序的启动过程要执行很多步骤，但是整体来看，主要分为以下五个阶段：

   一. Step1 - Step 11：Launcher通过Binder进程间通信机制通知ActivityManagerService，它要启动一个Activity；

   二. Step 12 - Step 16：ActivityManagerService通过Binder进程间通信机制通知Launcher进入Paused状态；

   三. Step 17 - Step 24：Launcher通过Binder进程间通信机制通知ActivityManagerService，它已经准备就绪进入Paused状态，于是ActivityManagerService就创建一个新的进程，用来启动一个ActivityThread实例，即将要启动的Activity就是在这个ActivityThread实例中运行；

   四. Step 25 - Step 27：ActivityThread通过Binder进程间通信机制将一个ApplicationThread类型的Binder对象传递给ActivityManagerService，以便以后ActivityManagerService能够通过这个Binder对象和它进行通信；

   五. Step 28 - Step 35：ActivityManagerService通过Binder进程间通信机制通知ActivityThread，现在一切准备就绪，它可以真正执行Activity的启动操作了。

## 应用程序内部启动新的Activity的过程

在应用程序内部启动新的Activity的过程要执行很多步骤，但是整体来看，主要分为以下四个阶段：
       一. Step 1 - Step 10：应用程序的MainActivity通过Binder进程间通信机制通知ActivityManagerService，它要启动一个新的Activity；
       二. Step 11 - Step 15：ActivityManagerService通过Binder进程间通信机制通知MainActivity进入Paused状态；
       三. Step 16 - Step 22：MainActivity通过Binder进程间通信机制通知ActivityManagerService，它已经准备就绪进入Paused状态，于是ActivityManagerService就准备要在MainActivity所在的进程和任务中启动新的Activity了；
       四. Step 23 - Step 29：ActivityManagerService通过Binder进程间通信机制通知MainActivity所在的ActivityThread，现在一切准备就绪，它可以真正执行Activity的启动操作了。



  参考链接：

https://blog.csdn.net/Luoshengyang/article/details/6685853

https://blog.csdn.net/luoshengyang/article/details/6689748

https://blog.csdn.net/luoshengyang/article/details/6703247