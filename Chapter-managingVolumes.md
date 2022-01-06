# 管理GlusterFS卷 

本章描述了如何执行公共的GlusterFS管理操作，包括如下：
+ 调整卷可选项
+ 配置卷传输类型
+ 扩容卷
+ 缩容卷
+ 替换brick
+ 重平衡卷
+ 停止卷
+ 删除卷
+ 复制卷上触发自愈
+ 非统一文件分配（NUFA）  

## 配置卷的传输类型  

卷的客户端和brick进程之间可以支持一个或多个类型的传输协议通信。支持的传输类型有tcp，rdma，和tcp，rdma三种。  

使用如下步骤修改卷支持的协议类型：
1. 使用下面命令卸载卷的所有客户端： 
   `# umount mount-point`  
2. 使用下面命令停止卷：
   `# gluster volume stop <VOLNAME>`  
3. 比如，执行下方命令改变卷的传输类型为同时支持tcp和rdma：
   `# gluster volume set test-volume config.transport tcp,rdma OR tcp OR rdma`  
4. 在客户端挂载卷。比如，通过下方命令通过rdma传输挂载卷。
   `# mount -t glusterfs -o transport=rdma server1:/test-volume /mnt/glusterfs`   

## 扩容卷  

当集群在线并且可用的时候，如果需要，可以扩容卷。比如，你想在分布式卷上添加一个brick，从而增加分布（distribution）并添加GlusterFS卷的容量。  

类似的，你或许想在分布式复制卷上添加一组brick，增加GlusterFS卷的容量。

> **注意**  
> 当你扩容分布式复制卷和分布式分散卷的时候，你需要添加brick的数量是副本或者分散数的整数倍。扩容一个有两个副本的分布式复制卷，你需要添加的brick是2的倍数（比如4，6，8等等） 

**扩容卷**  

1. 如果你还不是TSP（可信任存储池）的一部分，请使用以下命令探测包含要添加到卷中的brick的服务器。  
   `# gluster peer probe <SERVERNAME>`  
   比如：
   ```
   # gluster peer probe server4
     Probe successful
   ```  
2. 使用以下命令添加brick  
   `# gluster volume add-brick <VOLNAME> <NEW-BRICK>`  
   比如：  
   ```
   # gluster volume add-brick test-volume server4:/exp4  
   Add Brick successful
   ```  
3. 使用以下命令检查卷信息：  
   `# gluster volume info <VOLNAME>`  
   结果展示类似如下：
   ```
   Volume Name: test-volume
   Type: Distribute
   Status: Started
   Number of Bricks: 4
   Bricks:
   Brick1: server1:/exp1
   Brick2: server2:/exp2
   Brick3: server3:/exp3
   Brick4: server4:/exp4
   ```  
