# 如何编译指定版本的 kernel 成 RPM 安装包

**下载 linux kernel 4.14 包（以 4.14 为例）**

> 从 [https://mirrors.edge.kernel.org/pub/linux/kernel](https://links.jianshu.com/go?to=https%3A%2F%2Fmirrors.edge.kernel.org%2Fpub%2Flinux%2Fkernel) 中找到 4.14.124 并下载

```text
[root@aws-172-20-20-101 kernel]# wget https://mirrors.edge.kernel.org/pub/linux/kernel/v4.x/linux-4.14.124.tar.gz
```

**安装一些编译内核的依赖**

```text
[root@aws-172-20-20-101 kernel]# yum install -y ncurses-devel elfutils-libelf-devel openssl-devel bc
```

**安装 rpm 编译到依赖**

```text
[root@aws-172-20-20-101 kernel]# yum install -y gcc rpm-build rpm-devel rpmlint make python bash coreutils diffutils patch rpmdevtools
```

**解压 kernel 4.14 包**

```text
[root@aws-172-20-20-101 kernel]# tar -zxvf linux-4.14.124.tar.gz
```

**选择配置项，自定义内核编译**

执行 make menuconfig 将会弹出配置选项菜单，可定制化编译的内核模块，如果不打补丁，不做定制化需求，则直接 save 生成 .config 文件后 Exit 即可

```text
[root@aws-172-20-20-101 kernel]# make menuconfig
```

>

![](../../.gitbook/assets/image%20%2820%29.png)

**编译内核并生成 rpm 包**

make rpm 执行会自动生成 \*.spec 文件，编译完后会自动生成 rmp 安装包，编译时间比较长，建议使用配置较大的机器进行编译（4C16G的机器亲测30分钟内可编译完，1C1G一天都够呛），磁盘空间要保持在20G以上

```text
[root@aws-172-20-20-101 kernel]# make rpm &
# 或者
[root@aws-172-20-20-101 kernel]#  make rpm-pkg &
```

编译好后的 rmp 包路径会有提示

```text
[root@aws-172-20-20-101 kernel]#  cd /root/rpmbuild/RPMS/`uname -m`/
[root@aws-172-20-20-101 kernel]#  tree RPMS/
 RPMS/
  └── i386
  ├── kernel-4.14.124.x86_64.rpm
  ├── kernel-devel-4.14.124.x86_64.rpm
  └── kernel-headers-4.14.124.x86_64.rpm
```

**安装新编译好的内核**

编译好后的 rpm 即可随处使用了

```text
[root@aws-172-20-20-101 kernel]#  rpm -Uvh kernel-*-*.rpm
```

**安装完成后设置 4.14 位默认启动项**

```text
[root@aws-172-20-20-101 kernel]#  cat /boot/grub2/grub.cfg |grep menuentry
if [ x"${feature_menuentry_id}" = xy ]; then
  menuentry_id_option="--id"
  menuentry_id_option=""
export menuentry_id_option
menuentry 'CentOS Linux (4.14.124.x86_64) 7 (Core)' --class centos --class gnu-linux --class gnu --class os --unrestricted $menuentry_id_option 'gnulinux-3.10.0-862.3.2.el7.x86_64-advanced-8c1540fa-e2b4-407d-bcd1-59848a73e463' {
menuentry 'CentOS Linux (3.10.0-957.12.2.el7.x86_64) 7 (Core)' --class centos --class gnu-linux --class gnu --class os --unrestricted $menuentry_id_option 'gnulinux-3.10.0-862.3.2.el7.x86_64-advanced-8c1540fa-e2b4-407d-bcd1-59848a73e463' {
menuentry 'CentOS Linux (3.10.0-862.3.2.el7.x86_64) 7 (Core)' --class centos --class gnu-linux --class gnu --class os --unrestricted $menuentry_id_option 'gnulinux-3.10.0-862.3.2.el7.x86_64-advanced-8c1540fa-e2b4-407d-bcd1-59848a73e463' {
menuentry 'CentOS Linux (0-rescue-b30d0f2110ac3807b210c19ede3ce88f) 7 (Core)' --class centos --class gnu-linux --class gnu --class os --unrestricted $menuentry_id_option 'gnulinux-0-rescue-b30d0f2110ac3807b210c19ede3ce88f-advanced-8c1540fa-e2b4-407d-bcd1-59848a73e463’ {
```

**设置默认启动**

```text
[root@aws-172-20-20-101 kernel]#  grub2-set-default 'CentOS Linux (4.14.124.x86_64) 7 (Core)’
```

**验证**

```text
[root@aws-172-20-20-101 kernel]#  grub2-editenv list
saved_entry=CentOS Linux (4.14.124.x86_64) 7 (Core)
```

**重启机器**

```text
[root@aws-172-20-20-101 kernel]#  reboot
*****
[root@aws-172-20-20-101 kernel]#  uname -r
```

