---
layout: post
title: QEMU Develop Manual
categories: [QEMU]
tags: [QEMU,Develop]
---

本文主要分析QEMU的建模API

# Machine API

## Machine Register API

### 查看默认的Machine
{% highlight shell linenos %}
$ qemu-system-ppc -machine help
Supported machines are:
40p                  IBM RS/6000 7020 (40p)
bamboo               bamboo
g3beige              Heathrow based PowerMAC (default)
mac99                Mac99 based PowerMAC
mpc8544ds            mpc8544ds
none                 empty machine
ppce500              generic paravirt e500 platform
prep                 PowerPC PREP platform
ref405ep             ref405ep
taihu                taihu
virtex-ml507         Xilinx Virtex ML507 reference design
{% endhighlight %}

### 在Makefile中添加配置选项
在PowerPC的默认配置选项中添加：
{% highlight make linenos %}
defaults-configs/ppc-softmmu.mak

CONFIG_VMC=y
{% endhighlight %}

在具体编译对象中添加：
{% highlight make linenos %}
hw/ppc/Makefile.objs

# VMC
obj-$(CONFIG_VMC) += vmc.o
{% endhighlight %}

### 添加Machine源码
{% highlight c linenos %}
hw/ppc/vmc.c

static void ppc_vmc_init(MachineState *machine)
{
...
}

static int heathrow_kvm_type(const char *arg)
{
    /* Always force PR KVM */
    return 2;
}

static void vmc_class_init(ObjectClass *oc, void *data)
{
    MachineClass *mc = MACHINE_CLASS(oc);

    mc->desc               = "VMC PowerPC 755";
    mc->init               = ppc_vmc_init;
    mc->block_default_type = IF_PFLASH;
    mc->max_cpus           = 1;
    mc->default_boot_order = "cd";
    mc->kvm_type           = heathrow_kvm_type;
    mc->default_cpu_type   = POWERPC_CPU_TYPE_NAME("755_v2.8");
}

static const TypeInfo ppc_vmc_machine_info = {
    .name          = MACHINE_TYPE_NAME("VMC"),
    .parent        = TYPE_MACHINE,
    .class_init    = vmc_class_init
};

static void ppc_vmc_register_types(void)
{
    type_register_static(&ppc_vmc_machine_info);
}

type_init(ppc_vmc_register_types);
{% endhighlight %}

添加了这些代码，编译后在查看Machine
{% highlight shell linenos %}
$ qemu-system-ppc -machine help
Supported machines are:
40p                  IBM RS/6000 7020 (40p)
VMC                  VMC PowerPC 755
bamboo               bamboo
g3beige              Heathrow based PowerMAC (default)
mac99                Mac99 based PowerMAC
mpc8544ds            mpc8544ds
none                 empty machine
ppce500              generic paravirt e500 platform
prep                 PowerPC PREP platform
ref405ep             ref405ep
taihu                taihu
virtex-ml507         Xilinx Virtex ML507 reference design
{% endhighlight %}

MachineClass结构体定义：
{% highlight c linenos %}
include/hw/boards.h

struct MachineClass {
    /*< private >*/
    ObjectClass parent_class;
    /*< public >*/

    const char *family; /* NULL iff @name identifies a standalone machtype */
    char *name;
    const char *alias;
    const char *desc;

    void (*init)(MachineState *state);
    void (*reset)(void);
    void (*hot_add_cpu)(const int64_t id, Error **errp);
    int (*kvm_type)(const char *arg);

    BlockInterfaceType block_default_type;
    int units_per_default_bus;
    int max_cpus;
    int min_cpus;
    int default_cpus;
    unsigned int no_serial:1,
        no_parallel:1,
        use_virtcon:1,
        use_sclp:1,
        no_floppy:1,
        no_cdrom:1,
        no_sdcard:1,
        has_dynamic_sysbus:1,
        pci_allow_0_address:1,
        legacy_fw_cfg_order:1;
    int is_default;
    const char *default_machine_opts;
    const char *default_boot_order;
    const char *default_display;
    GArray *compat_props;
    const char *hw_version;
    ram_addr_t default_ram_size;
    const char *default_cpu_type;
    bool option_rom_has_mr;
    bool rom_file_has_mr;
    int minimum_page_bits;
    bool has_hotpluggable_cpus;
    bool ignore_memory_transaction_failures;
    int numa_mem_align_shift;
    const char **valid_cpu_types;
    bool auto_enable_numa_with_memhp;
    void (*numa_auto_assign_ram)(MachineClass *mc, NodeInfo *nodes,
                                 int nb_nodes, ram_addr_t size);

