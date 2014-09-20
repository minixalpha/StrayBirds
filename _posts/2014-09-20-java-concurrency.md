---
layout: default
title: Java 中的并发
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
