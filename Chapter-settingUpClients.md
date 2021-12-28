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
   3. 使用如下命令运行配置应用实例  
      `# ./configure`  
      ```
      GlusterFS configure summary
      ===========================
      FUSE client : yes
      Infiniband verbs : yes
      epoll IO multiplex : yes
      argp-standalone : no
      fusermount : no
      readline : yes
      ```  
      配置摘要展示了将使用Gluster本地客户端构建的组件。  
   4. 使用如下命令构建Gluster本地客户端。
      ```
      # make
      # make install`
      ```  
      1. 通过如下命令确认安装了正确版本的Gluster本地客户端。
        `# glusterfs --version`  

## 挂载卷  

安装Gluster本地客户端后，你需要挂载Gluster卷后才能访问数据。有两种方式你可以选择：
+ 手动挂载卷
+ 自动挂载卷
> **注意：**
> 在创建卷时选择的服务名应该是可以被客户端机器解析的。你可以使用适当的/etc/hosts条目，或者DNS服务器使服务名解析到IP地址。  

## 手动挂载

+ 使用如下命令挂载卷
  `# mount -t glusterfs HOSTNAME-OR-IPADDRESS:/VOLNAME MOUNTDIR`  
+ 比如：
  `# mount -t glusterfs server1:/test-volume /mnt/glusterfs`  
  > **注意：**
  > mount命令中指定的server只用于获取描述卷的gluster配置文件volfile。随后，客户端将直接和volfile中提到的服务器进行通信（甚至可能不包括用于挂载的服务器）。
  > 
  > 如果你看到一个帮助信息像"Usage: mount.glusterfs"，mount经常要求你创建一个目录用做挂载点。在你尝试去执行上面列出的mount命令之前运行"mkdir /mnt/glusters"。

**挂载选项**  

当使用`mount -t glusterfs`命令时，你可以指定下面的可选项。注意你需要使用逗号分隔所有可选项。  

```
backupvolfile-server=server-name

volfile-max-fetch-attempts=number of attempts

log-level=loglevel

log-file=logfile

transport=transport-type

direct-io-mode=[enable|disable]

use-readdirp=[yes|no]
```  

比如：
`# mount -t glusterfs -o backupvolfile-server=volfile_server2,use-readdirp=no,volfile-max-fetch-attempts=2,log-level=WARNING,log-file=/var/log/gluster.log server1:/test-volume /mnt/glusterfs`

在挂载fuse客户端时，如果 `backvolfile-server` 选项被添加。当第一个volfile服务器故障，将使用指定在`backvolfile-server`中的服务做为volfile服务器挂载客户端。  

在选项`volfile-max-fetch-attempts=X`选项中，指定了挂载卷时尝试去获取volfile的次数。当你尝试挂载服务使用多IP地址或者服务名被配置了轮训DNS的时候，这个可选项是有用的。  

如果设置`use-readdirp`为ON，将强制使用fuse内核模块中的readirp模式。  

### 自动挂载卷

你可以配置系统每次启动时自动挂载Gluster卷。  

mount命令中指定的server只用于获取描述卷信息的gluster配置文件volfile。随后，客户端将直接和volfile中提到的服务器进行通信（甚至可能不包括用于挂载的服务器）。  
+ 编辑/etc/fstab文件，添加如下行来挂载卷。  
  `HOSTNAME-OR-IPADDRESS:/VOLNAME MOUNTDIR glusterfs defaults,_netdev 0 0`  
  比如：
  `server1:/test-volume /mnt/glusterfs glusterfs defaults,_netdev 0 0`  

**挂载选项**  

在更新/etc/fstab文件的时候，你可以指定下面的可选项。注意你需要使用逗号分隔所有的可选项。

```
log-level=loglevel

log-file=logfile

transport=transport-type

direct-io-mode=[enable|disable]

use-readdirp=no
```  

比如：  

`HOSTNAME-OR-IPADDRESS:/VOLNAME MOUNTDIR glusterfs defaults,_netdev,log-level=WARNING,log-file=/var/log/gluster.log 0 0`  

## 测试挂载的卷  

测试挂载的卷  

+ 使用如下命令：
  `# mount`  
  如果gluster卷已经成功挂载，客户端机器上挂载命令的输出和下面的例子类似：
  ```
  server1:/test-volume on /mnt/glusterfs type fuse.glusterfs (rw,allow_other,default_permissions,max_read=131072
  ```  

