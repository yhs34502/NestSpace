# Parallel Space / multiDroid — Reverse-Engineering Notes and Implementation Blueprint

Status: research baseline for NestSpace replacement engine  
Target: Android 14 / API 34, Galaxy A52s 5G, ARM64  
Hard constraints: **No Root; no Shizuku**. Keep NestSpace's existing permission-selection and UI layers untouched.

## 1. What is confirmed, and what is not

### Confirmed from current product information
- The current Google Play listing identifies Parallel Space as built on its proprietary **multiDroid** application-virtualization engine.
- The listing says it includes software developed by the microG Project.
- The listing currently claims Android 15 compatibility and explicitly excludes apps declaring REQUIRE_SECURE_ENV.
- No public multiDroid source code or official internal architecture document was found. Therefore we must not claim that our proposed classes or internals are the actual implementation used by Parallel Space.

Sources:
- https://play.google.com/store/apps/details?id=com.lbe.parallel.intl
- https://www.cs.ucr.edu/~heng/pubs/sacmat2020.pdf

### Strong historical evidence about Parallel Space
A 2017 public reverse-engineering report for Parallel Space observed:
- a large pool of proxy Activities, Services and ContentProviders (reported as 100 of each);
- guest APK data stored beneath the host app's private data directory;
- Activity Manager calls intercepted and rewritten through proxy components;
- guest code loaded in the host-controlled runtime.

This is useful as a historical clue, not proof that the current multiDroid release still uses the exact same implementation:
- https://stackoverflow.com/questions/41570178/how-to-execute-apks-on-a-customized-data-directory/41702482

### Academic description of the app-virtualization design
The 2020 SACMAT paper describes a common HostApp architecture:
1. Launcher parses the guest APK manifest and resolves its launch component.
2. Host starts a declared dummy/stub component.
3. The guest runtime loads a hook module and communicates with the HostApp.
4. The guest APK gets a guest ClassLoader and guest Application/Context setup.
5. A virtual Activity Manager tracks guest component lifecycle.
6. Stub components in the host manifest stand in for guest components because Android's real Activity Manager cannot launch undeclared guest Activities.

The paper also describes native I/O redirection and hooks as compatibility mechanisms. This is the strongest public architectural reference, but it describes app-level virtualization generally and not a verified current multiDroid source listing:
- https://www.cs.ucr.edu/~heng/pubs/sacmat2020.pdf

## 2. Interpret the one-Recent observation correctly

User's device observation:
- NestSpace: opening five guest apps produces six Recent entries (five guest launches plus NestSpace).
- Parallel Space: opening several guest apps leaves one Recent entry.

The key target is **one Android task representing the virtualization host**, with guest Activities managed as an internal virtual stack.

Important distinction:
- One Recent entry does **not** prove there is only one Linux process.
- A robust design may use several host-owned processes for guest runtime slots while keeping their guest Activities associated with one host task.
- Process topology and Android task topology must be measured separately.

Design target:
- One host task / one Recents card.
- Guest Activities are represented by declared host proxy Activities.
- All guest Activity transitions are routed through a VirtualTaskManager.
- Guest requests for NEW_TASK, document tasks, task affinity and task reparenting are translated into virtual-task operations where safe.
- External intents (camera, chooser, browser, file picker, etc.) are explicitly routed outside the guest task when they require a real external Activity.

## 3. Proposed implementation modules (our design, not Parallel Space's internal class names)

Package root: <nestspace>.engine

### 3.1 Task and Activity virtualization — first implementation target

~~~kotlin
interface VirtualTaskManager {
    fun launch(
        guest: GuestComponent,
        intent: Intent,
        options: Bundle? = null
    ): LaunchResult

    fun onProxyCreated(token: ActivityToken, savedState: Bundle?)
    fun onProxyResumed(token: ActivityToken)
    fun onProxyPaused(token: ActivityToken)
    fun onProxyDestroyed(token: ActivityToken)
    fun finish(token: ActivityToken)
    fun moveToFront(taskId: VirtualTaskId)
    fun saveState(): Bundle
    fun restoreState(state: Bundle)
}
~~~

Data records:
- VirtualTaskRecord: virtualTaskId, guestPackage, rootComponent, activityStack, lastActiveTime.
- VirtualActivityRecord: token, guestComponent, originalIntent, proxySlot, lifecycleState, savedState.
- ProxySlot: manifest-declared host Activity class and assigned guest runtime slot.
- LaunchResult: started, routedExternal, unsupportedLaunchMode, or error.

Manifest strategy to prototype:
- Declare a host task anchor Activity.
- Declare a finite pool of proxy Activities in the host manifest.
- Keep guest proxy Activities in the host's chosen task affinity.
- Do not let each guest launch create a document task or independent task by default.
- Preserve one visible host task root in Recents; test task flags and root/base-intent behavior on API 34 rather than assuming excludeFromRecents alone solves it.

### 3.2 Guest APK metadata and loading

~~~kotlin
interface GuestPackageParser {
    fun parse(apk: File): GuestPackageInfo
    fun resolveLaunchActivity(info: GuestPackageInfo): GuestComponent?
    fun resolveComponent(info: GuestPackageInfo, intent: Intent): GuestComponent?
}

interface GuestRuntimeLoader {
    fun createRuntime(info: GuestPackageInfo, processSlot: Int): GuestRuntime
    fun loadApplication(runtime: GuestRuntime): Application
    fun createGuestContext(runtime: GuestRuntime): Context
}
~~~

