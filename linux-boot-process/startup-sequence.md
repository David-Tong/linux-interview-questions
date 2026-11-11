# Linux 操作系统启动顺序

本文以 **UEFI → GRUB → Linux 内核 → initramfs → systemd** 为主线说明启动过程。路径和文件名因发行版而异；传统 BIOS、其他引导程序及不使用 initramfs 的情况在对应章节说明。

```text
按下电源键
  → 固件初始化硬件，选择启动项
  → GRUB 加载内核、initramfs，并传入启动参数
  → 内核初始化，运行 initramfs 中的 /init（PID 1）
  → 早期用户空间找到并挂载真正的根文件系统
  → 切换根目录，交接给系统 init（常见为 systemd，仍为 PID 1）
  → init 系统启动服务，提供终端或图形登录环境
```

全文使用同一个教学布局，假设是 x86-64 机器，`/boot` 独立分区，内核文件采用 Ubuntu 风格的名称：

| 分区 | Linux 运行后的挂载点 | 主要内容 |
| --- | --- | --- |
| `/dev/sda1` | `/boot/efi` | EFI 系统分区（ESP），保存 EFI 启动程序 |
| `/dev/sda2` | `/boot` | 内核、initramfs 和 GRUB 菜单配置 |
| `/dev/sda3` | `/` | Linux 根文件系统，包含 `/etc`、`/usr` 等目录 |

实际系统也可以把 `/boot` 放在根文件系统中，或将 ESP 挂载到其他目录。

## 1. 固件初始化硬件并找到引导程序

按下电源键后，主板固件先运行，初始化处理器、内存和启动所需设备，并进行必要的硬件检查。固件通常保存在主板闪存中，在 Linux 之前运行。

**UEFI（Unified Extensible Firmware Interface，统一可扩展固件接口）**是一套定义固件与操作系统启动程序之间接口的标准。现代 PC 常使用实现该标准的固件；开机按 Del、F2 等键进入的设置界面，可以配置它的启动顺序等选项。

### 1.1 UEFI 具体怎么找到 GRUB？

常见安装过程会完成两件事：把 EFI 启动程序写入 ESP，并向固件注册一个启动项。ESP 通常使用 FAT32 文件系统，里面可以同时存放多个系统的启动程序。

固件的非易失变量中通常保存以下信息，断电后仍可保留：

| 变量 | 作用 |
| --- | --- |
| `BootOrder` | 启动项的尝试顺序，例如先尝试 `Boot0003`，再尝试 `Boot0001` |
| `Boot0003` 等 `Boot####` 项 | 一个启动项，包含显示名称、目标设备路径和 EFI 文件路径 |
| `BootNext`（若设置） | 指定下一次启动优先尝试的启动项，属于一次性覆盖 |

在已通过 UEFI 启动的 Linux 中，安装了 `efibootmgr` 且固件变量可访问时，可以查看：

```sh
sudo efibootmgr -v
```

输出可能包含以下内容，编号、GUID 和扇区位置均为示意：

```text
BootOrder: 0003,0001
Boot0003* ubuntu HD(1,GPT,12345678-1234-1234-1234-123456789abc,0x800,0x100000)/File(\EFI\ubuntu\shimx64.efi)
```

这条记录可以这样读：

1. `BootOrder` 指示固件优先尝试 `Boot0003`。
2. `ubuntu` 是固件启动菜单上显示的名称。
3. `HD(1,GPT,...)` 是设备路径中的分区描述，包含 GPT 分区唯一 GUID 等信息，用于定位目标分区，本例为 ESP。
4. `File(\EFI\ubuntu\shimx64.efi)` 指定要在该分区内读取并执行的文件。

**固件根据启动项记录的设备路径定位 ESP，再按文件路径加载 EFI 程序。** 它通常不需要扫描所有目录猜测 GRUB 在哪里，也不需要先读取 Linux 的 `/etc/fstab` 或 `grub.cfg`。

本例先加载 `shimx64.efi`，由 shim 再加载同目录中的 `grubx64.efi`。shim 常用于衔接 Secure Boot 信任链；其他安装配置可能让启动项直接指向 GRUB。Secure Boot 用于验证启动程序是否受信任，使用 UEFI 不代表一定启用它。

```text
主板固件中的 Boot0003
  → ESP 内的 \EFI\ubuntu\shimx64.efi
  → ESP 内的 \EFI\ubuntu\grubx64.efi
  → GRUB 读取启动配置
```

Linux 运行后，ESP 挂载到 `/boot/efi`，同一个 GRUB 文件才表现为 `/boot/efi/EFI/ubuntu/grubx64.efi`。**`/boot/efi` 是 Linux 的挂载点，不是固件查找文件时使用的路径。**

对于可移动介质，UEFI 定义了默认启动文件路径；x86-64 对应 `\EFI\BOOT\BOOTX64.EFI`。本地启动项缺失时，部分固件也会尝试这种路径，但具体恢复行为因固件而异。

### 1.2 与传统 BIOS 的区别

| 启动方式 | 常见的后续程序加载方式 |
| --- | --- |
| 传统 BIOS | 读取启动磁盘的引导扇区并执行其中的代码，再由后续阶段加载引导程序 |
| UEFI | 通过启动项中的设备与文件路径加载 EFI 可执行程序 |

UEFI 固件菜单用于选择 EFI 程序，下面的 GRUB 菜单用于选择内核和参数。某些配置还允许 UEFI 直接加载支持 EFI 启动的 Linux 内核。

## 2. GRUB 读取配置并把内核加载到内存

GRUB 运行后，读取启动配置，按设置显示菜单或等待默认启动。用户选择启动项，或等待超时后，GRUB 执行相应的加载指令。

### 2.1 配置文件之间的关系

| 文件或目录 | 用途 |
| --- | --- |
| `/etc/default/grub` | 配置生成工具的输入，保存等待时间、附加内核参数等变量 |
| `/etc/grub.d/` | 参与生成菜单的脚本 |
| `/boot/grub/grub.cfg` 或 `/boot/grub2/grub.cfg` | 常见的生成后菜单配置，包含 `menuentry` 等指令 |

GRUB 如何定位主配置由安装方式决定，例如使用安装时设置的路径，或先读取 ESP 中的小配置，再转到 `/boot` 上的主配置。