    HotplugHandler *(*get_hotplug_handler)(MachineState *machine,
                                           DeviceState *dev);
    CpuInstanceProperties (*cpu_index_to_instance_props)(MachineState *machine,
                                                         unsigned cpu_index);
    const CPUArchIdList *(*possible_cpu_arch_ids)(MachineState *machine);
    int64_t (*get_default_cpu_node_id)(const MachineState *ms, int idx);
};
{% endhighlight %}

block_default_type的类型：
{% highlight c linenos %}
include/sysemu/blockdev.h

typedef enum {
    IF_DEFAULT = -1,            /* for use with drive_add() only */
    /*
     * IF_NONE must be zero, because we want MachineClass member
     * block_default_type to default-initialize to IF_NONE
     */
    IF_NONE = 0,
    IF_IDE, IF_SCSI, IF_FLOPPY, IF_PFLASH, IF_MTD, IF_SD, IF_VIRTIO, IF_XEN,
    IF_COUNT
} BlockInterfaceType;
{% endhighlight %}

### 查看默认的CPU
{% highlight shell linenos %}
$ qemu-system-ppc -cpu help
...
PowerPC 755_v2.8         PVR 00083208
PowerPC 755              (alias for 755_v2.8)
...
PowerPC e500             (alias for e500v2_v22)
...
{% endhighlight %}

# 内存

## Memory
使用Gnome的GLib：
> GLib provides the core application building blocks for libraries and
applications written in C. It provides the core object system used in GNOME,
the main loop implementation, and a large set of utility functions for
strings and common data structures.

## Memory Allocation
[Memory Allocation](https://developer.gnome.org/glib/unstable/glib-Memory
-Allocation.html) — general memory-handling
{% highlight c linenos %}
#include <glib.h>

These functions provide support for allocating and freeing memory.

If any call to allocate memory using functions g_new(), g_new0(), g_renew(),
g_malloc(), g_malloc0(), g_malloc0_n(), g_realloc(), and g_realloc_n() fails,
the application is terminated. This also means that there is no need to check
if the call succeeded. On the other hand, g_try_...() family of functions
returns NULL on failure that can be used as a check for unsuccessful memory
allocation. The application is not terminated in this case.

