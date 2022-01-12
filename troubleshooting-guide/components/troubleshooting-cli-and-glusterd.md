# 问题诊断终端和glusterd  

## 问题诊断终端和glusterd  

glusterd后台进程运行在每个可信任服务节点上，并负责管理可信任存储池和卷。  

gluster cli发送命令到本地节点上的glusterd后台进程，后者执行操作并将结果返回给用户。  


## 调试glusterd

**日志**  

当你遇到问题，通过查看日志文件以获取导致问题的线索。Gluster日志的默认目录是`/var/log/glusterfs`。cli和glusterd日志如下：
+ glusterd: /var/log/glusterfs/glusterd.log
+ gluster CLI: /var/log/glusterfs/cli.log  

**状态转储（statedumps）**  

statedumps在调试内存泄漏和系统hang住的时候是有用的。查看[Statedump](https://docs.gluster.org/en/latest/Troubleshooting/statedump/)获取更多的细节。  


## 常见问题和解决方法  

***"Another transaction is in progress for volname" or "Locking failed on xxx.xxx.xxx.xxx"***  

由于Gluster是自然分布的，在执行操作的时候glusterd使用锁来保证对于卷的配置修改是原子性的在整个集群中。以下情况返回这些错误：

+ 多个事物争用一个锁  
  > 解决办法：这是一个短暂的错误，一旦重新尝试的时候另外的事务已经完成了，操作将会成功。
+ 一个节点上存在一个过时的节点
  > 解决办法：在过时的锁被清空前重复操作是没用的。重启使用这个锁的glusterd进程。
  + 检查glusterd.log 文件查找哪个节点持有这个过期的锁。查看如下信息：`lock being held by <uuid>`  
  + 执行`gluster peer status` 识别日志信息中该uuid的节点。
  + 在该节点上重启glusterd。  


***"Transport endpoint is not connected"*** **errors but all bricks are up**

这是非常常见的在一个brick进程没有干净的关闭时，保留过期数据在glusterd进程中。Gluster客户端向Glusterd查询brick正在监听的端口并尝试连接到该端口。如果Glusterd中的端口信息时错误的，客户端将不能连接到brick，即使brick是运行的。操作将需要访问brick或许失败并吐出信息"transport endpoint is not connected"。  

*解决办法:* 重启Glusterd服务。  


***"Peer Rejected"***

`gluster peer status` 一个节点返回 "Peer Rejected"  

```
Hostname: <hostname>
Uuid: <xxxx-xxx-xxxx>
State: Peer Rejected (Connected)
```  

这预示着这个节点上的卷配置没有和剩下的可信任存储池同步。你应该在运行peer status命令的节点的glusterd日志上看到如下信息：
```
Version of Cksums <vol-name> differ. local cksum = xxxxxx, remote cksum = xxxxyx on peer <hostname>
```  

*解决办法：* 更新cluster.op-version   
+ 执行`gluster volume get all cluster.max-op-version` 获取最新支持的op-version  
+ 通过执行`volume set all cluster.op-version <op-version>` 更新cluster.op-version到最新支持的op-version  

***"Accepted Peer Request"***  

当扩展集群的时候如果glusterd握手失败，集群的视图将会不一致。命令`gluster peer status`显示远程节点的状态将是"Accepted Peer Request"，随后的控制台命令将会失败，并显示错误比如：   
```
Volume create command will fail with "volume create: testvol: failed: Host <hostname> is not in 'Peer in Cluster' state
```  

在这种情况下，`/var/lib/glusterd/peers/<UUID>` 中的 state 字段的值将不是 3。  

*解决办法：*  
+ 停止glusterd
+ 打开`/var/lib/glusterd/peers/<UUID>`  
+ 改变state值为3
+ 启动glusterd