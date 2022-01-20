# gNFS故障诊断  

## 故障诊断Gluster NFS

本章描述了和NFS相关最常见的故障诊断问题。  

### 在NFS客户端挂载命令失败并显示"RPC Error: Program not registered"  

在NFS服务器上启动portmap或者rpcbind服务。  

这个错误在服务器没有正确启动的时候会遇到，在大多数Linux发行版启动portmap后修复。  
```
# /etc/init.d/portmap start
```  

在一些发行版中portman已经被替换为rpcbind，要求如下命令：   
```
# /etc/init.d/rpcbind start
```  

在启动portmap或者rpcbind后，Gluster NFS服务需要重启。  

### NFS服务启动失败并在日志文件显示"Port already is use"  

另外一个Gluster NFS服务运行在同一台机器上。  

这个错误可以出现在已经有个Gluster NFS服务运行在同一个机器上的情形下。这种情形可以通过日志文件确认，如果下面的错误行存在：  
```
[2010-05-26 23:40:49] E [rpc-socket.c:126:rpcsvc_socket_listen] rpc-socket: binding socket failed:Address already in use
[2010-05-26 23:40:49] E [rpc-socket.c:129:rpcsvc_socket_listen] rpc-socket: Port is already in use 
[2010-05-26 23:40:49] E [rpcsvc.c:2636:rpcsvc_stage_program_register] rpc-service: could not create listening connection 
[2010-05-26 23:40:49] E [rpcsvc.c:2675:rpcsvc_program_register] rpc-service: stage registration of program failed 
[2010-05-26 23:40:49] E [rpcsvc.c:2695:rpcsvc_program_register] rpc-service: Program registration failed: MOUNT3, Num: 100005, Ver: 3, Port: 38465 
[2010-05-26 23:40:49] E [nfs.c:125:nfs_init_versions] nfs: Program init failed 
[2010-05-26 23:40:49] C [nfs.c:531:notify] nfs: Failed to initialize protocols
```  

解决这个错误需要其中一个Gluster NFS服务被关闭。目前，不支持同一个机器上运行多个Gluster NFS服务。  

### 挂载命令失败并显示"rpc.statd"关联错误信息  

如果挂载命令失败并显示以下信息：  
```
mount.nfs: rpc.statd is not running but is required for remote locking.
mount.nfs: Either use '-o nolock' to keep locks local, or start statd.
```  

对于NFS客户端挂载NFS服务，rpc.statd服务必须运行在客户端上。运行如下命令启动rpc.statd服务：    
```
# rpc.statd
```  

### 挂载命令花费太长的时间完成  

**在NFS客户端启动rpcbind服务**  

这个问题是NFS客户端上rpcbind或者portmap服务没有运行。解决办法是执行以下命令来启动这些服务中的任何一个：  
```
# /etc/init.d/portmap start
```  

在一些发行中portmap已经被rpcbind替换，要求使用以下命令：  
```
# /etc/init.d/rpcbind start
```  

### NFS服务glusterfsd启动但是初始化失败并在日志中显示"nfsrpc-service:protmap registration of program failed"。  

NFS启动成功，但是初始化NFS服务仍然失败，阻止了客户端访问访问挂载点。这种情形可以通过日志文件中的如下错误信息确认：  
```
[2010-05-26 23:33:47] E [rpcsvc.c:2598:rpcsvc_program_register_portmap] rpc-service: Could notregister with portmap 
[2010-05-26 23:33:47] E [rpcsvc.c:2682:rpcsvc_program_register] rpc-service: portmap registration of program failed
[2010-05-26 23:33:47] E [rpcsvc.c:2695:rpcsvc_program_register] rpc-service: Program registration failed: MOUNT3, Num: 100005, Ver: 3, Port: 38465
[2010-05-26 23:33:47] E [nfs.c:125:nfs_init_versions] nfs: Program init failed
[2010-05-26 23:33:47] C [nfs.c:531:notify] nfs: Failed to initialize protocols
[2010-05-26 23:33:49] E [rpcsvc.c:2614:rpcsvc_program_unregister_portmap] rpc-service: Could not unregister with portmap
[2010-05-26 23:33:49] E [rpcsvc.c:2731:rpcsvc_program_unregister] rpc-service: portmap unregistration of program failed
[2010-05-26 23:33:49] E [rpcsvc.c:2744:rpcsvc_program_unregister] rpc-service: Program unregistration failed: MOUNT3, Num: 100005, Ver: 3, Port: 38465
```  

