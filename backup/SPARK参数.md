

## Driver

**spark.driver.cores**

driver端分配的核数，默认为1，thriftserver是启动thriftserver服务的机器，资源充足的话可以尽量给多。

**spark.driver.memory**

driver端分配的内存数，默认为1g，同上。

**spark.driver.maxResultSize**

driver端接收的最大结果大小，默认1GB，最小1MB，设置0为无限。
这个参数不建议设置的太大，如果要做数据可视化，更应该控制在20-30MB以内。过大会导致OOM。

**spark.extraListeners**

默认none，随着SparkContext被创建而创建，用于监听单参数、无参数构造函数的创建，并抛出异常。

## Executor

**spark.executor.memory**
每个executor分配的内存数，默认1g，会受到yarn CDH的限制，和memoryOverhead相加 不能超过总内存限制。

**spark.executor.cores**

每个executor的核数，默认yarn下1核，standalone下为所有可用的核。

**spark.default.parallelism**

默认RDD的分区数、并行数。
像reduceByKey和join等这种需要分布式shuffle的操作中，最大父RDD的分区数；像parallelize之类没有父RDD的操作，则取决于运行环境下得cluster manager：
如果为单机模式，本机核数；集群模式为所有executor总核数与2中最大的一个。

**spark.executor.heartbeatInterval**

executor和driver心跳发送间隔，默认10s，必须远远小于spark.network.timeout

**spark.files.fetchTimeout**

从driver端执行SparkContext.addFile() 抓取添加的文件的超时时间，默认60s

**spark.files.useFetchCache**

默认true，如果设为true，拉取文件时会在同一个application中本地持久化，被若干个executors共享。这使得当同一个主机下有多个executors时，执行任务效率提高。

**spark.broadcast.blockSize**

TorrentBroadcastFactory中的每一个block大小，默认4m
过大会减少广播时的并行度，过小会导致BlockManager 产生 performance hit.

**spark.files.overwrite**

默认false，是否在执行SparkContext.addFile() 添加文件时，覆盖已有的内容有差异的文件。

**spark.files.maxPartitionBytes**

单partition中最多能容纳的文件大小，单位Bytes 默认134217728 (128 MB)

**spark.files.openCostInBytes**

小文件合并阈值，小于该参数就会被合并到一个partition内。
默认4194304 (4 MB) 。这个参数在将多个文件放入一个partition时被用到，宁可设置的小一些，因为在partition操作中，小文件肯定会比大文件快。

**spark.storage.memoryMapThreshold**

从磁盘上读文件时，最小单位不能少于该设定值，默认2m，小于或者接近操作系统的每个page的大小。

## Shuffle

**spark.reducer.maxSizeInFlight**

默认48m。从每个reduce任务同时拉取的最大map数，每个reduce都会在完成任务后，需要一个堆外内存的缓冲区来存放结果，如果没有充裕的内存就尽可能把这个调小一点。。相反，堆外内存充裕，调大些就能节省gc时间。

**spark.reducer.maxBlocksInFlightPerAddress**

限制了每个主机每次reduce可以被多少台远程主机拉取文件块，调低这个参数可以有效减轻node manager的负载。（默认值Int.MaxValue）

**spark.reducer.maxReqsInFlight**

限制远程机器拉取本机器文件块的请求数，随着集群增大，需要对此做出限制。否则可能会使本机负载过大而挂掉。。（默认值为Int.MaxValue）

**spark.reducer.maxReqSizeShuffleToMem**

shuffle请求的文件块大小 超过这个参数值，就会被强行落盘，防止一大堆并发请求把内存占满。（默认Long.MaxValue）

**spark.shuffle.compress**

是否压缩map输出文件，默认压缩 true

**spark.shuffle.spill.compress**

shuffle过程中溢出的文件是否压缩，默认true，使用spark.io.compression.codec压缩。

**spark.shuffle.file.buffer**

在内存输出流中 每个shuffle文件占用内存大小，适当提高 可以减少磁盘读写 io次数，初始值为32k

**spark.shuffle.memoryFraction**

该参数代表了Executor内存中，分配给shuffle read task进行聚合操作的内存比例，默认是20%。
cache少且内存充足时，可以调大该参数，给shuffle read的聚合操作更多内存，以避免由于内存不足导致聚合过程中频繁读写磁盘。

**spark.shuffle.manager**

当ShuffleManager为SortShuffleManager时，如果shuffle read task的数量小于这个阈值（默认是200），则shuffle write过程中不会进行排序操作，而是直接按照未经优化的HashShuffleManager的方式去写数据，但是最后会将每个task产生的所有临时磁盘文件都合并成一个文件，并会创建单独的索引文件。

当使用SortShuffleManager时，如果的确不需要排序操作，那么建议将这个参数调大一些，大于shuffle read task的数量。那么此时就会自动启用bypass机制，map-side就不会进行排序了，减少了排序的性能开销。但是这种方式下，依然会产生大量的磁盘文件，因此shuffle write性能有待提高。