+ 使用如下命令：
  `# df`  
  客户端上df命令的输出将展示一个来自卷所有brick聚合的存储空间和下面的例子类似：
  ```
  # df -h /mnt/glusterfs
  Filesystem               Size Used Avail Use% Mounted on
  server1:/test-volume     28T 22T 5.4T 82% /mnt/glusterfs
  ```  
+ 输入如下命令跳转目录然后列出内容
  ```
  # cd MOUNTDIR
  # ls
  ```  

+ 比如： 
  ```
  # cd /mnt/glusterfs
  # ls
  ```

## NFS

你可以使用NFS v3协议访问gluster卷。在GNU/linux客户端和在其他操作系统比如FreeBSD，和Mac OS X，以及Windows7（专业版及以上），Windows Server 2003已经做了广泛的测试，其他Gluster NFS 服务实现也许可以有效。  

现在GlusterFS包含 network lock manager（NLM）v4。NLM允许在NFSv3客户端上的应用程序对NFS服务上的文件进行记录锁定。无论NFS服务是否在运行它都是启动的。   

你必须在客户端和服务端同时安装nfs-common包（仅仅针对Debian-based发行版）  

这部分描述了如何使用NFS协议挂载Gluster卷（包含手动以及自动挂载）和如何确认卷已经正常挂载。  

### 使用NFS协议挂载卷  

你可以使用以下方法的任意一种来挂载Gluster卷。
+ 手动使用NFS协议挂载卷
+ 自动使用NFS协议挂载卷

**先决条件：** 同时在服务端和客户端安装nfs-common包（仅针对Debian-based发行版），使用如下命令：
`$ sudo aptitude install nfs-common`  

#### 手动使用NFS协议挂载卷  

**手动使用NFS协议挂载Gluster卷**  
+ 使用如下命令挂载卷
  `# mount -t nfs -o vers=3 HOSTNAME-OR-IPADDRESS:/VOLNAME MOUNTDIR`  
  比如：
  `# mount -t nfs -o vers=3 server1:/test-volume /mnt/glusterfs`  

  > **注意**
  > Gluster NFS服务不支持UDP。如果你使用的NFS客户端默认使用UDP作为连接协议，将会显示如下信息：
  > `requested NFS version or transport protocol is not supported.`  

  **使用TCP协议连接**  
  + mount命令中添加如下可选项：
    `-o mountproto=tcp`  
    比如：  
    `# mount -o mountproto=tcp -t nfs server1:/test-volume /mnt/glusterfs`  

**使用Solaris客户端挂载Gluster NFS服务**  

+ 使用如下命令： 
  `# mount -o proto=tcp,vers=3 nfs://HOSTNAME-OR-IPADDRESS:38467/VOLNAME MOUNTDIR`  
  
  比如：
  `# mount -o proto=tcp,vers=3 nfs://server1:38467/test-volume /mnt/glusterfs`  

### 自动使用NFS协议挂载卷  

你可以配置每次系统启动时自动使用NFS协议挂载Gluster卷。  

**自动使用NFS协议挂载Gluster卷**  

+ 挂载卷可以配置/etc/fstab文件，添加如下行：
  `HOSTNAME-OR-IPADDRESS:/VOLNAME MOUNTDIR nfs defaults,_netdev,vers=3 0 0`  

  比如，  
  `server1:/test-volume /mnt/glusterfs nfs defaults,_netdev,vers=3 0 0`  

  > **注意**
  > Gluster NFS服务不支持UDP。如果你使用的NFS客户端默认使用UDP作为连接协议，将会显示如下信息：
  > `requested NFS version or transport protocol is not supported.`  

   **使用TCP协议连接**  
  + /etc/fstab中添加如下输入：
    `HOSTNAME-OR-IPADDRESS:/VOLNAME MOUNTDIR nfs defaults,_netdev,mountproto=tcp 0 0`  
    比如：  
    `server1:/test-volume /mnt/glusterfs nfs defaults,_netdev,mountproto=tcp 0 0`   

**自动挂载NFS挂载**

Gluster支持自动挂载NFS挂载的 *nix 标准方法。更新/etc/auto.master 和 /etc/auto.misc 并重启autofs服务。之后无论何时用户或者进程尝试访问目录的时候，它将在后台自动挂载上。  

#### 测试使用NFS协议的卷挂载  

