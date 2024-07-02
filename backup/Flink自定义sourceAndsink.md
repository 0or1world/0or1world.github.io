**连接kafka获取数据**

```
//flink流处理
StreamExecutionEnvironment exe = StreamExecutionEnvironment.getExecutionEnvironment();
        //将Properties创建出来
        Properties properties = new Properties();
        //将配置信息set到Properties
        properties.setProperty("bootstrap.servers","node4:9092");
        properties.setProperty("group.id","1");
        //使用FlinkKafkaConsumer011(自己的kafka版本[参数](z'ji't'y))
        FlinkKafkaConsumer011<String> jk = new FlinkKafkaConsumer011<>("jk", new SimpleStringSchema(),properties);
        //
        DataStreamSource<String> stringDataStreamSource = exe.addSource(jk);
```
**MySQLsource**
```
                      使用RichSourceFunction自定义source
public class MySqlSource extends RichSourceFunction<Tuple4<String, String, Double, Long>> {

    Connection conn;
    Statement stat;
    boolean flag = true;


    @Override
    //open内写连接方法
    public void open(Configuration parameters) throws Exception {
        //类反射mysqlDriver
        Class.forName("com.mysql.jdbc.Driver");
        //mysql的连接方式用户密码
        conn = DriverManager.getConnection("jdbc:mysql://node4:3306/1704e", "root", "123456");
        stat = conn.createStatement();
    }

    @Override
    //run内写具体的逻辑
    public void run(SourceContext<Tuple4<String, String, Double, Long>> ctx) throws Exception {
        while (flag) {
            ResultSet resultSet = stat.executeQuery("select * from t_dev where dev_state = 0");
            StringBuffer sb = new StringBuffer("(");
            int count = 0;
            while (resultSet.next()){
                count ++;
                long id = resultSet.getLong(1);
                String devId = resultSet.getString(2);
                String metric = resultSet.getString(3);
                double value = resultSet.getDouble(4);
                long timestamp = resultSet.getLong(5);
                sb.append(id+",");
                ctx.collect(Tuple4.of(devId, metric, value,timestamp));
            }
            String ids = sb.toString();
            ids = ids.substring(0, ids.length() - 1);
            ids = ids + ")";
            if(count != 0){
                String updateSql = "update t_dev set dev_state = 1 where id in "+ids;
                System.out.println(updateSql);
                stat.execute(updateSql);
            }
            Thread.sleep(10000);
        }
    }

    @Override
    public void cancel() {
        flag = false;
    }

    @Override
    public void close() throws Exception {
        if(conn != null){
            conn.close();
        }
    }
}
```
**HBaseSink**
```
                          //继承RichSinkFunction
public class HBaseSink extends RichSinkFunction<Tuple6<String, String, Double, Double, Double, Long>> {

    private org.apache.hadoop.conf.Configuration conf;
    private Connection conn;
    private Table table;
    private SimpleDateFormat sdf = new SimpleDateFormat("yyyyMMddHHmm");


    @Override
    //open内写连接方法
    public void open(Configuration parameters) throws Exception {
        //创建Hbase连接
        conf = HBaseConfiguration.create();
        //配置信息
        conf.set("hbase.zookeeper.quorum","node4:2181");
        System.out.println("long...........");
        conn = ConnectionFactory.createConnection(conf);
        System.out.println("ok....................");
        //获取表
        table = conn.getTable(TableName.valueOf("ns1:t_dev_data"));
    }

    @Override
    //invoke 具有逻辑实现
    public void invoke(Tuple6<String, String, Double, Double, Double, Long> value, Context context) throws Exception {
        String format = sdf.format(value.f5);
        String rowKeyStr = value.f0 + value.f1 + format;

        Put put = new Put(Bytes.toBytes(rowKeyStr));
        put.addColumn(Bytes.toBytes("f1"),Bytes.toBytes("max"),Bytes.toBytes(value.f2));
        put.addColumn(Bytes.toBytes("f1"),Bytes.toBytes("min"),Bytes.toBytes(value.f3));
        put.addColumn(Bytes.toBytes("f1"),Bytes.toBytes("avg"),Bytes.toBytes(value.f4));

        table.put(put);

    }

    @Override
    public void close() throws Exception {
        if(table != null) {
            table.close();
        }
        if(conn != null){
            conn.close();
        }
    }
}
