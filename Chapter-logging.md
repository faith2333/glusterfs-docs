# GlusterFS服务日志和位置  

下面列出了GlusterFS服务器中基于组件，服务和方法的日志。根据文件系统分层标准（FHS）所有的日志文件都放置在*/var/log*目录下。   

## Glusterd:  

glusterd日志位于`/var/log/glusterfs/glusterd.log`。每个服务一个glusterd日志文件。这个日志文件同样包含快照和用户日志。  


## Gluster命令行：  

gluster命令行日志位于`/var/log/glusterfs/cli.log`。在GlusterFS可信任存储池中节点上执行的Gluster命令存放在`/var/log/glusterfs/cmd_history.log`  


## Brick: 

brick日志位于`/var/log/glusterfs/bricks/<path extraction of brick path>.log`。服务器上的每个brick有一个日志文件。  


## 重平衡：

重平衡日志位于`/var/log/glusterfs/VOLNAME-rebalance.log`。服务器上的每个卷有一个日志文件。  


## 自愈守护进程：  

自愈守护进程日志在`/var/log/glusterfs/glustershd.log`。每个服务一个日志文件。  


## 配额（Quota）：  

`/var/log/glusterfs/quotad.log`是运行在每个节点上的分配守护进程的日志。`/var/log/glusterfs/quota-crawl.log` 无论何时启用配额（quota），就会执行文件系统抓取并将相应的日志存储在这个文件里。`/var/log/glusterfs/quota-mount- VOLNAME.log` 辅助FUSE客户端挂载在Glusterfs的/VOLNAME，相应的客户端日志可以在这个文件里发现。  
`One log file per server (and per volume from quota-mount.`  


## Gluster NFS:  

`/var/log/glusterfs/nfs.log` 每个服务器一个日志文件。


## SAMBA Gluster:  

`/var/log/samba/glusterfs-VOLNAME-<ClientIP>.log`。 如果客户端挂载在GlusterFS服务器节点上，实际日志文件或者挂载点上是找不到的。这个时候，需要考虑所有GlusterFS类型的挂载输出。  


## Ganesha NFS:  

`/var/log/nfs-ganesha.log`  


## FUSE 挂载:  

`/var/log/glusterfs/<mountpoint path extraction>.log`  


## Geo-replication:  

```
/var/log/glusterfs/geo-replication/<primary> /var/log/glusterfs/geo-replication-secondary
```  


## Gluster volume heal VOLNAME info 命令：  

`/var/log/glusterfs/glfsheal-VOLNAME.log`。 命令执行的每个服务器上一个日志文件。  


## Gluster-swift: 

`/var/log/messages`  

## SwiftKrbAuth:

`/var/log/httpd/error_log`