# pi4 alpine-linux headless installation

## 准备

- uart adapter
- ethernet connection
- sd card
  - mmcblk0p1 fat32 256M
  - mmcblk0p2 ext4/f2fs 200M+

- 下载解压到mmcblk0p1
  ><https://dl-cdn.alpinelinux.org/alpine/v3.14/releases/aarch64/alpine-rpi-3.14.1-aarch64.tar.gz>
- 修改cmdline.txt

  ```ini
  modules=loop,squashfs,sd-mod,usb-storage earlycon=uart8250,mmio32,0xfe215040 loglevel=7 console=ttyS0,115200
  ```

- 创建usercfg.txt

  ```ini
  enable_uart=1
  gpu_mem=32
  ```

## 安装

- 联网

  ```shell
  setup-interfaces
  setup-ntp
  ```

- 挂载mmcblk0p2

  ```shell
  mount /dev/mmcblk0p2 /mnt
  ```

- 安装基础包到mmcblk0p2

  ```shell
  apk -X https://mirrors.bfsu.edu.cn/alpine/latest-stable/main/ -p /mnt/ --initdb -U --allow-untrusted \
  add alpine-base linux-rpi4 e2fsprogs f2fs-tools util-linux dosfstools rng-tools
  ```

- 重新挂载mmcblk0p1

  ```shell
  mount -o remount,rw /media/mmcblk0p1
  ```

- 复制基础包中的kernel和initramfs到mmcblk0p1

  ```shell
  cp /mnt/boot/*-rpi4 /media/mmcblk0p1/
  ```

- 修改cmdline.txt

  ```ini
  earlycon=uart8250,mmio32,0xfe215040 loglevel=7 console=ttyS0,115200 root=/dev/mmcblk0p2 rootwait fsck.repair=yes
  ```

- 修改usercfg.txt

  ```ini
  enable_uart=1
  gpu_mem=32
  [pi4]
  enable_gic=1
  kernel=vmlinuz-rpi4
  initramfs initramfs-rpi4
  ```

- chroot

  ```shell
  chroot /mnt /bin/sh
  ```

- 基础服务

  ```shell
  for s in devfs procfs sysfs dmesg mdev hwdrivers; do rc-update add $s sysinit;done

  for s in swclock hostname modules sysctl bootmisc syslog rfkill rngd networking;do rc-update add $s boot;done

  for s in savecache killprocs mount-ro;do rc-update add $s shutdown;done
  ```

- 修改密码

  ```shell
  passwd
  ```

- 启用ttyS0

  ```shell
  #/etc/inittab
  ttyS0::respawn:/sbin/getty -L ttyS0 115200 vt100

  OR

  #https://github.com/OpenRC/openrc/blob/master/agetty-guide.md
  cd /etc/init.d && ln -s agetty agetty.ttyS0
  rc-update add agetty.ttyS0 boot
  ```

- 退出chroot并重启

  ```shell
  exit
  reboot
  ```

## post-install

- /boot

  ```shell
  # 清空mmcblk0p1只保留txt文件
  # 移动/boot内文件到mmcblk0p1
  # 之后将mmcblk0p1挂载为/boot
  mkdir -p /media/mmcblk0p1
  mount /dev/mmcblk0p1 /media/mmcblk0p1
  cp /media/mmcblk0p1/*.txt /boot/
  rm -rf /media/mmcblk0p1/*
  rm /boot/boot
  mv /boot/* /media/mmcblk0p1/
  echo -e '/dev/mmcblk0p1 /boot vfat defaults 0 0\n' >/etc/fstab
  reboot
  ```

- 开局

  ```shell
  setup-interfaces
  setup-ntp
  setup-apkrepos
  setup-timezone
  ```

- 疑似headless用不上的模块

  ```ini
  blacklist videodev
  blacklist videobuf2_common
  blacklist videobuf2_v4l2
  blacklist v4l2_mem2mem
  blacklist bcm2835_v4l2
  blacklist bcm2835_isp
  blacklist bcm2835_codec
  blacklist videobuf2_vmalloc
  blacklist videobuf2_dma_contig
  blacklist videobuf2_memops
  blacklist rpivid_mem
  blacklist bcm2835_mmal_vchiq
  blacklist mc
  blacklist vc_sm_cma
  ```

## backup/restore rootfs

```shell
sudo tar --acls --xattrs --xattrs-include='*' -cpf root.tar -C /mntpoint . #<--there's a dot
sudo tar --acls --xattrs --xattrs-include='*' -xpf root.tar -C /mntpoint/
```
