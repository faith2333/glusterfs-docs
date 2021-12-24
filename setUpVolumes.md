# 创建卷
**卷是brick(信任存储池(truseted storage pool)中的服务暴露的目录)的逻辑组合。在你的存储环境里创建新卷，需要指定组成卷的brick。创建卷后，必须要先启动才能尝试挂载。**  
  
如何创建brick请查看[挂载存储](https://docs.gluster.org/Administrator-Guide/setting-up-storage/)  
## 卷类型
+ 如下类型的卷可以在你的存储环境中创建
  + **分布式卷（distributed）**   分布式卷分发文件到卷中的所有brick中。在你需要规模存储且冗余是不重要或者已经由其他硬件或者软件提供了冗余的时候，你可以选择使用分布式卷。
  + **副本卷（Replicated）**   副本卷复制文件到卷中的所有brick中。你可以在对高可用(high-availability)和高可靠(high-reliability)要求最紧急的场景下使用副本卷。