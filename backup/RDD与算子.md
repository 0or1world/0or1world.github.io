## 什么是RDD
- RDD（Resilient Distributed Dataset）叫做弹性分布式数据集，是Spark中最基本的数据抽象，它代表一个不可变、可分区、里面的元素可并行计算的集合。RDD具有数据流模型的特点：自动容错、位置感知性调度和可伸缩性。RDD允许用户在执行多个查询时显式地将工作集缓存在内存中，后续的查询能够重用工作集，这极大地提升了查询速度。
## 什么是DataFrame
- DataFrame引入了schema和off-heap
schema : RDD每一行的数据, 结构都是一样的，这个结构就存储在schema中。 Spark通过schema就能够读懂数据, 因此在通信和IO时就只需要序列化和反序列化数据, 而结构的部分就可以省略了。
## 什么是DataSet
- DataSet结合了RDD和DataFrame的优点，并带来的一个新的概念Encoder。
当序列化数据时，Encoder产生字节码与off-heap进行交互，能够达到按需访问数据的效果，而不用反序列化整个对象。Spark还没有提供自定义Encoder的API，但是未来会加入。
![三者之间转换](https://upload-images.jianshu.io/upload_images/20370955-da315446437bf70f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## RDD的宽依赖和窄依赖
- 由于RDD是粗粒度的操作数据集，每个Transformation操作都会生成一个新的RDD，所以RDD之间就会形成类似流水线的前后依赖关系；RDD和它依赖的父RDD（s）的关系有两种不同的类型，即窄依赖（narrow dependency）和宽依赖（wide dependency）。如图所示显示了RDD之间的依赖关系。

![](https://upload-images.jianshu.io/upload_images/20370955-614444360f86b279.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 窄依赖：是指每个父RDD的一个Partition最多被子RDD的一个Partition所使用，例如map、filter、union等操作都会产生窄依赖；（独生子女）

- 宽依赖：是指一个父RDD的Partition会被多个子RDD的Partition所使用，例如groupByKey、reduceByKey、sortByKey等操作都会产生宽依赖；（超生）

## 依赖关系流下面的视图
>![](https://upload-images.jianshu.io/upload_images/20370955-ab1efcdbe3fc621a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在spark中，会根据RDD之间的依赖关系将DAG图（有向无环图）划分为不同的阶段，对于窄依赖，由于partition依赖关系的确定性，partition的转换处理就可以在同一个线程里完成，窄依赖就被spark划分到同一个stage中，而对于宽依赖，只能等父RDD shuffle处理完成后，下一个stage才能开始接下来的计算。

`因此spark划分stage的整体思路是`：从后往前推，遇到宽依赖就断开，划分为一个stage；遇到窄依赖就将这个RDD加入该stage中。因此在图2中RDD C,RDD D,RDD E,RDDF被构建在一个stage中,RDD A被构建在一个单独的Stage中,而RDD B和RDD G又被构建在同一个stage中。

在spark中，Task的类型分为2种：`ShuffleMapTask和ResultTask`；

简单来说，DAG的最后一个阶段会为每个结果的partition生成一个ResultTask，即每个Stage里面的Task的数量是由该Stage中最后一个RDD的Partition的数量所决定的！而其余所有阶段都会生成ShuffleMapTask；之所以称之为ShuffleMapTask是因为它需要将自己的计算结果通过shuffle到下一个stage中；也就是说上图中的stage1和stage2相当于mapreduce中的Mapper,而ResultTask所代表的stage3就相当于mapreduce中的reducer。

在之前动手操作了一个wordcount程序，因此可知，Hadoop中MapReduce操作中的Mapper和Reducer在spark中的基本等量算子是map和reduceByKey;不过区别在于：Hadoop中的MapReduce天生就是排序的；而reduceByKey只是根据Key进行reduce，但spark除了这两个算子还有其他的算子；因此从这个意义上来说，Spark比Hadoop的计算算子更为丰富。

## RDD编程API(Transformation和Action)
- ### Transformation(转换)
转换 | 含义
-|-
**map**(func) | 返回一个新的RDD，该RDD由每一个输入元素经过func函数转换后组成
**filter**(func) | 返回一个新的RDD，该RDD由经过func函数计算后返回值为true的输入元素组成
**flatMap**(func) |类似于map，但是每一个输入元素可以被映射为0或多个输出元素（所以func应该返回一个序列，而不是单一元素）
**mapPartitions**(func)|类似于map，但独立地在RDD的每一个分片上运行，因此在类型为T的RDD上运行时，func的函数类型必须是Iterator[T] => Iterator[U]
**mapPartitionsWithIndex**(func)|类似于mapPartitions，但func带有一个整数参数表示分片的索引值，因此在类型为T的RDD上运行时，func的函数类型必须是(Int, Interator[T]) => Iterator[U]
**sample**(withReplacement, fraction, seed)|根据fraction指定的比例对数据进行采样，可以选择是否使用随机数进行替换，seed用于指定随机数生成器种子
**union**(otherDataset)|对源RDD和参数RDD求并集后返回一个新的RDD
**intersection**(otherDataset)|对源RDD和参数RDD求交集后返回一个新的RDD
**distinct**([numTasks]))|对源RDD进行去重后返回一个新的RDD
**groupByKey**([numTasks])|在一个(K,V)的RDD上调用，返回一个(K, Iterator[V])的RDD
**reduceByKey**(func, [numTasks])|在一个(K,V)的RDD上调用，返回一个(K,V)的RDD，使用指定的reduce函数，将相同key的值聚合到一起，与groupByKey类似，reduce任务的个数可以通过第二个可选的参数来设置
**aggregateByKey**(zeroValue)(seqOp, combOp, [numTasks])|先按分区聚合 再总的聚合   每次要跟初始值交流 例如：aggregateByKey(0)(_+_,_+_) 对k/y的RDD进行操作
**sortByKey**([ascending], [numTasks])|在一个(K,V)的RDD上调用，K必须实现Ordered接口，返回一个按照key进行排序的(K,V)的RDD
**sortBy**(func,[ascending], [numTasks])|与sortByKey类似，但是更灵活 第一个参数是根据什么排序  第二个是怎么排序 false倒序   第三个排序后分区数  默认与原RDD一样
**join**(otherDataset, [numTasks])|在类型为(K,V)和(K,W)的RDD上调用，返回一个相同key对应的所有元素对在一起的(K,(V,W))的RDD  相当于内连接（求交集）
**cogroup**(otherDataset, [numTasks])|在类型为(K,V)和(K,W)的RDD上调用，返回一个(K,(Iterable<V>,Iterable<W>))类型的RDD
**cartesian**(otherDataset)|两个RDD的笛卡尔积  的成很多个K/V
**pipe**(command, [envVars])|调用外部程序
**coalesce**(numPartitions)|重新分区 第一个参数是要分多少区，第二个参数是否shuffle 默认false  少分区变多分区 true   多分区变少分区 false
**repartition**(numPartitions)|重新分区 必须shuffle  参数是要分多少区  少变多
**repartitionAndSortWithinPartitions**(partitioner)|重新分区+排序  比先分区再排序效率高  对K/V的RDD进行操作
**foldByKey**(zeroValue)(seqOp)|该函数用于K/V做折叠，合并处理 ，与aggregate类似   第一个括号的参数应用于每个V值  第二括号函数是聚合例如：_+_
**combineByKey**|合并相同的key的值 rdd1.combineByKey(x => x, (a: Int, b: Int) => a + b, (m: Int, n: Int) => m + n)
**partitionBy**（partitioner）|对RDD进行分区  partitioner是分区器 例如new HashPartition(2
`cache` `persist`|RDD缓存，可以避免重复计算从而减少时间，区别：cache内部调用了persist算子，cache默认就一个缓存级别MEMORY-ONLY ，而persist则可以选择缓存级别
**Subtract**（rdd）|返回前rdd元素不在后rdd的rdd
**leftOuterJoin**|leftOuterJoin类似于SQL中的左外关联left outer join，返回结果以前面的RDD为主，关联不上的记录为空。只能用于两个RDD之间的关联，如果要多个RDD关联，多关联几次即可。
**rightOuterJoin**|rightOuterJoin类似于SQL中的有外关联right outer join，返回结果以参数中的RDD为主，关联不上的记录为空。只能用于两个RDD之间的关联，如果要多个RDD关联，多关联几次即可
**subtractByKey**|substractByKey和基本转换操作中的subtract类似只不过这里是针对K的，返回在主RDD中出现，并且不在otherRDD中出现的元素    
  - ### Action(动作)
动作|含义
-|-
**reduce**(func)|通过func函数聚集RDD中的所有元素，这个功能必须是课交换且可并联的
**collect**()|在驱动程序中，以数组的形式返回数据集的所有元素
**count**()|返回RDD的元素个数
**first**()|返回RDD的第一个元素（类似于take(1)）
**take**(n)|返回一个由数据集的前n个元素组成的数组
**takeSample**(withReplacement,num, [seed])|返回一个数组，该数组由从数据集中随机采样的num个元素组成，可以选择是否用随机数替换不足的部分，seed用于指定随机数生成器种子
**takeOrdered**(n, [ordering])|*
**saveAsTextFile**(path)|将数据集的元素以textfile的形式保存到HDFS文件系统或者其他支持的文件系统，对于每个元素，Spark将会调用toString方法，将它装换为文件中的文本
**saveAsSequenceFile**(path) |将数据集中的元素以Hadoop sequencefile的格式保存到指定的目录下，可以使HDFS或者其他Hadoop支持的文件系统。
**saveAsObjectFile**(path) | *
**countByKey**()|针对(K,V)类型的RDD，返回一个(K,Int)的map，表示每一个key对应的元素个数。
**foreach**(func)|在数据集的每一个元素上，运行函数func进行更新。
**aggregate**|先对分区进行操作，在总体操作
**reduceByKeyLocally**|*
**lookup**|*
**top**|*
**fold**|*
**foreachPartition**|*
## WordCount代码编写
```
import org.apache.spark.rdd.RDD
import org.apache.spark.{SparkConf, SparkContext}

object SparkWordCountWithScala {
  def main(args: Array[String]): Unit = {

    val conf = new SparkConf()
    /**
      * 如果这个参数不设置，默认认为你运行的是集群模式
      * 如果设置成local代表运行的是local模式
      */
    conf.setMaster("local")
    //设置任务名
    conf.setAppName("WordCount")
    //创建SparkCore的程序入口
    val sc = new SparkContext(conf)
    //读取文件 生成RDD
    val file: RDD[String] = sc.textFile("E:\\hello.txt")
    //把每一行数据按照，分割
    val word: RDD[String] = file.flatMap(_.split(","))
    //让每一个单词都出现一次
    val wordOne: RDD[(String, Int)] = word.map((_,1))
    //单词计数
    val wordCount: RDD[(String, Int)] = wordOne.reduceByKey(_+_)
    //按照单词出现的次数 降序排序
    val sortRdd: RDD[(String, Int)] = wordCount.sortBy(tuple => tuple._2,false)
    //将最终的结果进行保存
    sortRdd.saveAsTextFile("E:\\result")

    sc.stop()
  }
```
 