---
layout: post
title: QEMU Module Infrastructure
categories: [QEMU]
tags: [QEMU,源码分析]
---

本文主要分析QEMU的Module框架机制

# QEMU Module Infrastructure
相关的Maintainer是IBM的Anthony Liguori <aliguori@us.ibm.com>。

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
{% endhighlight %}

这个module_init也是C语言的宏:
{% highlight c linenos %}
include/qemu/module.h

#define module_init(function, type)                                         \
static void __attribute__((constructor)) do_qemu_init_ ## function(void)    \
{                                                                           \
    register_module_init(function, type);                                   \
}
{% endhighlight %}

这个是利用了GNU Compiler Collection (GCC)的扩展属性，"GCC 7.3 Manual" ->
"6 Extensions to the C Language Family" -> "6.31 Declaring Attributes of
Functions" -> "6.31.1 Common Function Attributes"

> The **constructor** attribute causes the function to be called automatically
before execution enters main (). Similarly, the destructor attribute causes
the function to be called automatically after main () completes or exit ()
is called. Functions with these attributes are useful for initializing data
that is used implicitly during the execution of the program.

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