例如，`/etc/default/grub` 中可能有：

```sh
GRUB_TIMEOUT=5
GRUB_CMDLINE_LINUX=""
GRUB_CMDLINE_LINUX_DEFAULT="quiet"
```

这些变量控制启动等待时间及附加参数，具体行为还受菜单样式和发行版脚本影响。配置生成工具通常会探测根设备等信息，并结合这些输入生成启动项。修改后需要重新生成配置；Debian/Ubuntu 通常使用 `sudo update-grub`，其他发行版使用各自的工具和目标路径。直接修改生成的 `grub.cfg` 可能被下次生成覆盖。

### 2.2 完整菜单示例

下面是主 `grub.cfg` 中的一个教学启动项。两个 UUID 都是虚构占位值，内核版本也只作示例：

```text
menuentry 'Linux 示例' {
    search --no-floppy --fs-uuid --set=root aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee
    linux /vmlinuz-6.8.0-60-generic root=UUID=11111111-2222-3333-4444-555555555555 ro quiet
    initrd /initrd.img-6.8.0-60-generic
}
```

| 语句或参数 | 含义 |
| --- | --- |
| `menuentry 'Linux 示例' { ... }` | 定义一个名为“Linux 示例”的菜单项，选择后执行花括号内的指令 |
| `search --no-floppy --fs-uuid --set=root ...` | 跳过软盘，按文件系统 UUID 查找本例的 `/dev/sda2`，将设备保存到 GRUB 的 `root` 变量 |
| `linux /vmlinuz-6.8.0-60-generic ...` | 从 GRUB 当前根设备读取内核，并设置后面的内核命令行参数 |
| `root=UUID=11111111-...` | 指定真正的 Linux 根文件系统，本例位于 `/dev/sda3` |
| `ro` | 要求初次挂载真正的根文件系统时使用只读方式，后续通常按系统配置重新挂载为可写 |
| `quiet` | 减少启动时显示的日志 |
| `initrd /initrd.img-6.8.0-60-generic` | 将早期用户空间映像加载到内存；现代系统通常使用 initramfs 格式 |

UUID 是文件系统的标识，通常在创建文件系统时生成。这里 `aaaaaaaa-...` 对应 `/boot` 文件系统，`11111111-...` 对应 `/` 文件系统。文件系统 UUID 与 UEFI 启动项中用于定位 GPT 分区的分区 GUID（Linux 中可查看为 PARTUUID）属于不同标识。

**GRUB 的 `root` 用于找内核文件，内核参数 `root=` 用于指定最终的 Linux 根文件系统。** 两者在本例中指向不同分区。

GRUB 使用自身的文件系统支持读取文件，此时还没有 Linux 的挂载关系。因此，本例内核路径 `/vmlinuz-...` 对应 Linux 运行后的 `/boot/vmlinuz-...`。如果 `/boot` 是根文件系统中的普通目录，GRUB 选中该文件系统后，路径通常应写成 `/boot/vmlinuz-...`，initramfs 路径同理。

内核与 initramfs 加载到内存后，GRUB 把启动参数和映像相关信息交给内核，开始执行内核。

## 3. 内核初始化并启动 initramfs 中的 /init

GRUB 交出控制权后，内核开始执行。`vmlinuz` 是内核映像的文件名；加载启动后，运行的是内存中的内核代码。

### 3.1 建立内核运行环境

内核根据硬件架构和映像格式完成必要的解压与入口设置，逐步建立运行环境：

- 初始化页表、内存分配等内存管理机制。
- 配置中断控制器、定时器等硬件，建立中断处理、时间管理和任务调度机制。
- 建立进程管理、虚拟文件系统等核心设施。
- 初始化其他 CPU、内置设备驱动及内核子系统。

这里既有对硬件的配置，也有内核软件机制的建立。固件完成的是启动所需的基本硬件初始化，内核还需要建立自己管理设备和任务的方式。上述列表按职责概括，不是源码函数的严格执行顺序；动态驱动模块也可以在后续由用户空间请求加载。

### 3.2 建立 PID 1、PID 2 等任务

启动时已有一个 PID 为 0 的特殊初始任务。它执行早期工作，随后成为启动 CPU 的 idle（空闲）任务，在没有其他可运行任务时执行空闲处理。其他 CPU 也有各自的空闲任务。

内核启动代码还创建 PID 1 和 PID 2：

| 任务 | 来源与父子关系 | 后续用途 |
| --- | --- | --- |
| PID 0，启动任务/idle | 内核启动时建立 | 早期启动及启动 CPU 的空闲处理 |
| PID 1 | 内核启动代码创建，PPID 为 0 | 先继续内核初始化，随后执行用户空间 init |
| PID 2，kthreadd | 内核启动代码创建，PPID 为 0 | 处理许多内核线程的创建请求 |

**PID 0 不能等同于整个内核。** 内核包括调度、内存管理、驱动等代码；普通进程执行系统调用时也会运行内核代码。内核线程是参与调度、但不执行普通用户程序的任务，许多由 kthreadd 创建，PPID 为 2。

PPID 表示直接父进程的 PID，并非默认等于 1。例如 bash（PID 1000）创建 python（PID 2000），python 的 PPID 通常为 1000。创建操作由内核实现，不代表这些进程的父进程就是 PID 0 或 PID 1。

### 3.3 准备 initramfs：为什么先使用内存中的启动环境？

**initramfs 是早期用户空间的归档映像**，包含启动入口 `/init`，以及所需的工具、驱动模块和配置。GRUB 将映像装入内存后，内核将其内容解包到初始内存文件系统。内核文档中的 **rootfs** 指这个初始根文件系统；本文用“真实根文件系统”表示最终作为 `/` 使用的磁盘文件系统。

真实根文件系统此时可能还无法访问。例如，它位于加密的 LVM 卷上，需要先加载驱动、解锁设备、激活逻辑卷，才能读取里面的系统程序。如果准备设备所需的工具只放在这个尚不能访问的文件系统中，就形成了依赖循环。initramfs 将这些工具预先提供在内存中，解决这个问题。

这些准备工作由用户空间工具组织，内核提供驱动、设备映射、文件系统等基础能力。发行版可以通过重建 initramfs 调整工具和启动策略，不必把各种配置逻辑全部编入内核。initramfs 与旧式 initrd 实现不同，但文件名和 GRUB 指令仍常沿用 `initrd`。

