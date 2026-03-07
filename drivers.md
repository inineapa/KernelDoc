# Device Drivers and Kernel Modules in Linux 6.19

> Source base: `/home/inineapa/Lab/linux-6.19`

---

## Before You Begin

If you have written user-space programs on Linux, you have used device drivers without knowing it. Every `open("/dev/sda", ...)`, every `read()` on a terminal, every `ioctl()` on a GPU — these calls cross the syscall boundary and land in a device driver. A driver is just kernel code that knows how to talk to a specific piece of hardware (or a virtual device). A kernel module is the packaging mechanism that lets you load and unload that code at runtime without rebooting.

This document covers both: how modules work (loading, unloading, parameters, lifecycle), and how drivers are structured (the device model, character devices, PCI devices, platform devices, interrupt handling, and resource management). We trace the path from `insmod` to a working driver with exact source references to Linux 6.19.

---

## 1. Kernel Modules — Loading Code at Runtime

### 1.1 What Is a Module?

A kernel module is an ELF object file (`.ko`) that can be loaded into a running kernel. It extends the kernel without recompilation. Most device drivers, filesystems, and networking protocols are shipped as modules.

The alternative is building a driver directly into the kernel image (`obj-y` in Kbuild), which means it is always present. Modules give you flexibility — load only what the hardware needs.

### 1.2 The Minimal Module

Every module needs exactly two things: an init function and an exit function.

```c
#include <linux/module.h>
#include <linux/init.h>

static int __init hello_init(void)
{
    pr_info("hello: module loaded\n");
    return 0;   // 0 = success, negative errno = failure
}

static void __exit hello_exit(void)
{
    pr_info("hello: module unloaded\n");
}

module_init(hello_init);
module_exit(hello_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("You");
MODULE_DESCRIPTION("A minimal example module");
```

### 1.3 module_init() and module_exit()

These macros (`include/linux/module.h:131–144`) create aliases so the module loader can find your init and exit functions:

- **`module_init(fn)`** — when built as a module, this creates `int init_module(void)` as an alias to `fn`. When built into the kernel, it places `fn` into the initcall table (see `boot.md` Section 7).
- **`module_exit(fn)`** — creates `void cleanup_module(void)` as an alias to `fn`. Only effective when `CONFIG_MODULE_UNLOAD` is enabled.

### 1.4 __init and __exit Annotations

**`__init`** (`include/linux/init.h:45`):
```c
#define __init  __section(".init.text") __cold __latent_entropy
```

This places the function in the `.init.text` section. For built-in drivers, the kernel frees this section after boot (the "Freeing unused kernel memory" message you see during boot). For modules, it is freed after `module_init()` returns successfully.

**`__exit`** (`include/linux/init.h:79`):
```c
#define __exit  __section(".exit.text") __exitused __cold notrace
```

For built-in drivers, the exit function is never called (the driver cannot be removed), so the compiler discards the entire `.exit.text` section. For modules, it is kept and called on `rmmod`.

**`__initdata`** (`init.h:47`) — same idea for data: placed in `.init.data`, freed after init.

### 1.5 Module Metadata Macros

| Macro | Line | Purpose |
|-------|------|---------|
| `MODULE_LICENSE("GPL")` | `module.h:236` | Declares the license. Required — without it, the kernel is "tainted" |
| `MODULE_AUTHOR("name")` | `module.h:242` | Author information |
| `MODULE_DESCRIPTION("text")` | `module.h:245` | What the module does |
| `MODULE_VERSION("1.0")` | — | Version string |
| `MODULE_ALIAS("alias")` | — | Alternative names for autoloading |

The license string must be one of: `"GPL"`, `"GPL v2"`, `"Dual BSD/GPL"`, `"Dual MIT/GPL"`. Proprietary modules taint the kernel, which means kernel developers will refuse to debug any issues you report.

---

## 2. struct module — The Runtime Representation

When a module is loaded, the kernel creates a `struct module` (`include/linux/module.h:403–591`) to track it:

```c
struct module {
    enum module_state state;              // 404 — LIVE, COMING, GOING, UNFORMED
    struct list_head list;                // 407 — linked into global modules list
    char name[MODULE_NAME_LEN];           // 410 — module name (e.g., "e1000e")

    const struct kernel_symbol *syms;     // 425 — exported symbols
    unsigned int num_syms;

    struct kernel_param *kp;              // 438 — module parameters
    unsigned int num_kp;

    int (*init)(void);                    // 459 — init function pointer
    struct module_memory mem[MOD_MEM_NUM_TYPES]; // 461 — code/data memory

    void (*exit)(void);                   // 573 — exit function (CONFIG_MODULE_UNLOAD)
    ...
};
```

### 2.1 Module States

`enum module_state` (`module.h:312–317`):

| State | Value | Meaning |
|-------|-------|---------|
| `MODULE_STATE_LIVE` | 0 | Normal operation — init succeeded |
| `MODULE_STATE_COMING` | 1 | Module is being loaded, init is running |
| `MODULE_STATE_GOING` | 2 | Module is being removed |
| `MODULE_STATE_UNFORMED` | 3 | Still being set up (ELF parsing, relocation) |

You can see the current state of all modules via `/proc/modules`:

```bash
$ cat /proc/modules
e1000e 282624 0 - Live 0xffffffffa0000000
snd_hda_intel 57344 2 - Live 0xffffffffa0100000
```

Columns: name, size, refcount, dependencies, state, address.

---

## 3. Module Loading — From insmod to Running Code

### 3.1 The System Call Path

```
insmod hello.ko
  → open("hello.ko", O_RDONLY)
  → finit_module(fd, "", 0)            # syscall
    → SYSCALL_DEFINE3(finit_module)     [kernel/module/main.c:3734]
      → load_module()                   # the real work
        → 1. Read ELF from fd
        → 2. Parse sections, resolve symbols
        → 3. Allocate memory, apply relocations
        → 4. Call do_init_module()      [main.c:3010]
           → mod->init()               # your __init function
```

There are two syscalls for loading modules:

- **`finit_module(fd, uargs, flags)`** (`main.c:3734–3750`) — takes a file descriptor. This is what `insmod` uses.
- **`init_module(umod, len, uargs)`** (`main.c:3569–3590`) — takes a raw buffer in memory. Older interface.

