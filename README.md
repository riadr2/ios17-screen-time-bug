# ios17-screen-time-bug
Investigation into a bug in iOS 17.0 where FamilyControlsAgent crashes due to oversized Store.plist, causing apps using Screen Time API to freeze

---

If you’re on **iOS 17.0** and apps that use Screen Time features (like **Opal**, **One Sec**, etc.) are freezing on launch  here’s the fix:
1.	Open the **Files app** on your iPhone.
2.	Navigate to: /private/var/mobile/Library/com.apple.FamilyControlsAgent/
3. Locate the file named Store.plist Rename it to something like Store.plist.bak
4. Relaunch the app it should now work instantly.

This works because the .plist file had grown too large and was causing a system crash. Renaming it forces iOS to generate a fresh, clean version.

 **For anyone curious** who want to understand *why* this worked and what was really going on under the hood  keep reading below :)

---
---

I was still on **iOS 17.0** because I love **TrollStore**, and I wanted to try out some apps that help cut down screen time and boost productivity. So I installed **Opal**.
But as soon as I used the app it frozen. **stuck on the permissions screen** 

then tried a few other apps that use Apple’s **Screen Time API**  and every single one of them froze too. That seemed suspicious. 
also A friend of mine on iOS 17.0 had the exact same issue.

so I thought it might be something unstable with iOS 17.0 .

I plugged my iPhone into my Mac and opened the **Console app** to view system logs . Then I tried launching Opal again.

When the app froze, the last log message I saw was:
```log
default 14:37:53.769707+0100    Opal    
[0x281fc1500] activating connection: mach=true listener=false peer=false name=com.apple.FamilyControlsAgent

```
The app is activating connection. This is an XPC (Cross-Process Communication) call, which is how apps talk to system services.
The name of the service it's trying to connect to is com.apple.FamilyControlsAgent.

FamilyControlsAgent is the background daemon (a system process) that handles all the logic for the FamilyControls framework. This is the heart of the Screen Time API.

I confirmed that **every app** using the Screen Time API froze in the same way. They all got stuck right after trying to connect to FamilyControlsAgent.

So, since it’s the weekend and I’ve got some time (and I want to actually *use* these apps), I’ve decided to try tracking down the root of this bug. Should be fun.

---


so first of all my thinking was that :

the Screen Time API has a bug in iOS 17.0. 
the app makes a call to a function within the Screen Time API. Under the hood, that function in the FamilyControls framework creates an XPC connection to the com.apple.FamilyControlsAgent to do the actual work

Now, look at what happens immediately after. There are no more logs that indicate a successful connection or a response from **FamilyControlsAgent**. The app's own log stream essentially goes silent

the app's code, running on the main UI thread, made a call to the Screen Time API.
That API call is synchronous, meaning it blocks the main thread and waits for com.apple.FamilyControlsAgent to reply.
Because of the iOS 17.0 bug, the FamilyControlsAgent either crashes, fails to respond, or gets into its own deadlock.
the app is now stuck forever waiting for a reply that will never come. Since this is on the main thread, the entire UI freezes.

---

To confirm, I built a **simple iOS test app** that used the Screen Time API. It also froze immediately when:

**Requesting Authorization:**
```swift
try await AuthorizationCenter.shared.requestAuthorization(for: .individual)
```
**Checking Authorization Status:**
```swift
let status = AuthorizationCenter.shared.authorizationStatus
```

To gain more visibility, I **swizzled NSXPCConnection** to intercept calls to FamilyControlsAgent. When I tapped “Request Authorization,” I saw:

```
================= NSXPCConnection INTERCEPT =================
[+] Intercepted XPC call to: com.apple.FamilyControlsAgent
[+] Target Selector: getAuthorizationStatus:
===========================================================
```

Initially, I thought:
	The bug occurs in a *single shared path* (the getAuthorizationStatus: call) used by both 	authorizationStatus and requestAuthorization. It’s this call that hangs, not two separate issues.

That part was only partially true, as I learned later.

---
---

