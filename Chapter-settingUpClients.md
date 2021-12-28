# 访问数据 - 配置GlusterFS客户端
你可以通过多种方式访问gluster卷。你可以使用gluster原生客户端方法在GNU/Linux客户端中实现高并发，高性能和透明故障转移。你同样可以使用NFS v3协议连接gluster卷。已经对GNU/Linux客户端和其他操作系统中的NFS实现进行了广泛的测试，例如FreeBSD，Mac OS X,以及Windows7（Professional及以上）和Windows Server 2003。其他的NFS客户端实现可能可以和gluster NFS服务一起工作。  

在使用Microsoft Windows以及SAMBA客户端访问卷时，你可以使用CIFS协议。对于这种访问模式，Samba包需要在客户端侧存在。  

## Gluster本地客户端（Gluster Native Client）

Gluster本地客户端是一个基于FUSE的客户端运行在用户空间，当要求高并发和高写入性能时，推荐使用Gluster本地客户端方式访问卷。  

这部分介绍Gluster本地客户端并解释怎么在客户端机器上安装程序。这部分也介绍了如何在客户端（手动或者自动的）挂载卷，以及如何确认卷是否挂载成功。  

**安装Gluster本地客户端**  

在你开始安装Gluster本地客户端之前，你需要验证FUSE模块已经加载，并可以访问到所要求的所有模块，如下所示：
1.  在linux内核中添加FUSE可加载内核模块（LKM）：
    `# modprobe fuse`
2.  确认FUSE模块已经加载：
    ```
    # dmesg | grep -i fuse 
    fuse init (API version 7.13)
    ```  
### 在Red Hat Package Manager(RPM)发行版上安装  

在基于RPM分发的系统上安装Gluster本地客户端  

1.  使用如下命令在客户端机器上安装所需模块：
   ```
   # sudo yum -y install openssh-server wget fuse fuse-libs openib libibverbs
   ```  
2. 确保Gluster服务端上TCP和UDP端口24007和24008是开启的。除了这些端口，你还需要为每个brick打开一个从49152开始的端口（而不是像以前的版本从24009开始） 
   。brick端口分配方案现在符合IANA指南。比如：如果你有5个brick，你需要打开49152-49156的端口。  
   
   从Gluster-10开始，brick的端口将是随机的。分配给brick的端口将由定义在glusterd.vol文件中的从base_port到max_port范围内随机选择一个。比如：如果你有5个brick，你需要在base_port到max_port范围内至少打开5个端口。为了降低打开端口的数量（为了最佳安全实践），可以减小glusterd.vol中max_port值然后重启glusterd生效。  

   你可以使用如下iptables的链：
   ```
    $ sudo iptables -A RH-Firewall-1-INPUT -m state --state NEW -m tcp -p tcp --dport 24007:24008 -j ACCEPT
    $ sudo iptables -A RH-Firewall-1-INPUT -m state --state NEW -m tcp -p tcp --dport 49152:49156 -j ACCEPT
   ```  

**注意：**
如果你已经有了iptables chains，确保上面的ACCEPT规则在DROP规则之前。可以通过提供一个比DROP规则更小的规则号来实现。

1. 下载最新的glusterfs，gluster-fuse，和glusterfs-rdma RPM文件到每个客户端服务器上。glusterfs包包含了Gluster本地客户端。glusterfs-fuse包包含了在客户端系统上挂载需要用到的FUSE转换器，glusterfs-rdma包包含了OpenFabrics为Infiniband提供RDMA模块。
   你可以在[GlusterFS download page](http://www.gluster.org/download/) 中下载这些软件包。
2. 在客户端机器上安装Cluster本地客户端。  
  
**注意** 下面例子中列出的包的版本可能不是最新的版本。请转到下载页面确认你有最新的版本包。  
```
    $ sudo rpm -i glusterfs-3.8.5-1.x86_64
    $ sudo rpm -i glusterfs-fuse-3.8.5-1.x86_64
    $ sudo rpm -i glusterfs-rdma-3.8.5-1.x86_64
```  
> **注意：** 只有在使用Infiniband的时候再需要RDMA模块。  

### 在Debian发行版上安装  

在Debian发行版上安装Gluster本地客户端  

1. 使用如下命令在每个客户端机器上安装OpenSSH服务
   `$ sudo apt-get install openssh-server vim wget`  
2. 在每个客户端上下载最新的GlusterFS .deb文件和checksum
   你可以在[GlusterFS download page](http://www.gluster.org/download/) 中下载这些软件包。  
3. 对于每个.deb文件，获取校验（checksum）（使用如下命令），然后和在md5sum文件中的校验对比。  
   `$ md5sum GlusterFS_DEB_file.deb`  
   包的md5sum可以在此获取：[GlusterFS download page](https://download.gluster.org/pub/gluster/glusterfs/LATEST/)
4. 使用如下版本从客户端机器卸载GlusterFS v3.1 (或者更早的版本）
   `$ sudo dpkg -i GlusterFS_DEB_file`  
   例如：  
   `$ sudo dpkg -i glusterfs-3.8.x.deb`  
5. 确保在所有Gluster服务器上，TCP和UDP端口24007和24008是开放的。除了这些端口，你还需要从49152（而不是之前版本的24009）开始的端口中选择为每个brick开启一个端口。现在brick端口声明方法是兼容IANA指南的。比如：你有5个brick，你需要打开49152-49156端口。  

Gluster-10之前，brick端口将是随机的。随机的从定义在文件glusterd.vol中的base_port到max_port范围内选择端口分配给brick。比如：你有5个brick，你需要至少在给定的base_port到max_port范围内开放5个端口。为了减少开放端口的数量（为了最佳安全实践），可以减小glusterd.vol文件中max_port的值，然后重启glusterd服务使其生效。  

你可以在iptables中使用如下链：
```
$ sudo iptables -A RH-Firewall-1-INPUT -m state --state NEW -m tcp -p tcp --dport 24007:24008 -j ACCEPT
$ sudo iptables -A RH-Firewall-1-INPUT -m state --state NEW -m tcp -p tcp --dport 49152:49156 -j ACCEPT
```  
> **注意:**
> 如果你已经有了iptables链，确保上面的的ACCTEPT规则在DROP规则之前。可以通过提供比DROP规则更小的规则号实现。  

### 执行源码安装  

从源码编译和安装Gluster本地客户端。  

1. 使用如下命令创建一个新的目录：
   ```
   # mkdir glusterfs
   # cd glusterfs
   ```  
   1. 下载源码
    你可以[在此](http://www.gluster.org/download/)下载源码  
   2. 使用如下命令解压源码  
   `# tar -xvzf SOURCE-FILE`  
   3. 