1. 在NFS服务上启动portmap或者rpcbind服务。
   在大多数的Linux发行版上，可以通过使用如下命令启动portmap：
   ```
   # /etc/init.d/portmap start
   ```  

   在一些发行版中portmap已经被rpcbind替换，执行以下命令：  
   ```
   # /etc/init.d/rpcbind start
   ``` 
   在启动portmap或者rpcbind后，需要重启gluster NFS服务。

2. 停止在同一个机器上运行的另外一个NFS服务。  
   当另外一个NFS服务运行在同一个机器上但不是Gluster NFS服务的时候会看到这样的错误。在linux系统上，这个可能是内核NFS服务。解决办法包括停止另外一个NFS服务或者不要在这个机器上运行Gluster NFS服务。在停止内核NFS服务前，确保没有核心服务依赖访问这个NFS服务的暴露。   

   在linxu上，内核NFS服务可以使用以下依赖于发行版在使用的命令中的任意一种停止：    
   ```
   # /etc/init.d/nfs-kernel-server stop
   # /etc/init.d/nfs stop
   ```  

3. 重启Gluster NFS 服务  


### 挂载命令失败并显示NFS服务失败错误。  

挂载命令失败显示如下错误：  
```
mount: mount to NFS server '10.1.10.11' failed: timed out (retrying).
```  

执行以下之一去解决这个问题：  

1. **停用域名查询（name lookup）请求从NFS服务到DNS服务**  
   
   NFS服务尝试通过执行DNS查询匹配卷文件中的主机名和客户端IP地址认证NFS客户端。可能会有情形NFS服务连接不上DNS服务或者DNS服务花费太长的时间响应DNS请求。这些延迟会导致NFS服务到NFS客户端的延迟响应从而导致上面看到的超时错误。  

   NFS服务提供了一个解决办法，就是禁用DNS请求，转而仅依靠客户端IP地址进行认证。这种情况下，以下选项可以被添加以成功挂载。  
   `option rpc-auth.addr.namelookup off`  

   > **注意：** 记住禁用NFS服务强制客户端认证仅使用IP地址，并且如果卷文件的认证规则使用主机名，这些认证规则将失败并切这些客户端不允许挂载。  

   **或者**  

2. NFS客户端使用的NFS版本不是版本3  
   
   Gluster NFS服务支持NFS协议版本3。在最近的linux内核中，默认的NFS版本已经从3更改为4。这或许导致客户端机器无法连接到Gluster NFS服务，因为使用版本4的信息无法被Gluster NFS服务理解。这个超时可以通过强制NFS客户端使用版本3解决。**vers** 选项是挂载命令用于这个目的的：  
   ```
   # mount  -o vers=3
   ```  

### 显示挂载失败显示"clnt_create: RPC: Unable to receive"  

检查你的防火墙为portmap 请求/响应 和Gluster NFS服务 请求/响应 配置开启了端口111。Gluster NFS服务操作在以下端口数字：38465，38466和38467。  

### 应用失败显示"Invalid argument"或者"Value too large for defined data type" 错误。  

这两个错误一般发生在32位NFS客户端或者应用不支持64位inode数字或者大文件。从控制台使用如下选项去使NFS返回32位inode数字：`nfs.enable-ino32 \ <on/off>`  

将收益的应用是：  

+ 在32位机器上构建和运行，默认不支持大文件。  
+ 在64位系统上构建32位

这个选项是默认禁用，所以默认NFS返回64位inode数字。  

可以重新从源码构建的应用建议使用gcc如下标志：  
`-D_FILE_OFFSET_BITS=64`