# Hadoop介绍

Hadoop项目开发用于可靠的、可扩展的分布式计算开源软件。

> The Apache Hadoop software library is a framework that allows for the distributed processing of large data sets across clusters of computers using simple programming models. It is designed to scale up from single servers to thousands of machines, each offering local computation and storage. Rather than rely on hardware to deliver high-availability, the library itself is designed to detect and handle failures at the application layer, so delivering a highly-available service on top of a cluster of computers, each of which may be prone to failures.

## Hadoop包含的模块：

- **Hadoop Common**：通用模块
- **Hadoop Distributed File System (HDFS™)**： 分布式文件系统
- **Hadoop YARN**：用于作业调度和群资源管理的框架
- **Hadoop MapReduce**： 基于YARN的用于大数据集并行处理的系统

## 与Hadoop相关的项目：

- HBase
- Hive
- Spark
- Zookeeper

## 理论知识点：

- ### 存储模型

  - 文件线性按**字节**byte切割成块**block**，具有offset，id   （文件可以看成字节数组）
  - 文件与文件的block大小可以不一样
  - 一个文件除最后一个block，其他block大小一致
  - block的大小依据硬件的I/O特性调整     （可以变化）
  - block被分散存放在集群的节点中，具有location
  - block具有副本replication，没有主从概念，副本不能出现在同一节点
  - 副本是满足可靠性和性能的关键      （性能：同时在多节点进行计算的可能）
  - 文件上传是可以指定block大小和副本数，上传后只能修改副本数
  - 一次写入多次读取，不支持修改     （一旦修改某个block，后续block的offset就会无效，需要进行一次泛洪操作修改后续所有block的内容，代价太高）
  - 支持追加数据     （只在最后一个节点进行操作即可，不会有泛洪）
  - HDFS面向的**操作对象**是文件file，而不是文件中的单独某个块block

- ### 架构设计

  - HDFS是一个主从（Master/Slave）架构

  - 由一个NameNode和一些DataNode组成    

  - 面向文件包含：文件数据data和文件元数据metadata

  - NameNode负责存储和管理文件元数据，并维护了一个层次型的文件目录树

  - DataNode负责存储文件数据（block块）并提供block的读写

  - DataNode与NameNode维持心跳，并汇报自己持有的block信息

  - Client和NameNode交互文件元数据，和DataNode交互文件block数据

  - 角色即JVM进程

    ![HDFS架构](https://user-images.githubusercontent.com/17522733/95022734-6463be80-0679-11eb-9262-a65aa00df76e.png)

- ### 角色功能

  - NameNode：（整个集群的唯一的主人）
    - 完全基于**内存**存储文件元数据、目录结构、文件block的映射     （基于内存是因为要保证速度）
    - 需要持久化方案保证数据可靠性
    - 提供**副本放置策略**
  - DataNode：
    - 基于本地磁盘存储block（文件的形式）
    - 并保存block的校验和数据保证block的可靠性
    - 与NameNode保持心跳，汇报block列表状态

- ### 元数据持久化

  - 任何对文件系统元数据产生修改的操作，NameNode都会使用EditLog的事务日志记录下来     （append日志文件）
  - 使用FsImage存储内存所有的元数据状态                                                                                   （镜像、快照、dump）
  - 使用本地磁盘保存EditLog和FsImage
  - EditLog具有完整性，数据丢失少，但恢复速度慢，并有体积膨胀风险
  - FsImage具有恢复速度快，体积与内存数据相当，但是不能实时保存，数据丢失多
  - NameNode使用了FsImage+EditLog整合的方案：**滚动**将增量的EditLog更新到FsImage，以保证更近时点的FsImage和更小的EditLog体积

- ### 安全模式

  - HDFS搭建时会格式化，格式化操作产生一个空的FsImage
  - 当NameNode启动时，它从硬盘中读取EditLog和FsImage到内存中，把所有的EditLog中的事务作用到FsImage上，并将新版本的FsImage从内存保存到本地硬盘上，然后删除旧的EditLog，因为其事务均已通过新的FsImage被记录到硬盘上。
  - NameNode启动后会进入安全模式的特殊状态，不会进行数据块的复制。然后NameNode从所有的DataNode接受心跳信号和块状态报告，每当某个block的副本数目达到最小值，该block就会被认为是副本安全safely replicated的
  - 在一定百分比（可配置）的block被NameNode检测确认是安全之后，等待额外的30秒，就可以退出安全模式状态。然后它会确定还有哪些block的副本没有达到指定数目，并将这些block复制到其他的DataNode上。

- ### 副本放置策略

  - 第一个副本：放置在上传文件的DataNode；如果是集群外提交，则随机挑选一台磁盘不太满，CPU不太忙的节点     （保证副本写入磁盘速度）
  - 第二个副本：放置在与第一个副本不同的机架的节点上    （防止第一个机架宕机导致数据不可达）
  - 第三个副本：与第二个副本相同机架的节点上                  （限制成本，减少跨交换机次数）
  - 更多副本：随机节点

- ### HDFS写流程

  - client和NameNode连接创建文件元数据

  - NameNode判定元数据是否有效

  - NameNode**触发副本放置策略**，返回一个有序的DataNode列表    **【重点】**

  - client和DataNode**建立pipeline连接**                                               **【重点】**

  - client将block切分成packet（64KB），并使用`chunk（512B）+chunksum（4B）`填充

  - client将packet放入发送队列dataqueue中，并向第一个DataNode发送

  - 第一个DataNode收到packet后本地保存并发送给第二个DataNode

  - 第二个DataNode收到packet后本地保存并发送给第三个DataNode

  - 这个过程中上游节点同时发送下一个packet

  - 优点：

    - 这个副本传输过程及副本数对client来说是透明的。
    - 由于把文件进行分片传输，所以传输效率极高，因为可以同时在client和DataNode之间，以及DataNode之间进行同时传输。
    - block传输完成之后需要DataNode各自向NameNode进行汇报，因为这样才是可信的。如果有缺失，则可以命令他们之间进行互传。

    ![HDFS写流程](https://user-images.githubusercontent.com/17522733/95025060-66347e80-0687-11eb-8773-ad389564eaae.png)

- ### HDFS读流程

  - HDFS能够暴露文件的块信息，让每个单机上的程序根据offset获取到合理的文件分块，减少程序处理前的文件传输量！HDFS支持client给出文件的offset，自定义连接哪些block的DataNode，自定义获取数据。**这是支持计算层的分治+并行计算的核心**。  **【非常重要！！！】**

  - 为了降低整体的带宽消耗和读取延时，HDFS会尽量让读取程序读取离他最近的副本

  - 如果在读取程序的同一个机架上有一个副本，优先读取该副本

  - 如果一个HDFS集群跨域多个数据中心，则客户端优先读取本地数据中心副本

  - 语义：下载一个文件：

    - client和NameNode交互文件元数据获取fileBlockLocation
    - NameNode会按距离策略排序返回
    - client尝试下载block并校验数据完整性

    ![HDFS读流程](https://user-images.githubusercontent.com/17522733/95025432-02f81b80-068a-11eb-8aea-eb303bd4e6f4.png)