**spark.shuffle.consolidateFiles**

如果使用HashShuffleManager，该参数有效。如果设置为true，那么就会开启consolidate机制，会大幅度合并shuffle write的输出文件，对于shuffle read task数量特别多的情况下，这种方法可以极大地减少磁盘IO开销，提升性能。

如果的确不需要SortShuffleManager的排序机制，那么除了使用bypass机制，还可以尝试将spark.shuffle.manager参数手动指定为hash，使用HashShuffleManager，同时开启consolidate机制。

**spark.shuffle.io.maxRetries**

shuffle read task从shuffle write task所在节点拉取属于自己的数据时，如果因为网络异常导致拉取失败，是会自动进行重试的。该参数就代表了可以重试的最大次数。如果在指定次数之内拉取还是没有成功，就可能会导致作业执行失败。

对于那些包含了特别耗时的shuffle操作的作业，建议增加重试最大次数（比如60次），以避免由于JVM的full gc或者网络不稳定等因素导致的数据拉取失败。在实践中发现，对于针对超大数据量（数十亿~上百亿）的shuffle过程，调节该参数可以大幅度提升稳定性。

**spark.shuffle.io.retryWait**

同上，默认5s，建议加大间隔时长（比如60s），以增加shuffle操作的稳定性

## Compression and Serialization

**spark.broadcast.compress**

广播变量前是否会先进行压缩。默认true （spark.io.compression.codec）

**spark.io.compression.codec**

压缩RDD数据、日志、shuffle输出等的压缩格式 默认lz4

**spark.io.compression.lz4.blockSize**

使用lz4压缩时，每个数据块大小 默认32k

**spark.rdd.compress**

rdd是否压缩 默认false，节省memory_cache大量内存 消耗更多的cpu资源（时间）。

**spark.serializer.objectStreamReset**

当使用JavaSerializer序列化时，会缓存对象防止写多余的数据，但这些对象就不会被gc，可以输入reset 清空缓存。默认缓存100个对象，修改成-1则不缓存任何对象。

## Memory Management

**spark.memory.fraction**

执行内存和缓存内存（堆）占jvm总内存的比例，剩余的部分是spark留给用户存储内部源数据、数据结构、异常大的结果数据。
默认值0.6，调小会导致频繁gc，调大容易造成oom。

**spark.memory.storageFraction**

用于存储的内存在堆中的占比，默认0.5。调大会导致执行内存过小，执行数据落盘，影响效率；调小会导致缓存内存不够，缓存到磁盘上去，影响效率。

值得一提的是在spark中，执行内存和缓存内存公用java堆，当执行内存没有使用时，会动态分配给缓存内存使用，反之也是这样。如果执行内存不够用，可以将存储内存释放移动到磁盘上（最多释放不能超过本参数划分的比例），但存储内存不能把执行内存抢走。

**spark.memory.offHeap.enabled**

是否允许使用堆外内存来进行某些操作。默认false

**spark.memory.offHeap.size**

允许使用进行操作的堆外内存的大小，单位bytes 默认0

**spark.cleaner.periodicGC.interval**

控制触发gc的频率，默认30min

**spark.cleaner.referenceTracking**

是否进行context cleaning，默认true

**spark.cleaner.referenceTracking.blocking**

清理线程是否应该阻止清理任务，默认true

**spark.cleaner.referenceTracking.blocking.shuffle**

清理线程是否应该阻止shuffle的清理任务，默认false

**spark.cleaner.referenceTracking.cleanCheckpoints**

清理线程是否应该清理依赖超出范围的检查点文件（checkpoint files不知道怎么翻译。。）默认false

## Networking

**spark.rpc.message.maxSize**

executors和driver间消息传输、map输出的大小，默认128M。map多可以考虑增加。

**spark.driver.blockManager.port和spark.driver.bindAddress**

driver端绑定监听block manager的地址与端口。

**spark.driver.host和spark.driver.port**

driver端的ip和端口。

**spark.network.timeout**

网络交互超时时间，默认120s。如果
spark.core.connection.ack.wait.timeout
spark.storage.blockManagerSlaveTimeoutMs
spark.shuffle.io.connectionTimeout
spark.rpc.askTimeout orspark.rpc.lookupTimeout
没有设置，那么就以此参数为准。

**spark.port.maxRetries**

设定了一个端口后，在放弃之前的最大重试次数，默认16。 会有一个预重试机制，每次会尝试前一次尝试的端口号+1的端口。如 设定了端口为8000，则最终会尝试8000~(8000+16)范围的端口。

**spark.rpc.numRetries**

rpc任务在放弃之前的重试次数，默认3，即rpc task最多会执行3次。

**spark.rpc.retry.wait**

重试间隔，默认3s

