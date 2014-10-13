---
layout: default
title: Android中的进程
---

Android 运行在 Linux 上，Android 系统会为每个 App 创建一个 Linux 账号，并通过这个账号启动一个进程，这个进程运行一个 Dalvik 虚拟机，
每个 App 都运行在一个单独的 Dalvik 虚拟机内。

默认情况下，所有 Activity 都运行在同一进程下，不过可以通过对 AndroidManifest.xml 的配置改变 Activity 的运行方式，例如 

```
<activity android:name=".FooActivity" android:process="net.minixalpha.p2>
</activity>
```

此时， `FooActivity` 会运行在单独的进程内。