Both end up in `load_module()`, which:

1. Copies the ELF binary into kernel memory
2. Parses ELF sections (`.text`, `.data`, `.bss`, `.init.text`, `.rodata`, etc.)
3. Resolves symbol references against the kernel symbol table and other loaded modules
4. Applies relocations (patching addresses into the code)
5. Sets up `struct module` with all the metadata
6. Calls `do_init_module()` to run the init function

### 3.2 do_init_module()

`do_init_module()` (`kernel/module/main.c:3010`):

1. Sets module state to `MODULE_STATE_COMING`
2. Calls `mod->init()` — your `__init` function
3. If init returns 0 (success):
   - Sets state to `MODULE_STATE_LIVE`
   - Frees the `.init.text` and `.init.data` sections (no longer needed)
4. If init returns negative errno:
   - Sets state to `MODULE_STATE_GOING`
   - Cleans up everything and unloads the module

### 3.3 modprobe vs insmod

- **`insmod`** loads a single `.ko` file. If it depends on other modules, loading fails.
- **`modprobe`** reads `/lib/modules/$(uname -r)/modules.dep` (generated by `depmod`) and automatically loads dependencies first.

```bash
# insmod requires the full path and handles no dependencies:
sudo insmod /lib/modules/6.19.0/kernel/drivers/net/ethernet/intel/e1000e/e1000e.ko

# modprobe finds the module by name and loads dependencies:
sudo modprobe e1000e
```

### 3.4 Module Unloading

```
rmmod hello
  → delete_module("hello", 0)          # syscall
    → SYSCALL_DEFINE2(delete_module)    [kernel/module/main.c:1068]
      → Check refcount == 0
      → Set state to MODULE_STATE_GOING
      → mod->exit()                     # your __exit function
      → Free all module memory
```

A module cannot be unloaded if its reference count is nonzero. Other modules that depend on it, or open file descriptors using it, hold references.

---

## 4. Module Parameters

Module parameters let you configure a module at load time (or at runtime via sysfs).

### 4.1 Declaring Parameters

```c
static int debug_level = 0;
module_param(debug_level, int, 0644);
MODULE_PARM_DESC(debug_level, "Debug verbosity level (0-3)");

static char *device_name = "mydev";
module_param(device_name, charp, 0444);
MODULE_PARM_DESC(device_name, "Name of the device");
```

The `module_param()` macro (`include/linux/moduleparam.h:134`):

```c
#define module_param(name, type, perm)  \
    module_param_named(name, name, type, perm)
```

It creates a `struct kernel_param` (`moduleparam.h:77–89`):

```c
struct kernel_param {
    const char *name;                    // parameter name
    struct module *mod;                  // owning module
    const struct kernel_param_ops *ops;  // get/set operations
    const u16 perm;                      // sysfs permissions
    union {
        void *arg;                       // pointer to the variable
        const struct kparam_string *str;
        const struct kparam_array *arr;
    };
};
```

The `struct kernel_param_ops` (`moduleparam.h:55–64`) provides type-safe get/set functions:

```c
struct kernel_param_ops {
    int (*set)(const char *val, const struct kernel_param *kp);
    int (*get)(char *buffer, const struct kernel_param *kp);
    void (*free)(void *arg);
};
```

### 4.2 Supported Types

| Type | C Type | Description |
|------|--------|-------------|
| `bool` | `bool` | true/false, 1/0, y/n |
| `int` | `int` | Signed integer |
| `uint` | `unsigned int` | Unsigned integer |
| `long` | `long` | Signed long |
| `charp` | `char *` | String (kernel allocates a copy) |
| `short` | `short` | Short integer |

### 4.3 Using Parameters

```bash
# Set at load time:
sudo insmod mydriver.ko debug_level=2 device_name="eth_test"

# Or with modprobe:
sudo modprobe mydriver debug_level=2

# Read at runtime via sysfs:
cat /sys/module/mydriver/parameters/debug_level

# Write at runtime (if permission allows):
echo 3 > /sys/module/mydriver/parameters/debug_level
```

### 4.4 Array Parameters

```c
static int ports[4] = {0x3f8, 0x2f8, 0x3e8, 0x2e8};
static int num_ports;
module_param_array(ports, int, &num_ports, 0444);
```

The `module_param_array()` macro (`moduleparam.h:524`) declares an array parameter. The `num_ports` variable receives the actual count of values provided.

---

## 5. The Linux Device Model

The device model is a framework that organizes all devices and drivers in the system into a hierarchical structure. It provides:

- Automatic device-driver matching (probe)
- Power management (suspend/resume)
- Hot-plug support (USB, PCI hotplug)
- A user-space view via sysfs (`/sys/`)

### 5.1 The Core Abstractions

```
                    Bus Type
                 (struct bus_type)
                   /          \
                  /            \
            Device              Driver
       (struct device)    (struct device_driver)
              |                    |
         Hardware              Software
    "I exist on this bus"   "I can drive these devices"
              |                    |
              └──── match() ───────┘
                       │
                    probe()
```

### 5.2 struct device — The Hardware Side

`struct device` (`include/linux/device.h`, ~line 500+) represents any device in the system:

| Field | Purpose |
|-------|---------|
| `kobj` | sysfs representation (creates `/sys/devices/...` entry) |
| `parent` | Parent device (creates the device hierarchy) |
| `bus` | Which bus this device sits on |
| `driver` | Which driver is bound to this device (NULL if unbound) |
| `driver_data` | Opaque pointer for the driver's private data |
| `devres_head` | List of managed resources (devm_) |
| `of_node` | Device tree node (for DT-based platforms) |

### 5.3 struct device_driver — The Software Side

`struct device_driver` (`include/linux/device/driver.h:98–120`):

