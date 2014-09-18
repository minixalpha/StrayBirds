---
layout: default
title: Java中为什么不允许直接创建泛型数组
---

## 失败的方式 

在Java中，如果使用泛型数组，会出现编译错误, 例如 

```java
List<String>[] stringLists = new List<String>[1];
```

会提示

```
Error: Cannot create a generic array of List<String>
```

那么，在Java中为什么不允许创建泛型数组呢？  《Effective Java》第五章给出了一个解释。

使用泛型的作用是使得程序在编译期可以检查出与类型相关的错误，但是如果使用了泛型数组，这种能力就会受到破坏。

首先，假如我们可以声明这样一个泛型数组。

```java
List<String>[] stringLists = new List<String>[1];
```

那么，由于在 Java 中，数组是协变(covariant)的，这意味着基类类型的数组可以接收子类类型的数组，例如

```java
Object[] objects = stringLists;
```

一旦我们这样做之后，就可以通过objects向 stringLists中添加非List<String>类型的数据。

```java
List<Integer> intList = Arrays.asList(1);
objects[0] = intList
```

随后，再使用 stringList 时，stringList[0] 就会保存 intList, 而使用下面的代码，编译器不会提示错误，但运行时，就会出错。

```java
String str = stringList[0].get(0);
```

## 成功的方式

我们可以通过，先擦除类型，再强制转化的方式创建一个泛型数组。

```java
List<String>[] stringLists = (List<String>[])new List[10];
```

不过，随后你会发现，之前说的所有事情对这个通过编译的语句依然成立，你可以通过数组协变去掉泛型数组的类型，再给它添加一个不同类型
的数据，然后再访问这个数据。然后运行的时候就会出错 。。。


```java
		List<String>[] stringLists = (List<String>[])new List[10];
		Object[] objects = stringLists;
		List<Integer> intList = Arrays.asList(1);
		objects[0] = intList;
		String str = stringLists[0].get(0);
```


