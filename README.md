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

## zookeeper配置快照与事务日志自动清理
在zookeeper中，如果不配置自动清理，zookeeper不会对这两项的数据文件进行清理操作，从而导致将服务器磁盘打满的状况，进而引起依赖zookeeper的相关应用报错而停止工作。  
`autopurge.snapRetainCount=10`需要保留的文件数  
`autopurge.purgeInterval=48`单位：小时，清理频率。保留48小时内的日志  
## 关于nohup 与 &
使用了nohup之后，很多人就这样不管了，其实这样有可能在当前账户非正常退出或者结束的时候，命令还是自己结束了。所以在使用nohup命令后台运行命令之后，需要使用exit正常退出当前账户，这样才能保证命令一直在后台运行。
