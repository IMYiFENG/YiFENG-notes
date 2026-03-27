# 像安装 Arch 一样通过 debootstrap 手动安装 Debian 13 (Trixie)

这篇教程将带你从零开始，在 Live CD 环境下通过 `debootstrap` 手动打造一个极致纯净的 Debian 13 系统。我们将采用 Btrfs (开启 zstd 压缩) + systemd-boot 引导，并最小化安装 KDE Plasma 桌面与 Kitty 终端。

> **⚠️ 重要提示**：本教程假设你的 NVMe 硬盘为 `/dev/nvme0n1`，已存在的 Windows EFI 分区为 `/dev/nvme0n1p1`，空闲空间用于新建 Debian 根分区 `/dev/nvme0n1p3`。在执行涉及磁盘格式化的命令前，请务必使用 `lsblk` 确认你自己的设备路径，避免误伤数据。

### 第一步：启动 Debian Live CD 并使用 SSH 连接

#### 启动 Debian Live CD

前往 [USTC Open Source Software Mirror](https://mirrors.ustc.edu.cn/)

点击获取发行版镜像，下载 Debian Live CD（任意桌面环境均可），制作启动盘并进入 Live CD 环境

#### SSH 连接 Live CD 环境

打开终端

换源为任意国内镜像站

```bash
sudo nano /etc/apt/source.list
```

将 `deb.debian.org` 替换为任意国内镜像，如 `mirrors.ustc.edu.cn`，`ctrl+o` 保存，`ctrl+x` 退出

更新软件源

```bash
sudo apt update
```

安装 `openssh-server`

```bash
sudo apt install openssh-server
```

修改 ssh 配置文件，允许 root 用户登录

```bash
sudo nano /etc/ssh/sshd_config
```

调整 `PermitRootLogin` 为 `yes`，`PermitEmptyPasswords` 为 `yes`

开启 ssh 服务

```bash
sudo systemctl start sshd
```

在电脑上使用任意工具使用 ssh 连接 Live CD 环境，可以使用 `ip a` 查看自己的 ip 地址

### 第二步：磁盘分区与格式化

#### 在空闲空间创建新的 Linux 分区并格式化为 Btrfs

查看当前分区格式

```bash
lsblk
```

确认好自己的分区名称，进行分区

> ⚠️ 重要提示：这里务必要确认自己的分区名称，作者自己的电脑EFI分区为 `/dev/nvme0n1p1` 要安装Debian 的分区为 `/dev/nvme0n1p3`，务必确认好自己的分区名称再复制粘贴命令，切勿盲目复制粘贴执行！！！

```bash
cfdisk /dev/nvme0n1
```

使用方向键选择 `Free space`，并选择 `New` 回车，输入分区大小后回车确定，然后选择 `Type`，修改类型为 `Linux filesystem` 

#### 分配好空间后，将其格式化并创建子卷

格式化新分区为 btrfs

```bash
mkfs.btrfs -L debian /dev/nvme0n1p3
```

临时挂载以创建子卷

```bash
mount /dev/nvme0n1p3 /mnt

btrfs subvolume create /mnt/@

btrfs subvolume create /mnt/@home

umount /mnt
```

#### 挂载刚刚创建的子卷以及 EFI 分区

挂载根目录子卷（启用 zstd 压缩）

```bash
mount -o noatime,compress=zstd,subvol=@ /dev/nvme0n1p3 /mnt
```

创建必要的挂载点

```bash
mkdir -p /mnt/{home,boot/efi}
```

挂载 home 子卷

```bash
mount -o noatime,compress=zstd,subvol=@home /dev/nvme0n1p3 /mnt/home
```

挂载 EFI 分区（绝对不要格式化它！）

```bash
mount /dev/nvme0n1p1 /mnt/boot/efi
```

### 第二步：安装基础系统与进入 Chroot 环境

我们将借用 Arch Linux 的辅助脚本来简化 `fstab` 的生成。

```bash
apt update

apt install debootstrap arch-install-scripts
```

使用镜像站拉取 Trixie 基础系统

```bash
debootstrap --arch=amd64 trixie /mnt https://mirrors.ustc.edu.cn/debian/
```

生成 fstab 并进入 chroot 环境

```bash
genfstab -U /mnt >> /mnt/etc/fstab

arch-chroot /mnt
```

### 第三步：配置软件源与基础系统环境

#### 配置软件源

进入 chroot 后，首先替换为镜像站，并确保启用 `non-free-firmware`。

```bash
nano /etc/apt/source.list
```

配置软件源

```
deb https://mirrors.ustc.edu.cn/debian/ trixie main non-free-firmware contrib non-free
deb https://mirrors.ustc.edu.cn/debian/ trixie-updates main non-free-firmware contrib non-free
deb https://mirrors.ustc.edu.cn/debian-security/ trixie-security main non-free-firmware contrib non-free
```

更新系统

```bash
apt update
```

#### 配置时区、本地化和主机名

设置时区为上海

```bash
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```

安装并配置 locales

```bash
apt install locales

dpkg-reconfigure locales
```

> 提示：在界面中勾选 `en_US.UTF-8` 和 `zh_CN.UTF-8`，建议将默认语言设为英文，避免 tty 下中文显示乱码。

设置主机名

```bash
echo "debian-trixie" > /etc/hostname
```

设置 root 密码

```bash
passwd
```

### 第四步：固化内核启动参数

> **必须在安装内核和引导器之前执行此步骤**，否则自动生成的引导项会因为缺失 `root=` 参数导致无法开机

使用 `blkid` 获取 Btrfs 根分区的 UUID

```bash
blkid -s UUID -o value /dev/nvme0n1p3
```

将查到的 UUID 写入内核 cmdline 文件

```bash
mkdir -p /etc/kernel

echo "root=UUID=<你的UUID> rootflags=subvol=@ rw quiet" > /etc/kernel/cmdline
```

### 第五步：安装 systemd-boot 与系统内核

系统现在已经知道该如何引导了，可以放心地安装内核和引导器

```bash
apt install systemd-boot

bootctl install --esp-path=/boot/efi
```

安装内核及基础工具（安装内核时会自动读取刚才的 cmdline 生成引导项）

```bash
apt install linux-image-amd64 btrfs-progs openssh-server sudo vim
```

### 第六步：安装网络组件与无线网卡固件

> 为了确保重启后有网可用，必须一次性安装好 NetworkManager 以及必要的 Wi-Fi 认证后端和固件

安装网络管理器、认证后端与硬件开关控制工具

```bash
apt install network-manager wpasupplicant wireless-tools rfkill
```

安装通用固件包

```bash
apt install firmware-linux
```

针对你的网卡型号补充特定闭源固件

```bash
apt install firmware-iwlwifi     # Intel 网卡

apt install firmware-realtek     # Realtek 瑞昱网卡

apt install firmware-atheros     # Atheros 高通网卡

apt install firmware-brcm80211   # Broadcom 博通网卡
```

### 第七步：创建日常用户并配置服务开机自启

创建管理员用户

```bash
useradd -m -G sudo -s /bin/bash <你的用户名>

passwd <你的用户名>
```

启用 NetworkManager 与 SSH 开机自启

```bash
systemctl enable NetworkManager

systemctl enable sshd
```

### 第八步：退出并重启

清理环境并重启进入你的新系统

```bash
exit

umount -R /mnt

reboot
```

重启后可以再次使用工具连接 SSH

### 第九步：在安装桌面前创建系统快照

安装快照核心工具与图形化前端

```bash
sudo apt install snapper btrfs-assistant
```

初始化子卷的快照配置

```bash
sudo snapper -c root create-config /

sudo snapper -c home create-config /home
```

调整快照保留策略（可选）

```
sudo vim /etc/snapper/configs/root

sudo vim /etc/snapper/configs/home
```

创建快照

```bash
snapper -c root create -d "before desktop"

snapper -c home create -d "before desktop"
```

启用自动快照与清理服务

```bash
sudo systemctl enable --now snapper-timeline.timer

sudo systemctl enable --now snapper-cleanup.timer
```

### 第十步：安装 KDE Plasma 与 Kitty 终端

避开臃肿的元软件包，我们只安装最核心的组件来实现现代化的桌面体验。

```bash
sudo apt install kde-plasma-desktop sddm
```

> 如果想要 Wayland 会话，可以额外安装 `plasma-workspace-wayland`

安装音频组件和中文字体

```bash
sudo apt install pipewire-audio pipewire-pulse fonts-noto-cjk
```

安装终端

```bash
sudo apt install konsole
```

> 我个人更喜欢 kitty

启用 SDDM 登录管理器

```bash
sudo systemctl enable sddm
```

重启

```bash
sudo reboot
```
