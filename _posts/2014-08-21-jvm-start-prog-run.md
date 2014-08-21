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

要了解一个系统是如何运行的，光看是不行的，要实际地运行，调试，修改才能对系统的动作方式有所了解。我是在 windows 7 64位平台上，使用 Visual Studio 2010 来调试，运行的。按照 GitHub 上的一个项目 [OpenJDK-Research](https://github.com/codefollower/OpenJDK-Research) 中说的来就可以了。另外，在 linux 平台上用 GDB 调试，可以参考 [HotSpot 实战](http://book.douban.com/subject/25847620/)，如果想用 NetBeans 调试，可以参考 [深入理解Java虚拟机](http://book.douban.com/subject/24722612/) 这本书上的讲解。


## 调用栈
```
openjdk-8-src-b132-03_mar_2014\jdk\src\share\bin\main.c::WinMain/main
  openjdk-8-src-b132-03_mar_2014\jdk\src\share\bin\java.c::JLI_Launch
    openjdk-8-src-b132-03_mar_2014\jdk\src\windows\bin\java_md.c::LoadJavaVM # Load JVM Library: jvm.dll
    openjdk-8-src-b132-03_mar_2014\jdk\src\windows\bin\java_md.c::JVMInit # Create JVM
      openjdk-8-src-b132-03_mar_2014\jdk\src\share\bin\java.c::ContinueInNewThread
          openjdk-8-src-b132-03_mar_2014\jdk\src\windows\bin\java_md.c::ContinueInNewThread0(JavaMain, threadStackSize, (void*)&args);
            _beginthreadex(NULL, (unsigned)stack_size, JavaMain, args, 0, &thread_id)
              openjdk-8-src-b132-03_mar_2014\jdk\src\share\bin\java.c::JavaMain
                openjdk-8-src-b132-03_mar_2014\jdk\src\share\bin\java.c::InitializeJVM
                  openjdk-8-src-b132-03_mar_2014\hotspot\src\share\vm\prims\jni.cpp::JNI_CreateJavaVM
                        
                        
```


## 关键函数
```java
/*
 * Load a jvm from "jvmpath" and initialize the invocation functions.
 */
jboolean
LoadJavaVM(const char *jvmpath, InvocationFunctions *ifn)
{
    HINSTANCE handle;

    JLI_TraceLauncher("JVM path is %s\n", jvmpath);

    /*
     * The Microsoft C Runtime Library needs to be loaded first.  A copy is
     * assumed to be present in the "JRE path" directory.  If it is not found
     * there (or "JRE path" fails to resolve), skip the explicit load and let
     * nature take its course, which is likely to be a failure to execute.
     *
     */
    LoadMSVCRT();

    /* Load the Java VM DLL */
    if ((handle = LoadLibrary(jvmpath)) == 0) {
        JLI_ReportErrorMessage(DLL_ERROR4, (char *)jvmpath);
        return JNI_FALSE;
    }

    /* Now get the function addresses */
    ifn->CreateJavaVM =
        (void *)GetProcAddress(handle, "JNI_CreateJavaVM");
    ifn->GetDefaultJavaVMInitArgs =
        (void *)GetProcAddress(handle, "JNI_GetDefaultJavaVMInitArgs");
    if (ifn->CreateJavaVM == 0 || ifn->GetDefaultJavaVMInitArgs == 0) {
        JLI_ReportErrorMessage(JNI_ERROR1, (char *)jvmpath);
        return JNI_FALSE;
    }

    return JNI_TRUE;
}
```


```java
int JNICALL
JavaMain(void * _args)
{
    ...
    
    InvocationFunctions ifn = args->ifn;

    JavaVM *vm = 0;
    JNIEnv *env = 0;
    
    ...
    
    /* Initialize the virtual machine */
    start = CounterGet();
    if (!InitializeJVM(&vm, &env, &ifn)) {
        JLI_ReportErrorMessage(JVM_ERROR1);
        exit(1);
    }
    
    
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
