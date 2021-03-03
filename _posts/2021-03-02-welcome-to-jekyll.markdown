---
layout: default
---
# kernel编译

## 1. rpmbuild方式编译内核rpm包

### demo

我们现在演示编译一个centos8.3的内核，打上hygon的补丁,编译x86架构的内核rpm包

### 编译环境

centos8.3

### 编译依赖

#### 打开PowerTools.repo源

`/etc/yum.repos.d/CentOS-Linux-PowerTools.repo`

```shell
# CentOS-Linux-PowerTools.repo
#
# The mirrorlist system uses the connecting IP address of the client and the
# update status of each mirror to pick current mirrors that are geographically
# close to the client.  You should use this for CentOS updates unless you are
# manually picking other mirrors.
#
# If the mirrorlist does not work for you, you can try the commented out
# baseurl line instead.

[powertools]
name=CentOS Linux $releasever - PowerTools
mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=PowerTools&infra=$infra
#baseurl=http://mirror.centos.org/$contentdir/$releasever/PowerTools/$basearch/os/
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-centosofficial
```

#### 安装编译依赖

```shell
yum install -y gcc git java-devel kabi-dw libbabeltrace-devel libcap-devel libcap-ng-devel llvm-toolset m4 make ncurses-devel newt-devel nss-tools numactl-devel openssl-devel pciutils-devel perl perl-devel perl-generators pesign python3-devel python3-docutils xmlto xz-devel zlib-devel dwarves libbabeltrace-devel perl pesign dwarves libbabeltrace-devel libbpf-devel
```

#### 安装rpmbuild

```shell
yum install -y rpmbuild
```

#### 源码准备

1. 拷贝源码到`~/rpmbuild/SOURCES`下

2. 拷贝kernel.spec到`~/rpmbuild/SPECS/`下

3. 拷贝000_x86_add_hygon_family_18h.patch 到`~/rpmbuild/SOURCES`下（这里以hygon的内核补丁为例子）

4. 修改`~/rpmbuild/SPECS/`下的kernel.spec

   在Source或Patch999999增加一行`Patch9000: 000_x86_add_hygon_family_18h.patch`(Patch9000表示patch编号，不冲突即可)

```shell
...
# Sources for kernel-tools
Source2000: cpupower.service
Source2001: cpupower.config

Source9000: centos.pem

## Patches needed for building this package

# empty final patch to facilitate testing of kernel patches
Patch999999: linux-kernel-test.patch
Patch9000: 000_x86_add_hygon_family_18h.patch
...
```

​		在`ApplyOptionalPatch linux-kernel-test.patch` 后增加一行，`ApplyOptionalPatch linux-kernel-test.patch`（行业的代码对spec做了修改，只需增加）

```shell

...
%setup -q -n %{name}-%{rpmversion}-%{pkgrelease} -c
cp -v %{SOURCE9000} linux-%{rpmversion}-%{pkgrelease}/certs/rhel.pem
mv linux-%{rpmversion}-%{pkgrelease} linux-%{KVERREL}

cd linux-%{KVERREL}

ApplyOptionalPatch linux-kernel-test.patch
ApplyOptionalPatch 000_x86_add_hygon_family_18h.patch

# END OF PATCH APPLICATIONS
...
```
5. 编译

   ```shell
   pushd ~/rpmbuild/SPECS
   rpmbuild -ba kernel.spec
   ```

   等内核编译完，在`/home/gaochong/rpmbuild/RPMS/`产生内核rpm包，`/home/gaochong/rpmbuild/SPMS`下产生SRPM包

   <!--*如果编译报错，缺少依赖，yum安装缺少的依赖包即可*-->

## 2. tarball方式编译内核

### demo

从tar包编译**4.19.163**版本x86内核, 编译vmlinuz和module

### 下载源码

```shell
wget https://cdn.kernel.org/pub/linux/kernel/v4.x/linux-4.19.163.tar.xz
tar -xf linux-4.19.163.tar.xz
```

### 内核编译过程简述

#### make menuconfig、Kconfig、makefile 关系

