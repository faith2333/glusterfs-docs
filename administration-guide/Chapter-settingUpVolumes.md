# 创建卷
**卷是brick(信任存储池(truseted storage pool)中的服务暴露的目录)的逻辑组合。在你的存储环境里创建新卷，需要指定组成卷的brick。创建卷后，必须要先启动才能尝试挂载。**  
  
如何创建brick请查看[挂载存储](https://docs.gluster.org/Administrator-Guide/setting-up-storage/)  
## 卷类型
+ 如下类型的卷可以在你的存储环境中创建
  + **分布式卷（Distributed）**   分布式卷分发文件到卷中的所有brick中。在你需要可扩展存储且冗余是不重要或者已经由其他硬件或者软件提供了冗余的时候，选择使用分布式卷。
  + **复制卷（Replicated）**   复制卷跨卷的多个brick创建文件副本。你可以在对高可用（high-availability）和高可靠（high-reliability）至关重要的场景下使用复制卷。
  + **分布式复制卷（Distributed Replicated）**  分布式副本卷分发文件到卷的整个副本brick中。你可以在需要可扩展存储且高可靠性（high-reliability）至关重要的环境中使用分布式复制卷。在大多数场景下，他可以提供读的性能提升。
  + **分散卷（Dispersed）**  分散卷基于纠删码（erasure code），提供节省空间且防止磁盘和服务故障导致数据丢失的能力。它将原始文件的编码碎片存储到每个brick中，只需碎片的子集即可恢复原始文件。在不丢失数据访问权限的情况下可以丢失的brick数量由管理员在创建卷时配置。
  + **分布式分散卷（Distributed Dispersed）**  分布式分散卷在卷的分散卷子卷中分发文件。和分布式副本卷有同样的优，但是使用分散卷在brick中存储数据。
    
**创建卷**

+ 创建一个新的卷
    ```
    #gluster volume create <NEW-VOLNAME> [[replica <COUNT> [arbiter <COUNT>]]|[replica 2 thin-arbiter 1]] [disperse [<COUNT>]] [disperse-data <COUNT>] [redundancy <COUNT>] [transport <tcp|rdma|tcp,rdma>] <NEW-BRICK> <TA-BRICK>... [force]
    ```  
    例如，创建一个叫test-volume的卷，由server3:/exp3 和 server4:/exp4 组成:  
    ```
    # gluster volume create test-volume server3:/exp3 server4:/exp4
    Creation of test-volume has been successful
    Please start the volume to access data.
    ```  

## 创建分布式卷
分布式卷中，文件被随机的分布到到卷的brick中。在需要可扩展存储且冗余不重要或已经由其他硬件/软件层提供的场景下，你可以使用分布式卷。  
> *注意：基于目录和内容是随机的分布在卷的brick中，磁盘/服务故障会导致严重的数据丢失*  

![Distributed Volume](../images/distributed-volume.png)  
**创建分布式卷**  
1. 创建受信任存储池（trusted storage pool）
2. 创建分布式卷
```
#gluster volume create [transport tcp | rdma | tcp,rdma
```  

例：创建一个有四个存储服务使用tcp协议传输的分布式卷。
```
#gluster volume create test-volume server1:/exp1 server2:/exp2 server3:/exp3 server4:/exp4
```  
（可选）你可以使用如下查看卷的信息  
```
#gluster volume info
Volume Name: test-volume
Type: Distribute
Status: Created
Number of Bricks: 4
Transport-type: tcp
Bricks:
Brick1: server1:/exp1
Brick2: server2:/exp2
Brick3: server3:/exp3
Brick4: server4:/exp4
```  
例：创建一个有四个存储服务使用InfiniBand协议传输的分布式卷。
```
# gluster volume create test-volume transport rdma server1:/exp1 server2:/exp2 server3:/exp3 server4:/exp4
Creation of test-volume has been successful
Please start the volume to access data.
```  
如果传输类型没有被指定，tcp将作为默认类型。需要的话，你也可以设置其他选项，比如auth.allow 或 auth.reject。  
> *注意：确保在尝试挂载卷之前启动卷，否则挂载后的客户端操作将挂起。*
  
## 创建复制卷  

复制卷跨卷的多个brick创建文件副本。你可以在对高可用（high-availability）和高可靠（high-reliability）至关重要的场景下使用复制卷。  
> *注意：复制卷的brick数量应该和其副本数相等。为了避免磁盘和服务故障，建议brick来自不同的服务节点。*
![Replicated Volume](../images/replicated-volume.png)  
**创建复制卷**  
1. 创建信任存储池
2. 创建复制卷
    ```
    # gluster volume create [replica ] [transport tcp | rdma | tcp,rdma]
    ```  
    例：创建具有两个存储服务的复制卷
    ```
    # gluster volume create test-volume replica 2 transport tcp server1:/exp1 server2:/exp2
    Creation of test-volume has been successful
    Please start the volume to access data.
    ```  
    如果传输类型没有被指定，tcp将作为默认类型。需要的话，你也可以设置其他选项，比如auth.allow 或 auth.reject。
    > *注意：*
    > *确保在尝试挂载卷之前启动卷，否则挂载后的客户端操作将挂起。*
    > *如果一个副本集上超过一个brick在同一个远程节点上，GlusterFS将无法创建复制卷。如下所示，一个四节点的复制卷，其一个副本集中超过一个brick在同一个远程节点上*
    > ```
    > # gluster volume create <volname> replica 4 server1:/brick1 server1:/brick2 server2:/brick3 server4:/brick4
    > volume create: <volname>: failed: Multiple bricks of a replicate volume are present on the same server. This setup is not optimal. Use 'force' at the end of the command if you want to override this behavior.
    > ```  
    > *如果你仍然想使用这样的配置来创建卷，可以在命令的结尾使用 **force** 参数*  
    
    **复制卷的仲裁配置**
    仲裁卷是第三个brick做为仲裁brick的三副本复制卷。这种配置具有预防脑裂发生的机制。  
    可以使用如下命令创建仲裁卷：  
    ```
    # gluster volume create <VOLNAME> replica 2 arbiter 1 host1:brick1 host2:brick2 host3:brick3`
    ```  
    更多关于这种配置的信息可以在[Administrator-Guide : arbiter-volumes-and-quorum](https://docs.gluster.org/Administrator-Guide/arbiter-volumes-and-quorum/) 中查看。  
    注意：3副本的仲裁配置同样可以创建分布式复制卷  

## 创建分布式复制卷  
在卷中跨复制brick分发文件。你可以在对可扩展存储（scale storage）和好可靠（high-reliability）是至关重要的场景使用分布式复制卷。在大多数场景下，分布式复制卷也能提供读性能的提升。
> *注意：分布式复制卷的brick数量应该是副本数量的倍数。此外，指定brick时的顺序会对数据保护产生巨大影响。在你指定的brick列表中，每一个副本数（replica_count）的连续brick会形成一个副本集，所有的副本集组成了一个卷侧（volume-wide）分布式集。为确保副本集（replica-set）成员没有放置在同一个节点上，请列出每个节点上的第一个brick，然后按照相同的顺序列出第二个，以此类推*
![Distributed Replicated Volume](../images/disrtributed-replicated-volume.png)  

**创建分布式复制卷**  
1. 创建信任存储池
2. 创建分布式复制卷
    ```
    # gluster volume create [replica ] [transport tcp | rdma | tcp,rdma]
    ```  
    例：创建一个具有两份镜像（two-way）镜像的四节点分布式（复制）卷。  
    ```
    # gluster volume create test-volume replica 2 transport tcp server1:/exp1 server2:/exp2 server3:/exp3 server4:/exp4
    Creation of test-volume has been successful
    Please start the volume to access data.
    ```  
    例：创建一个具有两份（two-way）镜像的六节点分布式（复制）卷。  
    ```
    # gluster volume create test-volume replica 2 transport tcp server1:/exp1 server2:/exp2 server3:/exp3 server4:/exp4 server5:/exp5 server6:/exp6
    Creation of test-volume has been successful
    Please start the volume to access data.
    ```  
    如果传输类型没有被指定，tcp将作为默认类型。需要的话，你也可以设置其他选项，比如auth.allow 或 auth.reject。  
    > *注意：确保在尝试挂载卷之前启动卷，否则挂载后的客户端操作将挂起。*
    > + *如果一个副本集上超过一个brick在同一个远程节点上，GlusterFS将无法创建复制卷。如下所示，一个四节点的复制卷，其一个副本集中超过一个brick在同一个远程节点上*
    > ```
    > # gluster volume create <volname> replica 4 server1:/> brick1 server1:/brick2 server2:/brick3 server4:/brick4
    >volume create: <volname>: failed: Multiple bricks of a replicate volume are present on the same server. This setup is not optimal. Use 'force' at the end of the command if you want to override this behavior.
    > ```  
    > *如果你仍然想使用这样的配置来创建卷，可以在命令的结尾使用 **force** 参数*  

## 创建分散卷
分散卷是基于纠删码（erasure codes）。它跨卷的多个brick，条带化编码数据，并添加一些冗余信息。你可以使用分散卷来获得一个可配置的可靠性级别，同时最大限度的减少空间浪费。  

**冗余（Redundancy）**  

每一个分散卷在创建的时候定义冗余值。这个值决定了可以丢失多少个brick而不中断对volume的操作。它同样可以通过如下公式计算出整个volume的可用空间。
```
<Usable size> = <Brick size> * (#Bricks - Redundancy)
```  
分散集的所有brick应该有相同的容量，否则，当最小容量的brick满了以后，分散集将不允许添加额外的数据。

重点注意，一个配置了3 brick和1 冗余（rendundancy）（总物理空间66.7%）比配置了10 brick和1 冗余（rendundancy）（90%）的分散卷的可用空间要少。然而，第一个比第二个更安全（粗略计算，第二种配置卷故障的可能性是第一种的4.5倍）  

比如，一个由6个4TB的brick和2冗余（rendundancy）组成的分散卷在两个brick不可用后仍然可以提供完整的可操作性。然而第三个不可用brick将导致卷不可用，因为其既不可读也不可写。卷的可用空间等于16TB。  

GlusterFS中纠删码的实现将冗余值限制为小于#Bricks / 2 (或等同于, redundancy * 2 < #Bricks)。有冗余等于brick数量一半，将等同于副本数是2的复制卷，或许复制卷会表现的更好。  

**最佳卷（Optimal Volume）**  

纠删码在性能方面最糟糕的事情之一是RWM（Read-Modify-Write）周期。纠删码在一定大小的块中运行，它不能与较小的块一起使用。这意味着如果用户写入的文件没有填满整个块，它需要从文件的当前内容中读取剩余部分，合并它们，计算更新的编码块，最后，写入结果数据。  

当这种情形发生时将增加延迟和降低性能。对于一些工作负载（workloads）某些GlusterFS性能xlators 能够帮助降低甚至消除这些问题，但是当分散卷在针对特别的实例使用时应该考虑到这点。  

当前分散卷使用块大小的实现时依赖于brick和冗余（redundancy）的数量：512*（#Bricks - redundancy）bytes。此值也被称为条带大小。  

在大多数的工作负载中，分散卷使用#Bricks/redundancy 组合并设置条带大小为2的幂，会表现的更好，因为在2的倍数的块中写入数据更常见（比如数据库，虚拟机和许多应用程序）。  

这些组合被考虑为最佳的。  

比如，一个配置6 brick和2 redundancy的分散卷的条带大小为512*（6-2）= 2048 bytes，这被认为是最佳的（*optimal*）。一个配置7 brikc和2 redundancy的分散卷的条带大小为512*（7-2） = 2560 bytes，许多写入需要RMW周期（当然，这总是取决于用例）  

**创建分散卷**

1. 创建信任存储池
2. 创建分散卷：
   ```
   # gluster volume create [disperse [<count>]] [redundancy <count>] [transport tcp | rdma | tcp,rdma]
   ```  
   可以通过指定分散集的brick数量，指定冗余brick的数量或两者同时指定来创建分散卷。  

   如果*disperse*没有被指定，或者*count*不存在，整个卷将被认为是一个由命令行枚举的所有brick组成的单独的分散集。  

   如果*redundancy*没有被指定，它将被自动计算为最佳值，如果最佳值不存在，则假定为‘1’并显示告警信息：
   ```
   # gluster volume create test-volume disperse 4 server{1..4}:/bricks/test-volume
    There isn't an optimal redundancy value for this configuration. Do you want to create the volume with redundancy 1 ? (y/n)
   ```

   在*redundancy*是自动计算且结果不为1的时候，告警信息如下：
   ```
   # gluster volume create test-volume disperse 6 server{1..6}:/bricks/test-volume
    The optimal redundancy for this configuration is 2. Do you want to create the volume with this value ? (y/n)
   ```  

   *冗余（redundancy）*必须大于0，brick的总数也必须大于2\**redundancy*。这意味着一个分散卷要求的最少要求3 brick。   

   > *注意：确保在尝试挂载卷之前启动卷，否则挂载后的客户端操作将挂起。*
   > + *如果一个分散卷上超过一个brick在同一个远程节点上，GlusterFS将无法创建分散卷卷。
   > ```
   > # gluster volume create <volname> disperse 3 server1:/brick{1..3}
   > volume create: <volname>: failed: Multiple bricks of a disperse volume are present on the same server. This setup is not optimal. Bricks should be on different nodes to have best fault tolerant configuration. Use 'force' at the end of the command if you want to override this behavior.
   > ```  

## 创建分布式分散卷  

分布式分散卷等同于使用分散子卷而不是复制子卷的分布式复制卷。  

**创建分布式分散卷**  

1. 创建信任存储池
2. 创建分布式分散卷
   ```
   # gluster volume create disperse <count> [redundancy <count>] [transport tcp | rdma | tcp,rdma]
   ```  

   创建分布式分散卷，关键字*disperse*和<count\>是必须的。命令行中指定的brick数量必须是分散数的倍数。  

   *redundancy*和分散卷中完全一致  

    如果传输类型没有被指定，tcp将作为默认类型。需要的话，你也可以设置其他选项，比如auth.allow 或 auth.reject。  
    > *注意：*
    > + *确保在尝试挂载卷之前启动卷，否则挂载后的客户端操作将挂起。*
    > + *对于分布式分散卷如果brick不属于同一个子卷则可以托管在同一个节点上*
    >   ```
    >   # gluster volume create <volname> disperse 3 server1:/br1 server2:/br1 server3:/br1 server1:/br2 server2:/br2 server3:/br2
    >    volume create: : success: please start the volume to access data
    >   ``` 

## 启动卷
你在挂载卷之前必须先启动它。

**启动卷**
 + 开始卷：
    ```
    # gluster volume start <VOLNAME> [force]
    ```  

    比如：启动test-volume  
    ```
    # gluster volume start test-volume
    Starting test-volume has been successful
    ```