```c
struct device_driver {
    const char *name;                              // 99 — driver name
    const struct bus_type *bus;                     // 100 — bus this driver works on
    struct module *owner;                           // 101 — module owning this driver
    const struct of_device_id *of_match_table;     // 102 — device tree matching
    int (*probe)(struct device *dev);              // 104 — called when device matches
    void (*remove)(struct device *dev);            // 105 — called on unbind
    void (*shutdown)(struct device *dev);          // 106 — called on system shutdown
    int (*suspend)(struct device *dev, pm_message_t); // 107 — power management
    int (*resume)(struct device *dev);             // 108
    const struct attribute_group **groups;         // 109 — sysfs attributes
    const struct attribute_group **dev_groups;     // 110 — per-device sysfs attributes
    const struct dev_pm_ops *pm;                   // 111 — PM operations
};
```

### 5.4 struct bus_type — The Matchmaker

`struct bus_type` (`include/linux/device/bus.h:78–125`):

```c
struct bus_type {
    const char *name;                                  // 79 — bus name ("pci", "platform")
    const struct attribute_group **bus_groups;         // 81
    const struct attribute_group **dev_groups;         // 82

    int (*match)(struct device *, const struct device_driver *); // 85 — does this driver fit this device?
    int (*probe)(struct device *);                     // 87 — initialize after match
    void (*remove)(struct device *);                   // 89 — teardown
    int (*suspend)(struct device *, pm_message_t);    // 97
    int (*resume)(struct device *);                    // 98
    ...
};
```

The `match()` callback is the key — it decides whether a driver can handle a device. Different bus types implement this differently:
- **PCI bus**: matches vendor ID + device ID from the driver's `id_table`
- **Platform bus**: matches the device name or device tree `compatible` string
- **USB bus**: matches vendor/product ID, device class, etc.

### 5.5 The Probe Sequence

When a new device appears (or a new driver is registered), the bus core (`drivers/base/dd.c`) tries to match them:

```
1. bus->match(dev, drv)         — does this driver claim this device?
   ↓ returns 1 (yes)
2. try_module_get(drv->owner)   — pin the driver's module (prevent rmmod)
   ↓
3. dev->driver = drv            — bind the driver to the device
   ↓
4. drv->probe(dev)              — call the driver's probe function
   ↓ returns 0 (success)
5. Device is now live, driver is managing it
```

If `probe()` returns a negative errno, the binding fails — `dev->driver` is set back to NULL and the module reference is dropped.

### 5.6 The sysfs Hierarchy

The device model is fully visible in `/sys/`:

```bash
/sys/
├── bus/
│   ├── pci/
│   │   ├── devices/           # symlinks to all PCI devices
│   │   └── drivers/           # all PCI drivers
│   │       └── e1000e/        # a specific driver
│   │           ├── bind       # write a device address to force-bind
│   │           └── unbind     # write a device address to force-unbind
│   └── platform/
│       ├── devices/
│       └── drivers/
├── class/
│   ├── net/                   # network devices
│   │   └── eth0 → ../../devices/pci0000:00/.../net/eth0
│   └── block/
├── devices/
│   └── pci0000:00/            # device hierarchy
│       └── 0000:00:19.0/      # a PCI device
│           ├── vendor          # "0x8086"
│           ├── device          # "0x1502"
│           ├── driver → ../../../bus/pci/drivers/e1000e
│           └── net/eth0/
└── module/
    └── e1000e/
        └── parameters/        # module parameters
```

---

## 6. Platform Devices and Drivers

Platform devices represent hardware that is not discoverable by the bus — it is hardcoded in the board description or device tree. This includes SoC peripherals, memory-mapped I/O controllers, and embedded hardware.

### 6.1 struct platform_device

`struct platform_device` (`include/linux/platform_device.h:23–45`):

```c
struct platform_device {
    const char *name;                          // 24 — device name (used for matching)
    int id;                                    // 25 — instance ID (-1 for single)
    struct device dev;                         // 27 — embedded device struct
    u32 num_resources;                         // 30 — number of resources
    struct resource *resource;                 // 31 — I/O memory, IRQs, DMA channels
    const struct platform_device_id *id_entry; // 33 — matched ID table entry
    const char *driver_override;               // 38 — force specific driver
};
```

Resources describe the hardware's memory regions and interrupts:

```c
struct resource {
    resource_size_t start;   // start address (MMIO) or IRQ number
    resource_size_t end;     // end address
    const char *name;
    unsigned long flags;     // IORESOURCE_MEM, IORESOURCE_IRQ, etc.
};
```

### 6.2 struct platform_driver

`struct platform_driver` (`platform_device.h:239–256`):

```c
struct platform_driver {
    int (*probe)(struct platform_device *);          // 240 — device found
    void (*remove)(struct platform_device *);        // 241 — device removed
    void (*shutdown)(struct platform_device *);      // 242
    int (*suspend)(struct platform_device *, pm_message_t); // 243
    int (*resume)(struct platform_device *);         // 244
    struct device_driver driver;                     // 245 — embedded driver
    const struct platform_device_id *id_table;       // 246 — name matching
    bool prevent_deferred_probe;                     // 247
};
```

### 6.3 Registration

```c
// Register the driver:
platform_driver_register(&my_driver);    // platform_device.h:264

// Or use the convenience macro that handles module_init/exit for you:
module_platform_driver(my_driver);       // platform_device.h:294
```

`module_platform_driver()` expands to:

```c
static int __init my_driver_init(void) { return platform_driver_register(&my_driver); }
static void __exit my_driver_exit(void) { platform_driver_unregister(&my_driver); }
module_init(my_driver_init);
module_exit(my_driver_exit);
```

### 6.4 Device Tree Matching

On ARM and RISC-V platforms, devices are described in a Device Tree (`.dts` file). The driver declares which `compatible` strings it supports:

```c
static const struct of_device_id my_of_match[] = {
    { .compatible = "vendor,my-device" },
    { .compatible = "vendor,my-device-v2", .data = &v2_data },
    { /* sentinel */ }
};
MODULE_DEVICE_TABLE(of, my_of_match);

static struct platform_driver my_driver = {
    .probe  = my_probe,
    .remove = my_remove,
    .driver = {
        .name = "my-device",
        .of_match_table = my_of_match,
    },
};
```

The `struct of_device_id` is defined in `include/linux/mod_devicetable.h:282–286`:

