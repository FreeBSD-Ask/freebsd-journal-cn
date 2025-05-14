# 字符设备驱动教程（第三部分）

- [Character Device Driver Tutorial (Part 3)](https://freebsdfoundation.org/our-work/journal/browser-based-edition/downstreams/character-device-driver-tutorial-part-3)
- 作者：John Baldwin

在第 1 部分和第 2 部分中，我们实现了一款简单的字符设备驱动程序，该驱动程序支持了基本的 I/O 操作。在本系列的最后一篇文章中，我们将探讨字符设备如何为用户进程中的内存映射提供后备存储。与上一篇文章不同，我们不会扩展回显设备驱动程序，而是将实现新的驱动程序来演示内存映射。可以在与回显驱动程序相同的仓库中找到这些驱动程序，网址为 <https://github.com/bsdjhb/cdev_tutorial>。

### FreeBSD 中的内存映射

要理解字符设备中内存映射的工作原理，首先必须了解 FreeBSD 内核如何管理内存映射。FreeBSD 的虚拟内存子系统源自于 Mach 虚拟内存子系统，后者继承自 4.4BSD。虽然 FreeBSD 的虚拟内存（VM）在过去三十年里经历了重大的变化，但核心抽象依然保持不变。

在 FreeBSD 中，虚拟内存地址空间由虚拟内存映射（`struct vm_map`）表示。一个虚拟内存映射包含一组条目（`struct vm_map_entry`）。每个条目定义了一个连续地址空间范围的属性，包括权限和后备存储。虚拟内存对象（`struct vm_object`）用于描述映射的后备存储。一个虚拟内存对象拥有自己的逻辑地址空间页面。例如，磁盘上的每个常规文件都与一个虚拟内存对象相关联，其中虚拟内存对象中的页面逻辑地址对应文件中的偏移量，而逻辑页面的内容则是文件中给定偏移量处的文件内容。每个虚拟内存映射条目将其后备存储标识为从单个虚拟内存对象的特定偏移量开始的一系列逻辑连续页面。图 1 展示了如何使用单个虚拟内存映射条目将 C 运行时库的 `.data` 部分映射到进程的地址空间中。

**图 1：C 运行时库 `.data` 部分的映射**

![image](https://github.com/user-attachments/assets/ee1ef154-9fea-40d8-a829-48f41cc80dff)

每个虚拟内存对象（VM object）都与一个分页器（pager）相关联，分页器提供一组用于确定与虚拟内存对象关联的页面内容的函数。vnode 分页器用于与常规文件相关联的虚拟内存对象，这些文件来自块存储文件系统和网络文件系统。其函数从关联的文件中读取数据以初始化页面，并将修改后的页面写回关联的文件。交换分页器用于与常规文件无关的匿名虚拟内存对象。在首次使用时，系统会为这些对象分配填充为零的页面。如果系统内存不足，交换分页器会将使用较少的脏页面写入交换分区，直至它们再次被需要。

虚拟内存对象中的逻辑页面由虚拟内存页面（`struct vm_page`）表示。在启动时，内核分配了一个虚拟内存页面数组，使得每个物理内存页面都与一个虚拟内存页面对象相关联。虚拟内存页面通过使用特定架构的页表项（PTE）映射到地址空间中。受管理的虚拟内存页面通过使用特定架构的结构体（称为 PV 条目）维护一个映射链表。这个链表可以用于通过使关联的页表项无效来移除虚拟内存页面的所有映射，从而使虚拟内存页面能够重新用于表示不同的逻辑页面，不论是为另一个虚拟内存对象，还是为同一虚拟内存对象中的不同逻辑页面地址。

每次调用系统调用 [mmap(2)](https://man.freebsd.org/mmap/2) 时，都会在调用进程中创建一个新的虚拟内存映射条目。系统调用的参数提供了新条目的各种属性，包括权限、长度和偏移量，后者指向虚拟内存对象中的位置。文件描述符参数用于标识要映射到调用进程地址空间中的虚拟内存对象。为了映射字符设备的内存，进程将字符设备的打开文件描述符作为文件描述符参数传递给系统调用 `mmap()`。字符设备驱动程序的角色是决定哪个虚拟内存对象用于满足内存映射请求，并决定支持虚拟内存对象的页面内容。

## 默认字符设备分页器

4.4BSD 内置了一款设备虚拟内存（VM）分页器，用于支持字符设备内存映射。这个设备分页器旨在映射在操作系统运行时不会改变的物理内存区域。例如，它可以直接将 MMIO 区域（如帧缓冲区）暴露给用户空间。

设备分页器假设每个设备虚拟内存对象中的页面都映射到一个物理地址空间的页面。这个页面可以是个 RAM 页面，也可以与 MMIO 区域关联。重要的是，只要设备虚拟内存对象中的逻辑地址与物理页面关联，该映射就不能被改变。这个假设是双向的，因为设备分页器也假设只要物理地址空间中的页面与设备虚拟内存对象关联，该物理页面就永远不能被用于其他用途。因此，设备分页器使用的虚拟内存页面是无管理的（没有 PV 条目）。然而，这也意味着虚拟内存系统无法轻易找到这些虚拟内存页面的现有映射，以撤销现有的映射。尤其是通过 [`destroy_dev(9)`](https://man.freebsd.org/destroy_dev/9) 销毁字符设备并不会撤销现有的映射。

默认字符设备分页器使用字符设备的 `mmap` 方法来验证映射请求，并确定与每个逻辑页面地址关联的物理地址。`mmap` 方法应验证偏移量和保护参数。如果偏移量不是有效的逻辑页面地址，或者请求的保护不被支持，方法应通过返回错误代码来失败。否则，方法应将请求的偏移量的物理地址存储到物理地址参数中，并返回零。如果页面应以 `VM_MEMATTR_DEFAULT` 以外的内存属性进行映射，成功时也应返回该内存属性。当创建映射时，设备分页器在请求的每个逻辑页面地址上调用此方法以验证请求。对于逻辑页面地址的首次页面错误，设备分页器调用 `mmap` 方法以获取后备页面的物理地址和内存属性。

列表 1 显示了一个简单字符设备驱动程序的 `mmap` 方法，该驱动程序使用默认的设备分页器。此设备在加载时分配一个单独的 RAM 页面，并将该页面的指针保存在 `si_drv1` 字段中。由于字符设备分页器的限制，此驱动程序无法卸载。示例 1 展示了设备加载后的一些交互，使用 `maprw` 测试程序来读取和写入设备映射。

**列表 1：使用默认设备分页器**

```c
static int
mappage_mmap(struct cdev *dev, vm_ooffset_t offset, vm_paddr_t *paddr,
    int nprot, vm_memattr_t *memattr)
{
      if (offset != 0)
            return (EINVAL);

      *paddr = pmap_kextract((uintptr_t)dev->si_drv1);
      return (0);
}
```

**示例 1：使用 /dev/mappage 设备**

```c
# maprw read /dev/mappage 16 | hexdump
0000000 0000 0000 0000 0000 0000 0000 0000 0000
0000010
# jot -c -s "" 16 'A' | maprw write /dev/mappage 16
# maprw read /dev/mappage 16
ABCDEFGHIJKLMNOP
```

## 映射任意虚拟内存对象

由于默认字符设备分页器的限制，FreeBSD 扩展了对字符设备内存映射的支持。FreeBSD 8.0 引入了新的字符设备方法 `mmap_single`。每次在调用映射字符设备 `mmap()` 时，都会调用此方法。`mmap_single` 方法必须验证整个 `mmap()` 请求，包括偏移量、大小和请求的保护。如果请求有效，该方法应返回一个虚拟内存对象（VM object）的引用，以供映射使用。该方法可以创建一个新的虚拟内存对象，也可以返回对现有虚拟内存对象的额外引用。如果 `mmap_single` 方法返回 `ENODEV` 错误（默认行为），`mmap()` 将使用默认字符设备分页器。

`mmap_single` 方法还可以在返回虚拟内存对象时修改用于映射的偏移量（但不能修改大小）。这能让字符设备使用映射的初始偏移量作为标识特定虚拟内存对象的键。比如，驱动程序可能有两个内部虚拟内存对象，使用偏移量 0 映射第一个虚拟内存对象，并使用 `PAGE_SIZE` 的偏移量映射第二个虚拟内存对象。在第二种情况下，`mmap_single` 方法将重置有效偏移量为 0，以便生成的映射从第二个虚拟内存对象的开头开始。

然而，字符设备不一定需要使用多个虚拟内存对象来受益于 `mmap_single` 方法。使用其他分页器的虚拟内存对象的能力可能很有用。例如，物理分页器创建由物理 RAM 中的固定页面支持的虚拟内存对象。与默认设备分页器不同，这些页面是受管理的，可以在虚拟内存对象被销毁时安全地释放。列表 2 更新了先前的 `mappage` 设备驱动程序，改为使用物理分页器虚拟内存对象，而不是默认的字符设备分页器。此版本的设备驱动程序可以安全卸载，因为虚拟内存对象将在驱动程序卸载后继续存在，直到所有映射被销毁。

**列表 2：使用物理分页器**

```c
static int
mappage_mmap_single(struct cdev *cdev, vm_ooffset_t *offset, vm_size_t size,
    struct vm_object **object, int nprot)
{
      vm_object_t obj;

      obj = cdev->si_drv1;
      if (OFF_TO_IDX(round_page(*offset + size)) > obj->size)
            return (EINVAL);

      vm_object_reference(obj);
      *object = obj;
      return (0);
}

static int
mappage_create(struct cdev **cdevp)
{
      struct make_dev_args args;
      vm_object_t obj;
      int error;

      obj = vm_pager_allocate(OBJT_PHYS, NULL, PAGE_SIZE,
          VM_PROT_DEFAULT, 0, NULL);
      if (obj == NULL)
            return (ENOMEM);
      make_dev_args_init(&args);
      args.mda_flags = MAKEDEV_WAITOK | MAKEDEV_CHECKNAME;
      args.mda_devsw = &mappage_cdevsw;
      args.mda_uid = UID_ROOT;
      args.mda_gid = GID_WHEEL;
      args.mda_mode = 0600;
args.mda_si_drv1 = obj;
      error = make_dev_s(&args, cdevp, "mappage");
      if (error != 0) {
            vm_object_deallocate(obj);
            return (error);
      }
      return (0);
}

static void
mappage_destroy(struct cdev *cdev)
{
      if (cdev == NULL)
            return;

      vm_object_deallocate(cdev->si_drv1);
      destroy_dev(cdev);
}
```

## 每个打开的状态

在本系列的开篇文章中，我们演示了如何使用 `si_drv1` 字段支持每个实例的数据。某些字符设备驱动程序需要为每个打开的文件描述符维护独特的状态。也就是说，如果一个字符设备被多次打开，驱动程序希望对每个打开的引用提供不同的行为。

FreeBSD 通过一系列函数提供了这个功能。通常，字符设备驱动程序会在 `open` 方法中创建每个打开状态的新实例，并通过调用 [devfs_set_cdevpriv(9)](https://man.freebsd.org/devfs_set_cdevpriv/9) 将该实例与新的文件描述符关联。此函数接受一个 `void` 指针参数和一个析构函数回调。析构函数在文件描述符的最后一个引用被关闭时调用，用于清理每个打开状态。其他字符设备开关方法会调用 [devfs_get_cdevpriv(9)](https://man.freebsd.org/devfs_get_cdevpriv/9) 来检索与当前文件描述符关联的 `void` 指针。请注意，这些函数始终在由调用者上下文隐式确定的当前文件描述符上操作，驱动程序不会显式传递文件描述符的引用给这些函数。

列表 3 展示了一个新的字符设备驱动程序 `memfd` 的 `open` 和 `mmap_single` 方法，以及析构函数 `cdevpriv`。这个简单的驱动程序提供了类似于 FreeBSD [shm_open(2)](https://man.freebsd.org/shm_open/2) 实现中 `SHM_ANON` 扩展的功能。这个设备的每个打开的文件描述符都与一个匿名的虚拟内存对象（VM object）关联。当映射时，虚拟内存对象的大小会在必要时增长。虚拟内存对象可以通过共享文件描述符与其他进程共享，例如，通过在 UNIX 域套接字上传递文件描述符来实现。为了实现这一点，驱动程序在 `open` 方法中分配一个新的虚拟内存对象，并将该虚拟内存对象与新的文件描述符关联。`mmap_single` 方法获取当前文件描述符的虚拟内存对象，必要时增长它，并返回对该对象的引用。最后，析构函数会删除文件描述符对虚拟内存对象的引用。

**列表 3：每个打开的匿名内存**

```c
static int
memfd_open(struct cdev *cdev, int fflag, int devtype, struct thread *td)
{
      vm_object_t obj;
      int error;

      /* 只读和只写的打开方式没有意义。 */
      if ((fflag & (FREAD | FWRITE)) != (FREAD | FWRITE))
            return (EINVAL);

/*
 * 为每一个打开的文件描述符创建一个初始大小为 0 的匿名 VM 对象。
 */

      obj = vm_object_allocate_anon(0, NULL, td->td_ucred, 0);
      if (obj == NULL)
            return (ENOMEM);
      error = devfs_set_cdevpriv(obj, memfd_dtor);
      if (error != 0)
              vm_object_deallocate(obj);
      return (error);

}

static void
memfd_dtor(void *arg)
{
      vm_object_t obj = arg;

      vm_object_deallocate(obj);
}

static int
memfd_mmap_single(struct cdev *cdev, vm_ooffset_t *offset, vm_size_t size,
    struct vm_object **object, int nprot)
{
      vm_object_t obj;
      vm_pindex_t objsize;
      vm_ooffset_t delta;
      void *priv;
      int error;

      error = devfs_get_cdevpriv(&priv);
      if (error != 0)
            return (error);
      obj = priv;

/* 如有必要，扩展对象。 */
      objsize = OFF_TO_IDX(round_page(*offset + size));
      VM_OBJECT_WLOCK(obj);
      if (objsize > obj->size) {
            delta = IDX_TO_OFF(objsize - obj->size);
            if (!swap_reserve_by_cred(delta, obj->cred)) {
                 VM_OBJECT_WUNLOCK(obj);
                  return (ENOMEM);
            }
            obj->size = objsize;
            obj->charge += delta;
      }

      vm_object_reference_locked(obj);
      VM_OBJECT_WUNLOCK(obj);
      *object = obj;
      return (0);
}
```

## 扩展字符设备分页器

`mmap_single` 方法通过允许字符设备使用由任何分页器支持的虚拟内存对象（VM objects），并允许字符设备将不同的虚拟内存对象与不同的偏移量关联，从而缓解了默认字符设备分页器的一些限制。然而，也还存在一些限制。设备分页器在所有分页器中是独特的，因为它可以映射与物理 RAM 无关的物理地址，例如 MMIO 区域。由于使用了未管理的页面，因此无法撤销设备分页器的映射，也无法让驱动程序知道所有映射是否已被移除。FreeBSD 9.1 引入了一个新的设备分页器接口，提供了针对这两个问题的解决方案。

新的接口要求字符设备驱动程序显式地创建设备虚拟内存对象。这些虚拟内存对象随后由 `mmap_single` 方法使用，以提供映射的支持存储。在新的接口中，`mmap` 字符设备方法被一个新的方法结构体（`struct cdev_pager_ops`）所替代。该结构体包含以下方法：当虚拟内存对象被创建时调用的 `cdev_pg_ctor`，当发生页面故障并请求虚拟内存对象的页面时调用的 `cdev_pg_fault`，以及虚拟内存对象被销毁时调用的 `cdev_pg_dtor`。使用扩展设备分页器的虚拟内存对象通过调用 `cdev_pager_allocate()` 来创建。该函数的第一个参数是一个存储在新虚拟内存对象的句柄成员中的不透明指针。该指针还作为第一个参数传递给构造函数和析构函数分页器方法。`cdev_pager_allocate()` 的第二个参数是对象类型，可以是 `OBJT_DEVICE` 或 `OBJT_MGTDEVICE`。第三个参数是指向 `struct cdev_pager_ops` 实例的指针。

`cdev_pager_allocate()` 函数每次只为每个不透明指针创建一个虚拟内存对象。如果相同的不透明指针被传递给 `cdev_pager_allocate()` 的后续调用，函数将返回指向现有虚拟内存对象的指针，而不是创建一个新的虚拟内存对象。在这种情况下，虚拟内存对象的引用计数会增加，因此 `cdev_pager_allocate()` 总是返回指向返回的虚拟内存对象的新引用。

让我们利用这个接口扩展原始版本的 `mappage` 驱动程序（来自列表 1），使其在没有活动映射的情况下可以安全卸载。在该例中，我们将使用 `OBJT_DEVICE` 虚拟内存对象。这仍然使用驱动程序加载时分配的单个固定页面的未管理映射。然而，现在需要额外的状态来确定该分配的页面是否正在使用，因此这个版本的驱动程序定义了一个 `softc` 结构，包含指向页面的指针、一个布尔变量用于跟踪页面是否处于活动映射状态、一个布尔变量用于跟踪驱动程序是否正在卸载（在这种情况下，不允许新的映射），以及一个互斥锁来保护对布尔变量的访问。指向 `softc` 结构的指针存储在字符设备的 `si_drv1` 字段中，并作为虚拟内存对象的不透明句柄使用。`mmap_single` 字符设备方法验证每个映射请求（包括在卸载待处理时失败的请求），并调用 `cdev_pager_allocate()` 获取指向映射固定页面的虚拟内存对象的引用。请注意，`mmap_single` 方法不需要单独处理创建新虚拟内存对象或重用现有虚拟内存对象的情况。构造函数分页器方法将布尔变量 `mapped`（位于 `softc` 中）设置为 `true`。若移除虚拟内存对象的最后一个映射，并且虚拟内存对象被销毁，析构函数分页器方法就会被调用，将 `mapped` 设置为 `false`。如果在请求卸载时 `mapped` 成员为 `true`，则 `mappage_destroy()` 函数会因 `EBUSY` 错误而无法卸载。

页面故障分页器方法比它所替代的 `mmap` 字符设备方法更为复杂。页面故障方法与虚拟内存（VM）系统以及页面故障通常如何被虚拟内存分页器处理的方式更加直接。当发生页面故障时，虚拟内存系统会分配一个空闲的内存页面，并调用分页器方法将该页面填充为适当的内容。交换分页器和物理分页器会在此方法中将新页面填充为零，而 vnode 分页器则从相关联的文件中读取适当的内容。默认的设备分页器采取了不同的路线。由于它通常设计用于映射非 RAM 地址，例如 MMIO 区域，默认设备分页器分配一个与 `mmap` 方法返回的物理地址相关联的“伪”虚拟内存页面，并将虚拟内存系统分配的新虚拟内存页面替换为这个“伪”虚拟内存页面（新的虚拟内存页面会作为空闲页面返回给系统）。页面故障分页器方法通过传入虚拟内存系统分配的新虚拟内存页面的指针，允许驱动程序实现这两种方法中的任意一种。页面故障分页器方法负责要么将该页面填充为适当的内容，要么用一个“伪”虚拟内存页面替换它。对于我们的驱动程序，我们计算固定页面的物理地址与之前相同，但使用该物理地址来构造一个“伪”虚拟内存页面。

列表 4 显示了 `mmap_single` 字符设备方法、三个设备分页器方法以及在模块卸载期间调用的 `mappage_destroy()` 函数。在示例 2 中，我们暂停了 `maprw` 测试程序，当它映射了来自 `mappage` 设备的页面时，尝试卸载驱动程序，但失败。之后恢复测试程序，并让它通过退出来取消映射设备，驱动程序成功卸载。

**列表 4：使用扩展设备分页器**

```c
static struct cdev_pager_ops mappage_cdev_pager_ops = {
      .cdev_pg_ctor = mappage_pager_ctor,
      .cdev_pg_dtor = mappage_pager_dtor,
      .cdev_pg_fault = mappage_pager_fault,
};

static int
mappage_mmap_single(struct cdev *cdev, vm_ooffset_t *offset, vm_size_t size,
    struct vm_object **object, int nprot)
{
      struct mappage_softc *sc = cdev->si_drv1;
      vm_object_t obj;

      if (round_page(*offset + size) > PAGE_SIZE)
            return (EINVAL);

      mtx_lock(&sc->lock);
      if (sc->dying) {
            mtx_unlock(&sc->lock);
            return (ENXIO);
      }
      mtx_unlock(&sc->lock);

      obj = cdev_pager_allocate(sc, OBJT_DEVICE, &mappage_cdev_pager_ops,
          OFF_TO_IDX(PAGE_SIZE), nprot, *offset, curthread->td_ucred);
      if (obj == NULL)
            return (ENXIO);
/*
 * 如果在我们分配 VM 对象时开始卸载，
 * dying 将被设置，卸载线程将会在 destroy_dev() 中等待。
 * 只需释放 VM 对象并失败映射请求。
 */

      mtx_lock(&sc->lock);
      if (sc->dying) {
            mtx_unlock(&sc->lock);
            vm_object_deallocate(obj);
            return (ENXIO);
      }
       mtx_unlock(&sc->lock);

      *object = obj;
      return (0);
}

static int
mappage_pager_ctor(void *handle, vm_ooffset_t size, vm_prot_t prot,
    vm_ooffset_t foff, struct ucred *cred, u_short *color)
{
      struct mappage_softc *sc = handle;

      mtx_lock(&sc->lock);
      sc->mapped = true;
      mtx_unlock(&sc->lock);

      *color = 0;
      return (0);
}

static void
mappage_pager_dtor(void *handle)
{
      struct mappage_softc *sc = handle;

      mtx_lock(&sc->lock);
      sc->mapped = false;
      mtx_unlock(&sc->lock);
}

static int
mappage_pager_fault(vm_object_t object, vm_ooffset_t offset, int prot,
    vm_page_t *mres)
{
      struct mappage_softc *sc = object->handle;
      vm_page_t page;
      vm_paddr_t paddr;

      paddr = pmap_kextract((uintptr_t)sc->page + offset);

      /* 参见 device_pager.c 中 old_dev_pager_fault 的结尾部分。 */
      if (((*mres)->flags & PG_FICTITIOUS) != 0) {
            page = *mres;
            vm_page_updatefake(page, paddr, VM_MEMATTR_DEFAULT);
      } else {
            VM_OBJECT_WUNLOCK(object);
            page = vm_page_getfake(paddr, VM_MEMATTR_DEFAULT);
            VM_OBJECT_WLOCK(object);
            vm_page_replace(page, object, (*mres)->pindex, *mres);
            *mres = page;
      }
      vm_page_valid(page);
      return (VM_PAGER_OK);
}

...

static int
mappage_destroy(struct mappage_softc *sc)
{
      mtx_lock(&sc->lock);
      if (sc->mapped) {
            mtx_unlock(&sc->lock);
            return (EBUSY);
      }
      sc->dying = true;
      mtx_unlock(&sc->lock);

      destroy_dev(sc->dev);
      free(sc->page, M_MAPPAGE);
      mtx_destroy(&sc->lock);
      free(sc, M_MAPPAGE);
      return (0);
}
```

**示例 2：通过扩展设备分页器安全卸载**

```sh
# maprw write /dev/mappage 16
^Z
Suspended
# kldunload mappage
kldunload: can’t unload file: Device busy
# fg
maprw write /dev/mappage 16
maprw: empty read
# kldunload mappage
```

扩展设备分页器接口还增加了一种新的设备分页器类型。`OBJT_MGTDEVICE` 分页器与 `OBJT_DEVICE` 的不同之处在于，它总是使用受管理的页面进行映射，而不是使用未管理的页面。这意味着，即使页面已映射，也可以强制撤销页面的映射。对于映射非 RAM 页面（虚构页面），必须通过 `vm_phys_fictitious_reg_range()` 函数显式创建“伪”虚拟内存页面，然后才能在分页器中使用它们。

## 结论

在本文中，我们探讨了一些字符设备更罕见的使用案例，包括内存映射和每次打开状态。感谢您阅读此系列文章。希望它能为您提供有关 FreeBSD 中字符设备驱动程序的有用介绍。

---

**John Baldwin** 是一位系统软件开发人员。在过去的二十多年里，他在 FreeBSD 操作系统的各个部分（包括 x86 平台支持、SMP、各种设备驱动程序和虚拟内存子系统）以及用户空间程序中直接提交了修改。除了编写代码，John 还曾担任过 FreeBSD 核心团队和发布工程团队的成员。他还为 GDB 调试器做出了贡献。John 目前与妻子 Kimberly 和三个孩子 Janelle、Evan、Bella 一起住在弗吉尼亚州的阿什兰。
