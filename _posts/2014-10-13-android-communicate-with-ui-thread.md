---
layout: default
title: Android 中非 UI 线程与 UI 线程通信方式
---

由于更新 UI 是非线程安全的，所以对 UI 的更新操作只能在 UI 线程中做。但是有时候，我们的一些耗时操作必须在后台线程中执行，执行完
之后，再更新 UI。这意味着后台操作要在后台线程中执行， UI 操作要在 UI 线程中执行。后台线程需要与UI线程通信，告诉他后台任务的执行
状态，以便 UI 线程能够更新。

在 Android 中，非 UI 线程可以通过消息队列机制或者异步任务机制，与 UI 线程通信。

## 消息队列机制

消息队列机制的基本思想是，非 UI 线程执行任务后向消息队列中发送消息， UI 线程从消息队列中取得消息，根据消息的内容更新 UI。

具体的说，一个 UI 线程通过 Looper.prepare() 创建一个 Looper, 它在初始化时，会生成一个消息队列。同时，UI 线程还需要一个或者多个 
Handler，每个 Handler 都与 UI 线程的 Looper 相关联，并且需要实现 handleMessage 方法。非 UI 线程会保存 Handler 的引用，当非 UI 
线程需要与 UI 线程通信时，可以用 Handler 的引用向其关联的 Looper 的消息队列中发送一个消息，并在消息中保存当前 Handler 的引用。
而对 Looper 来说，当它调用Looper.loop()启动后，就会不断从消息队列中取出消息，再通过消息中包含的 Handler，确定它的
handleMessage 方法，调用这个方法更新 UI。

## 异步任务机制

Android 提供了 AsyncTask 类，通过继承这个类，可以创建一个后台任务，并且很容易更新 UI。

AsyncTask 是一个抽象类，其中包含4个重要的方法，分别负责运行后台任务，或者更新 UI。

* onPreExecute
在任务执行前，由 UI 线程中调用，这个方法通常用来做一些初始化工作，例如在 UI 中显示一个进度条
* doInBackgroud
在后台线程中执行，在 onPreExecute 被调用后执行，可以执行一些长时间运行的后台任务，在这个方法中，还可以调用 publishProgress 方法
通知 UI 线程当前的进度
* onProgressUpdate
在 UI 线程中执行，在调用 pushlishProgress 方法后执行
* onPostExecute
在 UI 线程中执行，在后台线程执行结束后这个方法被调用 
