# Archlinux安装配置指南

> author: satoru_filkli
>
> 多源于ArchWiki及个人安装配置经验
> 等下次有空优化一下，网站：arch-satoru.vercel.app



## Part.1 安装前的准备

### 1.1 准备安装映像

>  略

### 1.2 启动到 Live 环境

>  略

### 1.3 验证引导模式

要验证引导模式，请用下列命令列出 efivars 目录：

```bash
$ ls /sys/firmware/efi/efivars
```

如果命令结果显示了目录且没有报告错误，则系统以 UEFI 模式引导。 如果目录不存在，则系统可能以 BIOS 模式 (或 CSM 模式) 引导。如果系统未以您想要的模式引导启动，请参考您的主板说明书。

### 1.4 连接到互联网

要在 Live 环境中配置网络连接，请跟随以下步骤：

- 确保系统已经启用了网络接口，用 ip-link 检查：

```bash
$ ip link
```

- 对于无线局域网（Wi-Fi）和无线广域网（WWAN），请确保网卡未被 rfkill 禁用。

- 要连接到网络：
  - 有线以太网 —— 连接网线。
  - WiFi —— 使用 iwctl 验证无线网络。
  
- 配置网络连接：
  - DHCP：对于有线以太网、无线局域网（WLAN）和无线广域网（WWAN）网络接口来说，动态 IP 地址和 DNS 服务器分配（由 systemd-networkd 和 systemd-resolved 提供机能）能够开箱即用。

- 用 ping 检查网络连接：

- ```bash
  ping www.baidu.com
  ```

### 1.5 更新系统时间

使用 timedatectl 确保系统时间是准确的：

```bash
$ timedatectl set-ntp true
```

可以使用 *timedatectl status* 检查服务状态。

### 1.6 建立硬盘分区

系统如果识别到磁盘，就会将其分配为一个块设备，如 /dev/sda、/dev/nvme0n1 或 /dev/mmcblk0。可以使用 lsblk 或者 fdisk 查看：

```bash
$ fdisk -l
```

结果中以 rom、loop 或者 airoot 结尾的设备可以被忽略。

对于一个选定的设备，以下分区是必须要有的：

- 一个根分区（挂载在 根目录）/；
- 要在 UEFI 模式中启动，还需要一个 EFI 系统分区。

请使用 fdisk 或 parted 修改分区表。例如：

```bash
$ fdisk /dev/the_disk_to_be_partitioned（要被分区的磁盘）
```

### 1.7 格式化分区

创建分区后，必须使用合适的文件系统对每个新创建的分区进行格式化。

例如，要在根分区 /dev/root_partition 上创建一个 Ext4 文件系统，请运行：

```bash
$ mkfs.ext4 /dev/root_partition（根分区）
```

如果创建了交换分区，请使用 mkswap 将其初始化：

```bash
$ mkswap /dev/swap_partition（交换空间分区）
```

如果你要创建一个 EFI 系统分区，使用 mkfs.fat 将其格式化为 Fat32。

> **警告： 只有在分区步骤中创建 EFI 系统分区时才需要格式化。如果这个磁盘上已经有一个 EFI 系统分区了，将它重新格式化会破坏其他已安装操作系统的引导加载程序**。

```bash
$ mkfs.fat -F 32 /dev/efi_system_partition
```

### 1.8 挂载分区

将根磁盘卷挂载到 /mnt，例如：

```bash
$ mount /dev/root_partition（根分区） /mnt
```

然后使用 mkdir 创建其他剩余的挂载点（比如 /mnt/efi）并挂载其相应的磁盘卷。

对于 UEFI 系统，挂载 EFI 系统分区：

```bash
$ mount /dev/efi_system_partition /mnt/boot
```

如果创建了交换空间卷，请使用 swapon 启用它：

```bash
$ swapon /dev/swap_partition（交换空间分区）
```

稍后 genfstab 将自动检测挂载的文件系统和交换空间。

## Part.2 安装

### 1.1 选择镜像

文件 */etc/pacman.d/mirrorlist* 定义了软件包会从哪个镜像下载。在 LiveCD 启动的系统上，在连接到互联网后，reflector 会通过选择 20 个最新同步的 HTTPS 镜像并按下载速率对其进行排序来更新镜像列表。

在列表中越前的镜像在下载软件包时有越高的优先权。您或许想检查一下文件，看看是否满意。如果不满意，可以相应的修改 */etc/pacman.d/mirrorlist* 文件，并将地理位置最近的镜像源挪到文件的头部，同时也应该考虑一些其他标准。

这个文件接下来还会被 pacstrap 拷贝到新系统里，所以请确保设置正确。

### 1.2 安装必需的软件包

使用 pacstrap 脚本，安装 base 软件包和 Linux 内核以及常规硬件的固件：

