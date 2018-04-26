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
    mc->max_cpus           = MAX_CPUS;
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
PowerPC 601_v1           PVR 00010001
PowerPC 601_v0           PVR 00010001
PowerPC 601_v2           PVR 00010002
PowerPC 601              (alias for 601_v2)
PowerPC 601v             (alias for 601_v2)
PowerPC 603              PVR 00030100
PowerPC mpc8240          (alias for 603)
PowerPC vanilla          (alias for 603)
PowerPC 604              PVR 00040103
PowerPC ppc32            (alias for 604)
PowerPC ppc              (alias for 604)
PowerPC default          (alias for 604)
PowerPC 602              PVR 00050100
PowerPC 603e_v1.1        PVR 00060101
PowerPC 603e_v1.2        PVR 00060102
PowerPC 603e_v1.3        PVR 00060103
PowerPC 603e_v1.4        PVR 00060104
PowerPC 603e_v2.2        PVR 00060202
PowerPC 603e_v3          PVR 00060300
PowerPC 603e_v4          PVR 00060400
PowerPC 603e_v4.1        PVR 00060401
PowerPC 603e             (alias for 603e_v4.1)
PowerPC stretch          (alias for 603e_v4.1)
PowerPC 603p             PVR 00070000
PowerPC 603e7v           PVR 00070100
PowerPC vaillant         (alias for 603e7v)
PowerPC 603e7v1          PVR 00070101
PowerPC 603e7            PVR 00070200
PowerPC 603e7v2          PVR 00070201
PowerPC 603e7t           PVR 00071201
PowerPC 603r             (alias for 603e7t)
PowerPC goldeneye        (alias for 603e7t)
PowerPC 750_v1.0         PVR 00080100
PowerPC 740_v1.0         PVR 00080100
PowerPC 740e             PVR 00080100
PowerPC 750e             PVR 00080200
PowerPC 750_v2.0         PVR 00080200
PowerPC 740_v2.0         PVR 00080200
PowerPC 750_v2.1         PVR 00080201
PowerPC 740_v2.1         PVR 00080201
PowerPC 740_v2.2         PVR 00080202
PowerPC 750_v2.2         PVR 00080202
PowerPC 750_v3.0         PVR 00080300
PowerPC 740_v3.0         PVR 00080300
PowerPC 750_v3.1         PVR 00080301
PowerPC 750              (alias for 750_v3.1)
PowerPC typhoon          (alias for 750_v3.1)
PowerPC g3               (alias for 750_v3.1)
PowerPC 740_v3.1         PVR 00080301
PowerPC 740              (alias for 740_v3.1)
PowerPC arthur           (alias for 740_v3.1)
PowerPC 750cx_v1.0       PVR 00082100
PowerPC 750cx_v2.0       PVR 00082200
PowerPC 750cx_v2.1       PVR 00082201
PowerPC 750cx_v2.2       PVR 00082202
PowerPC 750cx            (alias for 750cx_v2.2)
PowerPC 750cxe_v2.1      PVR 00082211
PowerPC 750cxe_v2.2      PVR 00082212
PowerPC 750cxe_v2.3      PVR 00082213
PowerPC 750cxe_v2.4      PVR 00082214
PowerPC 750cxe_v3.0      PVR 00082310
PowerPC 750cxe_v3.1      PVR 00082311
PowerPC 755_v1.0         PVR 00083100
PowerPC 745_v1.0         PVR 00083100
PowerPC 755_v1.1         PVR 00083101
PowerPC 745_v1.1         PVR 00083101
PowerPC 755_v2.0         PVR 00083200
PowerPC 745_v2.0         PVR 00083200
PowerPC 755_v2.1         PVR 00083201
PowerPC 745_v2.1         PVR 00083201
PowerPC 745_v2.2         PVR 00083202
PowerPC 755_v2.2         PVR 00083202
PowerPC 755_v2.3         PVR 00083203
PowerPC 745_v2.3         PVR 00083203
PowerPC 755_v2.4         PVR 00083204
PowerPC 745_v2.4         PVR 00083204
PowerPC 745_v2.5         PVR 00083205
PowerPC 755_v2.5         PVR 00083205
PowerPC 755_v2.6         PVR 00083206
PowerPC 745_v2.6         PVR 00083206
PowerPC 755_v2.7         PVR 00083207
PowerPC 745_v2.7         PVR 00083207
PowerPC 745_v2.8         PVR 00083208
PowerPC 745              (alias for 745_v2.8)
PowerPC 755_v2.8         PVR 00083208
PowerPC 755              (alias for 755_v2.8)
PowerPC goldfinger       (alias for 755_v2.8)
PowerPC 750cxe_v2.4b     PVR 00083214
PowerPC 750cxe_v3.1b     PVR 00083311
PowerPC 750cxe           (alias for 750cxe_v3.1b)
PowerPC 750cxr           PVR 00083410
PowerPC 750cl_v1.0       PVR 00087200
PowerPC 750cl_v2.0       PVR 00087210
PowerPC 750cl            (alias for 750cl_v2.0)
PowerPC 750l_v2.0        PVR 00088200
PowerPC 750l_v2.1        PVR 00088201
PowerPC 750l_v2.2        PVR 00088202
PowerPC 750l_v3.0        PVR 00088300
PowerPC 750l_v3.2        PVR 00088302
PowerPC 750l             (alias for 750l_v3.2)
PowerPC lonestar         (alias for 750l_v3.2)
PowerPC 604e_v1.0        PVR 00090100
PowerPC 604e_v2.2        PVR 00090202
PowerPC 604e_v2.4        PVR 00090204
PowerPC 604e             (alias for 604e_v2.4)
PowerPC sirocco          (alias for 604e_v2.4)
PowerPC 604r             PVR 000a0101
PowerPC mach5            (alias for 604r)
PowerPC 7400_v1.0        PVR 000c0100
PowerPC 7400_v1.1        PVR 000c0101
PowerPC 7400_v2.0        PVR 000c0200
PowerPC 7400_v2.1        PVR 000c0201
PowerPC 7400_v2.2        PVR 000c0202
PowerPC 7400_v2.6        PVR 000c0206
PowerPC 7400_v2.7        PVR 000c0207
PowerPC 7400_v2.8        PVR 000c0208
PowerPC 7400_v2.9        PVR 000c0209
PowerPC 7400             (alias for 7400_v2.9)
PowerPC max              (alias for 7400_v2.9)
PowerPC g4               (alias for 7400_v2.9)
PowerPC 403ga            PVR 00200011
PowerPC 403gb            PVR 00200100
PowerPC 403gc            PVR 00200200
PowerPC 403              (alias for 403gc)
PowerPC 403gcx           PVR 00201400
PowerPC 401a1            PVR 00210000
PowerPC 401b2            PVR 00220000
PowerPC iop480           PVR 00220000
PowerPC 401c2            PVR 00230000
PowerPC 401d2            PVR 00240000
PowerPC 401e2            PVR 00250000
PowerPC 401f2            PVR 00260000
PowerPC 401g2            PVR 00270000
PowerPC 401              PVR 00270000
PowerPC g2               PVR 00810011
PowerPC mpc603           PVR 00810100
PowerPC g2hip3           PVR 00810101
PowerPC mpc8250_hip3     (alias for g2hip3)
PowerPC mpc8255_hip3     (alias for g2hip3)
PowerPC mpc8260_hip3     (alias for g2hip3)
PowerPC mpc8264_hip3     (alias for g2hip3)
PowerPC mpc8265_hip3     (alias for g2hip3)
PowerPC mpc8266_hip3     (alias for g2hip3)
PowerPC mpc8349a         PVR 00830010
PowerPC mpc8347ap        PVR 00830010
PowerPC mpc8347eap       PVR 00830010
PowerPC mpc8347p         PVR 00830010
PowerPC mpc8349          PVR 00830010
PowerPC mpc8343e         PVR 00830010
PowerPC mpc8343ea        PVR 00830010
PowerPC mpc8343          PVR 00830010
PowerPC mpc8347et        PVR 00830010
PowerPC mpc8347e         (alias for mpc8347et)
PowerPC mpc8349e         PVR 00830010
PowerPC mpc8347at        PVR 00830010
PowerPC mpc8347a         (alias for mpc8347at)
PowerPC mpc8349ea        PVR 00830010
PowerPC mpc8347eat       PVR 00830010
PowerPC mpc8347ea        (alias for mpc8347eat)
PowerPC mpc8343a         PVR 00830010
PowerPC mpc8347t         PVR 00830010
PowerPC mpc8347          (alias for mpc8347t)
PowerPC mpc8347ep        PVR 00830010
PowerPC e300c1           PVR 00830010
PowerPC e300c2           PVR 00840010
PowerPC e300c3           PVR 00850010
PowerPC e300             (alias for e300c3)
PowerPC mpc8379          PVR 00860010
PowerPC mpc8378e         PVR 00860010
PowerPC mpc8379e         PVR 00860010
PowerPC mpc8378          PVR 00860010
PowerPC mpc8377          PVR 00860010
PowerPC e300c4           PVR 00860010
PowerPC mpc8377e         PVR 00860010
PowerPC 750p             PVR 10080000
PowerPC conan/doyle      (alias for 750p)
PowerPC 740p             PVR 10080000
PowerPC cobra            PVR 10100000
PowerPC 460exb           PVR 130218a4
PowerPC 460ex            (alias for 460exb)
PowerPC 440epx           PVR 200008d0
PowerPC 405d2            PVR 20010000
PowerPC x2vp4            PVR 20010820
PowerPC x2vp7            (alias for x2vp4)
PowerPC x2vp20           PVR 20010860
PowerPC x2vp50           (alias for x2vp20)
PowerPC 405gpa           PVR 40110000
PowerPC 405gpb           PVR 40110040
PowerPC 405cra           PVR 40110041
PowerPC 405gpc           PVR 40110082
PowerPC 405gpd           PVR 401100c4
PowerPC 405gp            (alias for 405gpd)
PowerPC 405crb           PVR 401100c5
PowerPC 405crc           PVR 40110145
PowerPC 405cr            (alias for 405crc)
PowerPC 405gpe           (alias for 405crc)
PowerPC stb03            PVR 40310000
PowerPC npe4gs3          PVR 40b10000
PowerPC npe405h          PVR 414100c0
PowerPC npe405h2         PVR 41410140
PowerPC 405ez            PVR 41511460
PowerPC npe405l          PVR 416100c0
PowerPC 405d4            PVR 41810000
PowerPC 405              (alias for 405d4)
PowerPC stb04            PVR 41810000
PowerPC 405lp            PVR 41f10000
PowerPC 440epa           PVR 42221850
PowerPC 440epb           PVR 422218d3
PowerPC 440ep            (alias for 440epb)
PowerPC 405gpr           PVR 50910951
PowerPC 405ep            PVR 51210950
PowerPC stb25            PVR 51510950
PowerPC 750fx_v1.0       PVR 70000100
PowerPC 750fx_v2.0       PVR 70000200
PowerPC 750fx_v2.1       PVR 70000201
PowerPC 750fx_v2.2       PVR 70000202
PowerPC 750fl            PVR 70000203
PowerPC 750fx_v2.3       PVR 70000203
PowerPC 750fx            (alias for 750fx_v2.3)
PowerPC 750gx_v1.0       PVR 70020100
PowerPC 750gx_v1.1       PVR 70020101
PowerPC 750gx_v1.2       PVR 70020102
PowerPC 750gx            (alias for 750gx_v1.2)
PowerPC 750gl            PVR 70020102
PowerPC 440-xilinx       PVR 7ff21910
PowerPC 440-xilinx-w-dfpu PVR 7ff21910
PowerPC 7450_v1.0        PVR 80000100
PowerPC 7450_v1.1        PVR 80000101
PowerPC 7450_v1.2        PVR 80000102
PowerPC 7450_v2.0        PVR 80000200
PowerPC 7450_v2.1        PVR 80000201
PowerPC 7450             (alias for 7450_v2.1)
PowerPC vger             (alias for 7450_v2.1)
PowerPC 7441_v2.1        PVR 80000201
PowerPC 7441_v2.3        PVR 80000203
PowerPC 7441             (alias for 7441_v2.3)
PowerPC 7451_v2.3        PVR 80000203
PowerPC 7451             (alias for 7451_v2.3)
PowerPC 7451_v2.10       PVR 80000210
PowerPC 7441_v2.10       PVR 80000210
PowerPC 7455_v1.0        PVR 80010100
PowerPC 7445_v1.0        PVR 80010100
PowerPC 7445_v2.1        PVR 80010201
PowerPC 7455_v2.1        PVR 80010201
PowerPC 7445_v3.2        PVR 80010302
PowerPC 7445             (alias for 7445_v3.2)
PowerPC 7455_v3.2        PVR 80010302
PowerPC 7455             (alias for 7455_v3.2)
PowerPC apollo6          (alias for 7455_v3.2)
PowerPC 7455_v3.3        PVR 80010303
PowerPC 7445_v3.3        PVR 80010303
PowerPC 7455_v3.4        PVR 80010304
PowerPC 7445_v3.4        PVR 80010304
PowerPC 7447_v1.0        PVR 80020100
PowerPC 7457_v1.0        PVR 80020100
PowerPC 7457_v1.1        PVR 80020101
PowerPC 7447_v1.1        PVR 80020101
PowerPC 7447             (alias for 7447_v1.1)
PowerPC 7457_v1.2        PVR 80020102
PowerPC 7457             (alias for 7457_v1.2)
PowerPC apollo7          (alias for 7457_v1.2)
PowerPC 7457a_v1.0       PVR 80030100
PowerPC apollo7pm        (alias for 7457a_v1.0)
PowerPC 7447a_v1.0       PVR 80030100
PowerPC 7447a_v1.1       PVR 80030101
PowerPC 7457a_v1.1       PVR 80030101
PowerPC 7447a_v1.2       PVR 80030102
PowerPC 7447a            (alias for 7447a_v1.2)
PowerPC 7457a_v1.2       PVR 80030102
PowerPC 7457a            (alias for 7457a_v1.2)
PowerPC e600             PVR 80040010
PowerPC mpc8610          PVR 80040010
PowerPC mpc8641d         PVR 80040010
PowerPC mpc8641          PVR 80040010
PowerPC 7448_v1.0        PVR 80040100
PowerPC 7448_v1.1        PVR 80040101
PowerPC 7448_v2.0        PVR 80040200
PowerPC 7448_v2.1        PVR 80040201
PowerPC 7448             (alias for 7448_v2.1)
PowerPC 7410_v1.0        PVR 800c1100
PowerPC 7410_v1.1        PVR 800c1101
PowerPC 7410_v1.2        PVR 800c1102
PowerPC 7410_v1.3        PVR 800c1103
PowerPC 7410_v1.4        PVR 800c1104
PowerPC 7410             (alias for 7410_v1.4)
PowerPC nitro            (alias for 7410_v1.4)
PowerPC e500_v10         PVR 80200010
PowerPC mpc8540_v10      PVR 80200010
PowerPC mpc8540_v21      PVR 80200020
PowerPC mpc8540          (alias for mpc8540_v21)
PowerPC e500_v20         PVR 80200020
PowerPC e500v1           (alias for e500_v20)
PowerPC mpc8541_v10      PVR 80200020
PowerPC mpc8541e_v11     PVR 80200020
PowerPC mpc8541e         (alias for mpc8541e_v11)
PowerPC mpc8540_v20      PVR 80200020
PowerPC mpc8541e_v10     PVR 80200020
PowerPC mpc8541_v11      PVR 80200020
PowerPC mpc8541          (alias for mpc8541_v11)
PowerPC mpc8555_v10      PVR 80210010
PowerPC mpc8548_v10      PVR 80210010
PowerPC mpc8543_v10      PVR 80210010
PowerPC mpc8543e_v10     PVR 80210010
PowerPC mpc8548e_v10     PVR 80210010
PowerPC mpc8555e_v10     PVR 80210010
PowerPC e500v2_v10       PVR 80210010
PowerPC mpc8560_v10      PVR 80210010
PowerPC mpc8543e_v11     PVR 80210011
PowerPC mpc8548e_v11     PVR 80210011
PowerPC mpc8555e_v11     PVR 80210011
PowerPC mpc8555e         (alias for mpc8555e_v11)
PowerPC mpc8555_v11      PVR 80210011
PowerPC mpc8555          (alias for mpc8555_v11)
PowerPC mpc8548_v11      PVR 80210011
PowerPC mpc8543_v11      PVR 80210011
PowerPC mpc8547e_v20     PVR 80210020
PowerPC e500v2_v20       PVR 80210020
PowerPC mpc8560_v20      PVR 80210020
PowerPC mpc8545e_v20     PVR 80210020
PowerPC mpc8545_v20      PVR 80210020
PowerPC mpc8548_v20      PVR 80210020
PowerPC mpc8543_v20      PVR 80210020
PowerPC mpc8543e_v20     PVR 80210020
PowerPC mpc8548e_v20     PVR 80210020
PowerPC mpc8545_v21      PVR 80210021
PowerPC mpc8545          (alias for mpc8545_v21)
PowerPC mpc8548_v21      PVR 80210021
PowerPC mpc8548          (alias for mpc8548_v21)
PowerPC mpc8543_v21      PVR 80210021
PowerPC mpc8543          (alias for mpc8543_v21)
PowerPC mpc8544_v10      PVR 80210021
PowerPC mpc8543e_v21     PVR 80210021
PowerPC mpc8543e         (alias for mpc8543e_v21)
PowerPC mpc8544e_v10     PVR 80210021
PowerPC mpc8533_v10      PVR 80210021
PowerPC mpc8548e_v21     PVR 80210021
PowerPC mpc8548e         (alias for mpc8548e_v21)
PowerPC mpc8547e_v21     PVR 80210021
PowerPC mpc8547e         (alias for mpc8547e_v21)
PowerPC mpc8560_v21      PVR 80210021
PowerPC mpc8560          (alias for mpc8560_v21)
PowerPC e500v2_v21       PVR 80210021
PowerPC mpc8533e_v10     PVR 80210021
PowerPC mpc8545e_v21     PVR 80210021
PowerPC mpc8545e         (alias for mpc8545e_v21)
PowerPC mpc8533_v11      PVR 80210022
PowerPC mpc8533          (alias for mpc8533_v11)
PowerPC mpc8567e         PVR 80210022
PowerPC e500v2_v22       PVR 80210022
PowerPC e500             (alias for e500v2_v22)
PowerPC e500v2           (alias for e500v2_v22)
PowerPC mpc8533e_v11     PVR 80210022
PowerPC mpc8533e         (alias for mpc8533e_v11)
PowerPC mpc8568e         PVR 80210022
PowerPC mpc8568          PVR 80210022
PowerPC mpc8567          PVR 80210022
PowerPC mpc8544e_v11     PVR 80210022
PowerPC mpc8544e         (alias for mpc8544e_v11)
PowerPC mpc8544_v11      PVR 80210022
PowerPC mpc8544          (alias for mpc8544_v11)
PowerPC e500v2_v30       PVR 80210030
PowerPC mpc8572e         PVR 80210030
PowerPC mpc8572          PVR 80210030
PowerPC e500mc           PVR 80230020
PowerPC g2h4             PVR 80811010
PowerPC g2hip4           PVR 80811014
PowerPC mpc8241          (alias for g2hip4)
PowerPC mpc8245          (alias for g2hip4)
PowerPC mpc8250          (alias for g2hip4)
PowerPC mpc8250_hip4     (alias for g2hip4)
PowerPC mpc8255          (alias for g2hip4)
PowerPC mpc8255_hip4     (alias for g2hip4)
PowerPC mpc8260          (alias for g2hip4)
PowerPC mpc8260_hip4     (alias for g2hip4)
PowerPC mpc8264          (alias for g2hip4)
PowerPC mpc8264_hip4     (alias for g2hip4)
PowerPC mpc8265          (alias for g2hip4)
PowerPC mpc8265_hip4     (alias for g2hip4)
PowerPC mpc8266          (alias for g2hip4)
PowerPC mpc8266_hip4     (alias for g2hip4)
PowerPC g2le             PVR 80820010
PowerPC g2gp             PVR 80821010
PowerPC g2legp           PVR 80822010
PowerPC mpc5200_v10      PVR 80822011
PowerPC mpc5200_v12      PVR 80822011
PowerPC mpc52xx          (alias for mpc5200_v12)
PowerPC mpc5200          (alias for mpc5200_v12)
PowerPC g2legp1          PVR 80822011
PowerPC mpc5200_v11      PVR 80822011
PowerPC mpc5200b_v21     PVR 80822011
PowerPC mpc5200b         (alias for mpc5200b_v21)
PowerPC mpc5200b_v20     PVR 80822011
PowerPC g2legp3          PVR 80822013
PowerPC mpc82xx          (alias for g2legp3)
PowerPC powerquicc-ii    (alias for g2legp3)
PowerPC mpc8247          (alias for g2legp3)
PowerPC mpc8248          (alias for g2legp3)
PowerPC mpc8270          (alias for g2legp3)
PowerPC mpc8271          (alias for g2legp3)
PowerPC mpc8272          (alias for g2legp3)
PowerPC mpc8275          (alias for g2legp3)
PowerPC mpc8280          (alias for g2legp3)
PowerPC e200z5           PVR 81000000
PowerPC e200z6           PVR 81120000
PowerPC e200             (alias for e200z6)
PowerPC g2ls             PVR 90810010
PowerPC g2lels           PVR a0822010
{% endhighlight %}

{% highlight c linenos %}
{% endhighlight %}