**spark.rpc.askTimeout**

rpc任务超时时间，默认spark.network.timeout

**spark.rpc.lookupTimeout**

rpc任务查找时长

## Scheduling

**spark.scheduler.maxRegisteredResourcesWaitingTime**

在执行前最大等待申请资源的时间，默认30s。

**spark.scheduler.minRegisteredResourcesRatio**

实际注册的资源数占预期需要的资源数的比例，默认0.8

**spark.scheduler.mode**

调度模式，默认FIFO 先进队列先调度，可以选择FAIR。

**spark.scheduler.revive.interval**

work回复重启的时间间隔，默认1s

**spark.scheduler.listenerbus.eventqueue.capacity**

spark事件监听队列容量，默认10000，必须为正值，增加可能会消耗更多内存

**spark.blacklist.enabled**

是否列入黑名单，默认false。如果设成true，当一个executor失败好几次时，会被列入黑名单，防止后续task派发到这个executor。可以进一步调节spark.blacklist以下相关的参数：
（均为测试参数 Experimental）
spark.blacklist.timeout
spark.blacklist.task.maxTaskAttemptsPerExecutor
spark.blacklist.task.maxTaskAttemptsPerNode
spark.blacklist.stage.maxFailedTasksPerExecutor
spark.blacklist.application.maxFailedExecutorsPerNode
spark.blacklist.killBlacklistedExecutors
spark.blacklist.application.fetchFailure.enabled

**spark.speculation**

推测，如果有task执行的慢了，就会重新执行它。默认false，

详细相关配置如下：
**spark.speculation.interval**

检查task快慢的频率，推测间隔，默认100ms。

**spark.speculation.multiplier**

推测比均值慢几次算是task执行过慢，默认1.5

**spark.speculation.quantile**

在某个stage，完成度必须达到该参数的比例，才能被推测，默认0.75

**spark.task.cpus**

每个task分配的cpu数，默认1

**spark.task.maxFailures**

在放弃这个job前允许的最大失败次数，重试次数为该参数-1，默认4

**spark.task.reaper.enabled**

赋予spark监控有权限去kill那些失效的task，默认false
(原先有 job失败了但一直显示有task在running，总算找到这个参数了)

其他进阶的配置如下：
**spark.task.reaper.pollingInterval**

轮询被kill掉的task的时间间隔，如果还在running，就会打warn日志，默认10s。

**spark.task.reaper.threadDump**

线程回收是是否产生日志，默认true。

**spark.task.reaper.killTimeout**

当一个被kill的task过了多久还在running，就会把那个executor给kill掉，默认-1。

**spark.stage.maxConsecutiveAttempts**

在终止前，一个stage连续尝试次数，默认4。

## Dynamic Allocation

**spark.dynamicAllocation.enabled**

是否开启动态资源配置，根据工作负载来衡量是否应该增加或减少executor，默认false

以下相关参数：
**spark.dynamicAllocation.minExecutors**

动态分配最小executor个数，在启动时就申请好的，默认0

**spark.dynamicAllocation.maxExecutors**

动态分配最大executor个数，默认infinity

**spark.dynamicAllocation.initialExecutors**

动态分配初始executor个数默认值=spark.dynamicAllocation.minExecutors

**spark.dynamicAllocation.executorIdleTimeout**

当某个executor空闲超过这个设定值，就会被kill，默认60s

**spark.dynamicAllocation.cachedExecutorIdleTimeout**

当某个缓存数据的executor空闲时间超过这个设定值，就会被kill，默认infinity

**spark.dynamicAllocation.schedulerBacklogTimeout**

任务队列非空，资源不够，申请executor的时间间隔，默认1s

**spark.dynamicAllocation.sustainedSchedulerBacklogTimeout**

同schedulerBacklogTimeout，是申请了新executor之后继续申请的间隔，默认=schedulerBacklogTimeout

## Spark Streaming

**spark.streaming.stopGracefullyOnShutdown** （true / false）默认fasle

确保在kill任务时，能够处理完最后一批数据，再关闭程序，不会发生强制kill导致数据处理中断，没处理完的数据丢失

**spark.streaming.backpressure.enabled** （true / false） 默认false

开启后spark自动根据系统负载选择最优消费速率

**spark.streaming.backpressure.initialRate** （整数） 默认直接读取所有

在开启反压的情况下，限制第一次批处理应该消费的数据，因为程序冷启动队列里面有大量积压，防止第一次全部读取，造成系统阻塞

**spark.streaming.kafka.maxRatePerPartition** （整数） 默认直接读取所有

限制每秒每个消费线程读取每个kafka分区最大的数据量

**spark.streaming.unpersist**

自动将spark streaming产生的、持久化的数据给清理掉，默认true，自动清理内存垃圾。

**spark.streaming.ui.retainedBatches**
spark streaming 日志接口在gc时保留的batch个数，默认1000