4. 重平衡卷以保证文件分发到新的brick。
   你可以使用描述在[重平衡卷](#重平衡卷)的rebalance命令。  

## 缩容（shrinking）卷  

当集群是在线且可用的，在需要的时候，你可以缩容卷。比如，你需要移除卷中因为硬件或者网络故障导致不可用的brick。  
> **注意：**  
> 在Gluster挂载点将无法再访问驻留在你正在移除的brick上的数据。注意无论如何这只是移除了配置信息 - 你可以根据需要继续通过brick访问数据。  

当缩容分布式复制卷和分布式分散卷的时候，你需要删除副本数或者条带数整数倍的brick。比如：缩容一个两副本的分布式复制卷，你需要删除2的倍数的brick（比如4，6，8等）。另外，你删除的brick必须来自同一个子卷（同一个副本和分散集）  

使用*start*可选项执行删除brick（remove-brick）将自动触发重平衡操作，迁移来自卷中删除的brick（removed-bricks）的数据到其他brick中。  

**缩容卷**  

1. 使用以下命令删除brick：
   `# gluster volume remove-brick <VOLNAME> <BRICKNAME> start`   
   比如，删除 `server2:/exp2:`  
   ```
   # gluster volume remove-brick test-volume server2:/exp2 start
   volume remove-brick start: success
   ```  
2. 使用以下命令查看删除brick操作的状态:   
   `# gluster volume remove-brick <VOLNAME> <BRICKNAME> status`  
   比如，查看删除 server2:/exp2 brick的操作的状态：
   ```
   # gluster volume remove-brick test-volume server2:/exp2 status
                                Node  Rebalanced-files  size  scanned       status
                           ---------  ----------------  ----  -------  -----------
   617c923e-6450-4065-8e33-865e28d9428f               34   340      162   in progress
   ```  
3. 一但状态显示为"completed"，提交删除brick的操作
   `# gluster volume remove-brick <VOLNAME> <BRICKNAME> commit`  
   这个例子中：
   ```
   # gluster volume remove-brick test-volume server2:/exp2 commit
   Removing brick(s) can result in data loss. Do you want to Continue? (y/n) y
   volume remove-brick commit: success
   Check the removed bricks to ensure all files are migrated.
   If files with data are found on the brick path, copy them via a gluster mount point before re-purposing the removed brick.
   ```  

4. 使用以下命令检查卷信息  
   `# gluster volume info`   

   命令展示的信息和下面类似：

   ```
   # gluster volume info
   Volume Name: test-volume
   Type: Distribute
   Status: Started
   Number of Bricks: 3
   Bricks:
   Brick1: server1:/exp1
   Brick3: server3:/exp3
   Brick4: server4:/exp4
   ```  

## 替换故障brick  

**替换一个*纯*分布式卷中的brick**  

要替换仅分布式卷的brick，先添加新brick再删除你想替换的brick。这将触发重平衡操作把删除的brick上的数据移动出来。  

> 注意：使用'replace-brick'命令替换brick，只支持分布式复制卷或纯分布式卷（此处原文为replicate volumes，译者认为这错了，应该是指分布式卷）。  
  
删除brick Server1:/home/gfs/r2_1 和添加 Server1:/home/gfs/r2_2 的步骤如下： 

1. 这是初始卷配置：  
   ```
   Volume Name: r2
   Type: Distribute
   Volume ID: 25b4e313-7b36-445d-b524-c3daebb91188
   Status: Started
   Number of Bricks: 2
   Transport-type: tcp
   Bricks:
   Brick1: Server1:/home/gfs/r2_0
   Brick2: Server1:/home/gfs/r2_1
   ```  
2. mount出的文件展示如下：
   ```
   # ls
   1  10  2  3  4  5  6  7  8  9
   ```  
3. 添加新的brick - Server1:/home/gfs/r2_2 now:
   ```
   # gluster volume add-brick r2 Server1:/home/gfs/r2_2
   volume add-brick: success
   ```
4. 使用如下命令开始删除brick：  
   ```
   # gluster volume remove-brick r2 Server1:/home/gfs/r2_1 start
   volume remove-brick start: success
   ID: fba0a488-21a4-42b7-8a41-b27ebaa8e5f4
   ```   
5. 等到删除brick状态显示已经完成。
   ```
   # gluster volume remove-brick r2 Server1:/home/gfs/r2_1 status
                                Node Rebalanced-files          size       scanned      failures       skipped               status   run time in secs
                           ---------      -----------   -----------   -----------   -----------   -----------         ------------     --------------
                           localhost                5       20Bytes            15             0             0            completed               0.00
   ```  
6. 现在你可以安全的删除旧brick，所以提交修改：
   ```
   # gluster volume remove-brick r2 Server1:/home/gfs/r2_1 commit
   Removing brick(s) can result in data loss. Do you want to Continue? (y/n) y
   volume remove-brick commit: success
   ```  
7. 这是新卷配置。
   ```
   Volume Name: r2
   Type: Distribute
   Volume ID: 25b4e313-7b36-445d-b524-c3daebb91188
   Status: Started
   Number of Bricks: 2
   Transport-type: tcp
   Bricks:
   Brick1: Server1:/home/gfs/r2_0
   Brick2: Server1:/home/gfs/r2_2
   ```  
8. 检查挂载的内容：  
   ```
   # ls
   1  10  2  3  4  5  6  7  8  9
   ```

**替换副本/分布式副本卷的brick**  

本节描述了在有两个副本的复制卷r2上brick:`Server1:/home/gfs/r2_0`是如何被brick`Server1:/home/gfs/r2_5替换的。  
```
Volume Name: r2
Type: Distributed-Replicate
Volume ID: 24a0437a-daa0-4044-8acf-7aa82efd76fd
Status: Started
Number of Bricks: 2 x 2 = 4
Transport-type: tcp
Bricks:
Brick1: Server1:/home/gfs/r2_0
Brick2: Server2:/home/gfs/r2_1
Brick3: Server1:/home/gfs/r2_2
Brick4: Server2:/home/gfs/r2_3
```  

步骤：  

1. 确保新brick Server1:/home/gfs/r2_5 上没有数据。  
2. 检查除了即将被移除的（这个可以不是运行的）brick都是运行的。
3. 如果将要替换的brick不是下线的，请把它下线
   + 通过执行`gluster volume status`获取brick的PID  
      ```
      # gluster volume status
      Status of volume: r2
      Gluster process                        Port    Online    Pid
      ------------------------------------------------------------------------------
      Brick Server1:/home/gfs/r2_0            49152    Y    5342
      Brick Server2:/home/gfs/r2_1            49153    Y    5354
      Brick Server1:/home/gfs/r2_2            49154    Y    5365
      Brick Server2:/home/gfs/r2_3            49155    Y    5376
      ```   
   + 登录brick运行的机器，然后kill掉brick进程。  
      ```
      # kill -15 5342
      ```  
   + 确认该brick已经下线并且其他的brick正常运行。
      ```
      # gluster volume status
      Status of volume: r2
      Gluster process                        Port    Online    Pid
      ------------------------------------------------------------------------------
      Brick Server1:/home/gfs/r2_0            N/A      N    5342 <<---- brick is not running, others are running fine.
      Brick Server2:/home/gfs/r2_1            49153    Y    5354
      Brick Server1:/home/gfs/r2_2            49154    Y    5365
      Brick Server2:/home/gfs/r2_3            49155    Y    5376
      ```  
4. 使用Gluster卷FUSE挂载（本例中为：/mnt/r2）设置元数据，以便数据可以同步到新的brick（本例中为从`Server1:/home/gfs/r2_1` 到 `Server1:/home/gfs/r2_5`）  
   + 创建一个挂载点不存在的目录。然后删除目录，通过执行setfattr对元数据变更日志（metadata changelog）做相同操作。此操作标记待处理的变更日志，它将告诉自愈 damon/mounts 执行从/home/gfs/r2_1 到 /home/gfs/r2_5的自愈。  
   ```
   mkdir /mnt/r2/<name-of-nonexistent-dir>
   rmdir /mnt/r2/<name-of-nonexistent-dir>
   setfattr -n trusted.non-existent-key -v abc /mnt/r2
   setfattr -x trusted.non-existent-key  /mnt/r2
   ```  
   + 检查等待的xattrs在正在替换的副本brick上。  
   ```
   getfattr -d -m. -e hex /home/gfs/r2_1
   # file: home/gfs/r2_1
   security.selinux=0x756e636f6e66696e65645f753a6f626a6563745f723a66696c655f743a733000
   trusted.afr.r2-client-0=0x000000000000000300000002 <<---- xattrs are marked from source brick Server2:/home/gfs/r2_1
   trusted.afr.r2-client-1=0x000000000000000000000000
   trusted.gfid=0x00000000000000000000000000000001
   trusted.glusterfs.dht=0x0000000100000000000000007ffffffe
   trusted.glusterfs.volume-id=0xde822e25ebd049ea83bfaa3c4be2b440
   ```  
5. 卷治愈信息（heal info）将显示"/"需要治愈。（工作负载上可能有很多的列表，但是"/"必须存在）  
   ```
   # gluster volume heal r2 info
   Brick Server1:/home/gfs/r2_0
   Status: Transport endpoint is not connected

   Brick Server2:/home/gfs/r2_1
   /
   Number of entries: 1

   Brick Server1:/home/gfs/r2_2
   Number of entries: 0

   Brick Server2:/home/gfs/r2_3
   Number of entries: 0
   ```  
6. 使用'commit force'选项替换brick。请注意，不支持replica-brick命令的其他变体。
   + 执行 replace-brick 命令
      ```
      # gluster volume replace-brick r2 Server1:/home/gfs/r2_0 Server1:/home/gfs/r2_5 commit force
      volume replace-brick: success: replace-brick commit successful
      ```  
   + 现在检查新的brick是否在线：
      ```
      # gluster volume status
      Status of volume: r2
      Gluster process                        Port    Online    Pid
      ------------------------------------------------------------------------------
      Brick Server1:/home/gfs/r2_5            49156    Y    5731 <<<---- new brick is online
      Brick Server2:/home/gfs/r2_1            49153    Y    5354
      Brick Server1:/home/gfs/r2_2            49154    Y    5365
      Brick Server2:/home/gfs/r2_3            49155    Y    5376
      ```   
   + 用户可以使用`gluster volume heal [volname] info` 追踪自愈（self-heal）进度，一旦自愈完成，变更日志将被删除。
      ```
      # getfattr -d -m. -e hex /home/gfs/r2_1
      getfattr: Removing leading '/' from absolute path names
      # file: home/gfs/r2_1
      security.selinux=0x756e636f6e66696e65645f753a6f626a6563745f723a66696c655f743a733000
      trusted.afr.r2-client-0=0x000000000000000000000000 <<---- Pending changelogs are cleared.
      trusted.afr.r2-client-1=0x000000000000000000000000
      trusted.gfid=0x00000000000000000000000000000001
      trusted.glusterfs.dht=0x0000000100000000000000007ffffffe
      trusted.glusterfs.volume-id=0xde822e25ebd049ea83bfaa3c4be2b440
      ```  
   + `# gluster volume heal <VOLNAME> info` 命令将显示没有治愈需求。  
      ```
      # gluster volume heal r2 info
      Brick Server1:/home/gfs/r2_5
      Number of entries: 0

      Brick Server2:/home/gfs/r2_1
      Number of entries: 0

      Brick Server1:/home/gfs/r2_2
      Number of entries: 0

      Brick Server2:/home/gfs/r2_3
      Number of entries: 0
      ```  

## 重平衡卷  

使用add-brick命令扩容卷后，你可能需要重平衡服务中的数据。扩容或者缩容卷后创建的新目录将自动平均分配。对所有已经存在的目录，重新平衡布局和/或数据可以自动修复分布式。  

本节介绍了如何重平衡存储环境中的GlusterFS卷，使用以下常见场景。  

+ **修复布局** - 修复布局以使用新的卷拓扑，文件可以分发到新加的节点上。  
+ **修复布局和迁移数据** - 修复布局以使用新的卷拓扑和迁移存在的数据来重平衡卷。   

### 重平衡卷来修复布局改变  

修复布局是必要的，因为布局结构对于给定目录是静态的。即使卷添加了新的brick。在已经存在的目录中新创建的文件将仅存在于原始的brick中。命令`gluster volume rebalance <volname> fix-layout start` 将修复布局信息，所以文件可以被创建在新添加的brick中。此命令发布后，所有缓存的文件的状态信息将重新验证。  

GlusterFS 3.6以后，分配文件给brick将考虑brick的大小。比如，一个20TB大小的brick将会被分配10TB大小brick两倍的文件。在3.6版本以前，两个brick将是同等对待，忽视大小的，将被分配平等的文件份额。    

一个fix-layout重平衡，将仅修复布局变更不迁移数据。如果想迁移存在的数据，使用`gluster volume rebalance <volume> start`命令重平衡服务中的数据。  

**重平衡卷修复布局**  

+ 使用以下命令在任意Gluster服务上开始重平衡操作：  
  `# gluster volume rebalance <VOLNAME> fix-layout start`  
  
  比如：
  ```
  # gluster volume rebalance test-volume fix-layout start
   Starting rebalance on volume test-volume has been successful
  ```  

### 重平衡卷修复布局和迁移数据  

分别使用add-brick命令扩展卷后，你需要重平衡服务中的数据。remove-brick命令将自动触发重平衡。  

**重平衡卷修复布局和迁移已经存在的数据**  

+ 在任何服务器上使用以下命令开始重平衡操作：  
  `# gluster volume rebalance <VOLNAME> start`  
  比如：  
  ```
  # gluster volume rebalance test-volume start
  Starting rebalancing on volume test-volume has been successful
  ```  

+ 在任何服务器上使用以下命令强制迁移操作：  
  `# gluster volume rebalance <VOLNAME> start force`  
  比如：
  ```
  # gluster volume rebalance test-volume start force
   Starting rebalancing on volume test-volume has been successful
  ```  

重平衡操作将尝试跨节点平衡磁盘使用率。因此将跳过那些导致卷更不平衡的文件。这导致链接文件仍然保留在系统中，因此或许导致性能问题。该行为可以被force参数覆盖。  


### 展示重平衡操作的状态  

你可以在需要的时候显示重平衡卷操作的状态信息。  

+ 使用以下命令检查重平衡操作状态  
  `# gluster volume rebalance <VOLNAME> status`  
  比如：
  ```
  # gluster volume rebalance test-volume status
                                Node  Rebalanced-files  size  scanned       status
                           ---------  ----------------  ----  -------  -----------
   617c923e-6450-4065-8e33-865e28d9428f               416  1463      312  in progress
  ```  

  完成重平衡操作的时间依赖于卷中匹配文件大小的文件的数量。继续检查重平衡状态，确认重平衡的文件的数量或者总共扫描的保持增长的文件数量。  

  比如，再次执行状态命令，或许显示结果和下面类似：  
  ```
  # gluster volume rebalance test-volume status
                                Node  Rebalanced-files  size  scanned       status
                           ---------  ----------------  ----  -------  -----------
   617c923e-6450-4065-8e33-865e28d9428f               498  1783      378  in progress
  ```  

  当重平衡完成时，重平衡状态显示如下：
  ```
  # gluster volume rebalance test-volume status
                                Node  Rebalanced-files  size  scanned       status
                           ---------  ----------------  ----  -------  -----------
   617c923e-6450-4065-8e33-865e28d9428f               502  1873      334   completed
  ```  

### 停止正在执行的重平衡操作  

如果需要的话，你可以停止重平衡操作。  

+ 使用以下命令停止重平衡操作：  
  `# gluster volume rebalance <VOLNAME> stop`  
  比如：  
  ```
  # gluster volume rebalance test-volume stop
                                Node  Rebalanced-files  size  scanned       status
                           ---------  ----------------  ----  -------  -----------
   617c923e-6450-4065-8e33-865e28d9428f               59   590      244       stopped
   Stopped rebalance process on volume test-volume
  ```  

## 停止卷  

1. 使用以下命令停止卷：  
   `# gluster volume stop <VOLNAME>`  
   
   比如，停止test-volume卷  
   ```
   # gluster volume stop test-volume
   Stopping volume will make its data inaccessible. Do you want to continue? (y/n)
   ```  

2. 输入`y` 确认操作。命令输出显示如下：  
   `Stopping volume test-volume has been successful`  

## 删除卷  
1. 使用如下命令删除卷： 
   `# gluster volume delete <VOLNAME>`   
   比如，删除test-volume卷
   ```
   # gluster volume delete test-volume
   Deleting volume will erase all information about the volume. Do you want to continue? (y/n)
   ```

2. 输入`y`确认操作。命令输出显示如下：  
   `Deleting volume test-volume has been successful`  

## 复制卷触发自愈（self-heal）  

在复制模块中，以前当brick离线后重新上线时，你必须手动触发自愈，以使所有的副本同步。现在自愈进程主动运行在后台，诊断问题并每十分钟对需要治愈的文件启动自愈。  

你可以查看需要*治愈*的文件列表，当前/以前修复的文件列表，处于脑裂状态的文件列表，你可以手动触发整个卷的自愈或者仅仅需要治愈的文件。  

+ 触发仅治愈需要治愈的文件:  
  `# gluster volume heal <VOLNAME>`  
  比如，触发test-volume卷自愈需要*治愈*的文件:  
  ```
   # gluster volume heal test-volume
   Heal operation on volume test-volume has been successful
  ```  

+ 触发自愈在卷的所有文件:  
  `# gluster volume heal <VOLNAME> full`  
  比如，触发test-volume卷所有文件的自愈：
  ```
   # gluster volume heal test-volume full
   Heal operation on volume test-volume has been successful
  ```

+ 查看需要*治愈*的文件列表:  
  `# gluster volume heal <VOLNAME> info`  
  比如，查看test-volume卷需要*治愈*的文件列表：  
  ```
   # gluster volume heal test-volume info
   Brick server1:/gfs/test-volume_0
   Number of entries: 0

   Brick server2:/gfs/test-volume_1
   Number of entries: 101
   /95.txt
   /32.txt
   /66.txt
   /35.txt
   /18.txt
   /26.txt
   /47.txt
   /55.txt
   /85.txt
   ...
  ```  

+ 查看已经自愈完成的文件列表：
  `# gluster volume heal <VOLNAME> info healed`  
  比如，查看test-volume卷中自愈完成的卷：  
  ```
   # gluster volume heal test-volume info healed
   Brick Server1:/gfs/test-volume_0
   Number of entries: 0

   Brick Server2:/gfs/test-volume_1
   Number of entries: 69
   /99.txt
   /93.txt
   /76.txt
   /11.txt
   /27.txt
   /64.txt
   /80.txt
   /19.txt
   /41.txt
   /29.txt
   /37.txt
   /46.txt
   ...
  ```  

+ 查看特别的自愈失败的卷文件列表：
  `# gluster volume heal <VOLNAME> info failed`  
  比如，查看test-volume卷自愈失败的文件列表：  
  ```
   # gluster volume heal test-volume info failed
   Brick Server1:/gfs/test-volume_0
   Number of entries: 0

   Brick Server2:/gfs/test-volume_3
   Number of entries: 72
   /90.txt
   /95.txt
   /77.txt
   /71.txt
   /87.txt
   /24.txt
   ...
  ```

+ 查看特别的在脑裂状态的卷的文件列表：  
  `# gluster volume heal <VOLNAME> info split-brain`  
  比如，查看test-volume卷处于脑裂状态的文件列表：  
  ```
   # gluster volume heal test-volume info split-brain
   Brick Server1:/gfs/test-volume_2
   Number of entries: 12
   /83.txt
   /28.txt
   /69.txt
   ...

   Brick Server2:/gfs/test-volume_3
   Number of entries: 12
   /83.txt
   /28.txt
   /69.txt
   ...

  ```  

## 非统一文件分配（NUFA）  

NUFA转换器或者非统一文件访问转换器（Non Uniform File Access translator）旨在HPC类型的环境中使用时给予本地驱动器更高的优先级。可以应用为分布式和复制转换器；后一种情况下，如果空间允许，它确保一个副本时本地的。  

当客户端在服务端创建了文件，基于文件名把文件分配给卷的brick。这种分配或许不是理想的，因为对非本地brick或者暴露目录的读/写操作存在高延迟和不必要的网络开销。NUFA确保文件创建在服务本地暴露目录，因此，为服务访问文件降低延迟和节省带宽。对于运行在存储服务的挂载点上的应用同样有效。  

如果本地brick使用完了空间或者达到最小的磁盘空余限制，而不是分配文件给本地brick，如果同个卷的其他brick有充足空间，NUFA分发文件给它们。  

在卷上创建任何数据前应该启用NUFA。  

`# gluster volume set <VOLNAME> cluster.nufa enable`  

**重要**  

下列情形支持NUFA:  

+ 卷在每个服务上只有一个brick  
+ 使用FUSE客户端。**NFS和SMB模式不支持NUFA**  
+ 客户端挂载NUFA-enabled卷必须在可信任存储池中（TSP）  

在使用Unify转换器时，NUFA调度器同样存在，如下：  
```
volume bricks
  type cluster/nufa
  option local-volume-name brick1
  subvolumes brick1 brick2 brick3 brick4 brick5 brick6 brick7
end-volume
```  

#### NUFA附加选项  
+ lookup-unhashed  
  这是一个高级选项，如果文件在与文件名的哈希值匹配的子卷上丢失，则会在所有子卷中查找文件。默认为开启  
+ local-volume-name  
  考虑本地和首选文件创建的卷的名称，默认时搜索与系统主机名匹配的卷。  
+ subvolumes
  这个选项列出'cluster/nufa'卷的子卷，这个转换器要求超过一个自卷。  

## BitRot检测  




