---
layout:     post
title:      "Objective-C内存模型分析"
date:       2017-11-04 18:58:00
author:     "dingjingpisces"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - 技术
    - iOS开发
    - 源码 
---


# Objective-C内存模型分析

## 简介
---

在iOS开发中，我们知道iOS对象可以在运行时被操作，执行动态查找/执行方法等操作。通过阅读runtime源码，很容易将objc_class和Class联系在一起，认为一个类的存储结构是和objc_class一致。但是编译器如何将Objective-c的类编译成objc_class结构，objc_class中如method_list结构中存储的OC方法如何获取到函数地址，成员变量内存地址如何偏移等问题一直像一团迷雾一样阻隔在OC对象和其runtime结构之间，本文希望通过clang的rewrite-objc揭开这层神秘的面纱，加深对运行时机制的理解。

## 几个一直困扰的问题

---

下面是在研究Objective-C内存模型过程中一直思考和希望解决的问题，在文章的中会一一给出解答。

1. clang rewrite-objc做了什么。
2. 编译器怎么把OC类转换为C++类。
3. 运行时如何加载所有类。
4. 对象的成员变量如何排列。
5. 为啥category不能添加成员变量。


## rewrite-objc

为了研究编译器在编译OC代码的过程中，先利用rewrite-objc方法将OC的*[object.m](https://github.com/dingjingpisces2015/objc4-709/blob/master/RewriteToC/object.m)* 文件重写成 *[object.cpp](https://github.com/dingjingpisces2015/objc4-709/blob/master/RewriteToC/object.cpp)*.

拥有两个属性，一个方法的Test类

```
@interface Test : NSObject

@property NSString *str；
@property NSObject *obj;
- (void)doSomething;

@end

```

经过转换，首先可以看到定义了
```
typedef struct objc_object Test;
```
即Test的结构和objc_object相同，而objc_object本身只有一个```isa```指针

接着定义实例变量结构体,如下，可以看到继承自NSObject的成员变量排列在最前方（```NSObject_IVARS```），```Test```类中的成员变量按声明顺序排列

```
struct Test_IMPL {
     struct NSObject_IMPL NSObject_IVARS; 
     NSString *_str;
     NSObject *_obj;
};
```

之后将所有方法声明成静态变量，其实可以看做类似于C++对象的成员函数，

```
static void _I_Test_doSomething(Test * self, SEL _cmd) {}

static NSString * _I_Test_str(Test * self, SEL _cmd) { return (*(NSString **)((char *)self + OBJC_IVAR_$_Test$_str)); }
static void _I_Test_setStr_(Test * self, SEL _cmd, NSString *str) { (*(NSString **)((char *)self + OBJC_IVAR_$_Test$_str)) = str; }

static NSObject * _I_Test_obj(Test * self, SEL _cmd) { return (*(NSObject **)((char *)self + OBJC_IVAR_$_Test$_obj)); }
static void _I_Test_setObj_(Test * self, SEL _cmd, NSObject *obj) { (*(NSObject **)((char *)self + OBJC_IVAR_$_Test$_obj)) = obj; }
```

接着定义了实例变量列表结构，其中OBJC_IVAR_$_Test$_str 是实例变量_str所在偏移
```
extern "C" unsigned long int OBJC_IVAR_$_Test$_str __attribute__ ((used, section ("__DATA,__objc_ivar"))) = __OFFSETOFIVAR__(struct Test, _str);```


以下_OBJC_$_INSTANCE_VARIABLES_Test的内存布局和_ivar_list_t结构完全一致，记录了Test类中实例变量偏移位置。

```
static struct /*_ivar_list_t*/ 
     unsigned int entsize;  // sizeof(struct _prop_t)
     unsigned int count;
     struct _ivar_t ivar_list[2];
} _OBJC_$_INSTANCE_VARIABLES_Test __attribute__ ((used, section ("__DATA,__objc_const"))) = {
     sizeof(_ivar_t),
     2,
     {{(unsigned long int *)&OBJC_IVAR_$_Test$_str, "_str", "@\"NSString\"", 3, 8},
      {(unsigned long int *)&OBJC_IVAR_$_Test$_obj, "_obj", "@\"NSObject\"", 3, 8}}
};
```

以上即为实例变量布局记录在runtime的过程。

接下来看，可以发现编译后的代码还生成了方法结构体，和属性列表

```
//定义方法结构体
static struct /*_method_list_t*/ {
     unsigned int entsize;  // sizeof(struct _objc_method)
     unsigned int method_count;
     struct _objc_method method_list[5];
} _OBJC_$_INSTANCE_METHODS_Test __attribute__ ((used, section ("__DATA,__objc_const"))) = {
     sizeof(_objc_method),
     5,
     {{(struct objc_selector *)"doSomething", "v16@0:8", (void *)_I_Test_doSomething},
     {(struct objc_selector *)"str", "@16@0:8", (void *)_I_Test_str},
     {(struct objc_selector *)"setStr:", "v24@0:8@16", (void *)_I_Test_setStr_},
     {(struct objc_selector *)"obj", "@16@0:8", (void *)_I_Test_obj},
     {(struct objc_selector *)"setObj:", "v24@0:8@16", (void *)_I_Test_setObj_}}
};
```

```
//定义属性列表
static struct /*_prop_list_t*/ {
     unsigned int entsize;  // sizeof(struct _prop_t)
     unsigned int count_of_properties;
     struct _prop_t prop_list[2];
} _OBJC_$_PROP_LIST_Test __attribute__ ((used, section ("__DATA,__objc_const"))) = {
     sizeof(_prop_t),
     2,
     {{"str","T@\"NSString\",V_str"},
     {"obj","T@\"NSObject\",V_obj"}}
};
```

接下来cpp中定义类一个将方法列表，变量列表，属性列表整合起来的一个类,_OBJC_CLASS_RO_$_Test 但这个类的布局看起来和objc_class并不一致

```
static struct _class_ro_t _OBJC_CLASS_RO_$_Test __attribute__ ((used, section ("__DATA,__objc_const"))) = {
     0, __OFFSETOFIVAR__(struct Test, _str), sizeof(struct Test_IMPL),
     (unsigned int)0,
     0,
     "Test",
     (const struct _method_list_t *)&_OBJC_$_INSTANCE_METHODS_Test,
     0,
     (const struct _ivar_list_t *)&_OBJC_$_INSTANCE_VARIABLES_Test,
     0,
     (const struct _prop_list_t *)&_OBJC_$_PROP_LIST_Test,
};
```

之后便出现了真正的类定义，将上述_OBJC_CLASS_RO_$_Test赋值给OBJC_CLASS_$_Test，元类同理,当一个类定义文件没被引入工程时，常常能看到诸如
"_OBJC_CLASS_$_XXClass, referenced from:xxx "这类的错误提示，正是因为该类在符号表中的名字实际是_OBJC_CLASS_$_XXClass（下面的代码中可以看到通过dllexport在链接时导出了），而不是我们在.m文件中定义的名字。


```
extern "C" __declspec(dllexport) struct _class_t OBJC_CLASS_$_Test __attribute__ ((used, section ("__DATA,__objc_data"))) = {
     0, // &OBJC_METACLASS_$_Test,
     0, // &OBJC_CLASS_$_NSObject,
     0, // (void *)&_objc_empty_cache,
     0, // unused, was (void *)&_objc_empty_vtable,
     &_OBJC_CLASS_RO_$_Test,
};
```


再仔细观察，OBJC_CLASS_$_Test是struct _class_t结构，而struct _class_t结构如下,
和objc_class对比可以发现，

```
struct _class_t {
	struct _class_t *isa;
	struct _class_t *superclass;
	void *cache;
	void *vtable;
	struct _class_ro_t *ro;
};
//compare with

struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags
    
    class_rw_t *data() { 
        return bits.data();
    }
    void setData(class_rw_t *newData) {
        bits.setData(newData);
    }
    //method list 
}
```

- isa是一个指向_class_t结构的指针，而Class 也是一个指向objc_class的指针
-  superclass也同为指针
-  cache和vtable分别是两个指针大小，而cache_t结构为```struct cache_t {
    struct bucket_t *_buckets;
    mask_t _mask;
    mask_t _occupied;
``` 即cache可以对应buckets的指针，而mask_t结构为指针的一半，两个mask_t正好对应了vtable的指针位置
- 最后ro 就对应了 class_data_bits_t结构中的指针

对应Test类，ro指针正是集合了类中所有成员变量和方法的_OBJC_CLASS_RO_$_Test结构。








