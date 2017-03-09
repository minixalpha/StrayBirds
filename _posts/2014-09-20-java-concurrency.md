---
layout: post
title: Java 中的并发
category: 技术
comments: true
---


## 如何创建一个线程

按 Java 语言规范中的说法，创建线程只有一种方式，就是创建一个 Thread 对象。而从 HotSpot 虚拟机的角度看，创建一个虚拟机线程
有两种方式，一种是创建 Thread 对象，另一种是创建 一个本地线程，加入到虚拟机线程中。

如果从 Java 语法的角度。有两种方法。

第一是继承 Thread 类，实现 run 方法，并创建子类对象。

```java
	public void startThreadUseSubClass() {
		class MyThread extends Thread {
			public void run() {
				System.out.println("start thread using Subclass of Thread");
			}
		}

		MyThread thread = new MyThread();
		thread.start();
	}
```

另一种是传递给 Thread 构造函数一个 Runnable 对象。

```java
	public void startThreadUseRunnalbe() {
		Thread thread = new Thread(new Runnable() {
			public void run() {
				System.out.println("start thread using runnable");
			}
		});
		thread.start();
	}
```

当然， Runnalbe 对象，也不是只有这一种形式，例如如果我们想要线程执行时返回一个值，就需要用到另一种 Runnalbe 对象，它
对原来的 Runnalbe 对象进行了包装。

```java
	public void startFutureTask() {
		FutureTask<Integer> task = new FutureTask<>(new Callable<Integer>() {
			public Integer call() {
				return 1;
			}
		});

		new Thread(task).start();

		try {
			Integer result = task.get();
			System.out.println("future result " + result);
		} catch (InterruptedException e) {
			e.printStackTrace();
		} catch (ExecutionException e) {
			e.printStackTrace();
		}
	}
```

## 结束线程

## wait 与 sleep

sleep 会使得当前线程休眠一段时间，但并不会释放已经得到的锁。

wait 会阻塞住，并释放已经得到的锁。一直到有人调用 notify 或者 notifyAll，它会重新尝试得到锁，然后再唤醒。

## 线程池 

### 好处

* 复用

线程池中有一系列线程，这些线程在执行完任务后，并不会被销毁，而会从任务队列中取出任务，执行这些任务。这样，就避免为每个任务
都创建线程，销毁线程。 在有大量短命线程的场景下，如果创建线程和销毁线程的时间比线程执行任务的时间还长，显然是不划算的，这时候，使用线程池就会有明显
的好处。

* 流控

同时，可以设置线程数目，这样，线程不会增大到影响系统整体性能的程度。当任务太多时，可以在队列中排队，
如果有空闲线程，他们会从队列中取出任务执行。

### 使用

* 线程数目

那么，线程的数目要设置成多少呢？这需要根据任务类型的不同来设置，假如是大量计算型的任务，他们不会阻塞，那么可以将线程数目设置
为处理器数目。而如果任务中涉及大量IO，有些线程会阻塞住，这样就要根据阻塞线程数目与运行线程数目的比例，以及处理器数目来设置
线程总数目。例如阻塞线程数目与运行线程数目之比为n, 处理器数目为p，那么可以设置 n * (p + 1) 个线程，保证有 n 个线程处于运行
状态。

* Executors

JDK 的 java.util.concurrent.Executors 类提供了几个静态的方法，用于创建不同类型的线程池。

```java
ExecutorService service = Executors.newFixedThreadPool(10);
ArrayList<Future<Integer>> results = new ArrayList<>();
for (int i = 0; i < 14; i++) {
	Future<Integer> r = service.submit(new Callable<Integer>() {
		public Integer call() {
		return new Random().nextInt();
	});
	results.add(r);
}
```

`newFixedThreadPool` 可以创建固定数目的线程，一旦创建不会自动销毁线程，即便长期没有任务。除非显式关闭线程池。如果任务队列中有任务，就取出任务执行。

另外，还可以使用 `newCachedThreadPool` 方法创建一个不设定固定线程数目的线程池，它有一个特性，线程完成任务后，如果一分钟之内又有新任务，就会复用这个线程执行新任务。如果超过一分钟还没有任务执行，就会自动销毁。

另外，还提供了 `newSingleThreadExecutor` 创建有一个工作线程的线程池。

### 原理 

JDK 中的线程池通过 HashSet 存储工作者线程，通过 BlockingQueue 来存储待处理任务。

