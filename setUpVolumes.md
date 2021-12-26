# 创建卷
**卷是brick(信任存储池(truseted storage pool)中的服务暴露的目录)的逻辑组合。在你的存储环境里创建新卷，需要指定组成卷的brick。创建卷后，必须要先启动才能尝试挂载。**  
  
如何创建brick请查看[挂载存储](https://docs.gluster.org/Administrator-Guide/setting-up-storage/)  
## 卷类型
+ 如下类型的卷可以在你的存储环境中创建
  + **分布式卷（Distributed）**   分布式卷分发文件到卷中的所有brick中。在你需要可扩展存储且冗余是不重要或者已经由其他硬件或者软件提供了冗余的时候，选择使用分布式卷。
  + **复制卷（Replicated）**   复制卷跨卷的多个brick创建文件副本。你可以在对高可用（high-availability）和高可靠（high-reliability）至关重要的场景下使用复制卷。
  + **分布式副本卷（Distributed Replicated）**  分布式副本卷分发文件到卷的整个副本brick中。你可以在需要可扩展存储且高可靠性（high-reliability）至关重要的环境中使用分布式复制卷。在大多数场景下，他可以提供读的性能提升。
  + **分散卷（Dispersed）**  分散卷卷基于纠删码（erasure code），提供节省空间且防止磁盘和服务故障导致数据丢失的能力。它将原始文件的编码碎片存储到每个brick中，只需碎片的子集即可恢复原始文件。在不丢失数据访问权限的情况下可以丢失的brick数量由管理员在创建卷时配置。
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

![Distributed Volume](./image/distributed-vlume.png)  
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
![Replicated Volume](./image/replicated-volume.png)  
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
    ```
    # gluster volume create <volname> replica 4 server1:/brick1 server1:/brick2 server2:/brick3 server4:/brick4
    volume create: <volname>: failed: Multiple bricks of a replicate volume are present on the same server. This setup is not optimal. Use 'force' at the end of the command if you want to override this behavior.
    ```  
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
