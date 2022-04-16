This is PoC for CVE-2021-39749, which allows starting activities of other apps on Android 12L Beta regardless of their `permission` and `exported` settings

In Android 12L [TaskFragmentOrganizer access (intentionally) no longer requires `MANAGE_ACTIVITY_TASKS` permission](https://android.googlesource.com/platform/frameworks/base/+/d5555da37a3a7f9508efee06b5c7abbb380af55d%5E!/)

Using app provided here requires disabling [Hidden API Checks](https://developer.android.com/guide/app-compatibility/restrictions-non-sdk-interfaces), you can do so through `adb shell settings put global hidden_api_policy 1`. These are not security boundary and [there are known app-based bypasses](https://www.xda-developers.com/bypass-hidden-apis/)

Here are commits fixing this bug (and few related mentioned in original report):

1. [`startActivityInTaskFragment` no longer rely on `Binder.getCallingUid()`](https://android.googlesource.com/platform/frameworks/base/+/a3deece020fb5a2a6e54499d83076e4c783b96ab%5E!/)
2. [`ResolverActivity` now has `relinquishTaskIdentity` enabled](https://android.googlesource.com/platform/frameworks/base/+/674aed9fb5a2a4411660507a7edb6fe1e351ed9b%5E!/)
3. (Not needed for starting other activities, but allows repositioning them around screen and making them transparent and tap-jackable) [`SurfaceControl` of `TaskFragment` is no longer provided](https://android.googlesource.com/platform/frameworks/base/+/d45dba76ac1fe9726ec74a797441fdb853c1c4db%5E!/)
4. (Not shown in code here, issue only mentioned in original report) [Deciding whenever to send `ActivityRecord#appToken` to `TaskFragmentOrganizer` is now based on uid instead of pid](https://android.googlesource.com/platform/frameworks/base/+/9c906fbc942bbddba7fe3bc1c6e905281712a118%5E!/)

You can checkout `android-12.1.0_r4`, revert first 3 commits (or first 2, application will still be able shutdown device (by starting `ShutdownActivity`), but "Zoom and set alpha" checkbox won't work)

(First commit from that list will have merge conflicts in tests if you try to revert it, but you can ignore these)

# `Binder.getCallingUid()` that always returns system uid

[`Binder.getCallingUid()`](https://developer.android.com/reference/android/os/Binder#getCallingUid()) method returns uid of process that sent currently processed Binder transaction. That uid is stored in thread-local variable. Code handling transaction can call [`Binder.clearCallingIdentity()`](https://developer.android.com/reference/android/os/Binder#clearCallingIdentity()) to set that variable to uid of own process to indicate to methods called later during transaction handling that permission checks should be done against itself (code handling transaction) and not caller of Binder transaction

Sometimes there are `Binder.getCallingUid()` that are always called after `Binder.clearCallingIdentity()`, therefore always return uid of own process. [Sometimes this happens intentionally, for example in `ActivityTaskManagerService#startDreamActivity`](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java;l=1445-1447;drc=77d147be49bbacefa92cf8a0b33b2968125ce613) (although thats rather convoluted way of doing [`Process.myUid()`](https://developer.android.com/reference/android/os/Process#myUid()) or [`Os.getuid()`](https://developer.android.com/reference/android/system/Os#getuid()))

I've written for myself a ([Soot](https://soot-oss.github.io/soot/)-based) static analysis tool that reports such `Binder.getCallingUid()` calls (and other permission checks) that can only happen after `Binder.clearCallingIdentity()`. (I have custom logic handling Jimple/Shimple <abbr title="Intermediate Representation">IR</abbr> provided by Soot, although there might be better way to do so with Soot, but thats what I have now)

In Android 12L Beta that tool found one in [ActivityStartController#startActivityInTaskFragment](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/services/core/java/com/android/server/wm/ActivityStartController.java;l=511;drc=674aed9fb5a2a4411660507a7edb6fe1e351ed9b) (Note: source code was not available then as Beta releases are not open source, but Shimple is generally readable so I've been using Soot also as Java decompiler)

# How to call `startActivityInTaskFragment`

As part of static analysis report I've got call hierarchy from [`onTransact()`](https://developer.android.com/reference/android/os/Binder#onTransact(int,%20android.os.Parcel,%20android.os.Parcel,%20int)) implementation (where Binder call starts) to `startActivityInTaskFragment`:

1. `onTransact` in aidl-generated code of [`IWindowOrganizerController`](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/window/IWindowOrganizerController.aidl;drc=674aed9fb5a2a4411660507a7edb6fe1e351ed9b)
2. [`WindowOrganizerController#applyTransaction` (without `CallerInfo` argument)](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/services/core/java/com/android/server/wm/WindowOrganizerController.java;l=166;drc=674aed9fb5a2a4411660507a7edb6fe1e351ed9b)
3. [`WindowOrganizerController#applyTransaction` (with `CallerInfo` argument)](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/services/core/java/com/android/server/wm/WindowOrganizerController.java;l=380;drc=674aed9fb5a2a4411660507a7edb6fe1e351ed9b)
4. [`WindowOrganizerController#applyHierarchyOp`](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/services/core/java/com/android/server/wm/WindowOrganizerController.java;l=633;drc=674aed9fb5a2a4411660507a7edb6fe1e351ed9b)
5. [`ActivityStartController#startActivityInTaskFragment`](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/services/core/java/com/android/server/wm/ActivityStartController.java;l=511;drc=674aed9fb5a2a4411660507a7edb6fe1e351ed9b)

I've found that Binder calls to `applyTransaction` are present in [`TaskFragmentOrganizer` class](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/window/TaskFragmentOrganizer.java;drc=674aed9fb5a2a4411660507a7edb6fe1e351ed9b) and I've decided to use it as more convenient wrapper than doing all Binder calls directly (neither is public API so I had to use reflection anyway)

First of all, method "2." calls `enforceTaskPermission`, which on Android 12.0 checked `signature`-only `MANAGE_ACTIVITY_TASKS` permission which we couldn't get, however [on Android 12L rules were relaxed so certain transactions can be made without permissions](https://android.googlesource.com/platform/frameworks/base/+/d5555da37a3a7f9508efee06b5c7abbb380af55d%5E!/). It turned out that none of operations needed for doing `startActivityInTaskFragment` required a permission (if transaction had `TaskFragmentOrganizer` associated)

So we want to [perform `HIERARCHY_OP_TYPE_START_ACTIVITY_IN_TASK_FRAGMENT`](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/services/core/java/com/android/server/wm/WindowOrganizerController.java;l=621-641;drc=674aed9fb5a2a4411660507a7edb6fe1e351ed9b). In order to do so we must have our `TaskFragment` registered in `mLaunchTaskFragments` otherwise `"Not allowed to operate with invalid fragment token"` exception will be reported

We can register such `TaskFragment` through [`HIERARCHY_OP_TYPE_CREATE_TASK_FRAGMENT`](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/services/core/java/com/android/server/wm/WindowOrganizerController.java;l=588-593;drc=674aed9fb5a2a4411660507a7edb6fe1e351ed9b), which calls [`createTaskFragment()`](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/services/core/java/com/android/server/wm/WindowOrganizerController.java;l=1202-1240;drc=674aed9fb5a2a4411660507a7edb6fe1e351ed9b)

(In PoC code these transactions are sent in `SecondActivity`: `HIERARCHY_OP_TYPE_CREATE_TASK_FRAGMENT` is sent by `initOrganizerAndFragment()` and `HIERARCHY_OP_TYPE_START_ACTIVITY_IN_TASK_FRAGMENT` is sent by `startActivityInOrganizer`)

So that allows us to call `startActivityInTaskFragment` and Intents of Activities started here are considered to be coming from system uid, but it turns out that in itself it doesn't let us do anything: [activities started by system cannot do URI grants](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/services/core/java/com/android/server/uri/UriGrantsManagerService.java;l=1095-1111;drc=674aed9fb5a2a4411660507a7edb6fe1e351ed9b) and if we try to launch activity of another app we'll be stopped by [`canEmbedActivity` check](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/services/core/java/com/android/server/wm/ActivityStarter.java;l=1960-1997;drc=674aed9fb5a2a4411660507a7edb6fe1e351ed9b)

# Bypassing `canEmbedActivity`

Lets [take a look at `canEmbedActivity` again](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/services/core/java/com/android/server/wm/ActivityStarter.java;l=1969-1997;drc=674aed9fb5a2a4411660507a7edb6fe1e351ed9b): embedding is allowed if `taskFragment.getTask().effectiveUid` is uid of system or matches uid of launched app. We'll need to be in task whose `effectiveUid` is system

Also step back to [`createTaskFragment()`](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/services/core/java/com/android/server/wm/WindowOrganizerController.java;l=1202-1240;drc=674aed9fb5a2a4411660507a7edb6fe1e351ed9b): creation of TaskFragment was only allowed if `rootActivity.getUid() != ownerActivity.getUid()`. This means that our activity will need to be at bottom of back-stack of Task it is in

We'll need to launch new `Task` (through [`Intent.FLAG_ACTIVITY_NEW_TASK`](https://developer.android.com/reference/android/content/Intent#FLAG_ACTIVITY_NEW_TASK)) which will have Activity belonging to system uid (so `Task#effectiveUid` will be set to `AID_SYSTEM`) and then that Activity will start our Activity (within same Task) and [`finish()`](https://developer.android.com/reference/android/app/Activity#finish()) itself (so our Activity will become root of that task allowing us to use `createTaskFragment()`)

One of such Activities is [`ChooserActivity`](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/com/android/internal/app/ChooserActivity.java;drc=674aed9fb5a2a4411660507a7edb6fe1e351ed9b). Chooser is usually used for choosing to which app user wants to use after selecting "share" option. [`ChooserActivity` however does have `android:relinquishTaskIdentity="true"` set in `AndroidManifest.xml`](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/res/AndroidManifest.xml;l=5859-5864;drc=674aed9fb5a2a4411660507a7edb6fe1e351ed9b), which means that when it launches another Activity it will [overwrite `Task#effectiveUid` with uid of newly-launched app](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/services/core/java/com/android/server/wm/Task.java;l=1015-1030;drc=674aed9fb5a2a4411660507a7edb6fe1e351ed9b)

(`relinquishTaskIdentity` only works when used by first app in Task and only for system apps, so we cannot use `relinquishTaskIdentity` ourselves and launch system app to overwrite `Task#effectiveUid` of our Task)

Another such Activity (that can start our Activity and `finish()` itself) is [`ResolverActivity`](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/com/android/internal/app/ResolverActivity.java;drc=674aed9fb5a2a4411660507a7edb6fe1e351ed9b). It is used when starting implicit `Intent` that resolves to multiple Activities. Resolver (unlike Chooser) does offer option to remember choice, which is how you (as user of phone) can distinguish those two. `ResolverActivity` did not have `relinquishTaskIdentity` set, however Resolver uses own Intent to find what options are available (while Chooser takes Intent provided in Extras). This turns out to be a problem for exploitation because Intent flags used by Resolver when launching selected Activity will be same as those used to launch Resolver and:

* If we don't set `Intent.FLAG_ACTIVITY_NEW_TASK`, Resolver will be launched within our `Task` which already has `effectiveUid` permanently set
* If we do set `Intent.FLAG_ACTIVITY_NEW_TASK`, Resolver will launch selection into yet another `Task`, which will then have `effectiveUid` set to one belonging to launched app

The solution to these problems is to use both:

1. First we launch `ChooserActivity`: We provide to its Intent:
    * `Intent.FLAG_ACTIVITY_NEW_TASK`, so Chooser is launched in new Task (which will have `effectiveUid` of system but only until next Activity launch)
    * [`Intent.EXTRA_INTENT`](https://developer.android.com/reference/android/content/Intent#EXTRA_INTENT) set to an `Intent` which doesn't match any Activities and only options left in Chooser will come from `Intent.EXTRA_INITIAL_INTENTS`
    * [`Intent.EXTRA_INITIAL_INTENTS`](https://developer.android.com/reference/android/content/Intent#EXTRA_INITIAL_INTENTS) containing array with one-element: The Intent we want Chooser to launch (when there is only one option both Chooser and Resolver skip prompt and immediately launch only option and `finish()` itself)
2. Then `ResolverActivity` is launched by `ChooserActivity`:
    * Resolver didn't have `relinquishTaskIdentity` set, so now `Task#effectiveUid` is set to system and will stay that way regardless of next Activities launched in this Task
    * Intent does not have `Intent.FLAG_ACTIVITY_NEW_TASK`, so next Activity is launched within same Task
    * Intent action is set to non-standard one, matching only `<intent-filter>` we declared ourselves in our app, so Resolver immediately proceeds to launching our Activity
3. `ResolverActivity` launches our Activity
    * Now we're in Task whose `effectiveUid` is `AID_SYSTEM`, so `canEmbedActivity()` allows anything
    * Both Chooser and Resolver have finished themselves, so we're root Activity in Task and are allowed to use `createTaskFragment()`

(In PoC app preparation of these steps is performed in `FirstActivity`)

# Other tricks with TaskFragmentOrganizer

TaskFragmentOrganizer received a [`SurfaceControl`](https://developer.android.com/reference/android/view/SurfaceControl) through `onTaskFragmentAppeared` callback and using that `SurfaceControl` one can scale launched Activity and make it transparent, while it will still receive touch events and won't be considered obscured (so tap-jacking-protected elements can still be tapped)

You can see that by checking "Zoom and set alpha" checkbox in PoC app

[This is fixed by commit "3." from fixes list at top](https://android.googlesource.com/platform/frameworks/base/+/d45dba76ac1fe9726ec74a797441fdb853c1c4db%5E!/)

---

Another thing is that `ActivityRecord#appToken`-s of Activities running inside `TaskFragment` passed to `TaskFragmentOrganizer` callbacks. This list was filtered to only include tokens of Activities within same process, however check was done by comparing pid of TaskFragmentOrganizer with pid of Activity whose `appToken` we could get. I haven't actually checked but I think application could create `TaskFragmentOrganizer`, exit process used to initially create it and have its pid reused as pid of Activity of another app in order to get its `appToken`. [Here's commit ("4." in fixes list above) that switches verification from pid-based to uid-based (it looks like this commit was done independently of my report though (although after it))](https://android.googlesource.com/platform/frameworks/base/+/9c906fbc942bbddba7fe3bc1c6e905281712a118%5E!/)

Once attacker gets `appToken` of an Activity they can inject [`onActivityResult()`](https://developer.android.com/reference/android/app/Activity#onActivityResult(int,%20int,%20android.content.Intent)) calls (even if target app didn't call [`startActivityForResult()`](https://developer.android.com/reference/android/app/Activity#startActivityForResult(android.content.Intent,%20int)) themselves) and possibly tamper `savedInstanceState` (by calling [`activityStopped()`](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/services/core/java/com/android/server/wm/ActivityClientController.java;l=183;drc=674aed9fb5a2a4411660507a7edb6fe1e351ed9b), assuming attacker can win race with target application calling that method and additional call won't cause state to be lost due to crash)

I haven't checked if that can be done in this case, however previously, with [CVE-2020-0001](https://source.android.com/security/bulletin/2020-01-01#framework) (Yay, I've got fancy number), I was able to use `savedInstanceState` tampering and `onActivityResult()` injection to trick system settings app into enabling my `AccessibilityService` without user interaction, but that is a story for another time