```c
struct of_device_id {
    char name[32];
    char type[32];
    char compatible[128];    // matched against device tree "compatible" property
    const void *data;        // driver-specific data for this match
};
```

---

## 7. Character Device Drivers

Character devices are the most common driver type. They expose a file in `/dev/` that applications can `open()`, `read()`, `write()`, and `ioctl()`.

### 7.1 struct cdev

`struct cdev` (`include/linux/cdev.h:14–21`):

```c
struct cdev {
    struct kobject kobj;                     // 15 — sysfs representation
    struct module *owner;                    // 16 — owning module
    const struct file_operations *ops;       // 17 — file operations
    struct list_head list;                   // 18 — open inodes list
    dev_t dev;                               // 19 — device number (major:minor)
    unsigned int count;                      // 20 — number of minor numbers
};
```

### 7.2 struct file_operations

`struct file_operations` (`include/linux/fs.h:1918–1960`) is the core interface between VFS and your driver:

```c
struct file_operations {
    struct module *owner;
    loff_t  (*llseek)(struct file *, loff_t, int);
    ssize_t (*read)(struct file *, char __user *, size_t, loff_t *);
    ssize_t (*write)(struct file *, const char __user *, size_t, loff_t *);
    ssize_t (*read_iter)(struct kiocb *, struct iov_iter *);
    ssize_t (*write_iter)(struct kiocb *, struct iov_iter *);
    __poll_t (*poll)(struct file *, struct poll_table_struct *);
    long    (*unlocked_ioctl)(struct file *, unsigned int, unsigned long);
    int     (*mmap)(struct file *, struct vm_area_struct *);
    int     (*open)(struct inode *, struct file *);
    int     (*release)(struct inode *, struct file *);
    int     (*fsync)(struct file *, loff_t, loff_t, int);
    ...
};
```

When user space calls `read(fd, buf, count)`, the VFS looks up the file's `f_op->read` and calls it. The `char __user *` annotation tells the kernel that `buf` is a user-space pointer — you must use `copy_to_user()` / `copy_from_user()` to access it.

### 7.3 Device Numbers

Every device has a **major number** (identifies the driver) and a **minor number** (identifies the specific device instance). Together they form a `dev_t` — a 32-bit value with 12 bits for major and 20 bits for minor:

```c
MAJOR(dev_t dev);          // extract major number
MINOR(dev_t dev);          // extract minor number
MKDEV(int major, int minor); // combine into dev_t
```

### 7.4 Registration Steps

Creating a character device requires four steps:

```c
// 1. Allocate device numbers
dev_t devno;
alloc_chrdev_region(&devno, 0, 1, "mydev");   // fs/char_dev.c:236
// devno now contains your dynamically assigned major:minor

// 2. Initialize the cdev
struct cdev my_cdev;
cdev_init(&my_cdev, &my_fops);                // cdev.h:23
my_cdev.owner = THIS_MODULE;

// 3. Add the cdev to the system
cdev_add(&my_cdev, devno, 1);                 // cdev.h:29, impl: char_dev.c:470

// 4. Create a device node in /dev/ (via udev)
struct class *my_class = class_create("mydev");
device_create(my_class, NULL, devno, NULL, "mydev%d", 0);
// This triggers a uevent that tells udev to create /dev/mydev0
```

Cleanup is the reverse:

```c
device_destroy(my_class, devno);
class_destroy(my_class);
cdev_del(&my_cdev);
unregister_chrdev_region(devno, 1);            // char_dev.c:311
```

### 7.5 The VFS-to-Driver Path

```
User space:          read(fd, buf, 4096)
                         │
Syscall:             sys_read()              [fs/read_write.c]
                         │
VFS:                 vfs_read()
                         │
                     file->f_op->read()      ← your driver's read function
                         │
Driver:              my_read(file, buf, 4096, &pos)
                         │
                     copy_to_user(buf, kernel_buf, count)   ← safe user-space copy
```

### 7.6 A Complete Character Device Example

```c
#include <linux/module.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/uaccess.h>

static dev_t devno;
static struct cdev my_cdev;
static struct class *my_class;
static char message[256] = "Hello from kernel!\n";

static ssize_t my_read(struct file *filp, char __user *buf,
                        size_t count, loff_t *pos)
{
    size_t len = strlen(message);

    if (*pos >= len)
        return 0;   // EOF
    if (count > len - *pos)
        count = len - *pos;

    if (copy_to_user(buf, message + *pos, count))
        return -EFAULT;

    *pos += count;
    return count;
}

static const struct file_operations my_fops = {
    .owner = THIS_MODULE,
    .read  = my_read,
};

static int __init my_init(void)
{
    int ret;

    ret = alloc_chrdev_region(&devno, 0, 1, "mychardev");
    if (ret < 0)
        return ret;

    cdev_init(&my_cdev, &my_fops);
    my_cdev.owner = THIS_MODULE;

    ret = cdev_add(&my_cdev, devno, 1);
    if (ret < 0)
        goto err_region;

    my_class = class_create("mychardev");
    if (IS_ERR(my_class)) {
        ret = PTR_ERR(my_class);
        goto err_cdev;
    }

    device_create(my_class, NULL, devno, NULL, "mychardev");
    pr_info("mychardev: registered with major %d\n", MAJOR(devno));
    return 0;

err_cdev:
    cdev_del(&my_cdev);
err_region:
    unregister_chrdev_region(devno, 1);
    return ret;
}

static void __exit my_exit(void)
{
    device_destroy(my_class, devno);
    class_destroy(my_class);
    cdev_del(&my_cdev);
    unregister_chrdev_region(devno, 1);
}

module_init(my_init);
module_exit(my_exit);
MODULE_LICENSE("GPL");
```

Notice the `goto`-based error handling — this is the standard kernel pattern. Each `goto` label undoes resources in reverse order of acquisition.

---

## 8. PCI Drivers

PCI (Peripheral Component Interconnect) is the standard bus for GPUs, NICs, storage controllers, and most expansion cards. Unlike platform devices, PCI devices are discoverable — the kernel scans the PCI bus at boot and creates a `struct pci_dev` for each device found.

### 8.1 struct pci_driver

`struct pci_driver` (`include/linux/pci.h:1001–1018`):