你可以确认Gluster目录是挂载成功的。  

**测试挂载的卷**  

+ 输入如下使用mount命令：
  `# mount`  
  例如，客户端上mount命令输出将展示如下。
  `server1:/test-volume on /mnt/glusterfs type nfs (rw,vers=3,addr=server1)`  
+ 输入以下内容使用df命令：
  `# df`  
  客户端上df命令的输出将展示一个来自卷所有brick聚合的存储空间和下面的例子类似：
  ```
  # df -h /mnt/glusterfs
  Filesystem               Size Used Avail Use% Mounted on
  server1:/test-volume     28T 22T 5.4T 82% /mnt/glusterfs
  ```  
+ 输入如下命令跳转目录然后列出内容
  ```
  # cd MOUNTDIR
  # ls
  ```  
## CIFS  

在使用微软Windows和SAMBA客户端时可以使用CIFS协议访问卷。这种访问模式，客户端侧需要Samba包。你可以暴露glusterfs挂载点做为samba export，然后使用CIFS协议挂载。  

本节描述了在基于 Microsoft Windows客户端上如何挂载（手动和自动）CIFS分享，和如何确认卷已经挂载成功。  
> **注意**
> 不支持使用Mac OS X Finder访问CIFS，不过你可以通过Mac OS X命令行使用CIFS访问Gluster卷。  

### 使用CIFS协议挂载卷  

你可以使用如下任意方法挂载gluster卷：
+ 通过Samba暴露Gluster卷
+ 手动使用CIFS挂载卷
+ 自动使用CIFS挂载卷 
  
你同样可以通过CIFS协议使用Samba暴露Gluster卷。  

#### 通过Samba暴露Gluster卷  

我们建议使用Samba通过CIFS协议暴露Gluster卷。  

**通过CIFS协议暴露卷**  

1. 挂载一个Gluster卷
2. 建立Samba配置暴露Gluster卷挂载点。
   比如，如果一个Gluster卷挂载在/mnt/gluster，你必须编辑smb.conf文件去允许通过CIFS协议暴露。在编辑器中打开smb.conf文件并添加如下行以进行简单配置。  
   ```
    [glustertest]

    comment = For testing a Gluster volume exported through CIFS

    path = /mnt/glusterfs

    read only = no

    guest ok = yes
   ```  

   保存修改并使用系统初始化脚本（/etc/init.d/smb/start）启动smb服务。多次挂载需要执行上诉步骤。如果你只想samba挂载，你需要在你的smb.conf中添加：
   ```
    kernel share modes = no
    kernel oplocks = no
    map archive = no
    map hidden = no
    map read only = no
    map system = no
    store dos attributes = yes
   ```  
   > **注意**
   > 为了从可信任存储池中的任何服务挂载，你必须在每个Gluster节点上重复上诉步骤。有关更高级的配置，请查询Samba文档。  

### 使用CIFS手动挂载卷  

你可以在基于Microsoft Windows的客户端机器上手动使用CIFS挂载Gluster卷。  

**使用CIFS手动挂载Gluster卷**  

1. 使用Windows Explorer，从菜单选择**Tools > Map Network Drive...** 。**Map Network Drive**窗口将出现。
2. 使用**Drive**下拉列表选择drive。
3. 点击**Browse**，选择要映射到网络驱动器的卷，并点击**OK**。
4. 点击**FINISH**  

网络驱动（映射到卷）出现在计算机窗口。  

或者，使用CIFS手动的挂载Gluster卷，通过**Start > Run** 并手动的输入网络地址。  

### 使用CIFS协议自动挂载卷   

你可以配置基于Microsoft Windows的客户端在系统启动的时候使用CIFS自动挂载Gluster卷。  

**使用CIFS协议自动挂载Gluster卷**  

网络驱动（映射到卷）出现在计算机的窗口，后每次系统启动的时候会重新连接。  

1. 使用Windows Explorer，从菜单选择 **Tools > Map Network Drive...**。**Map Network Drive**窗口出现。
2. 使用**Drive**下拉列表选择drive。  
3. 点击**Browse**，选择要映射到网络驱动器的卷，并点击**OK**。
4. 在登录选择框点击**Reconnect**  
5. 点击**FINISH**  

### 测试使用CIFS挂载卷  

你可以通过Windows Explorer导航到Gluster目录来确认是否挂载成功。