```bash
$ pacstrap /mnt base linux linux-firmware
```

>提示：
>
>- 可以将 linux 替换为内核页面中介绍的其他内核软件包。
>- 在虚拟机或容器中安装时，可以不安装固件软件包。

base 软件包并没有包含 Live 环境中的全部程序。因此要获得一个功能齐全的基本系统，可能需要安装其他软件包。特别要考虑安装：

- 管理所用文件系统的用户工具（比如 XFS 和 Btrfs 对应的管理工具）；
- 访问RAID或LVM分区的工具；
- 未包含在 linux-firmware 中的额外固件(如用于声卡的sof-firmware)；
- 联网所需要的程序 (如网络管理器或DHCP客户端)；
- 文本编辑器（如：nano、vim 等）
- 访问 man 和 info 页面中文档的工具：man-db，man-pages 和 texinfo。

要安装其他软件包或软件包组（比如 base-devel），请将它们的名字追加到上文的 pacstrap 命令后 (用空格分隔)，或者也可以在 Chroot 进新系统后使用 pacman 手动安装软件包或软件包组。

## Part.3 配置系统

### 1.1 Fstab

用以下命令生成 fstab 文件 (用 *-U* 或 *-L* 选项设置 UUID 或卷标)：

```bash
$ genfstab -U /mnt >> /mnt/etc/fstab
```

**强烈建议**在执行完以上命令后，检查一下生成的 */mnt/etc/fstab* 文件是否正确。

### 1.2 Chroot

chroot 到新安装的系统：

```bash
$ arch-chroot /mnt
```

### 1.3 时区

要设置时区：

```bash
$ ln -sf /usr/share/zoneinfo/Region（地区名）/City（城市名） /etc/localtime
```

> 以要设置为上海时区为例，请运行 *# ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime*

然后运行 hwclock 以生成 /etc/adjtime：

```bash
$ hwclock --systohc
```

这个命令假定已设置硬件时间为 UTC 时间。

### 1.4 本地化

程序和库如果需要本地化文本，都依赖区域设置，后者明确规定了地域、货币、时区日期的格式、字符排列方式和其他本地化标准。

需在这两个文件设置：*locale.gen* 与 *locale.conf*。

编辑 */etc/locale.gen*，然后取消掉 *en_US.UTF-8 UTF-8* 和其他需要的 区域设置 前的注释（#）。

接着执行 *locale-gen* 以生成 locale 信息：

```bash
$ locale-gen
```

然后创建 locale.conf文件，并 编辑设定 LANG 变量，比如：

```bash
创建路径：/etc/locale.conf
内容：LANG=en_US.UTF-8
```

> **警告： 不推荐在此设置任何中文 locale，会导致 tty 乱码**。

### 1.5 网络配置

创建 hostname 文件:

```bash
创建路径：/etc/hostname
内容：myhostname（主机名）
```

### 1.6 Root 密码

设置 Root 密码：

```bash
$ passwd
```

### 1.7 安装Microcode

根据处理器，安装以下软件包：

- AMD 处理器安装 amd-ucode。
- Intel 处理器安装 intel-ucode。

### 1.8 安装引导程序

首先安装软件包 grub 和 efibootmgr。其中“GRUB”是启动引导器，“efibootmgr”被 GRUB 脚本用来将启动项写入 NVRAM。

执行下面的命令来将 GRUB EFI 应用 grubx64.efi 安装到 esp/EFI/GRUB/，并将其模块安装到 /boot/grub/x86_64-efi/。

```bash
$ grub-install --target=x86_64-efi --efi-directory=esp --bootloader-id=GRUB
```

上述安装完成后 GRUB 的主目录将位于 /boot/grub/。注意上述例子中，grub-install 还将在固件启动管理器中创建一个条目，名叫 GRUB。如果你的启动项已满，这个命令会执行失败。你需要使用 efibootmgr 来删除不必要的条目。

使用 grub-mkconfig 工具来生成 */boot/grub/grub.cfg*：

```bash
$ grub-mkconfig -o /boot/grub/grub.cfg
```

##### 探测其他操作系统

想要让 grub-mkconfig 探测其他已经安装的系统并自动把他们添加到启动菜单中，安装 软件包 os-prober 并 挂载 包含其它系统引导程序的磁盘分区。然后重新运行 grub-mkconfig。如果你得到以下输出：*Warning: os-prober will not be executed to detect other bootable partitions*，你需要编辑*/etc/default/grub*并取消下面这一行的注释，如果没有相应注释的话就在文件末尾添加上：

```bash
GRUB_DISABLE_OS_PROBER=false
```

然后运行 grub-mkconfig 再试一次。

