## ！！重要，redis集群配置
当写入大量数据的时候，并且每写入一个数据开启一个链接，之后立即断开，会引发redis链接不上的问题，比如报`unable to connect {ip}` `Cannot assign requested address`这一类的错误
这是由于客户端频繁的连服务器，由于每次连接都在很短的时间内结束，导致很多的TIME_WAIT，以至于用光了可用的端口号。  
因此，可以通过配置linux内核参数来解决这个问题  
写入文件  
`/etc/security/limits.conf`
`net.ipv4.tcp_syncookies = 1`  
`net.ipv4.tcp_tw_reuse = 1`  
`net.ipv4.tcp_tw_recycle = 1`  
`net.ipv4.tcp_fin_timeout = 30`  
配置说明：
net.ipv4.tcp_syncookies = 1 表示开启SYN Cookies。当出现SYN等待队列溢出时，启用cookies来处理，可防范少量SYN攻击，默认为0，表示关闭  
net.ipv4.tcp_tw_reuse = 1    表示开启重用。允许将TIME-WAIT sockets重新用于新的TCP连接，默认为0，表示关闭  
net.ipv4.tcp_tw_recycle = 1  表示开启TCP连接中TIME-WAIT sockets的快速回收，默认为0，表示关闭  
net.ipv4.tcp_fin_timeout=30修改系統默认的 TIMEOUT 时间  

# redis中 redis-trib 的使用
参考博客  
http://www.cnblogs.com/PatrickLiu/p/8484784.html?spm=5176.11156381.0.0.415979f5OT4Yjo  
redis-trib基本命令  
```
create          host1:port1 ... hostN:portN
              --replicas <arg>
check           host:port
info            host:port
fix             host:port
              --timeout <arg>
reshard         host:port
              --from <arg>
              --to <arg>
              --slots <arg>
              --yes
              --timeout <arg>
              --pipeline <arg>
rebalance       host:port
              --weight <arg>
              --auto-weights
              --use-empty-masters
              --timeout <arg>
              --simulate
              --pipeline <arg>
              --threshold <arg>
add-node        new_host:new_port existing_host:existing_port
              --slave
              --master-id <arg>
del-node        host:port node_id
set-timeout     host:port milliseconds
call            host:port command arg arg .. arg
import          host:port
              --from <arg>
              --copy
              --replace
```
create：创建集群  
check：检查集群  
info：查看集群信息  
fix：修复集群  
reshard：在线迁移slot  
rebalance：平衡集群节点slot数量  
add-node：将新节点加入集群  
del-node：从集群中删除节点  
set-timeout：设置集群节点间心跳连接的超时时间  
call：在集群全部节点上执行命令  
import：将外部redis数据导入集群  
## create 创建集群
redis-trib.rb  create  [--replicas <arg>]  host1:port1 ... hostN:portN  
Example : `./redis-trib.rb create --replicas 1 192.168.1.68:6001 192.168.1.68:6002 192.168.1.69:6003 192.168.1.69:6004 192.168.1.170:6005 192.168.1.170:6006`  
replicas 参数必须在host:port之前 代表着master主节点有多少个slaver从节点，如果省略，则最少会创建3个master主节点，也意味着redis集群至少要三个节点  
## check 检查集群
redis-trib.rb check host:port 这个地址可以是集群中任意一个地址
## info 查看集群信息
redis-trib.rb info 192.168.1.68:6001  
运行结果  
```
[root@localhost src]# ./redis-trib.rb info 192.168.1.68:6001
192.168.1.68:6001 (8bc31e0e...) -> 20 keys | 5461 slots | 1 slaves.
192.168.1.69:6003 (9726b0ff...) -> 117283 keys | 5462 slots | 1 slaves.
192.168.1.68:6002 (dd009699...) -> 15 keys | 5461 slots | 1 slaves.
[OK] 117318 keys in 3 masters.
7.16 keys per slot on average.
```
## fix 修复集群
redis-trib.rb fix --timeout <arg> host:port  
## reshard 迁移slot
此命令有以下几个子命令
--from <args> 需要迁移的slot来自于哪些源节点,可以指定多个源节点，args参数是节点的node id,也能直接传递all，代表指定集群中所有节点  
--to <arg> 指定接收slot的节点，只能填写一个节点  
--slots <arg> 需要迁移的slot数量
--yes 在打印完reshard计划后等待用户确认
--timeout <arg> 设置migrate命令的超时时间
--pipeline <arg> 定义cluster getkeysinslot命令一次取出的key数量，不传的话使用默认值为10