---
title: Extend Oracle Instance Root Volume Size
date: 2022-10-10 21:14:39

---

虽然 OCI Linux 系统默认驱动盘容量为46.6GB，但是可以在OCI中在线扩展引导卷的大小，而不会出现任何停机时间。


<!--more-->

# OCI Instance 系统磁盘大小
从下面的测试结果可知，ubuntu 和 Oracle Linux 7.9 的镜像，创建虚拟机之后，引导卷的大小即使创建虚拟机过程中指定定制的引导卷的大小。而 Oracle Linux 8 与 CentOS，虚拟机创建后均为默认的50GB大小，而不是等于在创建虚拟机过程中指定定制的引导卷的大小。

## 是否支持自动扩容？
测试用例，创建不同镜像的虚拟机，且指定定制的引导卷大小为100GB。创建后，查看磁盘空间信息如下：
### Oracle-Linux-8.6（不自动扩展）
Image: Oracle-Linux-8.6-2022.12.15-0
[opc@instance-20221228-0041 ~]$ df -h /
Filesystem                  Size  Used Avail Use% Mounted on
/dev/mapper/ocivolume-root   36G  7.8G   28G  22% /

### CentOS-7（不自动扩展）
Image: CentOS-7-2022.10.27-0
[opc@instance-20221228-0042 ~]$ df  -h /
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda3        39G  2.3G   36G   6% /

### Ubuntu-22.04（自动扩展）
Image: Canonical-Ubuntu-22.04-2022.11.06-0
ubuntu@instance-20221228-0051:~$ df -h  /
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        97G  2.1G   95G   3% /

### Oracle-Autonomous-Linux-7.9（自动扩展）
Image: Oracle-Autonomous-Linux-7.9-2022.12.16-0
[opc@instance-20221228-0052 ~]$ df -h /
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda3        92G  3.8G   89G   5% /

## 手动扩容
### Oracle Linux 镜像引导卷扩容
针对 Oracle Linux 镜像的虚拟机，请参考[此处](https://docs.oracle.com/en-us/iaas/oracle-linux/oci-utils/index.htm#oci-growfs)使用 oci-growfs 进行在线不停机扩容。

### Cent OS 镜像引导卷的扩容
sudo su -
lsblk
fdisk -l
yum -y install cloud-utils-growpart gdisk
growpart /dev/sda 3
reboot

# OCI instance 创建完成后，手动做引导卷的扩容
## 步骤1: 控制台，调整引导卷大小
(省略)
## 步骤2: 执行命令，重新扫描磁盘空间
sudo dd iflag=direct if=/dev/oracleoci/oraclevda of=/dev/null count=1
echo "1" | sudo tee /sys/class/block/`readlink /dev/oracleoci/oraclevda | cut -d'/' -f 2`/device/rescan

## 步骤3: 调整分区大小
### 针对 Oracle Linux
Linux 7提供了实用程序包 cloud-utils-growpart 来更改大小，但是首先使用下面的命令-检查这个包是否存在。

### 针对CentOS/RHEL/Fedora系统：
```
$ sudo yum -y install cloud-utils-growpart gdisk
$ sudo 
# growpart /dev/sda 3
# sudo xfs_growfs /dev/sda3
```
#### 针对Ubuntu/Debian系统：
```
$ sudo apt install cloud-guest-utils gdisk
```
安装gdisk工具之后，您现在应该可以使用growpart扩展磁盘的大小
```
growpart /dev/sda 1
xfs_growfs /dev/sda1

```
（Option）也可以直接使用下面的命令进行更改（无须安装其他插件）
```
sudo growpart /dev/sda 1
sudo resize2fs /dev/sda1
```

# 挂载大于 2 TB 的 磁盘（fdisk不适合大于2TB的volume）
## 分区
https://erdong.site/tools/parted-create-gpt-partition.html


## 挂载
标签
parted -s -a optimal -- /dev/nvme0n1 mklabel gpt

分区
parted -s -a optimal -- /dev/nvme0n1 mkpart primary 0% 100%

格式化
mkfs.xfs /dev/nvme0n1p1

挂载
mount /dev/nvme0n1p1 /data

参考文档：

https://erdong.site/tools/parted-create-gpt-partition.html


