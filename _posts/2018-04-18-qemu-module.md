---
layout: post
title: QEMU Module Infrastructure
categories: [QEMU]
tags: [QEMU,源码分析]
---

本文主要分析QEMU的Module框架机制

# QEMU Module Infrastructure
相关的Maintainer是IBM的Anthony Liguori (aliguori@us.ibm.com)。

## Module初始化

### 源码
定义了4种Module: BLOCK,OPTS,QOM,TRACE，其定义如下:
{% highlight c linenos %}
include/qemu/module.h

typedef enum {
    MODULE_INIT_BLOCK,
    MODULE_INIT_OPTS,
    MODULE_INIT_QOM,
    MODULE_INIT_TRACE,
    MODULE_INIT_MAX
} module_init_type;
{% endhighlight %}

针对每种Module的初始化接口，都是C语言的宏:
{% highlight c linenos %}
include/qemu/module.h

#define block_init(function) module_init(function, MODULE_INIT_BLOCK)
#define opts_init(function) module_init(function, MODULE_INIT_OPTS)
#define type_init(function) module_init(function, MODULE_INIT_QOM)
#define trace_init(function) module_init(function, MODULE_INIT_TRACE)

#define module_init(function, type)                                         \
static void __attribute__((constructor)) do_qemu_init_ ## function(void)    \
{                                                                           \
    register_module_init(function, type);                                   \
}
{% endhighlight %}

### type_init()源码分析
对应代码，对于每个type_init()都会生成一个对应的构造函数：
![函数调用流](/pictures/qemu-module-01.svg)

### 构造函数
这个是利用了GNU Compiler Collection (GCC)的扩展属性，"GCC 7.3 Manual" ->
"6 Extensions to the C Language Family" -> "6.31 Declaring Attributes of
Functions" -> "6.31.1 Common Function Attributes"

> The **constructor** attribute causes the function to be called automatically
before execution enters main (). Similarly, the destructor attribute causes
the function to be called automatically after main () completes or exit ()
is called. Functions with these attributes are useful for initializing data
that is used implicitly during the execution of the program.

### 源码
其register_module_init()函数定义如下:
{% highlight c linenos %}
util/module.c

typedef struct ModuleEntry
{
    void (*init)(void);
    QTAILQ_ENTRY(ModuleEntry) node;
    module_init_type type;
} ModuleEntry;

typedef QTAILQ_HEAD(, ModuleEntry) ModuleTypeList;

static ModuleTypeList init_type_list[MODULE_INIT_MAX];

static ModuleTypeList dso_init_list;

static void init_lists(void)
{
    static int inited;
    int i;

    if (inited) {
        return;
    }

    for (i = 0; i < MODULE_INIT_MAX; i++) {
        QTAILQ_INIT(&init_type_list[i]);
    }

    QTAILQ_INIT(&dso_init_list);

    inited = 1;
}


static ModuleTypeList *find_type(module_init_type type)
{
    init_lists();

    return &init_type_list[type];
}

void register_module_init(void (*fn)(void), module_init_type type)
{
    ModuleEntry *e;
    ModuleTypeList *l;

    e = g_malloc0(sizeof(*e));
    e->init = fn;
    e->type = type;

    l = find_type(type);

    QTAILQ_INSERT_TAIL(l, e, node);
}
{% endhighlight %}

### do_qemu_init_xxx()代码分析
每个构造函数都由OS在main之前调用： ![函数调用流](/pictures/qemu-module-02.svg)

每个构造函数生成一个节点，放入func和type，然后把节点插入到type对应的队列中。
数据结构为： ![数据结构](/pictures/qemu-module-03.svg)

## Module注册

### 源码
苹果的G3机器注册：
{% highlight c linenos %}
hw/ppc/mac_oldworld.c

static const TypeInfo ppc_heathrow_machine_info = {
    .name          = MACHINE_TYPE_NAME("g3beige"),
    .parent        = TYPE_MACHINE,
    .class_init    = heathrow_class_init
};

static void ppc_heathrow_register_types(void)
{
    type_register_static(&ppc_heathrow_machine_info);
}

type_init(ppc_heathrow_register_types);
{% endhighlight %}

苹果的G5机器注册：
{% highlight c linenos %}
static const TypeInfo core99_machine_info = {
    .name          = MACHINE_TYPE_NAME("mac99"),
    .parent        = TYPE_MACHINE,
    .class_init    = core99_machine_class_init,
};

static void mac_machine_register_types(void)
{
    type_register_static(&core99_machine_info);
}

type_init(mac_machine_register_types)
{% endhighlight %}

PowerPC的硬件参考平台注册：
{% highlight c linenos %}
hw/ppc/prep.c

DEFINE_MACHINE("prep", prep_machine_init)
{% endhighlight %}

其中的DEFINE_MACHINE()宏：
{% highlight c linenos %}
include/hw/boards.h