Required behavior:
- Parse base + split APKs and ABI metadata.
- Resolve activities, services, receivers, providers, permissions and metadata.
- Create isolated per-guest data directories under the host private directory.
- Use the correct native ABI and native-library search paths.
- Never assume a single DEX ClassLoader is sufficient for Google Play Services, WebView, split APKs or dynamically loaded native libraries.

### 3.3 Framework bridge

~~~kotlin
interface GuestFrameworkBridge {
    fun installActivityLaunchRouter()
    fun installPackageManagerFacade()
    fun installServiceManagerFacade()
    fun installInstrumentationBridge()
    fun installContentProviderRouter()
    fun installPendingIntentRouter()
}
~~~

Responsibilities:
- Intercept guest-originated component launches inside the guest runtime.
- Rewrite guest Activity intents to host-declared proxy Activities.
- Restore guest package/component identity for guest-side queries.
- Route permission checks, package queries, broadcasts, providers and PendingIntent operations consistently.
- Maintain callback and lifecycle ordering.

### 3.4 Native compatibility layer — later, not Milestone 1

Potential responsibilities:
- Native library loading and ABI selection.
- File path redirection for guest hard-coded paths.
- Native API hooks where Java-level interception cannot cover the operation.
- Media/graphics diagnostics only; do not assume a generic hook can manufacture DRM or trusted hardware support.

This layer is not part of the first Recent/Task proof of concept.

## 4. Android 14 / No-root feasibility boundaries

No-root means the engine runs with the host app's normal UID and cannot modify system_server, SELinux policy, kernel namespaces or global system services.

App-level virtualization still needs framework interception inside processes controlled by the host. This commonly relies on non-SDK Android internals and ART/native hooking. Android releases can change those internals, and hidden-API enforcement can break reflection-based bridges.

Consequences:
- Do not promise full compatibility just because the app can be loaded.
- Do not design around Root, Shizuku, shell grants or privileged permissions.
- Do not treat MediaCodec success as proof that protected Widevine playback will work.
- DRM, Play Integrity, hardware-backed identity and secure surfaces may remain unavailable to a guest even if ordinary video decoding works.

## 5. What VirtualApp tells us — and what it does not

Public VirtualApp documentation describes three layers:
- VA Space: guest package/data space.
- VA Framework: Java/framework proxy layer.
- VA Native: I/O redirection and native hooks.

It also describes separate host, server and guest-client process roles.

However, the public README explicitly says the GitHub code stopped updating in December 2017 and that the continuously updated Android 14-capable code is the commercial/business version. The public repository does not provide a clear open-source license. Therefore:
- use its architecture and change history as research material;
- do not copy or embed its code until licensing and the exact source version are verified;
- do not assume the public branch is a ready Android 14 engine.

References:
- https://github.com/asLody/VirtualApp
- https://github.com/asLody/VirtualApp/blob/master/README_eng.md

## 6. Concrete milestones and acceptance criteria

### M0 — Measure current Parallel Space on the actual phone
Collect evidence after opening multiple guest apps:
- dumpsys activity recents
- dumpsys activity activities
- ps -A / process list
- package name and process names
- taskId, rootActivity, baseIntent, affinity, realActivity and number of guest runtime processes

This will tell us whether one Recent card maps to one task, whether multiple guest activities share a task, and whether guest processes are separate. Do not infer process count from Recents.

### M1 — Single-Recent proof of concept
Only implement:
- one host task anchor;
- proxy Activity pool;
- virtual Activity stack;
- internal app switching;
- Back / finish / restore behavior;
- no guest app-specific compatibility patches.

Pass criteria:
- Launch at least three different guest APKs.
- Android Recents shows one NestSpace card.
- Switching between guest Activities does not create extra Recent cards.
- Back returns through the guest virtual stack correctly.
- Home then resume returns to the last guest screen.
- No Root and no Shizuku.

### M2 — Guest runtime bootstrap
- Application and Context setup.
- Package identity facade.
- split APK / ABI support.
- service/provider lifecycle.

### M3 — Media and Google service capability
Test ordinary video, hardware decode, audio focus, location, WebView, GMS and MicroG separately. Record failures by subsystem; do not patch individual apps as the strategy.

## 7. Immediate device evidence request

After opening at least three guest apps in Parallel Space, run these in **Windows CMD** and save the output:

~~~cmd
"%LOCALAPPDATA%\Android\Sdk\platform-tools\adb.exe" shell dumpsys activity recents > "%USERPROFILE%\Desktop\ps-recents.txt"
"%LOCALAPPDATA%\Android\Sdk\platform-tools\adb.exe" shell dumpsys activity activities > "%USERPROFILE%\Desktop\ps-activities.txt"
"%LOCALAPPDATA%\Android\Sdk\platform-tools\adb.exe" shell ps -A > "%USERPROFILE%\Desktop\ps-processes.txt"
"%LOCALAPPDATA%\Android\Sdk\platform-tools\adb.exe" shell pm list packages | findstr /i "parallel lbe microg"
~~~

These outputs are needed to distinguish the one-task observation from process architecture on the actual Android 14 device.

## 8. Decision

Proceed with a new No-root virtualization engine. Do not start by patching NestSpace's existing engine or by integrating the stale public VirtualApp branch. First complete the live Parallel Space task/process measurement, then implement M1 as a separately testable engine module.