- make menuconfig 通过调用script提供的工具和Kconfig提供配置，指定生成.ko和编译到内核的选项，然后在图形界面显示

- make menuconfig 弹出图形界面，交互生成.config文件

  详细关系，参考https://developer.aliyun.com/article/10404

#### make help

`make help`是makefile 内置的目标，打印内核make相关可以使用的参数

#### 执行make menuconfig

![](https://raw.githubusercontent.com/zhuzaifangxuele/PicgoBed/main/img/20201223131700.png)

```
Pressing <Y> includes, <N> excludes, <M> modularizes features.  Press <Esc><Esc> to exit, <?> for Help, </> for Search.  
Legend: [*] built-in  [ ] excluded  <M> module  < > module capable  
```

按Y表示包含进来，N表示不包含，M表示编译成模块，就是.ko

选择好，移动到Save保存，Esc按两次退出

***生成.config 文件（这里的config文件是通过menuconfig配置生成的），一般我们会使用其他系统里已经配置好内核配置项在`/boot/config-4.18.0-240.el8.x86_64`，把`/boot/config-4.18.0-240.el8.x86_64`***
***拷贝成.config，这样运行make menuconfig的时候，会从.config读取配置项，我们在.config的基础上修改配置后刷新.conifg，原来的配置文件回保存为.config.old***

#### 编译

##### 编译

```
make -j4
```

`-j4`表示线程，最大设置为处理器的核心数，超过反而回慢。

`make` 在缺省目标时，相当于`make all`，这个可以在Makefile 里找到`all：`目标，查看到编译依赖

`make ` = `make vmlinux` + `make modules`

##### 清空编译

```
make clean
```



```shell
Cleaning targets:
  clean           - Remove most generated files but keep the config and
                    enough build support to build external modules
  mrproper        - Remove all generated files + config + various backup files
  distclean       - mrproper + remove editor backup and patch files
```



#### 安装内核模块

```
make modules_install INSTALL_MOD_PATH=/e/x/a/m/p/l/e
```

安装内核模块`INSTALL_MOD_PATH`指定内核模块安装的目录，可以不指定，默认是`/` 下生成对应内核版本的lib/modules 

```
make install
```

#### 更改开机启动项

- Legacy模式

```shell
grub2-mkconfig -o /boot/grub2/grub.cfg
```

- UEFI模式

```
grub2-mkconfig -o /boot/efi/EFI/centos/grub.cfg
```

#### 重启

![](https://raw.githubusercontent.com/zhuzaifangxuele/PicgoBed/main/img/20201223141505.png)

![](https://raw.githubusercontent.com/zhuzaifangxuele/PicgoBed/main/img/20201223141543.png)

#### 卸载安装的内核

```
[gaochong@localhost x86_64]$ ls -l /boot/
total 495996
-rw-r--r--. 1 root root    189494 Sep 25 15:57 config-4.18.0-240.el8.x86_64
-rw-r--r--. 1 root root    182939 Dec 23 03:25 config-4.19.163
drwxr-xr-x. 3 root root        17 Dec 21 01:23 efi
drwx------. 4 root root        83 Dec 21 05:52 grub2
-rw-------. 1 root root 105462544 Dec 21 01:47 initramfs-0-rescue-468e9bcfa79d489885f2c06dbde1e7be.img
-rw-------. 1 root root  55163193 Dec 21 01:57 initramfs-4.18.0-240.el8.x86_64.img
-rw-------. 1 root root  19116813 Dec 21 03:34 initramfs-4.18.0-240.el8.x86_64kdump.img
-rw-------. 1 root root 110490576 Dec 23 04:11 initramfs-4.19.163.img
drwxr-xr-x. 3 root root        21 Dec 21 01:42 loader
lrwxrwxrwx. 1 root root        25 Dec 23 04:09 System.map -> /boot/System.map-4.19.163
-rw-------. 1 root root   4032815 Sep 25 15:57 System.map-4.18.0-240.el8.x86_64
-rw-r--r--. 1 root root   3755674 Dec 23 04:09 System.map-4.19.163
-rwxr-xr-x. 1 root root 182475078 Dec 23 03:19 vmlinux-4.19.163.bz2
lrwxrwxrwx. 1 root root        22 Dec 23 04:09 vmlinuz -> /boot/vmlinuz-4.19.163
-rwxr-xr-x. 1 root root   9514120 Dec 21 01:45 vmlinuz-0-rescue-468e9bcfa79d489885f2c06dbde1e7be
-rwxr-xr-x. 1 root root   9514120 Sep 25 15:57 vmlinuz-4.18.0-240.el8.x86_64
-rw-r--r--. 1 root root   7980928 Dec 23 04:09 vmlinuz-4.19.163
```

1. 删除/lib/modules 下对应的 kernel版本文件夹。 

2.  删除/boot/下面对应的内核vmlinz和System.map指向的软件

3. 调用grub2-mkconfig,实现grub菜单

   - uefi系统

     ```
     grub2-mkconfig -o /boot/efi/EFI/centos/grub.cfg 
     ```

   - legacy系统
   
     ```
     grub2-mkconfig -o /boot/grub2/grub.cfg
     ```

## 3. tarball 编译rpm或deb包

- 制作或copy centos的证书到`certs/rhel.pem`

- kernel的makefile回自动生成spec文件，调用rpmbuild 编译

```
make -j64 rpm-pkg 
```



- 编译过程

```
[gaochong@localhost linux-4.19.163]$ make -j64 rpm-pkg 
scripts/kconfig/conf  --syncconfig Kconfig
  UPD     include/config/kernel.release
make clean
/bin/sh ./scripts/package/mkspec >./kernel.spec
  TAR     kernel-4.19.163.tar.gz
rpmbuild  --target x86_64 -ta kernel-4.19.163.tar.gz \
--define='_smp_mflags %{nil}'
Building target platforms: x86_64
Building for target x86_64
Executing(%prep): /bin/sh -e /var/tmp/rpm-tmp.9oT1A4
+ umask 022
+ cd /home/gaochong/rpmbuild/BUILD
+ cd /home/gaochong/rpmbuild/BUILD
+ rm -rf kernel-4.19.163
+ /usr/bin/gzip -dc /home/gaochong/lab/linux-4.19.163/kernel-4.19.163.tar.gz
+ /usr/bin/tar -xof -
+ STATUS=0
+ '[' 0 -ne 0 ']'
+ cd kernel-4.19.163
+ /usr/bin/chmod -Rf a+rX,u+w,g-w,o-w .
+ exit 0
Executing(%build): /bin/sh -e /var/tmp/rpm-tmp.1ysQtM
+ umask 022
+ cd /home/gaochong/rpmbuild/BUILD
+ cd kernel-4.19.163
+ make KBUILD_BUILD_VERSION=1
```

```
/home/gaochong/rpmbuild/RPMS/x86_64/kernel-4.19.163-1.x86_64.rpm
/home/gaochong/rpmbuild/RPMS/x86_64/kernel-devel-4.19.163-1.x86_64.rpm
/home/gaochong/rpmbuild/RPMS/x86_64/kernel-headers-4.19.163-1.x86_64.rpm
```

- 编译后会产生3个rpm包

```
/home/gaochong/rpmbuild/RPMS/x86_64/kernel-4.19.163-1.x86_64.rpm
/home/gaochong/rpmbuild/RPMS/x86_64/kernel-devel-4.19.163-1.x86_64.rpm
/home/gaochong/rpmbuild/RPMS/x86_64/kernel-headers-4.19.163-1.x86_64.rpm
```

​	扩展：initrd 是安装kernel-4.19.163-1.x86_64.rpm后产生的，`%post`阶段调用`/sbin/installkernel` 脚本，脚本调用`/usr/lib/kernel/install.d/`里的`50-dracut.install`产生的`initramfs-${KERNEL_VERSION}.img` 



## 4. 考题

在欧拉的系统上，编译安装一个5.9的ARM内核，重启后默认从5.9内核上启动系统
