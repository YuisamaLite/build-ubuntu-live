# 从零开始制作 Ubuntu 22.04 Live CD

## 前言

最近想做一个基于原生 GNOME 桌面的 Ubuntu LiveCD，然后就想到了 Debian 系的 LiveCD 构建工具 live-build，就安装了一个 Ubuntu 版本试一试，发现太老了没法用。

然后我就在 Launchpad 上问了一下官方，他们的答复是 Ubuntu 现在已经不用 live-build 这种方式构建，所以 live-build for ubuntu 就不更新了。

于是我只能在官方文档里找教程，终于找到了，但是有一点老，很多东西不适用。我就参考官方文档和 rohhie.net 里的一篇制作 Ubuntu 20.04 的教程，琢磨了自己的 Ubuntu 22.04 Live CD 制作教程。详见参考资料，对两篇文档作者表示感谢。

教程以 x86-64 平台 + Ubuntu 22.04（Jammy）为例。这个教程建议有一定 Linux 基础的人使用，如果你不熟悉 Linux ，又进行了误操作，将可能会导致本机软件配置出现错误甚至是系统崩溃的危险。请慎重使用。

如果您要制作 Ubuntu 22.04 以下版本的 Live CD，本教程的部分内容可能需要进行一些调整。




基于 [Creative Commons Attribution-ShareAlike 3.0 License](http://creativecommons.org/licenses/by-sa/3.0/) 许可协议。



## 为什么要构建自己的 LiveCD

1. 进行一些本地化修改；
2. 对默认设置不满意；
3. 对镜像的预装不满意。



## 构建条件

1. 一个运行 Ubuntu 的设备（虚拟机也是可以的，但不推荐 WSL1/2，因为 WSL2 经过实际测试会出现一些问题）。

2. 确保已经安装 `dosfstools`、`genisoimage`、`squashfs-tools` 、`xorriso`、`grub-common`、`grub-pc-bin` 、`grub-efi-amd64-bin `软件包。还有 `nano` 文本编辑器。

```shell
apt install dosfstools genisoimage squashfs-tools xorriso grub-common grub-pc-bin grub-efi-amd64-bin nano
```

3. 打开终端。

4. 全程需要 root 权限，请输入 `sudo -s` 进入 root 权限。



## 准备目录

准备一个空文件夹作为工作目录。这个空文件夹所在分区不能是 NTFS、FAT32 格式。

然后在这个文件夹中创建 chroot、image 这两个文件夹。chroot 里面是目标系统，image 是 ISO 文件夹目录。

image 里面要创建 casper boot EFI preseed 等文件夹

在配置的时候你应当保持 Shell 位置的你的工作目录下面。



## 准备镜像

### 基础系统

构建基础系统有两种方式：

1. 利用 debootstrap 构建一个基础系统。
2. 解压 Ubuntu Base。


debootstrap 是 Debian 系发行版的一个实用工具，允许您基于在线的软件源构建一个属于自己的 Debian 系发行版的基本系统。

Ubuntu Base 是 Ubuntu 的基本系统，说通俗点，就是 Ubuntu 帮你打包好的 debootstrap 后的基本系统。我个人更倾向这一种，因为方便。


如果你想使用 debootstrap 工具构建一个基本系统，请运行：`debootstrap jammy chroot https://mirrors.ustc.edu.cn/ubuntu`

mirrors.ustc.edu.cn 是中国科学技术大学的开源镜像站，你可以换成阿里的、华为的或者清华源，如果你所在高校也有开源镜像站，换成你学校的效果更好。



如果是 Ubuntu Base，请在 http://mirrors.ustc.edu.cn/ubuntu-cdimage/ubuntu-base/ 里找对应的 .tar.gz 格式软件包，下载，然后进入到 chroot 目录，解压这个软件包：

```shell
cd chroot
tar -xvf ../ubuntu-base-22.04-base-amd64.tar.gz`
cd ..
```



### 基础系统的设置

你应该首先复制你本机的网络设置到 chroot/etc 下，所以你应该：

1. `mv chroot/etc/resolv.conf chroot/etc/resolv.conf.1`  这个命令是备份目标系统的网络配置

2. `cp /etc/resolv.conf chroot/etc/resolv.conf` 复制你的本机网络配置到目标系统。



应当修正目标系统的软件源设置，因为默认是不完善的。

输入 nano chroot/etc/apt/sources.list ，编辑软件源，下面给一个示例：

```
deb https://mirrors.ustc.edu.cn/ubuntu/ jammy main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ jammy main restricted universe multiverse

deb https://mirrors.ustc.edu.cn/ubuntu/ jammy-security main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ jammy-security main restricted universe multiverse

deb https://mirrors.ustc.edu.cn/ubuntu/ jammy-updates main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ jammy-updates main restricted universe multiverse

deb https://mirrors.ustc.edu.cn/ubuntu/ jammy-backports main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ jammy-backports main restricted universe multiverse
```

其中，`jammy` 是 Ubuntu 22.04 的代号。`main restricted` 之类的表明软件仓库的一些开关，你不熟悉的话可以不用管他，启用就行。



### 容器操作

#### 挂载特殊文件系统

你需要先挂载一些特殊的文件系统。比如 dev proc sys run。

```shell
mount --bind /dev chroot/dev
mount none -t proc chroot/proc
mount none -t sysfs chroot/sys
mount none -t devpts chroot/dev/pts
```



进入 chroot 容器，``` chroot chroot```

你会发现提示符变了，那么就顺利进入了chroot 容器，你可以像操作你的主机系统一样操作 chroot 容器，对其做出更改。



官网文档和参考资料中的教程里面都提到了应该将 /bin/true 链接到 /sbin/initctl，在这个教程里面我认为是不需要的，因为 systemd 已经作为新式的加载程序取代了 sysvinit 和 upstart，所以这个命令不再需要。



#### 更新软件源

我们先更新一下软件源，升级一下基础系统里的软件包。

```
apt update
apt upgrade
```



#### 安装 Linux 内核

这一步是必要的，安装的内核还将用于 ISO 镜像的引导。

``` shell
apt install linux-generic 
```

如果您对 hwe 内核有需求，也可以安装。

安装内核之后 GRUB 之类的会自动安装。



#### 安装 Ubuntu 标准套件

这是官方教程所推荐的，当然可以省略，但不推荐。

```shell
apt install ubuntu-standard
```

标准套件包括很多实用工具，是推荐安装的。



#### 安装 LiveCD 套件

包括硬件检测之类的组件，这一点非常重要，关乎到 LiveCD 能否成功启动。

```shell
apt install casper discover laptop-detect os-prober
```

如果是旧版本的 Ubuntu 还需要安装 `lupin-casper` 软件包。



安装 NetworkManager，顾名思义，网络套件。如果你选择在后续安装桌面环境，那么这一步可以省略。

```shell
apt-get install --no-install-recommends network-manager
```



### 安装 snap 等软件包管理器（可选）

snap 是新一代的 Ubuntu 软件包管理器，现在预装的 FireFox 等软件都是 snap 格式安装，所以您可以安装 snapd。 

```shell 
apt install snapd
```


Flatpak 也是新一代的软件包管理器，由 Red Hat 主导，但由于通用性强，所以也可以安装。

```shell 
apt install flatpak
```



### 给目标系统安装桌面

你应该给你的目标系统安装一个桌面环境，当然你也可以选择不安装，但是不推荐。

下面是几个套件，仅供参考：

1. ubuntu-desktop ，默认桌面
2. xubuntu-desktop，xfce。
3. kubuntu-desktop，KDE。

4. ubuntu-mate-desktop，Mate 桌面
5. lubuntu-dekstop，LXQT。

6. vanilla-gnome-desktop，原生GNOME桌面，但是安装后需要再配置。

如果你觉得套件里面有你不喜欢的组件，你也可以选择手动安装，比如 `apt install kde-standard`。



### 自定义配置

到这里结束之后，你可以在这个 chroot 容器里面进行自定义的设置，你想要预装什么软件，也可以操作。在容器里的操作和在本机操作的命令是相似甚至一致的。

比如我要预装编译套件，我可以：`apt install build-essential`。

BleachBit 是一款开源的垃圾清理工具，所以我可以：`apt install bleachbit`



比如，设定语言区域选项（图形环境）：`dpkg-reconfigure locales`

中文语言包：`apt install language-pack-zh-hans language-pack-gnome-zh-hans`

如果是 kde 语言包就换成 `language-pack-kde-zh-hans`

### 安装安装器

这一步不是必要的，但是如果你有将这个 LiveCD 安装到本机硬盘的需求，你可以这样做。

根据桌面环境的不同选择命令：

GNOME：

```shell
apt-get install ubiquity-frontend-gtk ubiquity-casper ubiquity-slideshow-ubuntu ubiquity-ubuntu-artwork ubiquity-frontend-debconf
```

KDE：

```shell
apt-get install ubiquity-frontend-kde ubiquity-casper ubiquity-slideshow-kubuntu ubiquity-ubuntu-artwork ubiquity-frontend-debconf
```

LXDE、MATE 之类以此类推，比如 ubiquity-slideshow-lubuntu





## 清理容器

到这一步，目标系统已经配置完毕了，下面应当进行清理。

输入 `apt clean`，清理一切软件源缓存。

输入 `rm /var/lib/dbus/machine-id` ，清理 machine-id。

清理临时文件：`rm -rf /tmp/*`



之前我们更改的 resolv.conf 也要恢复原状。

所以你应该 

```shell
rm /etc/resolv.conf
mv /etc/resolv.conf.1 /etc/resolv.conf
```



退出 chroot 容器：`exit`

进入目标系统的 /root 目录，清理掉 bash 历史记录： `rm chroot/root/.bash_history`。



取消特殊文件系统的挂载：

```shell
umount chroot/dev/pts
umount chroot/dev
umount chroot/proc
umount chroot/sys
```

如果取消挂载失败，可以看看是不是有什么程序占用了，如果还是不行，可以重启电脑，重启自动取消挂载。



## 打包容器

建立清单：

```shell
chroot chroot dpkg-query -W --showformat='${Package} ${Version}\n' | tee image/casper/filesystem.manifest
cp -v image/casper/filesystem.manifest image/casper/filesystem.manifest-desktop
REMOVE='ubiquity ubiquity-frontend-gtk ubiquity-frontend-kde casper live-initramfs user-setup discover1 xresprobe os-prober libdebian-installer4 ubiquity-slideshow-ubuntu ubiquity-ubuntu-artwork ubiquity-frontend-debconf'
for i in $REMOVE
do
        sed -i "/${i}/d" image/casper/filesystem.manifest-desktop
done
```

REMOVE 里面内容应当根据实际情况调整。


打包目标系统：

```
mksquashfs chroot image/casper/filesystem.squashfs
printf $(du -sx --block-size=1 chroot | cut -f1) > image/casper/filesystem.size
```

filesystem.squashfs 类似于 Windows 里面的 install.wim。



## ISO 镜像的制作

### 内核的创建、以及介质信息

1. 复制内核，将目标系统的 boot 文件夹里面的 vmlinuz-版本号和 initrd.img-版本号 这两个文件，复制到 ISO 文件夹的 casper 目录。版本要选 boot 文件夹里面最新的。

   如果你需要在 ISO 镜像里集成 `memtest`，可以加到 boot/grub 文件夹里面，同时在引导菜单里增加对应项目。

2. 【可选】创建介质信息，在 ISO 文件夹下面创建 .disk 文件夹，创建几个文件：

   base_installable、casper-uuid-generic、cd_type、info、release_notes_url

   cd_type 里面填 full_cd/single，releases_notes 里面填你的个人网址，info 是光盘信息（比如，我可以填写 `My Ubuntu Live Build 2022-09-14 `）。

   base_installable 文件留空，是空文件。
   
   

一些文档还会注明要在 ISO 文件夹根目录下创建 `README.diskdefines` 文件，但根据我的观察，Ubuntu 22.04 是可以不需要的，而且官方镜像也没有。

### 准备引导

Ubuntu 22.04 已经放弃了使用 ISOLINUX 作为 Legacy 方式引导，因此 Legacy 和 UEFI 模式都是基于 GRUB 的。如果你没有将镜像从 Legacy 方式引导的需求，那么也可以省略制作 Legacy 引导。

#### 创建 GRUB EFI 引导文件

1. 首先创建一个空白的 FAT32 格式的虚拟磁盘，`dd if=/dev/zero of=grub-efi-disk.img bs=12M count=1`

2. 格式化：`mkfs.vfat grub-efi-disk.img`

3. 创建一个挂载点：`mkdir mnt-grub`

4. 挂载虚拟磁盘，`mount grub-efi-disk.img mnt-grub`

5. 安装 GRUB EFI 到虚拟磁盘：`grub-install --target x86_64-efi --removable grub-efi-disk.img --efi-directory=mnt-grub --boot-directory=mnt-grub/boot --uefi-secure-boot`

6. 虚拟磁盘里有一文件，叫 EFI/boot/grub/grub.cfg，将该文件内容改为：

   ```
   search --set=root --file /casper/vmlinuz
   configfile /boot/grub/grub.cfg
   ```

7. 创建一个新的虚拟磁盘：`dd if=/dev/zero of=efi.img bs=4M count=1`

8. 格式化并挂载它：

   ```
   mkdir mnt-efi
   mkfs.vfat efi.img
   mount efi.img mnt-efi
   ```

9. 将 mnt-grub 下面的 EFI 文件夹全部复制到 mnt-efi 下面。将 boot 文件夹里的内容复制到 ISO 文件夹的 boot 下面。

10. 卸载虚拟磁盘: `umount mnt-grub` `umount mnt-efi`

11. 将虚拟磁盘 `efi.img` 复制到 ISO 文件夹的 boot/grub 下面。

#### 创建 GRUB LEGACY 引导文件

输入这个命令，创建一个仅仅保留必要项目的 Grub Legacy 镜像。

```shell
grub-mkstandalone --format=i386-pc --output=grub-pc-img.img --install-modules="linux linux16 normal iso9660 biosdisk memdisk search configfile tar ls" --modules="linux linux16 normal iso9660 biosdisk search configfile" --locales="" --fonts="" /boot/grub/grub.cfg=<ISO文件夹的绝对路径>/boot/grub/grub.cfg
```

​	`--locales --fonts` 应该留空，缩减体积。

等号后面必须是绝对路径，否则 GRUB 将无法识别配置文件。

合并 cdimage.img 和 core.img 构成一个完整的 legacy 引导文件，`cat /usr/lib/grub/i386-pc/cdboot.img grub-pc-img.img > image/boot/grub/boot.img`

#### 复制 GRUB 模块和创建配置

1. 复制 `/usr/lib/grub` 文件夹里的 `i386-pc` 文件夹，到 ISO 目录的 `boot/grub` 下面。
2. 准备一个引导菜单，也就是 `ISO工作目录/boot/grub/grub.cfg`，这里给出一个示例，来自 Ubuntu Desktop：

```
search --set=root --file /casper/vmlinuz

set timeout=30

menuentry "Try or Install Ubuntu" {
	set gfxpayload=keep
	linux	/casper/vmlinuz boot=casper quiet splash --- 
	initrd	/casper/initrd
}
menuentry "Ubuntu (safe graphics)" {
	set gfxpayload=keep
	linux	/casper/vmlinuz nomodeset boot=casper quiet splash --- 
	initrd	/casper/initrd
}
grub_platform
if [ "$grub_platform" = "efi" ]; then
menuentry 'Boot from next volume' {
	exit 1
}
menuentry 'UEFI Firmware Settings' {
	fwsetup
}
else
fi
```

如果您想在加载 LiveCD 时将系统载入内存，可以添加 toram 选项，但是这会比较耗费 RAM 空间。

### MD5 校验值的创建

```
cd image && find . -type f -print0 | xargs -0 md5sum | grep -v "\./md5sum.txt" > md5sum.txt
cd ..
```

这个步骤结束之后，清理一下上面步骤的临时文件。



### ISO 的生成

确保你在工作目录下，输入下列命令：

```
cd image
xorriso \
   -as mkisofs \
   -iso-level 3 \
   -full-iso9660-filenames \
   -volid "UBUNTU_LIVE_BUILD" \
   -output ../ubuntu-live.iso \
   -isohybrid-mbr /usr/lib/grub/i386-pc/boot_hybrid.img \
   --grub2-mbr /usr/lib/grub/i386-pc/boot_hybrid.img \
   --grub2-boot-info \
   -b boot/grub/boot.img \
      -no-emul-boot \
      -boot-load-size 4 \
      -boot-info-table \
      --eltorito-catalog boot/grub/boot.cat \
   -append_partition 2 0xef boot/grub/efi.img \
   -eltorito-alt-boot \
      -e boot/grub/efi.img \
      -no-emul-boot \
      -isohybrid-gpt-basdat \
      .
```



## 测试

运行你的虚拟机（VirtualBox、KVM 等），然后进行测试。



## 美化

这个 ISO 镜像的一些 UI 之类的元素是没有经过美化的，各位可以参照网络的教程进行适当的美化。



## 参考资料

1. https://help.ubuntu.com/community/LiveCDCustomizationFromScratch
2. https://help.ubuntu.com/community/LiveCDCustomization
3. grub-mkimage 的 man 手册；grub-mkstandalone 的 man 手册。
4. https://blog.csdn.net/a5nan/article/details/51361904
5. https://rohhie.net/ubuntu20-04-try-to-make-a-live-cd-from-scratch-basic
6. https://wiki.archlinux.org/title/GRUB