While my test app was frozen, I paused execution in Xcode and inspected the main thread’s call stack. Here’s a simplified trace:
```log
0  mach_msg2_trap
1  mach_msg2_internal
2  mach_msg_overwrite
3  mach_msg
4  _dispatch_mach_send_and_wait_for_reply
5  dispatch_mach_send_with_result_and_wait_for_reply
6  xpc_connection_send_message_with_reply_sync
7  __NSXPCCONNECTION_IS_WAITING_FOR_A_SYNCHRONOUS_REPLY__
```

This confirms the problem:
**A synchronous XPC call to FamilyControlsAgent never returns**, blocking the **main UI thread** and freezing the app entirely.

----

 I tried swizzling the XPC message to fake a response that says we’re not authorized yet, without actually sending the request to prevent the app from freezing. The good news is the app didn’t freeze, but nothing else happened either.

I also experimented with LLDB to skip over the authorization check in the FamilyControls framework and jump directly to the request logic. Still, nothing happened no freeze but also no respond 

That led me to think the issue might be tied to both the authorization check and the request process. One seems synchronous which causes the app to frezze and the other no

---

Why would the FamilyControlsAgent not reply?

The simplest explanation is that it crashed. i opened the system logs and searched for the agent.  Here’s what I found:

```
kernel: /System/Library/Frameworks/FamilyControls.framework/FamilyControlsAgent[1452] ==> temporary-sandbox
FamilyControlsAgent: activating connection: ... com.apple.cfprefsd.daemon
kernel: EXC_RESOURCE -> FamilyControlsAgent[1452] exceeded mem limit: InactiveHard 6 MB (fatal)
kernel: killing_specific_process pid 1452 [FamilyControlsAgent] (per-process-limit 10) 6144KBosanalyticshelper: Process FamilyControlsAgent [1452] killed by jetsam reason per-process-limit

kernel: /System/Library/Frameworks/FamilyControls.framework/FamilyControlsAgent[1454] ==> temporary-sandbox
```

The agent was stuck in a crash loop. The system would launch it, but it would immediately exceed its memory limit, triggering Jetsam (iOS’s memory manager) to kill it. Then, launchd would restart it and the cycle would repeat.

So the bug wasn’t in our app, the FamilyControls.framework, or even in how we interacted with the FamilyControlsAgent. The agent was already crashing on its own the whole time.

That’s why it could never respond no matter what we requested.
holding any app using the screen timne api unusable

----

**But why would FamilyControlsAgent exceed the 6MB memory limit in the first place?**

At first, I suspected a simple memory leak or some bad implementation inside the agent itself. But that didn’t make sense this issue wasn’t affecting everyone. So what was different?

I started to suspect maybe it is **something the agent was trying to read** during startup . And it turns out I was absolutely right.

i needed to know what the agent was doing right before it died. so i used fs_usage,
Most of the logs were standard system file access except for one line that appearies just moments before every single crash. 
```
open ... private/var/mobile/Library/com.apple.FamilyControlsAgent/Store.plist
read ...
close ...
... 0.13 seconds later ...
exit
```
That .plist file was **1.2MB**, unusually large for a plist file.

this was it. 
The agent was launching, reading its own big persistent storage file  Store.plist and this very act was causing the fatal memory error. 

---

The solution was suddenly, beautifully simple. If the Store.plist file was the problem, what would happen if it wasn't there?
I opened Filza app, navigated to: /private/var/mobile/Library/com.apple.FamilyControlsAgent/
found Store.plist, and simply renamed it to Store.plist.bak.

The moment I did, the crash loop in the system logs stopped. The FamilyControlsAgent launched and stayed alive. and created a new fresh Store.plist

I opened our demo app one last time. I tapped the "Request Authorization" button.
It worked. Instantly. The pop-up appeared. The freeze was gone.

All the other Screen Time-based apps started working again too 🎉🎉

---

It also looks like Apple quietly fixed the bug in iOS 17.6. I’m pretty sure they added logic to filter or trim the Store.plist file maybe by capping its size or removing old entries.

I considered reverse engineering the FamilyControlsAgent on iOS 17.6 to confirm what they changed…
But honestly, I think this is enough for now.