### 3.4 内核如何启动 /init，并进入用户态？

**第一个 init 入口位于解包后的内存文件系统顶层：`/init`。** 它可以是脚本、二进制程序，或指向 initramfs 内某个程序的符号链接。内核默认尝试执行它，也可以通过 `rdinit=` 指定其他入口。

本例中，PID 1 首次进入用户空间的过程如下：

1. PID 1 在内核态执行初始化代码，例如 `kernel_init()`。
2. 内核尝试加载 `/init`，源码中可以对应到 `run_init_process()` 和 `kernel_execve()`。
3. 加载逻辑建立用户地址空间、映射程序及所需解释器，准备用户栈、参数和程序入口。如果入口是脚本，则按照脚本指定的解释器执行。
4. 架构相关代码恢复准备好的用户执行上下文，使 CPU 以用户权限开始执行程序，PID 仍为 1。

```text
PID 1 执行内核初始化代码（内核态）
  → 加载 initramfs 中的 /init，准备用户执行环境
  → 进入用户态，运行早期启动程序
```

**由内核启动，不等于在内核态运行；文件存放在内存中，也不决定程序的权限状态。** 内核启动 `/init` 是加载用户程序，不是把它作为内核函数调用。initramfs 中的其他工具再由 `/init` 或它管理的程序启动。

早期程序执行自身代码时处于用户态，需要挂载文件系统或加载模块时，通过系统调用进入内核处理，然后返回用户态。例如：

```text
早期程序（用户态）：选择根设备及挂载参数
  → 发起挂载系统调用
内核（内核态）：检查权限，执行实际挂载
  → 返回用户态
早期程序：检查结果，继续启动
```

首次进入用户态发生在这里，不必等真实根文件系统准备好。后面的切根是文件系统视图的变化，不是首次从内核态进入用户态。

### 3.5 为什么早期 /init 也可以使用 systemd？

早期 `/init` 不必须使用 systemd：一些系统使用 shell 脚本和 BusyBox，另一些使用 systemd。使用 systemd 时，`/init` 可以链接到 **initramfs 内的 systemd**，由它组织设备等待、解锁、挂载等早期任务。

这样可以复用 systemd 的依赖管理、任务调度和日志能力。例如，“设备出现 → 解锁 → 激活逻辑卷 → 挂载”本身就是一组需要排序的任务。具体使用哪些 unit 和目标由生成的映像决定。

这仍是用户空间工作。systemd 即使以 root 身份、PID 1 运行，也不会因此成为内核线程；它调用工具和内核接口准备系统，之后再交接给真实根文件系统中的系统 init。

## 4. 挂载真实根文件系统，交接给系统 init

### 4.1 两处 init 分别在哪里？

本文这条启动路径中，先后使用两个不同文件系统中的入口：

| 对比项 | 早期 init | 系统 init |
| --- | --- | --- |
| 所在文件系统 | initramfs 解包后的初始内存文件系统 | 本例 `/dev/sda3` 上的真实根文件系统 |
| 入口路径 | `/init` | `/sbin/init`（常见默认入口） |
| 职责 | 准备根设备、挂载真实根文件系统并交接 | 组织正常系统服务和登录环境 |
| 可能使用的程序 | 脚本、自定义程序、systemd 等 | systemd、SysV init、BusyBox init 等 |

**两处入口不是同一个文件被改名。** 即使两边都使用 systemd，也是在不同文件系统中提供程序，并使用各自阶段的运行环境和配置。两者都是用户空间程序，先后沿用同一个 PID 1。

### 4.2 挂载到 /sysroot 后，如何成为 /？

早期启动程序读取 GRUB 传入的 `root=UUID=11111111-...`，找到本例 `/dev/sda3` 上的文件系统，准备好设备后，通过内核接口将它挂载到临时目录，例如 `/sysroot`。

此时 `/` 仍是初始内存文件系统，两处入口可以同时被访问：

```text
/init                  早期启动入口
/sysroot/sbin/init     真实系统的 init 入口
```

完成准备后，启动程序切换根目录，将真实根文件系统作为新的 `/`，并处理 `/dev`、`/proc`、`/sys` 等挂载点：

| 切根前 | 切根后 |
| --- | --- |
| `/sysroot` | `/` |
| `/sysroot/etc` | `/etc` |
| `/sysroot/sbin/init` | `/sbin/init` |
| `/sysroot/root` | `/root` |

**`/sysroot` 成为的是 `/`。** `/root` 通常是 root 用户家目录。部分 initramfs 使用 `/root` 作为临时挂载点，那是早期环境中的另一种路径约定。切根改变进程看到的文件系统根，不是逐个搬移文件。

### 4.3 /sbin/init 如何指向 systemd？

真实根文件系统中的 `/sbin/init` 可以是二进制文件，但在采用 systemd 的系统上，通常是软件包建立的符号链接，例如：

```text
/sbin/init → /lib/systemd/systemd
```

部分系统的 `/sbin`、`/lib` 也链接到 `/usr` 下的目录，最终路径可能是 `/usr/lib/systemd/systemd`。可以查看：

```sh
ls -l /sbin/init
readlink -f /sbin/init
# 示例结果：/usr/lib/systemd/systemd
```

**执行 `/sbin/init` 时，内核解析链接，加载的就是 systemd 二进制文件。** `/sbin/init` 是入口路径，systemd 是实际程序，init 是初始化程序的统称；这条符号链接通常由软件包提供，不是在 GRUB 中设置的。

### 4.4 两处 init 如何沿用同一个 PID 1？

切根交接使用 `exec` 替换当前进程的程序，保留 PID，而不是通过 `fork` 创建新进程。因此是同一个 PID 1 先后执行不同程序：

```text
PID 1：内核初始化代码（内核态）
  → initramfs 的 /init（用户态）
  → 准备根设备、挂载并切根
  → exec /sbin/init，解析链接到 systemd
  → 真实系统的 systemd（用户态，仍为 PID 1）
```

图中挂载、切根和执行程序会进入内核处理相应操作，然后以用户态继续执行程序；不是存在内核态和用户态两个 PID 1。

