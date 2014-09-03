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

注意虚拟机并不是直接用 instanceKlass 表示 Java 类了，而是又创建了一个镜像类，并且二者相互引用 。 至于为什么这样做，我还没有搞清楚。可以参考 RednaxelaFX 的这篇 [借助HotSpot SA来一窥PermGen上的对象](http://rednaxelafx.iteye.com/blog/730461) 中相应的描述。

 在源代码中 `ClassFileParser::parseClassFile` 方法负责解析 `.class` 文件。


```cpp
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

我们注意到， `create_mirror` 方法是在 `java_lang_Class` 类中。另外，我们还听过 Java 中的类都是 `java.lang.Class` 的实例，这
二者之间是不是有什么关系呢？到 `create_mirror` 里我们可以看到

```cpp
oop java_lang_Class::create_mirror(KlassHandle k, Handle protection_domain, TRAPS) {
   ...
    // Allocate mirror (java.lang.Class instance)
    Handle mirror = InstanceMirrorKlass::cast(SystemDictionary::Class_klass())->allocate_instance(k, CHECK_0);
    
    ...
    }
```

说明这个 mirror 确实是 `java.lang.Class` 的实例。另外，我们平时通过 `obj.getClass()` 得到的类，其实是通过 `obj->_klass->_java_mirror ` 得到的，参见 R 的另一篇 [借HSDB来探索HotSpot VM的运行时数据](http://rednaxelafx.iteye.com/blog/1847971). 也就是说，直接暴露给 Java 的 是 java_mirror, 而不是 InstanceKlass.


## 静态变量存储在哪里

首先，根据 R 在刚才提到的博客中的说法，静态变量存储在 `_java_mirror` 的尾部，我们再搜索一下，静态变量是如何从 `.class` 中提取
出来，分配空间并初始化的，又存储在哪里。

在上一节的代码中，我们可以看到这一句：

```cpp
  Array<u2>* fields = parse_fields(class_name,
                                     access_flags.is_interface(),
                                     &fac, &java_fields_count,
                                     CHECK_(nullHandle));
```

在 《HotSpot实战》中，3.2 节介绍类的加载过程时说过

```
这个函数用于读取字段信息，计算出域大小与偏移量，并根据域分配策略对
字段存储进行分配。这些信息会在后续步骤填入 instanceKlass  对象中成为类信息的一部分。
```

根据 Java 虚拟机规范 中的说明，字段(field) 由下列数据结构表示：

```cpp
field_info {
 u2 access_flag;
 u2 name_index;
 u2 descriptor_index;
 u2 attriutes_count;
 attribute_info attributes[attributes_count];
}
```

其中 `access_flag` 表示访问属性，例如 是否为 `public`, 是否为 `static` 等等。 `name_index` 表示这个字段的名字
在常量池中的索引。 `descriptor_index` 表示这个字段的描述符在常量池中的索引，描述符表示字段的语法符号，比如整形数
的语法符号是 'I', 一个一维数组的语法符号是 '['. 

我们来看一下这个函数，从中可以看到函数如何填充 `field_info`. 

```cpp
// OpenJDK-Research\hotspot\src\share\vm\classfile\classFileParser.cpp

Array<u2>* ClassFileParser::parse_fields(Symbol* class_name,
                                         bool is_interface,
                                         FieldAllocationCount *fac,
                                         u2* java_fields_count_ptr, TRAPS) {
  ...
  
  // Allocate a temporary resource array for field data. For each field,
  // a slot is reserved in the temporary array for the generic signature
  // index. After parsing all fields, the data are copied to a permanent
  // array and any unused slots will be discarded.
  ResourceMark rm(THREAD);
  u2* fa = NEW_RESOURCE_ARRAY_IN_THREAD(
             THREAD, u2, total_fields * (FieldInfo::field_slots + 1));
  
  for (int n = 0; n < length; n++) {
    ...
    
    AccessFlags access_flags;
    jint flags = cfs->get_u2_fast() & JVM_RECOGNIZED_FIELD_MODIFIERS;
    access_flags.set_flags(flags);
    
    
    u2 name_index = cfs->get_u2_fast();
    Symbol*  name = _cp->symbol_at(name_index); 
    
    
    u2 signature_index = cfs->get_u2_fast();
    Symbol*  sig = _cp->symbol_at(signature_index);
    
    ...
    
    FieldInfo* field = FieldInfo::from_field_array(fa, n);
    field->initialize(access_flags.as_short(),
                      name_index,
                      signature_index,
                      constantvalue_index);
                      
    // Remember how many oops we encountered and compute allocation type
    FieldAllocationType atype = fac->update(is_static, type);
    field->set_allocation_type(atype);
  }
  
  Array<u2>* fields = MetadataFactory::new_array<u2>(
          _loader_data, index * FieldInfo::field_slots + num_generic_signature,
          CHECK_NULL);
  _fields = fields; // save in case of error
  {
    int i = 0;
    for (; i < index * FieldInfo::field_slots; i++) {
      fields->at_put(i, fa[i]);
    }
    for (int j = total_fields * FieldInfo::field_slots;
         j < generic_signature_slot; j++) {
      fields->at_put(i++, fa[j]);
    }
    assert(i == fields->length(), "");
  }

```

先开辟数组空间 fa, 再从类文件中提取信息，然后将信息临时放在 `fa` 中的 `FieldInfo` 结构中，等解析完所有字段，将所有字段信息最终放在 `fields` 。另外，注意那句 `fac->update`，它会统计所有类型出现的次数。 

我们看看`FieldInfo` 的 `initialize` 方法:

```cpp
  void initialize(u2 access_flags,
                  u2 name_index,
                  u2 signature_index,
                  u2 initval_index) {
    _shorts[access_flags_offset] = access_flags;
    _shorts[name_index_offset] = name_index;
    _shorts[signature_index_offset] = signature_index;
    _shorts[initval_index_offset] = initval_index;
    _shorts[low_packed_offset] = 0;
    _shorts[high_packed_offset] = 0;
  }
```

我们注意到这里并没有计算偏移，计算偏移还是在 `parseClassFile` 方法中：

```cpp
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
  
  
  	// layout_fields 会为 fields 设置 offset
    FieldLayoutInfo info;
    layout_fields(class_loader, &fac, &parsed_annotations, &info, CHECK_NULL);
  }
  ```
  
  其中的 `layout_fields` 会根据之前统计的各个类型的数目 `fac`，计算各种类型数据的偏移。


```cpp
//

void ClassFileParser::layout_fields(Handle class_loader,
                                    FieldAllocationCount* fac,
                                    ClassAnnotationCollector* parsed_annotations,
                                    FieldLayoutInfo* info,
                                    TRAPS) {
                                    
                                    ...
                                    
  // Iterate over fields again and compute correct offsets.
  // The field allocation type was temporarily stored in the offset slot.
  // oop fields are located before non-oop fields (static and non-static).
  for (AllFieldStream fs(_fields, _cp); !fs.done(); fs.next()) {

    // skip already laid out fields
    if (fs.is_offset_set()) continue;

    // contended instance fields are handled below
    if (fs.is_contended() && !fs.access_flags().is_static()) continue;

    int real_offset;
    FieldAllocationType atype = (FieldAllocationType) fs.allocation_type();

    // pack the rest of the fields
    switch (atype) {
      case STATIC_OOP:
        real_offset = next_static_oop_offset;
        next_static_oop_offset += heapOopSize;
        break;
      case STATIC_BYTE:
        real_offset = next_static_byte_offset;
        next_static_byte_offset += 1;
        break;
      case STATIC_SHORT:
        real_offset = next_static_short_offset;
        next_static_short_offset += BytesPerShort;
        break;
      case STATIC_WORD:
        real_offset = next_static_word_offset;
        next_static_word_offset += BytesPerInt;
        break;
      case STATIC_DOUBLE:
        real_offset = next_static_double_offset;
        next_static_double_offset += BytesPerLong;
        break;
      
      ...
      
      default:
        ShouldNotReachHere();
    }
    fs.set_offset(real_offset);
  }
  
  
  ....
  
    // Pass back information needed for InstanceKlass creation
  info->nonstatic_oop_offsets = nonstatic_oop_offsets;
  info->nonstatic_oop_counts = nonstatic_oop_counts;
  info->nonstatic_oop_map_count = nonstatic_oop_map_count;
  info->total_oop_map_count = total_oop_map_count;
  info->instance_size = instance_size;
  info->static_field_size = static_field_size;
  info->nonstatic_field_size = nonstatic_field_size;
  info->has_nonstatic_fields = has_nonstatic_fields;
}
```

其中的 `set_offset` 会真正设置 `FieldInfo` 中的偏移，我们一路跟踪过去，会发现调用`FieldInfo` 的 这个函数设置了偏移。


```
  void set_offset(u4 val)                        {
    val = val << FIELDINFO_TAG_SIZE; // make room for tag
    _shorts[low_packed_offset] = extract_low_short_from_int(val) | FIELDINFO_TAG_OFFSET;
    _shorts[high_packed_offset] = extract_high_short_from_int(val);
  }
```

其中的 `info` 会将偏移信息记录下来，在创建 `InstanceKlass` 的时候会用上。

```cpp

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
  ...
  
```

注意 `info.static_field_size` 会被传进去，用于分配空间。

然后，再看看 `mirror` 的创建。

```cpp
    // Allocate mirror and initialize static fields
    java_lang_Class::create_mirror(this_klass, protection_domain, CHECK_(nullHandle));
```
  

