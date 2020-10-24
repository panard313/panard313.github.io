---
layout: article
title: qemu kernel debug
key: 20200403
tags:
  - qemu
  - kernel debug
  - cx
lang: zh-Hans
---

# Debug linux kernel with qemu and vscode

# 一　prepare ubuntu virtual machine

由于工作机权限, 版本较低等问题，　最好配置一个ubuntu虚拟机来进行所有工作，　本例中安装了ubuntu server 18.04 LTS版本, 使用x86_64位，　安装过程略过．


# 二　linux kernel 下载和编译

PS: 以下终端操作均在Ububtu虚拟机中进行

## 下载源码

可以从清华的镜像站下载linux 内核，　推荐使用git下载而不是下载压缩包，以便后期查看内核中的提交日志以及切换不同的内核版本．

```shell
git clone https://mirrors.tuna.tsinghua.edu.cn/git/linux.git
# 取回远程的tags
git fetch origin
# 以tag v4.19.100创建本地分支
git checkout -b ver-4.19.100 v4.19.100
```

Tips:

可以在现有的linux kernel代码中直接加入其它远程仓库，例如以下命令加入了cgroup 和block的开发仓库，　以便直观地查看相应模块的演变：

```shell
git remote add cgroup https://git.kernel.org/pub/scm/linux/kernel/git/tj/cgroup.git
git remote add block https://git.kernel.org/pub/scm/linux/kernel/git/axboe/linux-block.git
```

## 编译

最简单的方法，直接执行：

```shell
make localyesconfig
```
此方法可以直接以当前虚拟机的＂硬件＂配置给出一个最精简的内核配置文件，　保存在.config

### 检查kernel debug相关config

检查.config中以下config需要已打开：
```shell
CONFIG_DEBUG_INFO=y
CONFIG_DEBUG_KERNEL=y
CONFIG_GDB_SCRIPTS=y
```

### 执行 make -j4编译内核

检查生成文件：
- vmlinux
- vminux-gdb.py

## 配置qemu 与根文件系统：

### 安装以下软件包：
```shell
sudo apt install qemu gdb g++
```

Tips:
  - kernel gdb 对qemu, gdb版本均有要求，踩坑过程中看到有网友表示低版本ubuntu安装的qemu gdb会有问题，ubuntu server 18.04 LTS上实测直接安装的版本均可用．

### 为qemu kernel配置虚拟文件系统

#### 下载busybox
```shell
wget -c https://busybox.net/downloads/busybox-1.30.1.tar.bz2
tar xf busybox-1.30.1.tar.bz2
cd busybox-1.30.1
make menuconfig
# 选择静态链接，否则无法启动
make -j4 && make install
#检查编译成成的文件都在_install文件夹中
```

#### make image
```shell
qemu-img create qemu_rootfs.img  30g # 适当选择大小，１gb一般就够
mkfs.ext4 qemu_rootfs.img
mkdir qemu_rootfs
sudo mount -o loop qemu_rootfs.img  qemu_rootfs
cd qemu_rootfs
sudo mkdir proc sys dev etc etc/init.d var
cp -rv ../busybox-1.30.1/_install/* .
sudo vim etc/init.d/rcS
```

rcS文件内容如下：
```shell
#/etc/init.d/rcS
mount -t proc none /proc
mount -t sysfs none /sys
/sbin/mdev -s
```

修改一下权限：
```shell
sudo chmod a+x etc/init.d/rcS
```

### 测试内核和根文件系统是否可以启动

使用以下命令启动准备好的内核和根文件系统：
```shell
qemu-system-x86_64 -kernel ../linux-4.19.100/arch/x86_64/boot/bzImage -hda qemu_rootfs.img -append "root=/dev/sda rootfstype=ext4 rw  console=ttyS0" -m 512 --nographic
```

with buildroot:
```shell
qemu-system-x86_64 -kernel arch/x86_64/boot/bzImage -drive file=../buildroot-2020.02/output/images/rootfs.ext2,if=ide,format=raw -append "root=/dev/sda rootfstype=ext2 rw  console=ttyS0" -m 512 --nographic
```

qemu + buildroot + erofs:
```shell
qemu-system-x86_64 -kernel linux-4.19/arch/x86_64/boot/bzImage -drive file=buildroot-2020.02/output/images/rootfs.ext2,if=ide,format=raw -drive file=erofs.img,if=ide,format=raw -append "root=/dev/sda rootfstype=ext2 rw  console=ttyS0" -m 512 --nographic
```

可以看到内核能够正常启动并进入busybox自带的sh，　到这里一个基本的linux系统已经启动成功，可以执行busybox进的所有命令．