```c
struct pci_driver {
    const char *name;                                  // 1001
    const struct pci_device_id *id_table;              // 1002 — which devices we support
    int  (*probe)(struct pci_dev *, const struct pci_device_id *); // 1003
    void (*remove)(struct pci_dev *);                  // 1004
    int  (*suspend)(struct pci_dev *, pm_message_t);   // 1005
    int  (*resume)(struct pci_dev *);                  // 1006
    void (*shutdown)(struct pci_dev *);                // 1007
    const struct pci_error_handlers *err_handler;     // 1010
    const struct attribute_group **groups;             // 1011
    const struct attribute_group **dev_groups;         // 1012
    struct device_driver driver;                       // 1013 — embedded driver
    bool driver_managed_dma;                           // 1015
};
```

### 8.2 Device ID Table

The ID table tells the PCI bus which devices this driver can handle. `struct pci_device_id` (`include/linux/mod_devicetable.h:44–50`):

```c
struct pci_device_id {
    __u32 vendor, device;         // vendor and device ID
    __u32 subvendor, subdevice;   // subsystem IDs (or PCI_ANY_ID)
    __u32 class, class_mask;      // device class
    kernel_ulong_t driver_data;   // private driver data
    __u32 override_only;
};
```

Example:

```c
static const struct pci_device_id my_pci_ids[] = {
    { PCI_DEVICE(0x8086, 0x1502) },   // Intel 82579LM NIC
    { PCI_DEVICE(0x8086, 0x1503) },   // Intel 82579V NIC
    { 0, }                             // sentinel
};
MODULE_DEVICE_TABLE(pci, my_pci_ids);
```

`PCI_DEVICE()` (`pci.h:1034–1036`) sets vendor and device, fills subvendor/subdevice with `PCI_ANY_ID`.

`MODULE_DEVICE_TABLE()` exports the ID table so that `depmod` can build `/lib/modules/$(uname -r)/modules.alias` — this is how `udev` knows which driver to load when a device appears.

### 8.3 Registration

```c
pci_register_driver(&my_pci_driver);    // pci.h:1650
// Or the convenience macro:
module_pci_driver(my_pci_driver);       // pci.h:1664
```

### 8.4 The PCI Probe Function

A typical PCI probe function:

```c
static int my_pci_probe(struct pci_dev *pdev, const struct pci_device_id *id)
{
    int ret;
    struct my_device *dev;

    // 1. Enable the PCI device
    ret = pci_enable_device(pdev);
    if (ret)
        return ret;

    // 2. Request I/O regions (BARs)
    ret = pci_request_regions(pdev, "my_driver");
    if (ret)
        goto err_disable;

    // 3. Map BAR 0 into kernel virtual address space
    dev = devm_kzalloc(&pdev->dev, sizeof(*dev), GFP_KERNEL);
    if (!dev) {
        ret = -ENOMEM;
        goto err_regions;
    }

    dev->mmio = pci_iomap(pdev, 0, pci_resource_len(pdev, 0));
    if (!dev->mmio) {
        ret = -EIO;
        goto err_regions;
    }

    // 4. Set up DMA
    ret = dma_set_mask_and_coherent(&pdev->dev, DMA_BIT_MASK(64));
    if (ret)
        goto err_unmap;

    // 5. Enable bus mastering (allows device to DMA)
    pci_set_master(pdev);

    // 6. Request interrupt
    ret = devm_request_irq(&pdev->dev, pdev->irq, my_irq_handler,
                           IRQF_SHARED, "my_driver", dev);
    if (ret)
        goto err_unmap;

    // 7. Store private data
    pci_set_drvdata(pdev, dev);

    return 0;

err_unmap:
    pci_iounmap(pdev, dev->mmio);
err_regions:
    pci_release_regions(pdev);
err_disable:
    pci_disable_device(pdev);
    return ret;
}
```

### 8.5 PCI BARs (Base Address Registers)

PCI devices expose up to 6 memory or I/O regions through BARs (Base Address Registers). The kernel reads these from PCI configuration space:

```c
// Get the physical address and size of BAR n:
resource_size_t start = pci_resource_start(pdev, n);
resource_size_t len   = pci_resource_len(pdev, n);

// Map BAR into kernel virtual address space:
void __iomem *base = pci_iomap(pdev, n, len);

// Access hardware registers through the mapping:
u32 status = ioread32(base + STATUS_REG_OFFSET);
iowrite32(value, base + CONTROL_REG_OFFSET);
```

The `__iomem` annotation marks pointers that refer to I/O memory — you must use `ioread*()` / `iowrite*()` to access them, never dereference directly.

---

## 9. Interrupt Handling in Drivers

### 9.1 Requesting an Interrupt

`request_irq()` (`include/linux/interrupt.h:172–180`):

```c
static inline int request_irq(unsigned int irq,
                               irq_handler_t handler,
                               unsigned long flags,
                               const char *name,
                               void *dev_id)
{
    return request_threaded_irq(irq, handler, NULL,
                                flags | IRQF_COND_ONESHOT, name, dev_id);
}
```

Parameters:
- `irq` — the IRQ number (from PCI: `pdev->irq`, from device tree: `platform_get_irq()`)
- `handler` — the hardirq handler function
- `flags` — `IRQF_SHARED`, `IRQF_ONESHOT`, trigger type, etc.
- `name` — appears in `/proc/interrupts`
- `dev_id` — opaque pointer, passed back to the handler (must be unique for shared IRQs)

### 9.2 IRQ Flags

Key flags from `include/linux/interrupt.h`:

| Flag | Line | Purpose |
|------|------|---------|
| `IRQF_SHARED` | 74 | Multiple devices share this IRQ line |
| `IRQF_ONESHOT` | 80 | Keep IRQ disabled until threaded handler finishes |
| `IRQF_NO_SUSPEND` | 81 | Don't disable this IRQ during system suspend |
| `IRQF_TRIGGER_RISING` | 32 | Trigger on rising edge |
| `IRQF_TRIGGER_FALLING` | 33 | Trigger on falling edge |
| `IRQF_TRIGGER_HIGH` | 34 | Trigger on high level |
| `IRQF_TRIGGER_LOW` | 35 | Trigger on low level |

