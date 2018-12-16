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