> 注意：
>
> - 分区挂载点并不重要，os-prober读取mtab信息来确认并搜索引导程序的位置。
> - 记得每次运行 grub-mkconfig 之前都把包含其他操作系统引导程序的分区挂载上，以免这些操作系统的启动项丢失。



> **警告： 这是安装的最后一步也是至关重要的一步，请按上述指引正确安装好引导加载程序后再重新启动。否则重启后将无法正常进入系统**。

### 1.9 重启

输入 *exit* 或按 *Ctrl+d* 退出 chroot 环境。

可选用 *umount -R /mnt* 手动卸载被挂载的分区：这有助于发现任何「繁忙」的分区，并通过 fuser 查找原因。

最后，通过执行 *reboot* 重启系统，systemd 将自动卸载仍然挂载的任何分区。不要忘记移除安装介质，然后使用 root 帐户登录到新系统。

## Part.4 安装后的工作

##### 添加新用户

```bash
$ useradd -m -g users <用户名>
$ vim /etc/sudoers
# 在root ALL=(ALL)ALL下面添加 <用户名>ALL=(ALL)ALL，保存退出
```



##### 安装桌面环境

所有桌面环境都需要依赖xorg。所以先要安装xorg组。

```bash
$ pacman -S xorg
```

安装xfce4桌面和附带的软件包：

```bash
$ pacman -S xfce4 xfce4-goodies
```

安装LightDM登录管理器(显示管理器)

```bash
$ pacman -S lightdm lightdm-gtk-greeter
```

> 其配置文件为：*/etc/lightdm/lightdm.conf*

```bash
$ systemctl enable lightdm.service
```

可重启进入xfce4图形界面，然后在图形界面中使用终端来继续以下配置步骤.

##### 安装显卡驱动

查看显卡型号

```bash
$ lspci -k | grep -A 2 -E "(VGA|3D)"
```

``` bash
$ sudo pacman -S nvidia 
$ sudo pacman -S xf86-video-intel
```

##### 安装中文字体

```bash
$ pacman -S wqy-microhei
```

##### 软件安装

添加archlinuxcn源

*sudo vim /etc/pacman.conf*  在末尾添加一个或多个archlinuxcn源：

```bash
[archlinuxcn]
Server = https://mirrors.zju.edu.cn/archlinuxcn/$arch
Server = https://mirrors.ustc.edu.cn/archlinuxcn/$arch
```

保存退出。

```bash
$ pacman -Syy
$ pacman -S archlinuxcn-keyring
$ pacman -S archlinuxcn-mirrorlist-git
```

```bash
$ pacman -S yay
$ yay -S google-chrome
$ yay -S icalingua++
$ yay -S LibreOffice
$ yay -S yesplaymusic
$ yay -S v2raya v2ray
$ yay -S motrix
$ yay -S ulauncher
$ yay -S redshift
$ yay -S putty
$ yay -S telegram-desktop
$ yay -S lightdm-gtk-greeter-settings
```

###### fcitx5输入法

```bash
$ sudo pacman -S fcitx5-im fcitx5-rime
$ yay -S fcitx5-pinyin-zhwiki-rime
```

编辑 /etc/environment 并添加以下几行，然后重新登录: 

```bash
GTK_IM_MODULE=fcitx
QT_IM_MODULE=fcitx
XMODIFIERS=@im=fcitx
SDL_IM_MODULE=fcitx
GLFW_IM_MODULE=ibus
```

想要 fcitx5 开机自启，执行

```bash
$ cp /usr/share/applications/org.fcitx.Fcitx5.desktop ~/.config/autostart/
```

安装fcitx5主题

```bash
$ git clone https://github.com/tonyfettes/fcitx5-nord.git
$ mkdir -p ~/.local/share/fcitx5/themes/
$ cd fcitx5-nord
$ cp -r Nord-Dark/ Nord-Light/ ~/.local/share/fcitx5/themes/
```

在 *~/.config/fcitx5/conf/classicui.conf* 中, 添加：

```bash
Theme=Nord-Dark
# or
Theme=Nord-Light
```

重启fcitx5

```bash
$ fcitx5 -r
```

###### zsh安装配置
```bash
$ sudo pacman -S zsh
$ chsh -s /bin/zsh
$ sh -c "$(curl -fsSL https://gitee.com/mirrors/oh-my-zsh/raw/master/tools/install.sh)"
$ git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
$ git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting 

在 *.zshrc* 中修改
plugins=(
    git
    zsh-autosuggestions
    zsh-syntax-highlighting
)

```
###### redshift配置

```bash
$ sudo vim ~/.config/redshift/redshift.conf

[redshift]
temp-day=5700
temp-night=4000
fade=1
gamma=0.8
location-provider=manual
adjustment-method=randr
[manual]
lat=39.56
lon=116.17
```