### 9.3 The Handler Function

```c
static irqreturn_t my_irq_handler(int irq, void *dev_id)
{
    struct my_device *dev = dev_id;
    u32 status = ioread32(dev->mmio + IRQ_STATUS);

    if (!(status & MY_IRQ_PENDING))
        return IRQ_NONE;       // not our interrupt (shared IRQ)

    // Acknowledge the interrupt at the hardware level
    iowrite32(status, dev->mmio + IRQ_ACK);

    // Schedule bottom-half processing if needed
    // (tasklet, workqueue, or threaded IRQ)

    return IRQ_HANDLED;
}
```

The return type `irqreturn_t` is one of:
- `IRQ_NONE` — this wasn't our interrupt (critical for shared IRQs)
- `IRQ_HANDLED` — we handled it
- `IRQ_WAKE_THREAD` — wake the threaded handler (for `request_threaded_irq()`)

### 9.4 Threaded Interrupts

For handlers that need to sleep (e.g., I2C transfers), use `request_threaded_irq()`:

```c
request_threaded_irq(irq,
    my_hardirq_handler,     // runs in hardirq context (fast, no sleeping)
    my_threaded_handler,    // runs in a kernel thread (can sleep)
    IRQF_ONESHOT,           // keep IRQ masked until thread finishes
    "my_driver",
    dev);
```

The hardirq handler returns `IRQ_WAKE_THREAD` to schedule the threaded handler. See `interrupt.md` Section 6 for the full interrupt model.

### 9.5 Managed Interrupts

`devm_request_irq()` (`interrupt.h:228–234`) automatically calls `free_irq()` when the device is unbound:

```c
devm_request_irq(&pdev->dev, pdev->irq, my_handler,
                 IRQF_SHARED, "my_driver", dev);
// No need to call free_irq() in remove() or error paths
```

---

## 10. Managed Resources (devm_)

### 10.1 The Problem: Error Path Cleanup

A driver's `probe()` function typically acquires many resources: memory, I/O mappings, IRQs, clocks, regulators, GPIOs. If step 5 of 8 fails, you need to clean up steps 4, 3, 2, and 1 — in reverse order. This leads to deeply nested goto chains.

### 10.2 The Solution: Automatic Cleanup

The `devm_` (device-managed) API ties resource lifetime to the device. When the device is unbound from the driver (or `probe()` fails), all managed resources are automatically freed in LIFO (last-in, first-out) order.

Internally, each `devm_` call allocates a `struct devres` and links it to `dev->devres_head`. Each entry records a cleanup function. On unbind, the kernel walks this list and calls each cleanup function.

### 10.3 Common devm_ Functions

**Memory** (`include/linux/device/devres.h`):

| Function | Line | Purpose |
|----------|------|---------|
| `devm_kmalloc()` | 48 | Allocate memory, auto-freed on unbind |
| `devm_kzalloc()` | 52 | Zero-initialized allocation |
| `devm_kcalloc()` | 65 | Array allocation with overflow check |
| `devm_kfree()` | 80 | Explicit early free (rarely needed) |

**I/O Mapping**:

| Function | Purpose |
|----------|---------|
| `devm_ioremap_resource()` | Map MMIO resource, auto-unmapped |
| `devm_platform_ioremap_resource()` | Platform device convenience variant |

**Interrupts**:

| Function | Purpose |
|----------|---------|
| `devm_request_irq()` | Request IRQ, auto-freed |
| `devm_request_threaded_irq()` | Threaded IRQ, auto-freed |

**Clocks, regulators, GPIO**:

| Function | Purpose |
|----------|---------|
| `devm_clk_get()` | Get clock reference |
| `devm_regulator_get()` | Get regulator reference |
| `devm_gpiod_get()` | Get GPIO descriptor |

### 10.4 Before and After devm_

**Without devm_ — manual cleanup:**

```c
static int my_probe(struct platform_device *pdev)
{
    dev->base = ioremap(res->start, resource_size(res));
    if (!dev->base)
        return -EIO;

    ret = request_irq(irq, handler, 0, "my", dev);
    if (ret)
        goto err_unmap;

    dev->clk = clk_get(&pdev->dev, "main");
    if (IS_ERR(dev->clk)) {
        ret = PTR_ERR(dev->clk);
        goto err_irq;
    }

    return 0;

err_irq:
    free_irq(irq, dev);
err_unmap:
    iounmap(dev->base);
    return ret;
}
```

**With devm_ — automatic cleanup:**

```c
static int my_probe(struct platform_device *pdev)
{
    dev->base = devm_ioremap_resource(&pdev->dev, res);
    if (IS_ERR(dev->base))
        return PTR_ERR(dev->base);

    ret = devm_request_irq(&pdev->dev, irq, handler, 0, "my", dev);
    if (ret)
        return ret;      // ioremap is automatically undone

    dev->clk = devm_clk_get(&pdev->dev, "main");
    if (IS_ERR(dev->clk))
        return PTR_ERR(dev->clk);   // irq and ioremap both undone

    return 0;
}
// remove() can be empty — everything is auto-cleaned
```

---

## 11. sysfs — Exposing Driver Data to User Space

### 11.1 struct attribute

`struct attribute` (`include/linux/sysfs.h:30–38`):

```c
struct attribute {
    const char *name;      // filename in /sys/
    umode_t mode;          // permissions (0444, 0644, etc.)
};
```

### 11.2 struct device_attribute

`struct device_attribute` (`include/linux/device.h:105–111`):

```c
struct device_attribute {
    struct attribute attr;
    ssize_t (*show)(struct device *dev, struct device_attribute *attr, char *buf);
    ssize_t (*store)(struct device *dev, struct device_attribute *attr,
                     const char *buf, size_t count);
};
```

### 11.3 Creating Attributes

The `DEVICE_ATTR()` macro family (`device.h:139`):

