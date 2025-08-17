系统架构图


从HBase的架构图上可以看出，HBase中的组件包括Client、Zookeeper、HMaster、HRegionServer、HRegion、Store、MemStore、StoreFile、HFile、HLog等，接下来介绍他们的作用

Client
1、HBase 有两张特殊表：
.META.：记录了用户所有表拆分出来的的 Region 映射信息，.META.可以有多个 Regoin

-ROOT-：记录了.META.表的 Region 信息，-ROOT-只有一个 Region，无论如何不会分裂

2、Client 访问用户数据前需要首先访问 ZooKeeper，找到-ROOT-表的 Region 所在的位置，然 后访问-ROOT-表，接着访问.META.表，最后才能找到用户数据的位置去访问，中间需要多次 网络操作，不过 client 端会做 cache 缓存。

zookeeper
ZooKeeper 为 HBase 提供 Failover 机制，选举 Master，避免单点 Master 单点故障问题
存储所有 Region 的寻址入口：-ROOT-表在哪台服务器上。-ROOT-这张表的位置信息
实时监控 RegionServer 的状态，将 RegionServer 的上线和下线信息实时通知给 Master
存储 HBase 的 Schema，包括有哪些 Table，每个 Table 有哪些 Column Family
Master
为 RegionServer 分配 Region
负责 RegionServer 的负载均衡
发现失效的 RegionServer 并重新分配其上的 Region
HDFS 上的垃圾文件（HBase）回收
处理 Schema 更新请求（表的创建，删除，修改，列簇的增加等等）
RegionServer
RegionServer 维护 Master 分配给它的 Region，处理对这些 Region 的 IO 请求
RegionServer 负责 Split 在运行过程中变得过大的 Region，负责 Compact 操作
可以看到，client 访问 HBase 上数据的过程并不需要 master 参与（寻址访问 zookeeper 和 RegioneServer，数据读写访问 RegioneServer），Master 仅仅维护者 Table 和 Region 的元数据信息，负载很低。
.META.存的是所有的 Region 的位置信息，那么 RegioneServer 当中 Region 在进行分裂之后 的新产生的 Region，是由 Master 来决定发到哪个 RegioneServer，这就意味着，只有 Master 知道 new Region 的位置信息，所以，由 Master 来管理.META.这个表当中的数据的 CRUD
所以结合以上两点表明，在没有 Region 分裂的情况，Master 宕机一段时间是可以忍受的。

HRegion
table在行的方向上分隔为多个Region。Region是HBase中分布式存储和负载均衡的最小单元，即不同的region可以分别在不同的Region Server上，但同一个Region是不会拆分到多个server上。
Region按大小分隔，每个表一般是只有一个region。随着数据不断插入表，region不断增大，当region的某个列族达到一个阈值时就会分成两个新的region。
每个region由以下信息标识：< 表名,startRowkey,创建时间>
由目录表(-ROOT-和.META.)记录该region的endRowkey
Store
每一个region由一个或多个store组成，至少是一个store，hbase会把一起访问的数据放在一个store里面，即为每个 ColumnFamily建一个store，如果有几个ColumnFamily，也就有几个Store。一个Store由一个memStore和0或者 多个StoreFile组成。 HBase以store的大小来判断是否需要切分region
MemStore
memStore 是放在内存里的。保存修改的数据即keyValues。当memStore的大小达到一个阀值（默认128MB）时，memStore会被flush到文 件，即生成一个快照。目前hbase 会有一个线程来负责memStore的flush操作。
StoreFile
memStore内存中的数据写到文件后就是StoreFile，StoreFile底层是以HFile的格式保存。
HFile
HBase中KeyValue数据的存储格式，HFile是Hadoop的 二进制格式文件，实际上StoreFile就是对Hfile做了轻量级包装，即StoreFile底层就是HFile
HLog
HLog(WAL log)：WAL意为write ahead log，用来做灾难恢复使用，HLog记录数据的所有变更，一旦region server 宕机，就可以从log中进行恢复。
HLog文件就是一个普通的Hadoop Sequence File， Sequence File的value是key时HLogKey对象，其中记录了写入数据的归属信息，除了table和region名字外，还同时包括sequence number和timestamp，timestamp是写入时间，sequence number的起始值为0，或者是最近一次存入文件系统中的sequence number。 Sequence File的value是HBase的KeyValue对象，即对应HFile中的KeyValue。
读流程
客户端通过 ZooKeeper 以及-ROOT-表和.META.表找到目标数据所在的 RegionServer(就是 数据所在的 Region 的主机地址)
联系 RegionServer 查询目标数据
RegionServer 定位到目标数据所在的 Region，发出查询请求
Region 先在 Memstore 中查找，命中则返回
如果在 Memstore 中找不到，则在 Storefile 中扫描 为了能快速的判断要查询的数据在不在这个 StoreFile 中，应用了 BloomFilte
BloomFilter，布隆过滤器：迅速判断一个元素是不是在一个庞大的集合内，但是他有一个 弱点：它有一定的误判率
写流程
Client 先根据 RowKey 找到对应的 Region 所在的 RegionServer
Client 向 RegionServer 提交写请求
RegionServer 找到目标 Region
Region 检查数据是否与 Schema 一致
如果客户端没有指定版本，则获取当前系统时间作为数据版本
将更新写入 WAL Log
将更新写入 Memstore
判断 Memstore 的是否需要 flush 为 StoreFile 文件。