![图一：](/images/qemu/qemu-linux-terminal.png)


## 开始debug

### 使kernel启动并等待debug连接：
```shell
qemu-system-x86_64 -kernel ../linux-4.19.100/arch/x86_64/boot/bzImage -hda qemu_rootfs.img -append "root=/dev/sda rootfstype=ext4 rw  console=ttyS0 nokaslr" -m 512 --nographic -s -S
```

PS: 注意添加了nokaslr和-s -S参数．

- nokaslr为了保证内核中的起始地址不随机化，不加此参数会导致debug时gdb并不能跟踪到内核当前的代码位置
- -s -S让内核启动后立即停住，等待gdb连接

### 使用gdb连接qemu中的内核

```shell
echo add-auto-load-safe-path /path/to/kernel/source > ~/.gdbinit
```
ps: 以便让gdb启动后直接加载内核编译路径下的vmlinux-gdb.py, 省去执行autoload之类的操作．

启动gdb:
```shell
gdb \
    -ex "add-auto-load-safe-path /path/to/kernel/source" \
    -ex "file vmlinux" \
    -ex 'set arch i386:x86-64:intel' \
    -ex 'target remote localhost:1234' \
    -ex 'break start_kernel' \
    -ex 'continue' \
    -ex 'disconnect' \
    -ex 'set arch i386:x86-64' \
    -ex 'target remote localhost:1234'
```

内核启动后停住：

![图二:](/images/qemu/qemu-kernel-hangup.png)

gdb连接后可以看到内核停在init/main.c start_kernel()

![图三:](/images/qemu/qemu-kernel-bp.png)

### gdb常规操作：

此时，后续的操作与应用程序调试时的gdb没有区别．当前的状态已经可以直接进行内核启动流程的debug.

若需要让系统完成启动，可以直接在gdb中continue，qemu中就可以看到terminal. 

## let vscode instead of gdb

gdb仅仅是满足了基本功能，难看且难用，好在有vscode.

- 由于这里的内核源码与qemu均在虚拟机中，首先需要安装并配置remote插件，在插件中搜索remote　development，　安装．

- 配置ubuntu虚拟机的端口转发，以暴露出虚拟机的ssh端口

- 安装cpptools，　以便进行代码跳转和语法解析（安装在远程）

- 直接以ubuntu虚拟机中的kernel source建立一个
workspace, 以便vscode能够进行kernel全域的代码跳转

- 安装gdb debug插件（安装在远程）

- Run --> open configuration:

修改为：

```json
{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "kernel-debug",
            "type": "cppdbg",
            "request": "launch",
            "miDebuggerServerAddress": "127.0.0.1:1234",
            "program": "${workspaceFolder}/vmlinux",
            "args": [],
            "stopAtEntry": false,
            "cwd": "${workspaceFolder}",
            "environment": [],
            "externalConsole": false,
            "logging": {
                "engineLogging": false
            },
            "MIMode": "gdb",
        }
    ]
}
```
- 按上述方法启动qemu kernel后，　点击run --> start debugging, 即可

如下图：

![vscode](/images/qemu/vscode-debug.png)

#### vscode settings:

append:

```json
"files.watcherExclude": {
    "**/.git/objects/**": true,
    "**/.git/subtree-cache/**": true,
    "**/node_modules/*/**": true,
    "**/drivers/*/**": true,
    "**/*.o": true,
    "**/*.cmd": true,
    "**/*.map": true,
    "**/*.txt": true,
},
"files.exclude": {
    "**/.git": true,
    "**/.DS_Store": true,
    "jspm_packages": true,
    "node_modules": true,
    "**/**/node_modules": true,
    "**/drivers": true,
    "**/firmware": true,
    "**/net": true,
    "**/sound": true,
    "**/vmlinux*": true,
    "**/.zip": true,
    "**/.sh": true,
    "**/.idea": true,
    "**/*.o": true,
    "**/*.cmd": true,
    "**/*.map": true,
},
"search.exclude": {
    "**/node_modules": true,
    "**/bower_components": true,
    "**/.idea": true,
    "**/*.o": true,
    "**/*.cmd": true,
    "**/*.map": true,
},
"debug.console.fontFamily": "ubuntu mono",
"C_Cpp.dimInactiveRegions": false,
```

#### .gdbinit

> add-auto-load-safe-path /home/panard/linux-4.19.100


#### links

[How to debug the Linux kernel with GDB and QEMU?](https://stackoverflow.com/questions/11408041/how-to-debug-the-linux-kernel-with-gdb-and-qemu)


## extened

#### create img

qemu-img create -f qcow2 -o preallocation=metadata qemu.qcow2 1G