```c
// Read-write attribute:
static ssize_t debug_level_show(struct device *dev,
                                 struct device_attribute *attr, char *buf)
{
    struct my_device *mydev = dev_get_drvdata(dev);
    return sysfs_emit(buf, "%d\n", mydev->debug_level);
}

static ssize_t debug_level_store(struct device *dev,
                                  struct device_attribute *attr,
                                  const char *buf, size_t count)
{
    struct my_device *mydev = dev_get_drvdata(dev);
    int val;

    if (kstrtoint(buf, 10, &val))
        return -EINVAL;

    mydev->debug_level = val;
    return count;
}

static DEVICE_ATTR_RW(debug_level);   // creates dev_attr_debug_level

// Read-only attribute:
static ssize_t status_show(struct device *dev,
                            struct device_attribute *attr, char *buf)
{
    return sysfs_emit(buf, "running\n");
}

static DEVICE_ATTR_RO(status);        // creates dev_attr_status
```

### 11.4 Attribute Groups

Group attributes together for bulk registration:

```c
static struct attribute *my_attrs[] = {
    &dev_attr_debug_level.attr,
    &dev_attr_status.attr,
    NULL,   // sentinel
};

static const struct attribute_group my_attr_group = {
    .attrs = my_attrs,
};

static const struct attribute_group *my_attr_groups[] = {
    &my_attr_group,
    NULL,
};
```

Then set `driver.dev_groups = my_attr_groups` in your driver structure — the attributes are automatically created when the device is bound and removed when unbound.

### 11.5 User-Space View

```bash
# Read:
cat /sys/devices/.../debug_level
3

# Write:
echo 1 > /sys/devices/.../debug_level
```

---

## 12. A Real Kernel Driver: dummy-irq

Let us examine a real (tiny) driver from the kernel tree: `drivers/misc/dummy-irq.c`. It registers a shared interrupt handler on a user-specified IRQ — useful for debugging interrupt problems.

```c
/* drivers/misc/dummy-irq.c — abbreviated */

#include <linux/module.h>
#include <linux/interrupt.h>

static int irq = -1;

static irqreturn_t dummy_interrupt(int irq, void *dev_id)     // line 20
{
    static int count = 0;
    if (count == 0) {
        printk(KERN_INFO "dummy-irq: interrupt occurred on IRQ %d\n", irq);
        count++;
    }
    return IRQ_NONE;    // we didn't really handle it (shared)
}

static int __init dummy_irq_init(void)                        // line 33
{
    if (irq < 0) {
        printk(KERN_ERR "dummy-irq: no IRQ given.  Use irq=N\n");
        return -EIO;
    }
    if (request_irq(irq, &dummy_interrupt,
                    IRQF_SHARED, "dummy_irq", &irq)) {       // line 39
        printk(KERN_ERR "dummy-irq: cannot register IRQ %d\n", irq);
        return -EIO;
    }
    printk(KERN_INFO "dummy-irq: registered for IRQ %d\n", irq);
    return 0;
}

static void __exit dummy_irq_exit(void)                       // line 47
{
    printk(KERN_INFO "dummy-irq unloaded\n");
    free_irq(irq, &irq);                                     // line 50
}

module_init(dummy_irq_init);                                  // line 53
module_exit(dummy_irq_exit);                                  // line 54

MODULE_LICENSE("GPL");                                        // line 56
module_param_hw(irq, uint, irq, 0444);                       // line 58
MODULE_PARM_DESC(irq, "The IRQ to register for");
MODULE_DESCRIPTION("Dummy IRQ handler driver");               // line 60
```

Usage:

```bash
sudo insmod dummy-irq.ko irq=16
cat /proc/interrupts | grep dummy_irq
sudo rmmod dummy-irq
```

This driver demonstrates: module parameters, `__init`/`__exit`, `request_irq()`/`free_irq()`, shared IRQ handling, and proper error checking.

---

## 13. DMA — Direct Memory Access

### 13.1 What Is DMA?

DMA allows hardware devices to read from and write to system memory without CPU intervention. Instead of the CPU copying data byte-by-byte between device registers and memory, the device does it directly. This is essential for high-performance devices — a NIC receiving gigabit traffic or a storage controller doing disk I/O.

### 13.2 The DMA API

The kernel provides a portable DMA API (`include/linux/dma-mapping.h`) that handles the complexities of cache coherency, IOMMU mapping, and address translation:

**Coherent DMA** — memory that is always consistent between CPU and device:

```c
// Allocate coherent DMA memory (for descriptor rings, etc.)
void *cpu_addr = dma_alloc_coherent(&pdev->dev, size,
                                     &dma_handle, GFP_KERNEL);
// cpu_addr  = virtual address for CPU access
// dma_handle = DMA address for device programming

// Free when done:
dma_free_coherent(&pdev->dev, size, cpu_addr, dma_handle);
```

**Streaming DMA** — for one-shot data transfers (packet buffers):

```c
// Map a buffer for DMA:
dma_addr_t dma = dma_map_single(&pdev->dev, buffer, size, DMA_TO_DEVICE);
// Now the device can read from 'dma'

// After the transfer is complete:
dma_unmap_single(&pdev->dev, dma, size, DMA_TO_DEVICE);
// Now the CPU can access 'buffer' again
```

### 13.3 DMA Direction

| Direction | Meaning |
|-----------|---------|
| `DMA_TO_DEVICE` | CPU → device (transmit) |
| `DMA_FROM_DEVICE` | Device → CPU (receive) |
| `DMA_BIDIRECTIONAL` | Both directions |

### 13.4 DMA Mask

A DMA mask tells the kernel which physical addresses the device can reach:

```c
// Device can address 64 bits of physical memory:
dma_set_mask_and_coherent(&pdev->dev, DMA_BIT_MASK(64));

// Older device limited to 32-bit addresses (4 GB):
dma_set_mask_and_coherent(&pdev->dev, DMA_BIT_MASK(32));
```

If the device has a limited DMA mask, the kernel's IOMMU or bounce buffer mechanism ensures data is placed in reachable memory.

---

## 14. Power Management

### 14.1 The PM Callbacks

Drivers participate in system suspend/resume through `struct dev_pm_ops`:

