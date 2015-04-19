---
layout: default
title: Java 增强for循环
comments: true
---
##增强for循环

学习java有一段时间了，或多或少看到过增强for循环的使用例子，这次就整理了一下增强for循环的使用细节。
###一、增强for循环的基本使用介绍
1、数组
```java	   
String[] list={"1","2","3"};
//普通for循环
for(int i=0;i<list.length;i++){
	System.out.print(list[i]);
}
//增强for循环
for(String num:list){
	System.out.print(num);
}
``` 
2、ArrayList与泛型的结合
```java	
ArrayList<String> list=new ArrayList<String>();
list.add("1");	
list.add("2");
list.add("3");
//普通for循环
for(int i=0;i<list.size();i++){
	System.out.print(list.get(i));
}
//增强for循环
for(String num:list){
	System.out.print(num);
}
```
3、set等集合
```java
Set<String> set=new HashSet<String>();
set.add("1");
set.add("2");
set.add("3");
//普通循环
Iterator<String> it = set.iterator();
while(it.hasNext()){
	System.out.print(it.next());
}
//增强for循环
for(String num:set){
	System.out.print(num);
}
```
###二、使用增强for循环的场合
在网上查询了很多增强for循环的相关知识以及API，增强for循环主要用在没有具体下标的集合中，比如set，map等，可以减少使用iterator迭代器的代码。而在数组，以及ArrayList（建立在数组上的集合）上，如果要遍历则也可以使用增强for循环，不然还是使用普通循环来的方便。
###三、注意点
增强for循环在使用时注意不要对集合本身进行操作，如集合内数据的增加删除等。
如:
```java
for(String num:set){
	set.remove(num);
}
```
代码报错
