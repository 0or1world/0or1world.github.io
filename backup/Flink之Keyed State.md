**源码解析**
``` public class CountWindowAverage extends RichFlatMapFunction<Tuple2<Long, Long>, Tuple2<Long, Long>> {

    /**
     * The ValueState handle. The first field is the count, the second field a running sum.
     */
    /*
      ValueState 运行时保存在Taskmanager内存里
      checkkpoint的时候,把state保存在远端文件系统里
      当flink开启checkkpoint的时候,默认state保存在taskmanagerd    的内存里checkkpoint保存在jobmanager
      生产模式,state保存在taskManager的rocksdb文件系统里,checkkpoint保存在hdfs里
    */ 
    private transient ValueState<Tuple2<Long, Long>> sum;

    @Override
    public void flatMap(Tuple2<Long, Long> input, Collector<Tuple2<Long, Long>> out) throws Exception {

        // sum可以访问里面的数据
        Tuple2<Long, Long> currentSum = sum.value();

        // 元组下标0的+1
        currentSum.f0 += 1;

        // 元组下标1 = (传入元组下标1的+1)
        currentSum.f1 += input.f1;

        // 将Valuestate更新到sum里
        sum.update(currentSum);

        // 当currentSum.f0 >= 2时输出平均数
        if (currentSum.f0 >= 2) {
            out.collect(new Tuple2<>(input.f0, currentSum.f1 / currentSum.f0));
            //清空sum
           // sum.clear();
        }
    }

    @Override
    public void open(Configuration config) {
        ValueStateDescriptor<Tuple2<Long, Long>> descriptor =
                      //定义ValueState描述
                new ValueStateDescriptor<>(
                        "average", // the state name
                        TypeInformation.of(new TypeHint<Tuple2<Long, Long>>() {}), // 类型是Tuple2
                        Tuple2.of(0L, 0L)); // 默认值
                  //通过描述获得sumstate
        sum = getRuntimeContext().getState(descriptor);
    }
}

// 例子
        env.enableCheckpointing(2000);



// advanced options:

// set mode to exactly-once (this is the default)
        env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);

// make sure 500 ms of progress happen between checkpoints
        env.getCheckpointConfig().setMinPauseBetweenCheckpoints(500);

// checkpoints have to complete within one minute, or are discarded
        env.getCheckpointConfig().setCheckpointTimeout(60000);

// allow only one checkpoint to be in progress at the same time
        env.getCheckpointConfig().setMaxConcurrentCheckpoints(1);

// enable externalized checkpoints which are retained after job cancellation
        env.getCheckpointConfig().enableExternalizedCheckpoints(CheckpointConfig.ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION);

        //无重启策略
        //env.setRestartStrategy(RestartStrategies.noRestart());


// allow job recovery fallback to checkpoint when there is a more recent savepoint
       env.setStateBackend(new FsStateBackend("file:///C:\\Users\\19191\\Desktop\\test"));

env.fromElements(Tuple2.of(1L, 3L), Tuple2.of(1L, 5L), Tuple2.of(1L, 7L), Tuple2.of(1L, 4L), Tuple2.of(1L, 2L))
        .keyBy(0)
                      //新建
        .flatMap(new CountWindowAverage())
        .print();

// the printed output will be (1,4) and (1,5)
//打包运行
bin/flink run -c [全限定名]  [jar包位置]

//恢复
$ bin/flink run -s :checkpointMetaDataPath [matedata文件路径] -c  [全限定名]  [jar包位置]