脚本式 initramfs 的交接可能类似 `exec switch_root /sysroot /sbin/init`：保留 PID 执行切根工具，再由工具执行系统 init。使用 systemd 的 initramfs 通过自身流程交接，不一定运行这条命令。

### 4.5 init 入口可以配置吗？

可以，系统 init 不必须是 `/sbin/init`。默认实现由安装的软件包决定，实际启动还受下列配置影响：

| 配置位置 | 作用 |
| --- | --- |
| 内核命令行 `rdinit=` | 指定初始内存文件系统的入口，通常默认 `/init` |
| 内核命令行 `init=` | 请求使用指定的真实系统 init；经过 initramfs 时，其处理方式取决于实现 |
| initramfs 的生成配置和启动程序 | 决定设备准备、切根和交接逻辑；修改后通常需要重建映像 |
| 真实系统的 `/sbin/init` 文件或链接 | 决定使用该入口时执行的程序 |

例如，可以在 GRUB 的 `linux` 行追加 `init=/usr/lib/systemd/systemd`，前提是真实系统存在该路径，且 initramfs 支持按此参数交接。持久修改通常在 `/etc/default/grub` 中配置附加参数，再重新生成菜单。`init=` 不表示跳过早期 `/init`。

如果不经过 initramfs 的 `/init`，内核需要具备直接访问根设备的支持，并使用它能解析的根设备标识，不能假定前面的文件系统 UUID 写法直接适用。此时由内核直接挂载根文件系统并执行系统 init。

这条直接启动路径中，显式 `init=` 执行失败会导致启动失败；未指定时，可以先尝试编译配置 `CONFIG_DEFAULT_INIT`，之后依次尝试 `/sbin/init`、`/etc/init`、`/bin/init`、`/bin/sh`。其中 shell 可以用于部分调试场景，但不会自动完成正常系统服务启动。

## 5. 系统 init 启动服务：从 SysV 对照理解 systemd

前面讲的是如何执行系统 init。从这里开始，实际使用哪种 init，就决定了如何读取配置、组织服务。你熟悉的 SysV init 与 systemd 都可以作为 PID 1，但它们使用不同的配置体系。

### 5.1 /etc/inittab 在 systemd 下还有用吗？

**systemd 本身不读取 `/etc/inittab` 来组织启动。** 即使这个文件还留在磁盘上，修改其中的 `initdefault`、`respawn` 等条目，也不会按 SysV 的方式改变 systemd 的启动行为。若某个额外工具自行读取它，则属于该工具的行为。

`inittab` 没有被一个新文件完整替代，而是由多类 unit 和配置分别接替其职责：

| SysV 中熟悉的配置 | systemd 中对应的机制 | 常见文件或目录位置 |
| --- | --- | --- |
| `id:3:initdefault:`：默认运行级别 | `default.target` 默认目标，通常链接到文本或图形目标 | 管理员设置通常是 `/etc/systemd/system/default.target` 符号链接；目标文件例如 `/usr/lib/systemd/system/multi-user.target` |
| `si::sysinit:...`：早期系统初始化脚本 | `sysinit.target` 及其依赖的挂载、设备和初始化服务 | `/usr/lib/systemd/system/sysinit.target`、`/usr/lib/systemd/system/sysinit.target.wants/`；具体初始化逻辑分布在被依赖的 unit 中 |
| `l3:3:wait:/etc/init.d/rc 3`：组织该级别的服务 | target 及各 unit 的依赖、排序关系 | 例如 `/usr/lib/systemd/system/multi-user.target` 及相关 `.service` 文件的 `[Unit]` 段；管理员覆盖通常位于 `/etc/systemd/system/<unit名>.d/*.conf` |
| `/etc/rc3.d/S...`：哪些服务随级别启动 | target 的 `.wants/`、`.requires/` 链接，以及 unit 中的依赖声明 | 例如 `/etc/systemd/system/multi-user.target.wants/example.service` 链接；软件包也可在 `/usr/lib/systemd/system/` 下提供依赖链接 |
| `/etc/init.d/example start`：如何启动服务 | `example.service` 的 `ExecStart=` 等设置 | 软件包通常提供 `/usr/lib/systemd/system/example.service`；管理员自建服务通常为 `/etc/systemd/system/example.service`，命令写在 `[Service]` 段 |
| `respawn`：进程退出后再次启动 | service 的 `Restart=` 等重启策略，语义不完全相同 | 对应 `.service` 文件的 `[Service]` 段，或 `/etc/systemd/system/example.service.d/override.conf` 中的覆盖设置 |
| `agetty` 条目：终端登录 | `getty@.service`、`serial-getty@.service` 模板及实例，配合 `getty.target` | `/usr/lib/systemd/system/getty@.service`、`serial-getty@.service`、`getty.target`；启用链接可位于 `/etc/systemd/system/getty.target.wants/`，生成器也可在 `/run/systemd/` 下创建相关配置 |
| `ctrlaltdel` 动作 | `ctrl-alt-del.target` | 通常为 `/usr/lib/systemd/system/ctrl-alt-del.target`，常链接到 `reboot.target`；管理员可在 `/etc/systemd/system/` 下覆盖 |

表中软件包目录统一写作 `/usr/lib/systemd/system/`，部分发行版使用 `/lib/systemd/system/`。默认目标也可能由软件包目录提供，因此 `/etc/systemd/system/default.target` 不一定存在。模板实例如 `getty@tty1.service` 可以直接使用 `getty@.service` 的定义，不要求磁盘上有一个同名实例文件。

查看本机实际使用哪个文件，不必猜路径：

```sh
systemctl cat default.target                  # 默认目标解析后的 unit 内容及来源
systemctl cat sysinit.target                  # 系统初始化目标
systemctl cat getty@tty1.service               # 终端实例对应的模板与覆盖配置
systemctl cat ctrl-alt-del.target             # Ctrl+Alt+Del 对应目标
systemctl show -p FragmentPath -p DropInPaths example.service # 主文件与覆盖文件路径
```

这些是职责上的对应，不是逐行转换规则。systemd 可以直接管理服务进程、挂载点、设备、socket、定时器等对象，而不仅是执行一组 rc 脚本。

### 5.2 从 runlevel 到 target：默认启动到哪里？