```c
static int my_suspend(struct device *dev)
{
    struct my_device *mydev = dev_get_drvdata(dev);

    // Save hardware state
    mydev->saved_reg = ioread32(mydev->base + SOME_REG);

    // Disable the device
    // ...

    return 0;
}

static int my_resume(struct device *dev)
{
    struct my_device *mydev = dev_get_drvdata(dev);

    // Restore hardware state
    iowrite32(mydev->saved_reg, mydev->base + SOME_REG);

    // Re-enable the device
    // ...

    return 0;
}

static DEFINE_SIMPLE_DEV_PM_OPS(my_pm_ops, my_suspend, my_resume);

static struct platform_driver my_driver = {
    .driver = {
        .name = "my-device",
        .pm   = pm_sleep_ptr(&my_pm_ops),
    },
    .probe  = my_probe,
    .remove = my_remove,
};
```

### 14.2 Runtime PM

Runtime PM allows devices to power down when idle, without a full system suspend:

```c
// In probe():
pm_runtime_enable(&pdev->dev);

// Before using the device:
pm_runtime_get_sync(&pdev->dev);    // wake up, increment usage count

// When done:
pm_runtime_put(&pdev->dev);         // decrement usage, may power down

// In remove():
pm_runtime_disable(&pdev->dev);
```

---

## 15. Function Quick Reference

| Function | File:Line | Role |
|----------|-----------|------|
| `module_init()` | `include/linux/module.h:131` | Declare module init function |
| `module_exit()` | `include/linux/module.h:139` | Declare module exit function |
| `module_param()` | `include/linux/moduleparam.h:134` | Declare a module parameter |
| `alloc_chrdev_region()` | `fs/char_dev.c:236` | Dynamically allocate device numbers |
| `register_chrdev_region()` | `fs/char_dev.c:200` | Register fixed device numbers |
| `unregister_chrdev_region()` | `fs/char_dev.c:311` | Release device numbers |
| `cdev_init()` | `include/linux/cdev.h:23` | Initialize a cdev struct |
| `cdev_add()` | `fs/char_dev.c:470` | Add cdev to the system |
| `cdev_del()` | `include/linux/cdev.h:35` | Remove a cdev |
| `platform_driver_register()` | `include/linux/platform_device.h:264` | Register a platform driver |
| `module_platform_driver()` | `platform_device.h:294` | Helper combining init/exit |
| `pci_register_driver()` | `include/linux/pci.h:1650` | Register a PCI driver |
| `module_pci_driver()` | `pci.h:1664` | Helper combining init/exit |
| `request_irq()` | `include/linux/interrupt.h:172` | Register an interrupt handler |
| `request_threaded_irq()` | `kernel/irq/manage.c` | Register threaded interrupt handler |
| `free_irq()` | `include/linux/interrupt.h:227` | Free an interrupt handler |
| `devm_request_irq()` | `interrupt.h:228` | Managed interrupt registration |
| `devm_kmalloc()` | `include/linux/device/devres.h:48` | Managed memory allocation |
| `devm_kzalloc()` | `devres.h:52` | Managed zero-initialized allocation |
| `devm_ioremap_resource()` | `devres.h:121` | Managed MMIO mapping |
| `dma_alloc_coherent()` | `include/linux/dma-mapping.h` | Allocate coherent DMA buffer |
| `dma_map_single()` | `include/linux/dma-mapping.h` | Map buffer for streaming DMA |

---

## 16. Debugging Drivers

### 16.1 Kernel Messages

```bash
# Watch kernel messages in real time:
dmesg -w

# Filter by module name:
dmesg | grep my_driver

# Set log level:
echo 7 > /proc/sys/kernel/printk    # show all messages
```

Use `pr_info()`, `pr_err()`, `pr_debug()` for module messages, or `dev_info()`, `dev_err()`, `dev_dbg()` for device-specific messages (they prefix the device name automatically).

### 16.2 Module Information

```bash
# List loaded modules:
lsmod

# Show module info from .ko file:
modinfo e1000e.ko

# Module parameters at runtime:
ls /sys/module/e1000e/parameters/
cat /sys/module/e1000e/parameters/debug

# Module dependencies:
modprobe --show-depends e1000e
```

### 16.3 Device and Driver Binding

```bash
# List all devices on a bus:
ls /sys/bus/pci/devices/

# See which driver is bound:
readlink /sys/bus/pci/devices/0000:00:19.0/driver

# Force unbind:
echo "0000:00:19.0" > /sys/bus/pci/drivers/e1000e/unbind

# Force rebind:
echo "0000:00:19.0" > /sys/bus/pci/drivers/e1000e/bind
```

### 16.4 Tracing Driver Activity

```bash
# Trace probe/remove events:
sudo trace-cmd record -e driver:driver_bound \
                      -e driver:driver_probe_done \
                      sleep 5
trace-cmd report

# Trace a specific function:
echo my_probe > /sys/kernel/tracing/set_ftrace_filter
echo function > /sys/kernel/tracing/current_tracer
cat /sys/kernel/tracing/trace

# Trace IRQ handlers:
sudo trace-cmd record -e irq:irq_handler_entry \
                      -e irq:irq_handler_exit \
                      sleep 2
trace-cmd report
```

### 16.5 Common Pitfalls

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Missing `MODULE_LICENSE("GPL")` | Kernel tainted, GPL symbols unavailable | Add the macro |
| Dereferencing `__iomem` pointer directly | Sparse warnings, wrong values on some architectures | Use `ioread32()` / `iowrite32()` |
| Sleeping in hardirq context | "BUG: scheduling while atomic" | Use threaded IRQ or workqueue |
| Forgetting `copy_to_user()` / `copy_from_user()` | Kernel oops or security vulnerability | Always use for user pointers |
| Not checking return values | Silent failures, resource leaks | Check every allocation and registration |
| Missing sentinel in ID table | Match beyond array bounds | End `pci_device_id[]` / `of_device_id[]` with `{ 0 }` or `{ }` |
| Using `kfree()` on `devm_` memory | Double free on unbind | Use `devm_kfree()` or let auto-cleanup handle it |

---

*This document covers the Linux 6.19 device driver and kernel module subsystems. For in-depth driver examples, study `drivers/misc/` for simple cases and `drivers/net/ethernet/intel/e1000e/` for a full production NIC driver. The `Documentation/driver-api/` directory in the kernel source has API-specific guides.*
