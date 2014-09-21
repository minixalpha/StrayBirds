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


我们可以在调试版本的虚拟机中开启 ' -XX:+TraceClassLoading -XX:+TraceClassLoadingPreorder -XX:+Verbose' ，然后就可以看到我们平时喜闻乐见的一些类的加载过程了。

```
[Bootstrap loader class path=d:\jdk8\jre\lib\resources.jar;d:\jdk8\jre\lib\rt.ja
r;d:\jdk8\jre\lib\sunrsasign.jar;d:\jdk8\jre\lib\jsse.jar;d:\jdk8\jre\lib\jce.ja
r;d:\jdk8\jre\lib\charsets.jar;d:\jdk8\jre\lib\jfr.jar;d:\jdk8\jre\classes]
[Meta index for d:\jdk8\jre\lib\charsets.jar=sun/nio sun/awt]
[Meta index for d:\jdk8\jre\lib\jce.jar=javax/crypto sun/security META-INF/ORACL
E_J.RSA META-INF/ORACLE_J.SF]
[Meta index for d:\jdk8\jre\lib\jfr.jar=oracle/jrockit/ jdk/jfr com/oracle/jrock
it/]
[Meta index for d:\jdk8\jre\lib\jsse.jar=sun/security com/sun/net/]
[Meta index for d:\jdk8\jre\lib\rt.jar=com/sun/java/util/jar/pack/ java/ org/iet
f/ com/sun/beans/ com/sun/tracing/ com/sun/java/browser/ com/sun/corba/ com/sun/
media/ com/sun/awt/ com/sun/management/ sun/ com/sun/jmx/ com/sun/demo/ com/sun/
imageio/ com/sun/net/ com/sun/rmi/ org/w3c/ com/sun/swing/ com/sun/activation/ c
om/sun/nio/ com/sun/rowset/ org/jcp/ com/sun/istack/ jdk/ com/sun/naming/ org/xm
l/ org/omg/ com/sun/security/ com/sun/image/ com/sun/xml/ com/sun/java/swing/ co
m/oracle/ com/sun/java_cup/ com/sun/jndi/ com/sun/accessibility/ com/sun/org/ ja
vax/]
[Opened d:\jdk8\jre\lib\rt.jar]
[Loading java.lang.Object from d:\jdk8\jre\lib\rt.jar]
[Loaded java.lang.Object from d:\jdk8\jre\lib\rt.jar]
[Loading java.lang.String from d:\jdk8\jre\lib\rt.jar]

...

[Loading sun.misc.Launcher$ExtClassLoader from d:\jdk8\jre\lib\rt.jar]
[Loaded sun.misc.Launcher$ExtClassLoader from d:\jdk8\jre\lib\rt.jar]

...

[Loading sun.misc.Launcher$AppClassLoader from d:\jdk8\jre\lib\rt.jar]
[Loaded sun.misc.Launcher$AppClassLoader from d:\jdk8\jre\lib\rt.jar]

...

[Loading HelloWorld from file:/D:/HotSpotResearch/testcase/]
```

但是，我没有从中发现 lib\ext 目录下的扩展类被加载的过程。

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

使用双亲委派机制的优点，一般来说有两点，一是避免重复加载，二是出于安全性的考虑，如果用户自己定义的类加载器加载了JDK的核心类，
就可能对系统安全性造成破坏。

但是，如果有时候，我们想要自己加载类，不使用双亲委派机制，怎么办呢？可以用线程上下文加载器来加载这些类。


## 参考资料

1. [Java虚拟机类加载机制浅谈](http://computerdragon.blog.51cto.com/6235984/1223354)
