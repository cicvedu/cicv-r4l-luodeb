我的博客：<https://www.luodeb.top/2ed0cc28/>

# 作业1 编译内核

```bash
cd linux
make x86_64_defconfig
make LLVM=1 menuconfig # [*] Rust support
make LLVM=1 -j$(nproc)
ls
```
![](https://imgfiles-debin.oss-cn-hangzhou.aliyuncs.com/md_imgfiles/202311131506517.png)

# 作业2 对Linux内核进行一些配置

``` bash
./build_image.sh
```

这个时候网卡是默认启用的，然后我们禁用网卡以后，再次

```bash
make LLVM=1 menuconfig # [ ] Intel devices, Intel(R) PRO/1000 Gigabit Ethernet support
make LLVM=1 -j$(nproc)
./build_image.sh
insmod r4l_e1000_demo.ko
ip link set eth0 up
ip addr add broadcast 10.0.2.255 dev eth0
ip addr add 10.0.2.15/255.255.255.0 dev eth0 
ip route add default via 10.0.2.1
ping 10.0.2.2
```
![](https://imgfiles-debin.oss-cn-hangzhou.aliyuncs.com/md_imgfiles/202311131513532.png)

成功启用rust驱动，ping通了

 - 编译成内核模块，是在哪个文件中以哪条语句定义的？
```text
Kbuild文件里的obj-m := r4l_e1000_demo.o
```

 - 该模块位于独立的文件夹内，却能编译成Linux内核模块，这叫做out-of-tree module，请分析它是如何与内核代码产生联系的？
```text
OOT模块需要使用内核的头文件来确保与内核代码的一致性。这些头文件包含了内核的数据结构、函数原型和常量定义等，使得OOT模块可以正确地使用内核的API和数据类型。在编译OOT模块时，需要指定内核源代码树的路径，以便编译器能够找到这些头文件。
其次，OOT模块通过调用内核提供的模块加载API（如module_init和module_exit）来注册自己的初始化和清理函数。当模块被加载时，内核会调用这些函数，从而执行OOT模块的初始化和清理操作。这些函数可以与内核的其他部分进行交互，注册驱动程序、添加新的系统调用等。
```

# 作业3 使用rust编写一个简单的内核模块并运行

编辑./linux/samples/rust/rust_helloworld.rs，打印Hello World from Rust module
在Makefile中添加obj-$(CONFIG_SAMPLES_RUST_HELLOWORLD)   += rust_helloworld.o
在Kconfig中添加
```
config SAMPLES_RUST_HELLOWORLD
    tristate "Print Helloworld in Rust"
    depends on RUST
    help
      Say Y here if you want to compile the Rust "Hello, World!"
      example module. Say M if you want to build it as a module.

      If unsure, say N.
```

更改配置文件
```bash
make LLVM=1 menuconfig # <M>Print Helloworld in Rust (NEW)
```
然后使用`make LLVM=1 -j$(nproc)`编译模块
![](https://imgfiles-debin.oss-cn-hangzhou.aliyuncs.com/md_imgfiles/202311131526870.png)

![](https://imgfiles-debin.oss-cn-hangzhou.aliyuncs.com/md_imgfiles/202311131529595.png)

# 作业4 为e1000网卡驱动添加remove代码

1.首先编写释放申请的tx/rx descriptor ring的内存方法
```rust
    /// 释放之前设置的所有TX资源。
    fn e1000_cleanup_tx_resources(data: &NetDevicePrvData) {
        let mut tx_ring_guard = data.tx_ring.lock();
        if let Some(tx_ring) = tx_ring_guard.take() {
            // 释放TX描述符的DMA空间
            drop(tx_ring); 
        }
    }

    /// 释放之前设置的所有RX资源。
    fn e1000_cleanup_rx_resources(data: &NetDevicePrvData) {
        let mut rx_ring_guard = data.rx_ring.lock();
        if let Some(mut rx_ring) = rx_ring_guard.take() {
            // 清理与RX描述符相关联的DMA映射和缓冲区
            for entry in rx_ring.buf.borrow_mut().iter_mut() {
                if let Some((dma_map, skb)) = entry.take() {
                    drop(dma_map); 
                    drop(skb); 
                }
            }
            drop(rx_ring); 
        }
    }
```

在设备down的时候，执行这个方法
```rust
    fn stop(_dev: &net::Device, _data: &NetDevicePrvData) -> Result {
        pr_info!("Rust for linux e1000 driver demo (net device stop)\n");

        Self::e1000_cleanup_tx_resources(_data);
        Self::e1000_cleanup_rx_resources(_data);

        // 获取irq_handler的指针
        let irq_handler_ptr = _data._irq_handler.load(core::sync::atomic::Ordering::Relaxed);

        // 确保指针不为空，然后释放资源
        if !irq_handler_ptr.is_null() {
            unsafe {
                // 将裸指针转换回 `Box`
                let _irq_handler_box = Box::from_raw(irq_handler_ptr);
                // 离开作用域时，`_irq_handler_box` 将被丢弃，`drop` 方法将被调用
            }

            // 重置 `AtomicPtr`
            _data._irq_handler.store(core::ptr::null_mut(), core::sync::atomic::Ordering::Relaxed);
        }

        _dev.netif_stop_queue();
        _dev.netif_carrier_off();

        _data.e1000_hw_ops.e1000_reset_hw();
        _data.napi.disable();

        Ok(())
    }
```

向E1000DrvPrvData中增加字段，使得我们可以保持对pci设备的引用，并将E1000DrvPrvData声明为unsafe
```rust
struct E1000DrvPrvData {
    _netdev_reg: net::Registration<NetDevice>,
    bars:i32,
    dev_ptr: *mut bindings::pci_dev,
}

unsafe impl Send for E1000DrvPrvData {}
unsafe impl Sync for E1000DrvPrvData {}
```

修改probe函数，将bars和pci_dev写入E1000DrvPrvData
```rust
    fn probe(dev: &mut pci::Device, id: core::option::Option<&Self::IdInfo>) -> Result<Self::Data> {
        pr_info!("Rust for linux e1000 driver demo (probe): {:?}\n", id);
        ...
        Ok(Box::try_new(
            E1000DrvPrvData{
                // Must hold this registration, or the device will be removed.
                _netdev_reg: netdev_reg,
                bars,
                dev_ptr: dev.as_ptr(),
            }
        )?)
    }
```

在remove中将pcidev还原出来，然后将它释放掉
```rust
    fn remove(data: &Self::Data) {
        pr_info!("Rust for linux e1000 driver demo (remove)\n");
        
        let netdev = data._netdev_reg.dev_get();
        let bars = data.bars;
        let pcidev_ptr = data.dev_ptr;
        let netdev_reg = &data._netdev_reg;

        unsafe { bindings::pci_release_selected_regions(pcidev_ptr, bars) };
        unsafe { bindings::pci_clear_master(pcidev_ptr) };
        unsafe { bindings::pci_disable_device(pcidev_ptr) };

        drop(netdev_reg);
        drop(data);
    }
```

将E1000DrvPrvData中的netdev_reg字段释放
```rust
    fn device_remove(&self) {
        pr_info!("Rust for linux e1000 driver demo (device_remove)\n");

        drop(&self._netdev_reg);
    }
```

最后，将self.dev释放
```rust
impl Drop for E1000KernelMod {
    fn drop(&mut self) {
        pr_info!("Rust for linux e1000 driver demo (exit)\n");

        drop(&self._dev);
    }
}
```
结果如下
首先尝试加载模块后，并释放模块，随后再加载，可以看到PCI设备的内存空间被释放了
```bash
insmod r4l_e1000_demo.ko
rmmod r4l_e1000_demo.ko
insmod r4l_e1000_demo.ko
```
![](https://imgfiles-debin.oss-cn-hangzhou.aliyuncs.com/md_imgfiles/202311131540697.png)

然后尝试启动设备后，配置网卡，正常ping通
```bash
ip link set eth0 up
ip addr add broadcast 10.0.2.255 dev eth0
ip addr add 10.0.2.15/255.255.255.0 dev eth0 
ip route add default via 10.0.2.1
ping 10.0.2.2
```
![](https://imgfiles-debin.oss-cn-hangzhou.aliyuncs.com/md_imgfiles/202311131543704.png)

最后移除模块，再重复上面的步骤，也可以正常ping通
```bash
rmmod r4l_e1000_demo.ko
insmod r4l_e1000_demo.ko
ip link set eth0 up
ip addr add broadcast 10.0.2.255 dev eth0
ip addr add 10.0.2.15/255.255.255.0 dev eth0 
ip route add default via 10.0.2.1
ping 10.0.2.2
```

![](https://imgfiles-debin.oss-cn-hangzhou.aliyuncs.com/md_imgfiles/202311150246762.png)

# 作业5 注册字符设备

更改配置：
```bash
make LLVM=1 menuconfig # <*>Character device (NEW)
```

编写字符设备代码,完成read和write方法
```rust
    fn write(
        this: &Self,
        _file: &file::File,
        reader: &mut impl kernel::io_buffer::IoBufferReader,
        _offset: u64,
    ) -> Result<usize> {
        let buffer = &mut this.inner.lock();
        let mut _size = reader.len();
        if _size > GLOBALMEM_SIZE {
            _size = GLOBALMEM_SIZE;
        }
        reader.read_slice(&mut buffer[.._size])?;
        Ok(_size)
    }

    fn read(this: &Self, _file: &file::File, writer: &mut impl kernel::io_buffer::IoBufferWriter, offset:u64,) -> Result<usize> {
        let data = &mut *this.inner.lock();
        if offset as usize >= GLOBALMEM_SIZE {
            return Ok(0);
        }
        writer.write_slice(&data[offset as usize..])?;
        Ok(data.len())
    }
```

编译模块，并测试
```bash
make LLVM=1 -j$(nproc)
cp samples/rust/rust_chrdev.ko ../src_e1000/rootfs/
cd ../src_e1000/
./build_image.sh
insmod rust_chrdev.ko
echo "Hello" > /dev/cicv
cat /dev/cicv
```
![](https://imgfiles-debin.oss-cn-hangzhou.aliyuncs.com/md_imgfiles/202311131551473.png)

 - 作业5中的字符设备/dev/cicv是怎么创建的？它的设备号是多少？它是如何与我们写的字符设备驱动关联上的？
```text
1.通过src_e1000/rootfs/etc/init.d/rcS中的`mknod /dev/cicv c 248 0`命令创建

2.它的设备号是248，次设备号0

3.字符设备通过chedrv的`register`方法注册，这个函数将设备号与设备驱动程序关联起来，使得当应用程序打开/dev/cicv设备文件时，内核能够找到正确的驱动程序来处理该设备的操作。

```
![](https://imgfiles-debin.oss-cn-hangzhou.aliyuncs.com/md_imgfiles/202311131552771.png)