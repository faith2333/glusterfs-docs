# 异地复制（Geo-replication）故障排除

本章介绍了和GlusterFS异地复制相关最常见的故障诊断场景。  

## 查找日志文件  

对每个异地复制会话，以下三个日志文件是相关的（四个，如果二级是gluster卷）。  

+ **Primary-log-file** - 监控基础卷的进程日志文件
+ **Secondary-log-file** - 初始化二级修改的进程日志文件
+ **Primary-gluster-log-file** - 管理Geo-replication模块使用去监控基础卷的挂载点的日志文件
+ **Secondary-gluster-log-file** -  是二级的对应物  
  
**初级日志文件（Primary-log-file）**  

使用以下命令，获取geo-replication的初级日志文件：  
```
gluster volume geo-replication <session> config log-file
```  

比如：  
```
# gluster volume geo-replication Volume1 example.com:/data/remote_dir config log-file
```  

**二级日志文件（Secondary-log-file）**  

使用如下命令，获取geo-replication的二级日志文件（glusterd必须在二级机器上运行）：  
1. 在初级机器上，使用如下命令：  
   ```
   # gluster volume geo-replication Volume1 example.com:/data/remote_dir config session-owner 5f6e5200-756f-11e0-a1f0-0800200c9a66
   ```  
   显示会话所有者细节。  

2. 在二级节点上，执行如下命令：  
   ```
   # gluster volume geo-replication /data/remote_dir config log-file /var/log/gluster/${session-owner}:remote-mirror.log
   ```  

3. 替换会话所有者细节（步骤1的输出）到步骤2获取日志文件的位置。  
   ```
   /var/log/gluster/5f6e5200-756f-11e0-a1f0-0800200c9a66:remote-mirror.log
   ```  

## 循环记录（Rotating）异地复制日志  

需要的话，超级管理员可以循环记录一个具体的primary-secondary 会话日志文件。当你执行geo-replications的`log-rotate`命令时，使用文件名后缀的当前时间戳备份日志文件，并将信号发送到gsyncd以开始记录新的日志。  

**循环记录异地复制日志文件**  

+ 使用如下命令循环特定的primary-secondary会话日志文件：
  ```
  # gluster volume geo-replication  log-rotate
  ```  
  比如，循环初级`Volume1`和二级`example.com:/data/remote_dir` 日志文件：  
  ```
  # gluster volume geo-replication Volume1 example.com:/data/remote_dir log rotate
  log rotate successful
  ```  
+ 使用如下命令，循环一个初级卷的所有会话的日志文件：
  ```
  # gluster volume geo-replication  log-rotate
  ```   
  比如，循环初级`Volume1`的日志文件：  
  ```
  # gluster volume geo-replication Volume1 log rotate
  log rotate successful
  ```   
+ 使用以下命令循环所有会话的日志文件：
  ```
  # gluster volume geo-replication log-rotate
  ```  
  比如，循环所有会话的日志文件：  
  ```
  # gluster volume geo-replication log rotate
  log rotate successful
  ```   

## 同步未完成  

**描述：** GlusterFS异地复制数据没有完成同步，但是异地复制状态显示是OK。  

**解决办法：** 你可以通过擦除索引和重启GlusterFS异地复制来强制执行全量同步。重启后，GlusterFS异地复制开始同步所有的数据。所有的文件比较校验和（checksum），大数据集时将会消耗时间和资源非常多。  


## 数据同步问题  

**描述：** 异地复制显示状态OK，但是文件没有被同步，仅仅目录和链接被同步了并在日志中显示如下错误信息：  
```
[2011-05-02 13:42:13.467644] E [primary:288:regjob] GMaster: failed to
sync ./some\_file\`
```  

**解决办法：** 异地复制引用远程和本地主机上的3.0.0或者更高版本的rsync。你必须确认安装了要求的版本。  

## 异地复制高频率显示Faulty状态  

**描述：** 异地复制非常频繁的显示faulty并带着回溯信息如下：   
```
2011-04-28 14:06:18.378859] E [syncdutils:131:log\_raise\_exception]
\<top\>: FAIL: Traceback (most recent call last): File
"/usr/local/libexec/glusterfs/python/syncdaemon/syncdutils.py", line
152, in twraptf(\*aa) File
"/usr/local/libexec/glusterfs/python/syncdaemon/repce.py", line 118, in
listen rid, exc, res = recv(self.inf) File
"/usr/local/libexec/glusterfs/python/syncdaemon/repce.py", line 42, in
recv return pickle.load(inf) EOFError
```  

**解决办法：** 这个错误意味着在初级gsyncd模块和二级gsyncd模块间的RPC通信断了，很多因素都可以导致这个错误。检查是否满足以下所有的先决条件：  

+ 在本地和远程主机间执行了合适的免密SSH。
+ FUSE安装在机器上，因为异地复制模块使用FUSE挂载GlusterFS卷来同步数据。
+ 如果二级是一个卷，检查卷是启动的。
+ 如果二级是一个空白目录，确认目录按照要求的权限创建了。
+ 如果GlusterFS3.2或者更高的版本没有安装在默认的位置（在初级节点上）并且已经作为前提被安装在一个自定义位置，请为其配置`gluster-command`并指向具体的位置。
+ 如果GlusterFS3.2或者更高的版本没有安装在默认的位置（在二级节点上）并且已经作为前提被安装在一个自定义位置，请为其配置`gluster-command`并指向具体的位置。

## 中间节点进入Faulty状态  

**描述：** 在级联设置中，中间节点进入失败状态并显示如下日志：  
```
raise RuntimeError ("aborting on uuid change from %s to %s" % \\
RuntimeError: aborting on uuid change from af07e07c-427f-4586-ab9f-
4bf7d299be81 to de6b5040-8f4e-4575-8831-c4f55bd41154
```  

**解决办法：** 级联设置中，中间节点对原始节点是忠诚的。以上日志意味着异地复制模块在原始节点上已经探测到了改变。如果这是一个所需的行为，删除从中间节点初始化会话的配置选项volume-id。