#define DEFINE_MACHINE(namestr, machine_initfn) \
    static void machine_initfn##_class_init(ObjectClass *oc, void *data) \
    { \
        MachineClass *mc = MACHINE_CLASS(oc); \
        machine_initfn(mc); \
    } \
    static const TypeInfo machine_initfn##_typeinfo = { \
        .name       = MACHINE_TYPE_NAME(namestr), \
        .parent     = TYPE_MACHINE, \
        .class_init = machine_initfn##_class_init, \
    }; \
    static void machine_initfn##_register_types(void) \
    { \
        type_register_static(&machine_initfn##_typeinfo); \
    } \
    type_init(machine_initfn##_register_types)
{% endhighlight %}

而其中的内部注册函数：
{% highlight c linenos %}
qom/object.c

static TypeImpl *type_new(const TypeInfo *info)
{
    TypeImpl *ti = g_malloc0(sizeof(*ti));
    int i;

    g_assert(info->name != NULL);

    if (type_table_lookup(info->name) != NULL) {
        fprintf(stderr, "Registering `%s' which already exists\n", info->name);
        abort();
    }

    ti->name = g_strdup(info->name);
    ti->parent = g_strdup(info->parent);

    ti->class_size = info->class_size;
    ti->instance_size = info->instance_size;

    ti->class_init = info->class_init;
    ti->class_base_init = info->class_base_init;
    ti->class_finalize = info->class_finalize;
    ti->class_data = info->class_data;

    ti->instance_init = info->instance_init;
    ti->instance_post_init = info->instance_post_init;
    ti->instance_finalize = info->instance_finalize;

    ti->abstract = info->abstract;

    for (i = 0; info->interfaces && info->interfaces[i].type; i++) {
        ti->interfaces[i].typename = g_strdup(info->interfaces[i].type);
    }
    ti->num_interfaces = i;

    return ti;
}

static TypeImpl *type_register_internal(const TypeInfo *info)
{
    TypeImpl *ti;
    ti = type_new(info);

    type_table_add(ti);
    return ti;
}

TypeImpl *type_register(const TypeInfo *info)
{
    assert(info->parent);
    return type_register_internal(info);
}

TypeImpl *type_register_static(const TypeInfo *info)
{
    return type_register(info);
}
{% endhighlight %}

module_call_init()函数
{% highlight c linenos %}
util/module.c

void module_call_init(module_init_type type)
{
    ModuleTypeList *l;
    ModuleEntry *e;

    l = find_type(type);

    QTAILQ_FOREACH(e, l, node) {
        e->init();
    }
}
{% endhighlight %}

TypeInfo数据结构
{% highlight c linenos %}
include/qom/object.h

struct TypeInfo
{
    const char *name;
    const char *parent;

    size_t instance_size;
    void (*instance_init)(Object *obj);
    void (*instance_post_init)(Object *obj);
    void (*instance_finalize)(Object *obj);

    bool abstract;
    size_t class_size;

    void (*class_init)(ObjectClass *klass, void *data);
    void (*class_base_init)(ObjectClass *klass, void *data);
    void (*class_finalize)(ObjectClass *klass, void *data);
    void *class_data;

    InterfaceInfo *interfaces;
};
{% endhighlight %}

TypeImpl数据结构
{% highlight c linenos %}
qom/object.c

struct TypeImpl
{
    const char *name;

    size_t class_size;

    size_t instance_size;

    void (*class_init)(ObjectClass *klass, void *data);
    void (*class_base_init)(ObjectClass *klass, void *data);
    void (*class_finalize)(ObjectClass *klass, void *data);

    void *class_data;

    void (*instance_init)(Object *obj);
    void (*instance_post_init)(Object *obj);
    void (*instance_finalize)(Object *obj);

    bool abstract;

    const char *parent;
    TypeImpl *parent_type;

    ObjectClass *class;

    int num_interfaces;
    InterfaceImpl interfaces[MAX_INTERFACES];
};
{% endhighlight %}

### type_register_static()源码分析
从前面的分析，可以得知：
* 每个type_init()都会生成一个对应的构造函数
* 每个构造函数都由OS在main之前调用
* 每个构造函数生成一个节点，放入func和type，然后把节点插入到type对应的队列中

而这每个节点中func(即type_init()的参数)是怎么调用的呢？

原来在main()中，会先后初始化TRACE/QOM/OPTS：
{% highlight c linenos %}
vl.c

module_call_init(MODULE_INIT_TRACE);

module_call_init(MODULE_INIT_QOM);

module_call_init(MODULE_INIT_OPTS);
{% endhighlight %}

以这个QOM初始化为例，module_call_init()遍历QOM队列中的每个节点，并执行节点中的func函数。
而每个Type注册函数调用type_register_static()来完成TypeInfo到TypeImpl的转换：
 ![函数调用流](/pictures/qemu-module-04.svg)

而已经有了TypeInfo，为什么还要用TypeImpl，可能是为了速度(各个TypeInfo单独存在，
而TypeImpl是采用的Hash组织起来的)：
![数据结构](/pictures/qemu-module-05.svg)

### TypeImpl的操作
{% highlight c linenos %}
qom/object.c

static GHashTable *type_table_get(void)
{
    static GHashTable *type_table;

    if (type_table == NULL) {
        type_table = g_hash_table_new(g_str_hash, g_str_equal);
    }

    return type_table;
}

static bool enumerating_types;

static void type_table_add(TypeImpl *ti)
{
    assert(!enumerating_types);
    g_hash_table_insert(type_table_get(), (void *)ti->name, ti);
}

static TypeImpl *type_table_lookup(const char *name)
{
    return g_hash_table_lookup(type_table_get(), name);
}
{% endhighlight %}

即TypeImpl采用的关键字为对象名字与对象之间建立Hash关系。