SysV 中，init 读取 `initdefault` 决定默认运行级别，再按 `inittab` 执行对应命令。systemd 则启动 **`default.target`**，并递归拉起它依赖的 unit。

```text
SysV：
/etc/inittab 的 initdefault
  → 进入指定 runlevel
  → 执行 rc 脚本
  → 处理该级别的服务链接

systemd：
default.target（例如链接到 graphical.target）
  → 拉起目标依赖的 unit
  → 继续展开各 unit 的依赖
  → 按排序关系调度启动任务
```

target 是一组依赖关系的组织节点，通常不直接执行程序。常见目标包括：

| target | 用途与 SysV 对照 |
| --- | --- |
| `sysinit.target` | 组织早期系统初始化工作，不是一个传统运行级别 |
| `basic.target` | 组织基础运行环境，不是一个传统运行级别 |
| `multi-user.target` | 多用户、非图形环境，职责类似很多发行版的级别 3 |
| `graphical.target` | 图形登录环境，通常在多用户环境上加入显示管理器等，类似很多发行版的级别 5 |
| `rescue.target` | 系统维护环境，职责接近单用户维护模式 |
| `emergency.target` | 更精简的紧急维护环境 |

不同 SysV 发行版对级别 2、3、4、5 的定义不同，不能把数字机械地套到所有系统。target 也不是互斥的数字状态：图形环境运行时，`graphical.target` 和它依赖的 `multi-user.target` 可以同时处于 active 状态。

```sh
systemctl get-default                     # 查看默认目标
systemctl cat default.target              # 查看默认目标解析后的配置
systemctl list-dependencies default.target # 查看依赖树
systemctl list-units --type=target         # 查看当前加载且活跃等状态的目标
```

设置以后默认启动到文本环境，可以使用 `sudo systemctl set-default multi-user.target`；这是修改默认配置，不会立即切换当前环境。`systemd.unit=` 内核参数可以覆盖本次启动目标。

`systemctl isolate multi-user.target` 则是立即切换到该目标及其依赖，可能停止图形界面和其他服务，应与 `set-default` 区分。systemd 还提供一些 `runlevelN.target` 兼容别名，但实际应以本机 target 定义为准。

### 5.3 从 rc 链接到 unit 依赖：哪些服务启动，先后怎么排？

SysV 的 rc 脚本通常通过 `S20...`、`S80...` 等链接名组织服务启动。systemd 将两个问题分开表达：

| 要解决的问题 | systemd 配置 | 含义 |
| --- | --- | --- |
| 是否一同启动其他 unit | `Wants=` | 拉起对方，但对方启动失败通常不会单凭此关系导致本 unit 启动失败 |
| 更强的需求关系 | `Requires=` | 拉起对方，并按规则传播失败或停用影响；本身不保证先后顺序 |
| 谁先完成启动，谁后开始 | `After=`、`Before=` | 当相关 unit 都有启动任务时，规定顺序，不负责拉起对方 |

例如 `After=database.service` 只规定排序，不会自动启动数据库；需要同时表达拉起与排序时，通常组合 `Wants=` 或 `Requires=` 与 `After=`。无须排序的服务可以并行启动。

注意，排在另一个 service “启动完成”之后，不一定代表应用业务已经可用，取决于该服务的 `Type=` 和就绪通知方式。`After=network.target` 也不等于可以访问互联网；需要启动期网络就绪时，还要结合 `network-online.target` 和本机的等待服务。

unit 的依赖还可以来自 target 的链接目录。例如：

```text
/etc/systemd/system/multi-user.target.wants/example.service
  → /etc/systemd/system/example.service
```

这表示 `multi-user.target` 会希望启动 `example.service`，职责类似于把服务加入某个运行级别的启动集合。

#### 对照 SysV 的 S 编号：怎样查看依赖树、启动顺序和耗时？

传统 SysV rc 脚本通常按启动链接中的数字排序，例如先处理 `S20database`，再处理 `S80web`。具体是否等待服务就绪、是否并行，还取决于 rc 脚本及发行版实现，不能只凭编号认定业务已经准备完成。

systemd 不使用一套全局的 S 编号，而是根据 unit 的依赖和 `After=`、`Before=` 等约束调度任务。因此需要区分三种视图：

| 想看什么 | 命令 | 如何理解 |
| --- | --- | --- |
| 默认启动目标依赖哪些 unit | `systemctl list-dependencies --all default.target` | 递归展开依赖树，包括非 target 单元 |
| 某个服务依赖哪些 unit | `systemctl list-dependencies --all example.service` | 查看该服务的依赖，不是执行步骤清单 |
| 哪些已加载 unit 依赖它 | `systemctl list-dependencies --all --reverse example.service` | 反向查看依赖关系 |
| 某个服务的前后排序约束 | `systemctl show example.service -p After -p Before` | 显示管理器解析后的排序关系；不等于这些 unit 都会启动 |
| 某个服务的定义和覆盖配置 | `systemctl cat example.service` | 查看显式依赖声明；默认和隐式依赖还可能由管理器添加 |
| 实际启动中的时间关键链 | `systemd-analyze critical-chain` | 查看通向默认目标的时间关键链，不是全部服务清单 |
| 指定服务的时间关键链 | `systemd-analyze critical-chain example.service` | 查看影响它启动时间的链条 |
| 各 unit 的激活耗时 | `systemd-analyze blame` | 按耗时排序，不表示启动先后，也不等于对总启动时间的贡献 |
| 实际启动时间轴 | `systemd-analyze plot > boot.svg` | 生成 SVG，查看各 unit 的启动时间和重叠区间 |

**依赖树不是严格的启动执行顺序。** 树中同层或不同分支的 unit 可能并行启动；`Wants=` 和 `Requires=` 说明拉起关系，`After=` 和 `Before=` 才表达相应排序约束。即使有排序约束，应用何时真正可用还取决于服务类型和就绪通知。

例如，假设 web 的 unit 同时声明 `Requires=database.service` 和 `After=database.service`：

```text
依赖与排序：
web 需要拉起 database，并排在 database 的启动步骤之后

可能的实际时间轴：
database  [----------]
日志服务  [----]                 ← 与 database 没有排序要求时可以并行
web                  [-----]
```

