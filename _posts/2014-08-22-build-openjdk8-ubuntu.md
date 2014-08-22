---
layout: default
title: 在Ubuntu 12.04 上编译 openjdk8
---


## 前言

现在看的资料都是编译 openjdk7 的，openjdk8好像已经 openjdk7 编译方式大一样，按照前辈的文章使用 

```
make sanity
```

会提示找到 sanity 规则，然后的编译过程其实基本就直接 

```
./configure
make all
```

其实最好就是读官方的 README,

下面记录下过程


## 下载代码 


```
hg clone http://hg.openjdk.java.net/jdk8/jdk8 YourOpenJDK 
cd YourOpenJDK 
bash ./get_source.sh
```

然后下载代码，进入代码目录：

```
cd YourOpenJDK
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

这样生成相应配置

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
* Boot JDK:       java version "1.7.0_17" Java(TM) SE Runtime Environment (build 1.7.0_17-b02) Java HotSpot(TM) Server VM (build 23.7-b01, mixed mode)  (at /home/zhaoxk/Software/jdk1.7.0_17)
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

可以看出提示缺少 ccache 包，按提示安装就可以了。从提示可以看出，


## 编译
