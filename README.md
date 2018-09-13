## SSH 免密登录
在本机上执行 `   ` ,一路回车之后会生成一个秘钥  
然后执行此命令`ssh-copy-id -i ~/.ssh/id_rsa.pub 用户名@ip`  
  
## 端口占用情况查询
`lsof -i:[端口号]`  
例如：`lsof -i:8080`   

## 添加用户
useradd [用户名]  
## 给新建的用户赋予密码
passwd [用户名]  
## 彻底删除用户
userdel -r [用户名]

## 批量移动特定文件到某个文件夹
ls /opt/*.tar | xargs -n1 -I {} mv {} /opt/installer  

## ！！重要，redis集群配置
当写入大量数据的时候，并且每写入一个数据开启一个链接，之后立即断开，会引发redis链接不上的问题，比如报`unable to connect {ip}` `Cannot assign requested address`这一类的错误
这是由于客户端频繁的连服务器，由于每次连接都在很短的时间内结束，导致很多的TIME_WAIT，以至于用光了可用的端口号。  
因此，可以通过配置linux内核参数来解决这个问题  
写入文件`/etc/security/limits.conf`
`net.ipv4.tcp_syncookies = 1`  
`net.ipv4.tcp_tw_reuse = 1`  
`net.ipv4.tcp_tw_recycle = 1`  
`net.ipv4.tcp_fin_timeout = 30`  
配置说明：
net.ipv4.tcp_syncookies = 1 表示开启SYN Cookies。当出现SYN等待队列溢出时，启用cookies来处理，可防范少量SYN攻击，默认为0，表示关闭  
net.ipv4.tcp_tw_reuse = 1    表示开启重用。允许将TIME-WAIT sockets重新用于新的TCP连接，默认为0，表示关闭  
net.ipv4.tcp_tw_recycle = 1  表示开启TCP连接中TIME-WAIT sockets的快速回收，默认为0，表示关闭  
net.ipv4.tcp_fin_timeout=30修改系統默认的 TIMEOUT 时间  
