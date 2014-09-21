---
layout: default
title: Java 类加载机制
---

## 加载过程

假如我们运行一个程序，那么，Java 是如何进行类加载过程的呢？

```
java HelloWorld
```

基本的过程

1. 查找 jvm.dll ，初始化 JVM
2. 产生 Bootstrap Loader (启动类加载器)，并加载 JAVA_HOME/lib 下的Java核心API
3. Bootstrap Loader 加载 Extended Loader (标准扩展类加载器), Extended Loader 加载 JAVA_HOME/lib/ext 下的扩展类
4. Bootstrap Loader 加载 AppClass Loader (系统加载器), 并将其父加载器设置为 Extended Loader
5. AppClass Loader 加载 CLASSPATH 目录下的 HelloWorld 类

## 双亲委派机制

双亲委派机制的基本思想是，一个类加载器加载类时，首先委派给父加载器去加载，只有父加载器无法加载时，才自己去加载。

看下JDK中一个体现双亲委派机制的函数 ` java.lang.ClassLoader.loadClass` ： 

```java
    protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // First, check if the class has already been loaded
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    long t1 = System.nanoTime();
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }
```

算法的基本思想是，先看这个类是否被加载过了，如果没有被加载过，看下父加载器是否为空，如果不为空，让父加载器加载，否则，
使用 Bootstrap Loader 加载。如果之前的尝试都失败了，则当前类加载器调用 findClass 加载类。


## 参考资料

1. [Java虚拟机类加载机制浅谈](http://computerdragon.blog.51cto.com/6235984/1223354)
