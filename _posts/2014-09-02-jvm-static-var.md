---
layout: default
title: Java 虚拟机如何访问类的静态变量
---



要明白 Java 虚拟机如何访问类的静态变量，首先要明白下面几个问题：

* 虚拟机内部是如何表示一个 Java 类的
* 静态变量存储在哪里
* 虚拟机如何访问到这些静态变量

这篇文章也从这围绕这三个问题展开，并结合 OpenJDK 中 HotSpot 的源代码作分析。


## Java 虚拟机内部如何表示类

HotSpot 虚拟机在内部使用两组类来表示 Java 的类和对象 

* OOP(ordinary object pointer)，用来描述对象实例信息
* Klass,用来描述Java类，是虚拟机内部Java类的对等体

例如，constantPoolOop 表示在 Class 文件中描述 的常量池， instanceOop 表示 一个Java类型实例; instanceKlass 在虚拟机层面
描述一个 Java 类。

虚拟机在拿到一个 `.class` 文件后，会对它进行解析，并创建相应的 instanceKlass。其中主要包含:

* 读取魔数
* 读取版本号
* 解析常量池 (parse_constant_pool)
* 解析字段信息 (parse_fields)
* 创建 Java 镜像类 (create_mirror)

注意虚拟机并不是直接用 instanceKlass 表示 Java 类了，而是又创建了一个镜像类，并且二者相互引用 。 至于为什么这样做，我还没有搞清楚。可以参考 RednaxelaFX 的这篇 [博客](http://rednaxelafx.iteye.com/blog/730461) 中相应的描述。

 在源代码中 `ClassFileParser::parseClassFile` 方法负责解析 `.class` 文件。


```c++
//OpenJDK-Research\hotspot\src\share\vm\classfile\classFileParser.cpp

instanceKlassHandle 
ClassFileParser::parseClassFile(
  Symbol* name,
  ClassLoaderData* loader_data,
  Handle protection_domain,
  KlassHandle host_klass,
  GrowableArray<Handle>* cp_patches,
  TempNewSymbol& parsed_name,
  bool verify,
  TRAPS) {
  
  ...
  
  // Magic value
  u4 magic = cfs->get_u4_fast();
  

  // Version numbers
  u2 minor_version = cfs->get_u2_fast();
  u2 major_version = cfs->get_u2_fast();
  
  // Constant pool
  constantPoolHandle cp = parse_constant_pool(CHECK_(nullHandle));
  
  Array<u2>* fields = parse_fields(class_name,
                                     access_flags.is_interface(),
                                     &fac, &java_fields_count,
                                     CHECK_(nullHandle));
                                     
                                     
  // We can now create the basic Klass* for this klass
  _klass = InstanceKlass::allocate_instance_klass(loader_data,
                                                    vtable_size,
                                                    itable_size,
                                                    info.static_field_size,
                                                    total_oop_map_size2,
                                                    rt,
                                                    access_flags,
                                                    name,
                                                    super_klass(),
                                                    !host_klass.is_null(),
                                                    CHECK_(nullHandle));
  
// Allocate mirror and initialize static fields
java_lang_Class::create_mirror(this_klass, protection_domain, CHECK_(nullHandle));
  ...

}

```