`list-dependencies` 主要根据 systemd 当前加载的 unit 展示关系；`--all` 表示递归展开非 target 单元，不代表扫描磁盘上所有已安装 unit 并给出一张完整启动总表。耗时工具也有测量边界：不同服务类型的启动完成定义不同，尤其短启动阶段或后台异步工作不一定在耗时图中完整反映。

如果你记得的是 `pstree`，它在两种 init 系统上都可以查看，但显示的是**进程父子树**，与服务依赖树、启动时间轴分别是不同信息。命令含义可参考 [systemctl 手册](https://www.freedesktop.org/software/systemd/man/latest/systemctl.html)和 [systemd-analyze 手册](https://www.freedesktop.org/software/systemd/man/latest/systemd-analyze.html)。

### 5.4 从 init.d 脚本到 service 文件：怎样启动具体程序？

SysV 的服务脚本自行解释 `start`、`stop` 等参数，通常还处理后台化、PID 文件等工作。systemd 的 service 文件描述启动方式和进程生命周期，由服务管理器执行和跟踪。

常见配置目录如下：

| 目录 | 用途 |
| --- | --- |
| `/usr/lib/systemd/system/` 或 `/lib/systemd/system/` | 软件包提供的 unit |
| `/etc/systemd/system/` | 管理员创建的 unit、覆盖配置和启用链接 |
| `/run/systemd/system/` | 运行时配置，通常不跨重启保留 |

对于这些目录中的同名主文件，通常 `/etc` 优先于 `/run`，再优先于软件包目录；完整搜索路径还包括生成器等目录。可以用 `systemctl cat example.service` 查看主文件和 drop-in 配置，使用 `systemctl edit example.service` 为已有服务添加覆盖，避免直接修改软件包文件。

下面是一个自定义常驻服务示例，假设 `/usr/local/bin/example-server` 是已安装、以前台方式运行的程序。创建 `/etc/systemd/system/example.service`：

```ini
[Unit]
Description=Example server
After=network.target

[Service]
Type=exec
ExecStart=/usr/local/bin/example-server
Restart=on-failure
RestartSec=3

[Install]
WantedBy=multi-user.target
```

- `[Unit]`：服务说明及依赖、排序关系。
- `Type=exec`：程序成功执行后认为启动步骤完成，不代表业务已就绪；程序应保持前台运行，不要额外加 `&`。
- `ExecStart=`：指定要执行的程序和参数，默认不会经过 shell 解释管道、重定向等语法。
- `Restart=on-failure`：异常退出等失败情况下尝试重启，仍受启动频率限制；不是直接照搬 `respawn`。
- `[Install]` 的 `WantedBy=`：告诉 `enable` 应向哪个 target 添加依赖链接，单独写这一行并不会启用或启动服务。

保存后，在目标 Linux 系统上执行：

```sh
sudo systemctl daemon-reload          # 重读新增或修改的 unit 定义
sudo systemctl enable example.service # 建立随目标启动的链接
sudo systemctl start example.service  # 立即启动
systemctl status example.service
journalctl -b -u example.service
```

`enable` 与 `start` 是两件事，可以用 `systemctl enable --now example.service` 同时完成。常驻服务也不都采用 `Type=exec`：支持就绪通知的程序可以使用 `Type=notify`；传统自行后台化的程序可能使用 `Type=forking`，应遵循该程序的要求。

若只是运行一次开机命令，可以使用 `Type=oneshot`，例如 `ExecStart=/usr/bin/echo Boot note`。未设置 `RemainAfterExit=yes` 时，成功执行后回到 inactive 状态属于正常情况。

### 5.5 常用管理动作怎么对应？

| SysV 中的操作或习惯 | systemd 中对应的操作 |
| --- | --- |
| `service example start` 或 init.d 脚本的 `start` | `systemctl start example.service` |
| 停止、重启、查看状态 | `systemctl stop/restart/status example.service`（分别执行对应子命令） |
| `chkconfig`、`update-rc.d` 等配置开机启动 | `systemctl enable/disable example.service`（按需选择子命令） |
| 修改启动脚本后重新使用 | 修改 unit 后通常先 `systemctl daemon-reload`；运行中的进程不会因此自动重启 |
| 要求服务重读自己的配置 | 服务支持时执行 `systemctl reload example.service`，与 `daemon-reload` 不同 |
| 查看服务启动输出 | `journalctl -b -u example.service`，应用也可能另外写日志文件 |

`disable` 只移除启用链接，不会自动停止运行中的服务，也不阻止它被其他依赖或手动启动。`mask` 则用于阻止常规方式启动该 unit，含义更强。

### 5.6 getty 和 /etc/inittab 的 respawn 条目去了哪里？

你熟悉的 SysV 条目可能是：

```text
1:2345:respawn:/sbin/agetty --noclear tty1 linux
```

systemd 通常使用模板 `getty@.service`，具体终端实例是 `getty@tty1.service`。串口控制台通常使用 `serial-getty@.service`。这些服务结合 `getty.target`、生成器和发行版策略启动，自动虚拟终端还可能由 logind 按需管理。

```sh
systemctl cat getty@tty1.service
systemctl status getty@tty1.service
```

实际的 `ExecStart=`、`Restart=` 等设置以本机 unit 为准。登录提示、认证和 shell 仍由 getty、login 及用户配置等组件完成，不是由 systemd 自己提供一个 shell。

### 5.7 旧的 init.d 脚本和 rc.local 还会执行吗？

**不读取 `/etc/inittab`，不等于所有旧脚本必然失效。** 部分 systemd 版本及发行版提供 SysV 兼容生成器，把 `/etc/init.d/` 脚本包装为临时 service；兼容支持正在随版本和发行版变化，不能假设所有系统都有。可以通过 `systemctl cat example.service` 查看本机服务究竟来自原生 unit 还是生成的包装。

`rc.local` 同样是可选兼容机制：提供支持的系统通常由 `rc-local.service` 按执行条件运行 `/etc/rc.local` 或 `/etc/rc.d/rc.local`。它不保证排在所有服务之后，也不是 systemd 必经的最后阶段。用 `systemctl cat rc-local.service` 查看本机规则。

新增服务应使用原生 unit；已有 SysV 脚本是否仍能运行，则检查本机兼容机制。systemd 启动依赖及目标定义可参考 [systemd Bootup](https://www.freedesktop.org/software/systemd/man/latest/bootup.html)和 [systemd.special](https://www.freedesktop.org/software/systemd/man/latest/systemd.special.html)。

### 5.8 登录时运行哪些脚本？换成 systemd 后变了吗？

**`/etc/profile`、`~/.bash_profile`、`~/.bashrc` 等文件仍然由 shell 按自己的规则读取，并没有被 systemd unit 取代。** 从 SysV 换到 systemd，改变的是服务和会话的组织方式；登录脚本具体读哪些，主要取决于使用哪个 shell，以及它是否是登录式、交互式 shell。

以本地文本终端登录、使用 Bash 为例：

```text
systemd 启动 getty@tty1.service
  → agetty 提供登录提示，启动 login
  → login 完成认证与会话设置（通常通过 PAM）
  → 启动用户配置的登录 shell，例如 Bash
  → Bash 读取自己的启动文件
  → 出现命令提示符
```

这里 systemd 负责组织 getty 服务，login/PAM 负责登录相关处理，Bash 负责读取 shell 启动脚本。SysV 下最后这部分职责划分也基本相同。

#### Bash 启动文件的读取顺序

以下以 Bash 按 `bash` 名称、使用正常默认选项启动为前提；以 `sh` 名称启动或显式禁用启动文件时，规则不同。

| 场景 | Bash 读取什么 | 常见例子 |
| --- | --- | --- |
| 交互式登录 shell | 先读 `/etc/profile`，再按顺序查找 `~/.bash_profile`、`~/.bash_login`、`~/.profile`，只读取第一个存在且可读的用户文件 | 文本终端登录、通常的 SSH 交互登录、`bash --login` |
| 交互式非登录 shell | 读取 `~/.bashrc`；某些发行版另有系统级 bashrc 规则 | 桌面中打开普通终端、在 shell 中再运行 `bash` |
| 普通非交互式 Bash | 不走上述常规登录文件链；设置了 `BASH_ENV` 时读取它指定的文件 | `bash script.sh`、`bash -c '命令'` |
| 登录 Bash 退出 | 通常读取 `~/.bash_logout`（若存在） | 退出交互式登录 Bash |

**`~/.bash_profile`、`~/.bash_login`、`~/.profile` 不是依次全部执行。** 例如存在可读的 `~/.bash_profile` 时，Bash 不会再自动读取 `~/.profile`。登录 Bash 也不会仅因为文件存在就自动读取 `~/.bashrc`，常见做法是由用户的 profile 显式加载：

```sh
# ~/.bash_profile 中的常见内容
if [ -f ~/.bashrc ]; then
    . ~/.bashrc
fi
```

发行版还会通过这些文件组织系统配置：

| 文件或目录 | 通常由谁读取 | 常见用途 |
| --- | --- | --- |
| `/etc/profile` | 登录 shell | 系统范围的登录环境设置 |
| `/etc/profile.d/*.sh` | 通常由 `/etc/profile` 中的代码加载 | 软件包拆分存放的登录环境设置；Bash 不会自动扫描这个目录 |
| `/etc/bash.bashrc` 或 `/etc/bashrc` | 发行版的 Bash 启动规则或 profile、bashrc 中的加载代码 | 系统范围的交互式 Bash 设置；路径和调用链因发行版而异 |
| `~/.bash_profile`、`~/.profile` 等 | 对应 shell 的登录启动规则 | 用户登录环境设置 |
| `~/.bashrc` | 交互式非登录 Bash，或其他文件显式加载 | 别名、提示符、交互功能等 |

其他 shell 如 Zsh、Fish 使用不同的启动文件。SSH 远程执行命令还存在 Bash 针对远程 shell 的特殊启动规则，不能简单认为所有 SSH 操作都执行登录 profile。

#### PAM 和 systemd 用户服务有什么变化？

这里的 SSH 服务程序叫 **`sshd`**。`login` 与 `sshd` 是两种登录入口，通常不是“sshd 调用 login”的上下级关系。它们可以分别调用 PAM，复用认证、账号检查和会话设置机制。

**本地文本终端：getty → login → shell。**

```text
systemd（或 SysV init）
  → 启动 agetty，管理 tty1 等终端并显示登录提示
  → agetty 取得用户名，执行 login 程序
  → login 调用 PAM，使用 /etc/pam.d/login 配置
  → 认证、账号检查及会话准备完成
  → 以目标用户身份启动其登录 shell
  → shell 按自身规则读取 profile 等文件
```

agetty 主要负责终端初始化、提示与转交，通常不是由它检查密码；login 才是本地文本登录中调用 PAM 的程序。getty 的 systemd unit 负责启动和管理这个终端登录入口，不负责直接解析 shell 的 profile 文件。

**SSH 远程登录：sshd → shell，不经过 getty，通常也不执行 login 程序。**

```text
SSH 客户端通过网络连接
  → sshd 及其会话处理进程完成 SSH 协议和用户认证
  → 若启用 PAM，则使用 /etc/pam.d/sshd 等相应 PAM 服务配置
  → 完成账号检查和会话准备
  → 按请求分配伪终端，启动用户 shell、命令或子系统
```

sshd 已经承担了远程登录的认证与会话组织工作，所以通常不需要先运行 `/bin/login`。SSH 分配的伪终端（例如 `/dev/pts/0`）也不需要 agetty 来提示用户名。上图表达职责顺序，不是固定的进程父子树；OpenSSH 的进程拆分方式随版本和配置而异。

| 组件 | 职责 | 常见配置位置 |
| --- | --- | --- |
| agetty | 本地或串口终端初始化、登录提示、执行 login | `getty@.service`、`serial-getty@.service` 及其覆盖配置 |
| login | 本地文本登录认证与会话建立，然后启动用户 shell | `/etc/pam.d/login`；其他登录设置因实现而异 |
| sshd | SSH 网络协议、远程认证和会话处理 | `/etc/ssh/sshd_config` 及其 `Include` 文件；PAM 通常使用 `/etc/pam.d/sshd` |
| PAM | 供登录程序调用的模块化认证、账号和会话接口，不是独立登录守护进程 | `/etc/pam.d/` 中的服务配置及其包含的公共配置 |
| 用户 shell | 运行交互命令，读取自己的启动文件 | Bash 的 profile、bashrc 等 |

Linux 上使用支持 PAM 的 OpenSSH 时，`UsePAM yes` 启用相应 PAM 处理。密码或键盘交互认证可使用 PAM；**公钥认证通常由 OpenSSH 完成，但仍可以继续执行 PAM 的账号检查和会话模块**。所以“使用密钥登录”不等于“不经过 PAM”。实际流程还受认证方法及 PAM 配置影响。

PAM 配置中的 `auth` 负责认证，`account` 负责账号使用条件检查，`session` 负责会话打开、关闭等处理，`password` 负责认证令牌更新。它们可能包含公共配置，例如 Debian/Ubuntu 的 `common-*`，或其他发行版的 `system-auth`、`password-auth` 等，并不是每次登录都会执行所有类别。**`/etc/environment` 是环境变量配置，不是 shell 脚本**，某些系统通过 PAM 等组件读取，不能在里面写任意 shell 命令。

systemd 系统常通过 `pam_systemd` 将会话注册到 logind，并按策略启动或接入用户管理器 `systemd --user`，对应系统服务实例 `user@<UID>.service`。它与用户的登录 shell 是不同进程，通常由同一用户的多个会话共享；不是每打开一个终端就新建一个用户管理器。

也就是说，本地 login 和远程 sshd 可以通过各自 PAM 配置最终包含的 `pam_systemd.so` 接入同一套会话管理机制。`systemd-logind` 管理会话信息，`systemd --user` 管理用户服务，二者都不替代 login/sshd 的全部职责，也不负责读取 `.bashrc`。

```text
本地：agetty → login → PAM 的 login 配置 ─┐
                                       ├→ pam_systemd → logind / 用户管理器
远程：sshd → PAM 的 sshd 配置 ──────────┘
```

这张图表示调用和接入关系，不表示 shell 是由 logind 或用户管理器创建的。是否启用该模块以及创建何种会话，取决于本机 PAM 栈、会话类别和 systemd 配置。

终端与 login 的关系见 [agetty 手册](https://man7.org/linux/man-pages/man8/agetty.8.html)；Linux OpenSSH 的 PAM 行为见 [sshd_config 手册的 UsePAM 说明](https://manpages.debian.org/unstable/openssh-server/sshd_config.5.en.html)。

| 想配置的内容 | 常见位置或方式 |
| --- | --- |
| Bash 的别名、提示符等交互设置 | `~/.bashrc` |
| 登录 shell 的环境设置 | `~/.bash_profile` 或 `~/.profile`，按实际 shell 规则选择 |
| 用户后台服务 | `~/.config/systemd/user/*.service`，使用 `systemctl --user` 管理 |
| 图形桌面登录后自动启动应用 | 桌面的自动启动机制，例如 `~/.config/autostart/*.desktop`，具体以桌面实现为准 |

用户服务不是通过登录 Bash 启动的，因此不能假定它自动获得 `~/.bashrc` 中的变量，也不应在 `.bashrc` 中启动希望每用户只运行一份的守护进程。图形登录也不保证执行某一套 Bash 登录文件，其环境设置取决于显示管理器和桌面会话；打开终端后则再按终端启动的 shell 类型读取配置。

可以在当前 Bash 中查看：

```sh
shopt -q login_shell && echo "登录 shell" || echo "非登录 shell"
case $- in *i*) echo "交互式 shell" ;; *) echo "非交互式 shell" ;; esac
ps -p $$ -o pid,ppid,args
systemctl --user status  # 查看当前用户管理器状态，需要用户会话环境可用
```

Bash 的文件读取规则见 [GNU Bash 启动文件文档](https://www.gnu.org/software/bash/manual/html_node/Bash-Startup-Files)；systemd 的会话衔接见 [pam_systemd 文档](https://www.freedesktop.org/software/systemd/man/latest/pam_systemd.html)。

## 6. 在运行中的 Linux 上核对启动过程

以下命令供目标 Linux 系统使用，未在当前工作环境中执行；部分工具需要安装或相应权限。

```sh
# 固件启动方式与启动项
test -d /sys/firmware/efi && echo "本次通过 UEFI 启动"
sudo efibootmgr -v

# 分区、文件系统标识和挂载关系
lsblk -o NAME,FSTYPE,UUID,PARTUUID,MOUNTPOINTS
findmnt -T /boot/efi
findmnt -T /boot
findmnt /

# 实际传入的启动参数与内核文件
cat /proc/cmdline
ls -l /boot/vmlinuz* /boot/init*

# init 入口与当前 PID 1
ls -l /sbin/init
readlink -f /sbin/init
ps -p 1 -o pid,ppid,comm,args
readlink -f /proc/1/exe

# systemd 配置及本次启动日志
systemctl get-default
systemctl cat example.service
journalctl -b
```

`/proc/cmdline` 记录本次启动实际收到的内核参数，重新生成 GRUB 配置不会改变它；`/proc/1/exe` 指向当前 PID 1 的实际程序。内核和 init 通常是二进制文件，查看其配置和运行状态比直接 `cat` 程序文件更有意义。

## 参考资料

- [UEFI Boot Manager 规范](https://uefi.org/specs/UEFI/2.11/03_Boot_Manager.html)：启动变量、设备路径和默认启动行为。
- [GNU GRUB 配置文档](https://www.gnu.org/software/grub/manual/grub/html_node/Simple-configuration.html)：配置生成及内核参数。
- [GRUB search 指令](https://www.gnu.org/software/grub/manual/grub/html_node/search.html)：按文件系统 UUID 查找设备。
- [Linux 内核文档：initramfs](https://docs.kernel.org/filesystems/ramfs-rootfs-initramfs.html)：早期用户空间机制。
- [Linux 内核源码 init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c)：init 程序的执行逻辑。
- [Linux 内核命令行参数](https://docs.kernel.org/admin-guide/kernel-parameters.html)：`root=`、`init=` 等参数。
- [systemd Bootup](https://www.freedesktop.org/software/systemd/man/latest/bootup.html)：启动目标及阶段关系。
- [systemd.unit](https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html)：unit 依赖和配置。
- [SysV init 的 inittab 手册](https://github.com/slicer69/sysvinit/blob/main/man/inittab.5)：传统 init 配置语法。