It's important to match g_malloc() (and wrappers such as g_new()) with
g_free(), g_slice_alloc() (and wrappers such as g_slice_new()) with
g_slice_free(), plain malloc() with free(), and (if you're using C++)
new with delete and new[] with delete[]. Otherwise bad things can happen,
since these allocators may use different memory pools (and new/delete call
constructors and destructors).

functions

#define             g_new(struct_type, n_structs)

Allocates n_structs elements of type struct_type . The returned pointer is
cast to a pointer to the given type. If n_structs is 0 it returns NULL. Care
is taken to avoid overflow when calculating the size of the allocated block.

Since the returned pointer is already casted to the right type, it is
normally unnecessary to cast it explicitly, and doing so might hide
memory allocation errors.

Parameters
    struct_type the type of the elements to allocate
    n_structs   the number of elements to allocate

Returns
    a pointer to the allocated memory, cast to a pointer to struct_type
{% endhighlight %}


## MemoryRegion
{% highlight c linenos %}
include/exec/memory.h

struct MemoryRegion {
    Object parent_obj;

    /* All fields are private - violators will be prosecuted */

    /* The following fields should fit in a cache line */
    bool romd_mode;
    bool ram;
    bool subpage;
    bool readonly; /* For RAM regions */
    bool rom_device;
    bool flush_coalesced_mmio;
    bool global_locking;
    uint8_t dirty_log_mask;
    bool is_iommu;
    RAMBlock *ram_block;
    Object *owner;

    const MemoryRegionOps *ops;
    void *opaque;
    MemoryRegion *container;
    Int128 size;
    hwaddr addr;
    void (*destructor)(MemoryRegion *mr);
    uint64_t align;
    bool terminates;
    bool ram_device;
    bool enabled;
    bool warning_printed; /* For reservations */
    uint8_t vga_logging_count;
    MemoryRegion *alias;
    hwaddr alias_offset;
    int32_t priority;
    QTAILQ_HEAD(subregions, MemoryRegion) subregions;
    QTAILQ_ENTRY(MemoryRegion) subregions_link;
    QTAILQ_HEAD(coalesced_ranges, CoalescedMemoryRange) coalesced;
    const char *name;
    unsigned ioeventfd_nb;
    MemoryRegionIoeventfd *ioeventfds;
};
{% endhighlight %}

## RAMBlock
{% highlight c linenos %}
include/exec/ram_addr.h

struct RAMBlock {
    struct rcu_head rcu;
    struct MemoryRegion *mr;
    uint8_t *host;
    ram_addr_t offset;
    ram_addr_t used_length;
    ram_addr_t max_length;
    void (*resized)(const char*, uint64_t length, void *host);
    uint32_t flags;
    /* Protected by iothread lock.  */
    char idstr[256];
    /* RCU-enabled, writes protected by the ramlist lock */
    QLIST_ENTRY(RAMBlock) next;
    QLIST_HEAD(, RAMBlockNotifier) ramblock_notifiers;
    int fd;
    size_t page_size;
    /* dirty bitmap used during migration */
    unsigned long *bmap;
    /* bitmap of pages that haven't been sent even once
     * only maintained and used in postcopy at the moment
     * where it's used to send the dirtymap at the start
     * of the postcopy phase
     */
    unsigned long *unsentmap;
    /* bitmap of already received pages in postcopy */
    unsigned long *receivedmap;
};
{% endhighlight %}

## get_system_memory()
{% highlight c linenos %}
exec.c

MemoryRegion *get_system_memory(void)
{
    return system_memory;
}
{% endhighlight %}

## memory_region_allocate_system_memory()
头文件
{% highlight c linenos %}
include/hw/boards.h

/**
 * memory_region_allocate_system_memory - Allocate a board's main memory
 * @mr: the #MemoryRegion to be initialized
 * @owner: the object that tracks the region's reference count
 * @name: name of the memory region
 * @ram_size: size of the region in bytes
 *
 * This function allocates the main memory for a board model, and
 * initializes @mr appropriately. It also arranges for the memory
 * to be migrated (by calling vmstate_register_ram_global()).
 *
 * Memory allocated via this function will be backed with the memory
 * backend the user provided using "-mem-path" or "-numa node,memdev=..."
 * if appropriate; this is typically used to cause host huge pages to be
 * used. This function should therefore be called by a board exactly once,
 * for the primary or largest RAM area it implements.
 *
 * For boards where the major RAM is split into two parts in the memory
 * map, you can deal with this by calling memory_region_allocate_system_memory()
 * once to get a MemoryRegion with enough RAM for both parts, and then
 * creating alias MemoryRegions via memory_region_init_alias() which
 * alias into different parts of the RAM MemoryRegion and can be mapped
 * into the memory map in the appropriate places.
 *
 * Smaller pieces of memory (display RAM, static RAMs, etc) don't need
 * to be backed via the -mem-path memory backend and can simply
 * be created via memory_region_allocate_aux_memory() or
 * memory_region_init_ram().
 */
void memory_region_allocate_system_memory(MemoryRegion *mr, Object *owner,
                                          const char *name,
                                          uint64_t ram_size);
{% endhighlight %}

函数体
{% highlight c linenos %}
numa.c

void memory_region_allocate_system_memory(MemoryRegion *mr, Object *owner,
                                          const char *name,
                                          uint64_t ram_size)
{
    uint64_t addr = 0;
    int i;

    if (nb_numa_nodes == 0 || !have_memdevs) {
        allocate_system_memory_nonnuma(mr, owner, name, ram_size);
        return;
    }

    memory_region_init(mr, owner, name, ram_size);
    for (i = 0; i < nb_numa_nodes; i++) {
        uint64_t size = numa_info[i].node_mem;
        HostMemoryBackend *backend = numa_info[i].node_memdev;
        if (!backend) {
            continue;
        }
        MemoryRegion *seg = host_memory_backend_get_memory(backend,
                                                           &error_fatal);

        if (memory_region_is_mapped(seg)) {
            char *path = object_get_canonical_path_component(OBJECT(backend));
            error_report("memory backend %s is used multiple times. Each "
                         "-numa option must use a different memdev value.",
                         path);
            exit(1);
        }

        host_memory_backend_set_mapped(backend, true);
        memory_region_add_subregion(mr, addr, seg);
        vmstate_register_ram_global(seg);
        addr += size;
    }
}
{% endhighlight %}

{% highlight c linenos %}
{% endhighlight %}
