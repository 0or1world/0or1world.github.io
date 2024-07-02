**源码刨析** **:**
设置水印时间防止数据乱序
```
public class BoundedOutOfOrdernessGenerator implements AssignerWithPeriodicWatermarks<MyEvent> {

    private final long maxOutOfOrderness = 10000; //最大允许乱序时间

    private long currentMaxTimestamp;//现在最大的时间戳

    @Override
    //extractTimestamp抽取数据的时间戳
    public long extractTimestamp(MyEvent element, long previousElementTimestamp) {
        //从element获取时间
        long timestamp = element.getCreationTime();
        //从timestamp和最大时间戳取最大时间
        currentMaxTimestamp = Math.max(timestamp, currentMaxTimestamp);
        return timestamp;
    }

    @Override
    //getCurrentWatermark得到当前水印时间(100ms调用一次)
    public Watermark getCurrentWatermark() {
        // 水印=当前最大的时间戳 - 最大允许乱序时间
        return new Watermark(currentMaxTimestamp - maxOutOfOrderness);
    }
}
```
**模拟Watermarks** **:**
```
public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);

        env.setParallelism(1);

        DataStreamSource<String> text = env.socketTextStream("localhost", 8888, "\n");

        SingleOutputStreamOperator<Tuple2<String, Long>> inputmap = text.map(new MapFunction<String, Tuple2<String, Long>>() {
            @Override
            public Tuple2<String, Long> map(String value) throws Exception {
                String[] arr = value.split(",");
                return new Tuple2<String, Long>(arr[0], Long.parseLong(arr[1]));
            }
        });

        SingleOutputStreamOperator<Tuple2<String, Long>> watermarkStream = inputmap.assignTimestampsAndWatermarks(new AssignerWithPeriodicWatermarks<Tuple2<String, Long>>() {


            SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS");
            Long currentMaxTimestamp = 0L;
            final Long maxOutOfOrderness = 10000L;// 最大允许的乱序时间是10s

            //默认100ms被调用一次
            @Nullable
            @Override
            public Watermark getCurrentWatermark() {
                return new Watermark(currentMaxTimestamp - maxOutOfOrderness);
            }

            //定义如何提取timestamp
            @Override
            public long extractTimestamp(Tuple2<String, Long> element, long previousElementTimestamp) {
                long timestamp = element.f1;
                currentMaxTimestamp = Math.max(timestamp, currentMaxTimestamp);
                long id = Thread.currentThread().getId();
                System.out.println("currentThreadId:" + id + ",key:" + element.f0 + ",eventtime:[" + element.f1 + "|" + sdf.format(element.f1) + "],currentMaxTimestamp:[" + currentMaxTimestamp + "|" +
                        sdf.format(currentMaxTimestamp) + "],watermark:[" + getCurrentWatermark().getTimestamp() + "|" + sdf.format(getCurrentWatermark().getTimestamp()) + "]");
                return timestamp;
            }
        });

        //保存被丢弃的数据
        OutputTag<Tuple2<String, Long>> outputTag = new OutputTag<Tuple2<String, Long>>("late-data"){};
        SingleOutputStreamOperator<String> applied = watermarkStream.keyBy(0)
                .timeWindow(Time.seconds(3))
                //.allowedLateness(Time.seconds(2))//允许数据迟到2秒
                .sideOutputLateData(outputTag)
                .apply(new WindowFunction<Tuple2<String, Long>, String, Tuple, TimeWindow>() {
                    @Override
                    public void apply(Tuple tuple, TimeWindow window, Iterable<Tuple2<String, Long>> input, Collector<String> out) throws Exception {
                        String key = tuple.toString();
                        List<Long> arrarList = new ArrayList<Long>();
                        Iterator<Tuple2<String, Long>> it = input.iterator();
                        while (it.hasNext()) {
                            Tuple2<String, Long> next = it.next();
                            arrarList.add(next.f1);
                        }
                        Collections.sort(arrarList);
                        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS");
                        String result = key + "," + arrarList.size() + "," + sdf.format(arrarList.get(0)) + "," + sdf.format(arrarList.get(arrarList.size() - 1))
                                + "," + sdf.format(window.getStart()) + "," + sdf.format(window.getEnd());
                        out.collect(result);
                    }
                });

        //把迟到的数据暂时打印到控制台，实际中可以保存到其他存储介质中
        DataStream<Tuple2<String, Long>> sideOutput = applied.getSideOutput(outputTag);
        applied.print();

        env.execute();

    }
```


> 输入数据
                  0001,1538359882000
 0001,1538359886000
0001,1538359892000
0001,1538359893000
0001,1538359894000

> 输出数据
     eventTime*************/ currentMaxTimestamp / Watermark 
     2018-10-01 10:11:**22** / 2018-10-01 10:11:22 / 2018-10-01 10:11:12
    2018-10-01 10:11:26 / 2018-10-01 10:11:26 / 2018-10-01 10:11:16
    2018-10-01 10:11:32 / 2018-10-01 10:11:32 / 2018-10-01 10:11:22
    2018-10-01 10:11:33 / 2018-10-01 10:11:33 / 2018-10-01 10:11:23
    2018-10-01 10:11:34 / 2018-10-01 10:11:34 / 2018-10-01 10:11:**24**
   
**设置的.timeWindow(Time.seconds(3))是系统划分的时间段,遵循前包后不包原则.**
**当Watermark>=窗口的结束时间就触发并且窗口内有数据.**
## 对于迟到数据处理
1. 丢弃
2. 使用.allowedLateness(Time.seconds(2))允许数据迟到2秒(会对时间段内数据进行重新处理耗费内存/计算资源)
1. 对迟到数据进行统一回收.sideOutputLateData(**outputTag**) 
```        
 //把迟到的数据打印到控制台，实际中可以保存到其他存储介质中
OutputTag<Tuple2<String, Long>> outputTag = new OutputTag<Tuple2<String, Long>>("late-data"){};

.timeWindow(Time.seconds(3)).sideOutputLateData(outputTag)

 DataStream<Tuple2<String, Long>> sideOutput = applied.getSideOutput(outputTag);

 applied.print();