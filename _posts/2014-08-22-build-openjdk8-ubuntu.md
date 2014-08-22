---
layout: default
title: 在Ubuntu 12.04 上编译 openjdk8
---


## 前言

现在看的资料都是编译 openjdk7 的，openjdk8好像已经 openjdk7 编译方式大一样，按照前辈的文章使用 

```
make sanity
```

会提示找不到 sanity 规则，然后编译过程其实基本就直接 

```
./configure
make all
```

官方的 README 写的很清楚。

下面记录下过程


## 下载代码 


```
hg clone http://hg.openjdk.java.net/jdk8u/jdk8u jdk8u
cd jdk8u 
bash ./get_source.sh
```

然后下载代码，进入代码目录：

```
cd jdk8u
```


## 安装依赖 

```
sudo aptitude build-dep openjdk-7 
sudo aptitude install openjdk-7-jdk
```


## 配置 

* 环境变量 

```
export LANG=C 
export PATH="/usr/lib/jvm/java-7-openjdk/bin:${PATH}"
```

* 配置编译选项

```
bash ./configure
```

这样生成相应默认配置，如果有需要，比如想编译出调试版本的，可以给 `configure` 加参数。

```
A new configuration has been successfully created in
/home/minix/SourceCode/openjdk8/jdk8u/build/linux-x86-normal-server-release
using default settings.

Configuration summary:
* Debug level:    release
* JDK variant:    normal
* JVM variants:   server
* OpenJDK target: OS: linux, CPU architecture: x86, address length: 32

Tools summary:
* Boot JDK:       java version "1.7.0_17" Java(TM) SE Runtime Environment (build 1.7.0_17-b02) Java HotSpot(TM) Server VM (build 23.7-b01, mixed mode)  (at /home/minix/Software/jdk1.7.0_17)
* C Compiler:     gcc-4.6 (Ubuntu/Linaro 4.6.3-1ubuntu5) version 4.6.3 (at /usr/bin/gcc-4.6)
* C++ Compiler:   g++-4.6 (Ubuntu/Linaro 4.6.3-1ubuntu5) version 4.6.3 (at /usr/bin/g++-4.6)

Build performance summary:
* Cores to use:   3
* Memory limit:   3878 MB
* ccache status:  not installed (consider installing)

Build performance tip: ccache gives a tremendous speedup for C++ recompilations.
You do not have ccache installed. Try installing it.
You might be able to fix this by running 'sudo apt-get install ccache'.
```

可以看出提示缺少 ccache 包，按提示安装就可以了。从提示可以看出，编译级别是 `release`，另外还有几种编译级别，可以在调试时候提供更多的信息。例如：

```
bash ./configure --enable-debug
```

这样会生成 `fastdebug` 版本的配置信息：

```
A new configuration has been successfully created in
/home/minix/openjdk8/jdk8u/build/linux-x86-normal-server-fastdebug
using configure arguments '--enable-debug'.

Configuration summary:
* Debug level:    fastdebug
* JDK variant:    normal
* JVM variants:   server
* OpenJDK target: OS: linux, CPU architecture: x86, address length: 32

Tools summary:
* Boot JDK:       java version "1.7.0_17" Java(TM) SE Runtime Environment (build 1.7.0_17-b02) Java HotSpot(TM) Server VM (build 23.7-b01, mixed mode)  (at /home/minix/Software/jdk1.7.0_17)
* C Compiler:     gcc-4.6 (Ubuntu/Linaro 4.6.3-1ubuntu5) version 4.6.3 (at /usr/bin/gcc-4.6)
* C++ Compiler:   g++-4.6 (Ubuntu/Linaro 4.6.3-1ubuntu5) version 4.6.3 (at /usr/bin/g++-4.6)

Build performance summary:
* Cores to use:   3
* Memory limit:   3878 MB
* ccache status:  installed and in use
```

注意编译的级别已经变成 `fastdebug` 了。


## 编译

编译直接 

```
make
```

就可以了，如果提示

```
No CONF given, but more than one configuration found in /home/minix/openjdk8/jdk8u//build.
Available configurations:
* linux-x86-normal-server-fastdebug
* linux-x86-normal-server-release
Please retry building with CONF=<config pattern> (or SPEC=<specfile>)

```

需要指定使用哪个编译配置：

```
make CONF=linux-x86-normal-server-fastdebug
```

最后编译成功后，会提示：

```
----- Build times -------
Start 2014-08-22 10:56:52
End   2014-08-22 11:16:31
00:00:30 corba
00:13:38 hotspot
00:00:22 jaxp
00:00:30 jaxws
00:04:10 jdk
00:00:29 langtools
00:19:39 TOTAL
```

查看 `build` 目录，可以看到 `linux-x86-normal-server-fastdebug`

切换到 `jdk/bin` 目录：

```
cd linux-x86-normal-server-fastdebug/jdk/bin/
```

运行可执行文件 java

```
./java -version
```

会得到提示

```
openjdk version "1.8.0-internal-fastdebug"
OpenJDK Runtime Environment (build 1.8.0-internal-fastdebug-minix_2014_08_22_10_56-b00)
OpenJDK Server VM (build 25.40-b05-fastdebug, mixed mode)
```

## 调试 

下面展示一个启动 `GDB`, 加断点，并运行一个 Java 程序的过程。


```
$ gdb java

GNU gdb (Ubuntu/Linaro 7.4-2012.04-0ubuntu2.1) 7.4-2012.04
Copyright (C) 2012 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "i686-linux-gnu".
For bug reporting instructions, please see:
<http://bugs.launchpad.net/gdb-linaro/>...
Reading symbols from /home/minix/openjdk8/jdk8u/build/fastdebug/jdk/bin/java...done.

(gdb) b main
Breakpoint 1 at 0x8048410: file /home/minix/openjdk8/jdk8u/jdk/src/share/bin/main.c, line 94.

(gdb) r -classpath PossibleReordering

Starting program: /home/minix/openjdk8/jdk8u/build/fastdebug/jdk/bin/java -classpath PossibleReordering
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/i386-linux-gnu/libthread_db.so.1".

Breakpoint 1, main (argc=3, argv=0xbfffeca4)
    at /home/minix/openjdk8/jdk8u/jdk/src/share/bin/main.c:94

```
