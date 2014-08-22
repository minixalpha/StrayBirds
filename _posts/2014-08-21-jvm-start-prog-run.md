---
layout: default
title: Java虚拟机的启动与程序的运行
---

这篇文章是从 OpenJDK 源代码的角度讲当我们运行了 

```
java -classpath . hello
```

之后，java.exe 如何从 main 函数开始执行，启动虚拟机，并执行字节码中的代码。


## 实验环境

要了解一个系统是如何运行的，光看是不行的，要实际地运行，调试，修改才能对系统的动作方式有所了解。


起初我是按照 GitHub 上的一个项目 [OpenJDK-Research](https://github.com/codefollower/OpenJDK-Research) 在 windows 7 64位平台上，使用 Visual Studio 2010 来调试，运行的。但是后来发现，这个项目仅仅编译了HotSpot虚拟机， `java.exe` 并没有编译。

这里我们首先弄明白 `java.exe` 和虚拟机之间的关系。我们使用 Visual Studio 编译出的 HotSpot 是虚拟机，是作为动态链接库的形式被 `java.exe` 加载的。`java.exe` 负责解析参数，加载虚拟机链接库，它需要调用虚拟机中的函数来完成执行 Java 程序的功能。所以，你在HotSpot的源代码中找不到启动的程序的 `main` 函数，本来在 openjdk7 中，虚拟机是带有一个启动器的，在目录 `openjdk/hotspot/src/share/tools/launcher/java.c` 中可以找到 main 函数，但是在 openjdk8 中，这个启动器不见了，被放在 `openjdk/jdk` 目录下，而不是 `openjdk/hotspot` 目录下了，给我们的学习过程造成了伤害。 

所以我后来就在 linux 平台上调试了，因为在 windows 平台上，我始终没有把整个 openjdk8 编译成功，编译不出 `java.exe`, 仅仅编译了 `hotspot`，是看不到从 main 函数开始的执行的。关于如何在 linux 平台下编译调试 openjdk8，可以参考我的另一篇文章 [在Ubuntu 12.04 上编译 openjdk8](http://minixalpha.github.io/StrayBirds/2014/08/22/build-openjdk8-ubuntu.html). 



## 调用栈
```
jdk8u/jdk/src/share/bin/main.c::WinMain/main
  jdk8u/jdk/src/share/bin/java.c::JLI_Launch
    jdk8u/jdk/src/solaris/bin/java_md_solinux.c::LoadJavaVM # Load JVM Library: libjvm.so
    jdk8u/jdk/src/solaris/bin/java_md_solinux.c::JVMInit # Create JVM
      jdk8u/jdk/src/share/bin/java.c::ContinueInNewThread
        jdk8u/jdk/src/solaris/bin/java_md_solinux.c::ContinueInNewThread0(JavaMain, threadStackSize, (void*)&args);
          pthread_create(&tid, &attr, (void *(*)(void*))continuation, (void*)args)
            jdk8u/jdk/src/share/bin/java.c::JavaMain
              jdk8u/jdk/src/share/bin/java.c::InitializeJVM
                jdk8u\hotspot\src\share\vm\prims\jni.cpp::JNI_CreateJavaVM
                        
                        
```

## 执行过程

* main.c (jdk8u/jdk/src/share/bin/main.c)

```c
#ifdef JAVAW

char **__initenv;

int WINAPI
WinMain(HINSTANCE inst, HINSTANCE previnst, LPSTR cmdline, int cmdshow)
{
    int margc;
    char** margv;
    const jboolean const_javaw = JNI_TRUE;

    __initenv = _environ;
#else /* JAVAW */
int
main(int argc, char **argv)
{
    int margc;
    char** margv;
    const jboolean const_javaw = JNI_FALSE;
#endif /* JAVAW */
#ifdef _WIN32
    {
        int i = 0;
        if (getenv(JLDEBUG_ENV_ENTRY) != NULL) {
            printf("Windows original main args:\n");
            for (i = 0 ; i < __argc ; i++) {
                printf("wwwd_args[%d] = %s\n", i, __argv[i]);
            }
        }
    }
    JLI_CmdToArgs(GetCommandLine());
    margc = JLI_GetStdArgc();
    // add one more to mark the end
    margv = (char **)JLI_MemAlloc((margc + 1) * (sizeof(char *)));
    {
        int i = 0;
        StdArg *stdargs = JLI_GetStdArgs();
        for (i = 0 ; i < margc ; i++) {
            margv[i] = stdargs[i].arg;
        }
        margv[i] = NULL;
    }
#else /* *NIXES */
    margc = argc;
    margv = argv;
#endif /* WIN32 */
    return JLI_Launch(margc, margv,
                   sizeof(const_jargs) / sizeof(char *), const_jargs,
                   sizeof(const_appclasspath) / sizeof(char *), const_appclasspath,
                   FULL_VERSION,
                   DOT_VERSION,
                   (const_progname != NULL) ? const_progname : *margv,
                   (const_launcher != NULL) ? const_launcher : *margv,
                   (const_jargs != NULL) ? JNI_TRUE : JNI_FALSE,
                   const_cpwildcard, const_javaw, const_ergo_class);
}

```

这就是传说中的 `main` 函数的真身，可以看出，它针对操作系统是否使用 Windows ，执行了不同的代码段，最终调用 `JLI_Launch` 函数。

* JLI_Lanuch(jdk8u/jdk/src/share/bin/java.c)

```c
int
JLI_Launch(int argc, char ** argv,              /* main argc, argc */
        int jargc, const char** jargv,          /* java args */
        int appclassc, const char** appclassv,  /* app classpath */
        const char* fullversion,                /* full version defined */
        const char* dotversion,                 /* dot version defined */
        const char* pname,                      /* program name */
        const char* lname,                      /* launcher name */
        jboolean javaargs,                      /* JAVA_ARGS */
        jboolean cpwildcard,                    /* classpath wildcard*/
        jboolean javaw,                         /* windows-only javaw */
        jint ergo                               /* ergonomics class policy */
)
{

...

    if (!LoadJavaVM(jvmpath, &ifn)) {
        return(6);
    }

...

    return JVMInit(&ifn, threadStackSize, argc, argv, mode, what, ret);

}
```

从这里可以看出 JLI_Lanuch 的各个参数的含义, 我列出了关键代码， 其中 `LoadJavaVM` 完成载入虚拟机动态链接库，并初始化 `ifn` 中的函数指针，HotSpot虚拟机就是这样向启动器 `java` 提供功能。

* LoadJavaVM (jdk8u/jdk/src/solaris/bin/java_md_solinux.c)

这个函数涉及动态链接库，不同操作系统有不同接口，这里是针对 linux 的。

```c
jboolean
LoadJavaVM(const char *jvmpath, InvocationFunctions *ifn)
{
    ...
    
    libjvm = dlopen(jvmpath, RTLD_NOW + RTLD_GLOBAL);
    
    ...
    
    ifn->CreateJavaVM = (CreateJavaVM_t)
    dlsym(libjvm, "JNI_CreateJavaVM");
    
    ifn->GetDefaultJavaVMInitArgs = (GetDefaultJavaVMInitArgs_t)
    dlsym(libjvm, "JNI_GetDefaultJavaVMInitArgs");

    ifn->GetCreatedJavaVMs = (GetCreatedJavaVMs_t)
        dlsym(libjvm, "JNI_GetCreatedJavaVMs");

  ...
  
```

从这里可以看出载入动态链接库以及初始化 ifn 数据结构的代码。在我的调试版本中，`javapath` 指向之前编译出的动态链接库 `jdk8u/build/fastdebug/jdk/lib/i386/server/libjvm.so`. 


* JVM_Init(jdk8u/jdk/src/solaris/bin/java_md_solinux.c)

回到 `JLI_Lanuch` 函数，我们最终进入 `JVM_Init`, 这个函数会启动一个新线程。


```c
int
JVMInit(InvocationFunctions* ifn, jlong threadStackSize,
        int argc, char **argv,
        int mode, char *what, int ret)
{
    ShowSplashScreen();
    return ContinueInNewThread(ifn, threadStackSize, argc, argv, mode, what, ret);
}
```

`ContinueInNewThread` 会调用另一个函数 `ContinueInNewThread0` 启动线程，执行 `JavaMain` 函数:


```c
int
ContinueInNewThread0(int (JNICALL *continuation)(void *), jlong stack_size, void * args) {

...

    if (pthread_create(&tid, &attr, (void *(*)(void*))continuation, (void*)args) == 0) {
      void * tmp;
      pthread_join(tid, &tmp);
      rslt = (int)tmp;
    } else {
     /*
      * Continue execution in current thread if for some reason (e.g. out of
      * memory/LWP)  a new thread can't be created. This will likely fail
      * later in continuation as JNI_CreateJavaVM needs to create quite a
      * few new threads, anyway, just give it a try..
      */
      rslt = continuation(args);
    }

...


```


* JavaMain(jdk8u/jdk/src/share/bin/java.c)

这个函数会初始化虚拟机，加载各种类，并执行应用程序中的 `main` 函数。注释很详细。


```c

int JNICALL
JavaMain(void * _args)
{
    JavaMainArgs *args = (JavaMainArgs *)_args;
    int argc = args->argc;
    char **argv = args->argv;
    int mode = args->mode;
    char *what = args->what;
    InvocationFunctions ifn = args->ifn;

    JavaVM *vm = 0;
    JNIEnv *env = 0;
    jclass mainClass = NULL;
    jclass appClass = NULL; // actual application class being launched
    jmethodID mainID;
    jobjectArray mainArgs;
    int ret = 0;
    jlong start, end;

    RegisterThread();

    /* Initialize the virtual machine */
    start = CounterGet();
    if (!InitializeJVM(&vm, &env, &ifn)) {
        JLI_ReportErrorMessage(JVM_ERROR1);
        exit(1);
    }

    ...

    ret = 1;

    /*
     * Get the application's main class.
     *
     * See bugid 5030265.  The Main-Class name has already been parsed
     * from the manifest, but not parsed properly for UTF-8 support.
     * Hence the code here ignores the value previously extracted and
     * uses the pre-existing code to reextract the value.  This is
     * possibly an end of release cycle expedient.  However, it has
     * also been discovered that passing some character sets through
     * the environment has "strange" behavior on some variants of
     * Windows.  Hence, maybe the manifest parsing code local to the
     * launcher should never be enhanced.
     *
     * Hence, future work should either:
     *     1)   Correct the local parsing code and verify that the
     *          Main-Class attribute gets properly passed through
     *          all environments,
     *     2)   Remove the vestages of maintaining main_class through
     *          the environment (and remove these comments).
     *
     * This method also correctly handles launching existing JavaFX
     * applications that may or may not have a Main-Class manifest entry.
     */
    mainClass = LoadMainClass(env, mode, what);
    CHECK_EXCEPTION_NULL_LEAVE(mainClass);
    /*
     * In some cases when launching an application that needs a helper, e.g., a
     * JavaFX application with no main method, the mainClass will not be the
     * applications own main class but rather a helper class. To keep things
     * consistent in the UI we need to track and report the application main class.
     */
    appClass = GetApplicationClass(env);
    NULL_CHECK_RETURN_VALUE(appClass, -1);
    /*
     * PostJVMInit uses the class name as the application name for GUI purposes,
     * for example, on OSX this sets the application name in the menu bar for
     * both SWT and JavaFX. So we'll pass the actual application class here
     * instead of mainClass as that may be a launcher or helper class instead
     * of the application class.
     */
    PostJVMInit(env, appClass, vm);
    /*
     * The LoadMainClass not only loads the main class, it will also ensure
     * that the main method's signature is correct, therefore further checking
     * is not required. The main method is invoked here so that extraneous java
     * stacks are not in the application stack trace.
     */
    mainID = (*env)->GetStaticMethodID(env, mainClass, "main",
                                       "([Ljava/lang/String;)V");
    CHECK_EXCEPTION_NULL_LEAVE(mainID);

    /* Build platform specific argument array */
    mainArgs = CreateApplicationArgs(env, argv, argc);
    CHECK_EXCEPTION_NULL_LEAVE(mainArgs);

    /* Invoke main method. */
    (*env)->CallStaticVoidMethod(env, mainClass, mainID, mainArgs);

    /*
     * The launcher's exit code (in the absence of calls to
     * System.exit) will be non-zero if main threw an exception.
     */
    ret = (*env)->ExceptionOccurred(env) == NULL ? 0 : 1;
    LEAVE();
}

```

注意 InitializeJVM 函数，它会调用之前初始化的 `ifn` 数据结构中的 `CreateJavaVM` 函数，这个函数指向虚拟机动态链接库中的 `JNI_CreateJavaVM` 函数，这个函数会真正创建虚拟机。

这个函数位于 `jdk8u\hotspot\src\share\vm\prims\jni.cpp`。

我之前在 Windows 下调试，直接调试的 HotSpot 动态链接库，可以看到的第一个函数就是 `JNI_CreateJavaVM`, 之前的调用都位于 `java.exe` 代码中。因为 Windows 中 `java.exe` 不是我们自己编译的，看不到其中调用关系。如下图所示：

![invoke_stack](../images/invoke_stack.png)

同时可以看到两个线程

![threads](../images/thread.png)


