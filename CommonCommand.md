# 使用到的Linux命令

`fdisk -l` 列出所有分区表  
`mount /dev/sdb4 /mnt/usbfile` 挂载分区至指定目录，此处，挂载`/dev/sdb4`分区到目录`/mnt/usbfile`目录，此处的sdb4是U盘的分区  
`umount /mnt/usbfile` 卸载挂载分区  
`rpm -qa` 列出已安装的软件包  
`rpm -e --nodeps {包名}` 删除软件包，但不删除依赖  
`timedatectl | grep 'Time zone'`查看时区  
`date -s 2018/12/06` 设置年月日  
`date -s 19:00:00` 设置时分秒  
`hwclock -w` 将当前时间写入硬件，避免重启失效  
`ssh-keygen -t rsa -P ''`生成RSA秘钥  
`hostname set-hostname cdh01` 设置主机名(Centos)  
`tar -zxvf {包名} -C {解压路径}`  解压  
`grep 'pattern' filepath` 在一批文件中查找匹配的字符串  
`route add default gw 192.168.0.1 dev wlo1` 添加默认路由(重启后失效)  
`route add -net 192.168.0.0 netmask 255.255.255.0 gw 192.168.1.1 dev wlo1` 添加网络路由(重启后失效) tips:删除的话将add替换为del即可  
`dhclient wlo1` 使用dhcp为wlo1网卡获得IP地址  
`chown [OPTION]... [OWNER][:[GROUP]] FILE...` 更改文件的属主和属组  
`mkdir -p /usr/bin/test/s`  创建目录，如果父级目录没创建则会一并创建  
`useradd -d /app/hadoop hadoop` 创建hadoop用户，并制定用户主目录为`/app/hadoop`  
`ssh-copy-id -i ~/.ssh/id_rsa.pub 用户名@ip`  讲自己的公钥传到目标机器，达到免密的作用  
`ls /opt/*.tar | xargs -n1 -I {} mv {} /opt/installer `批量移动特定文件到某个文件夹  
`awk '/queueTime/{gsub(/^.*queueTime\s*/,"queueTime");print $0}' counter.backup`  提取hbase日志中的命令响应时间情况
```text
PUT value: "{\"9351000\":\"\346\210\220\345\212\237\"}" } stale: false partial: false } queueTime: 1 processingTime: 76 totalTime: 77
```
`sed -n '1341,1655p' file > result` 将file文件中的第1341行至1655行的内容存入result中

## 创建LVM
> 背景：Centos下有3块硬盘，每个20G，现在都要将这三块硬盘都挂载在app下，因此，需要使用LVM将三块硬盘合在一起

使用**lvm2**进行相关操作，`yum install lvm2 -y`  
1、先使用`fdisk -l`确定三块硬盘的名字，这里分别是`/dev/vdb /dev/vdc /dev/vdd`
2、`pvcreate`命令 用于将物理硬盘分区初始化为物理卷，以便LVM使用  
使用该命令来创建物理卷`pvcreate /dev/vd{b,c,d}`  
3、`vgcreate`命令用于创建LVM卷组创建并物理卷加入卷组  
使用该命令来创建卷组`vgcreate hadoop /dev/vdb /dev/vdc /dev/vdd` 其中hadoop是卷组名  
4、`lvcreate`命令用于在LVM卷组上创建逻辑卷  
使用该命令来创建逻辑卷`lvcreate -n hadooplv -l 100%VG hadoop` 其中hadooplv是逻辑卷的名字，-l 后跟百分数表示分配卷组空间的百分比给这个逻辑卷，hadoop是物理卷名，表示在此卷组上分配逻辑卷  
5、`mkfs.ext4`命令用于格式化逻辑卷为ext4格式  
使用该命令格式化逻辑卷hadooplv，`mkfs.ext4 -m 0.01 /dev/hadoop/hadooplv`,此处，`-m`表示调整保留空间为总空间的0.01% ，`/dev/hadoop/hadooplv`是需要格式化的逻辑卷名。  
在ext4分区中，如果未指定`-m`参数，则默认是设置保留空间为总空间的**5%**  
6、使用`mount`命令将新创建的逻辑卷挂载到指定目录下`mount /dev/hadoop/hadooplv /opt`  
7、 blkid是查看卷的唯一UUID，用于开启自动挂载  
8、编辑`/etc/fstab`文件，将新挂载的硬盘添加进去，就能开机挂载`UUID=f68612d6-bf27-46d3-81da-1a5f395e9541  /opt     ext4   defaults    0 0`
该文件内第一列是卷的UUID  
第二列是挂载点  
第三列是此分区的文件系统类型  
第四列是挂载的选项  
> auto: 系统自动挂载，fstab默认就是这个选项  
defaults: rw, suid, dev, exec, auto, nouser, and async.  
noauto 开机不自动挂载  
nouser 只有超级用户可以挂载  
ro 按只读权限挂载  
rw 按可读可写权限挂载  
user 任何用户都可以挂载  
请注意光驱和软驱只有在装有介质时才可以进行挂载，因此它是noauto

第五列是dump备份设置,当其值设置为1时，将允许dump备份程序备份；设置为0时，忽略备份操作  
第六列是fsck磁盘检查设置，其值是一个顺序。当其值为0时，永远不检查；而 / 根目录分区永远都为1。其它分区从2开始，数字越小越先检查，如果两个分区的数字相同，则同时检查  