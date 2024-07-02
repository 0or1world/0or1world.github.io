### Hive倾斜之group by聚合倾斜
**原因：**
- 分组的维度过少，每个维度的值过多，导致处理某值的reduce耗时很久；

- 对一些类型统计的时候某种类型的数据量特别多，其他的数据类型特别少。当按照类型进行group by的时候，会将相同的group by字段的reduce任务需要的数据拉取到同一个节点进行聚合，而当其中每一组的数据量过大时，会出现其他组的计算已经完成而这个reduce还没有计算完成，其他的节点一直等待这个节点的任务执行完成，所以会一直看到map 100% reduce99%的情况；

**解决方法：**
- set hive.map.aggr=true;

- set hive.groupby.skewindata=true;

**原理：**
- hive.map.aggr=true 这个配置代表开启map端聚合；

- hive.groupby.skewindata=true，当选项设定为true，生成的查询计划会有两个MR Job。当第一个MR Job中，Map的输出结果结合会随机分布到Reduce中，每个Reduce做部分聚合操作，并输出结果。这样处理的结果是相同的Group By Key有可能被分发到不同的Reduce中，从而达到负载均衡的目的。第二个MR Job再根据预处理的数据结果按照Group By Key分布到reduce中，这个过程可以保证相同的key被分到同一个reduce中，最后完成最终的聚合操作;

### Hive倾斜之Map和Reduce优化
1. 原因：当出现小文件过多，需要合并小文件。可以通过set hive.merge.mapredfiles=true来解决；
2. 原因：输入数据存在大块和小块的严重问题，比如 说：一个大文件128M，还有1000个小文件，每 个1KB。 解决方法：任务输入前做文件合并，将众多小文件合并成一个大文件。通过set hive.merge.mapredfiles=true解决；
3. 原因：单个文件大小稍稍大于配置的block块的大小，此时需要适当增加map的个数。解决方法：set mapred.map.tasks的个数；
4. 原因：文件大小适中，但是map端计算量非常大，如：select id,count(*),sum(case when...),sum(case when ...)...需要增加map个数。解决方法：set mapred.map.tasks个数，set mapred.reduce.tasks个数；

### Hive倾斜之HQL中包含count(distinct)时
- 如果数据量非常大，执行如select a,count(distinct b) from t group by a;类型的sql时，会出现数据倾斜的问题。

- 解决方法：使用sum...group by代替。如：select a,sum(1) from(select a,b from t group by a,b) group by a;

### Hive倾斜之HQL中join优化
当遇到一个大表和一个小表进行join操作时。使用mapjoin将小表加载到内存中。如：select /*+ MAPJOIN(a) */ a.c1, b.c1 ,b.c2 from a join b where a.c1 = b.c1;
遇到需要进行join，但是关联字段有数据为null，如表一的id需要和表二的id进行关联；
解决方法1：id为null的不参与关联
比如：
```
select * from log a 
　join users b 
on a.id is not null and a.id = b.id 
union all 
select * from log a 
where a.id is null;
```
解决方法2： 给null值分配随机的key值
比如：
```
select * from log a 
left outer join users b 
on 
case when a.user_id is null 
then concat(‘hive’,rand() ) 
else a.user_id end = b.user_id; 
```
### 合理设置Map数

1. 通常情况下，作业会通过input的目录产生一个或者多个map任务。
主要的决定因素有：input的文件总个数，input的文件大小，集群设置的文件块大小。
2. 是不是map数越多越好？
答案是否定的。如果一个任务有很多小文件（远远小于块大小128m），则每个小文件也会被当做一个块，用一个map任务来完成，而一个map任务启动和初始化的时间远远大于逻辑处理的时间，就会造成很大的资源浪费。而且，同时可执行的map数是受限的。
3. 是不是保证每个map处理接近128m的文件块，就高枕无忧了？
答案也是不一定。比如有一个127m的文件，正常会用一个map去完成，但这个文件只有一个或者两个小字段，却有几千万的记录，如果map处理的逻辑比较复杂，用一个map任务去做，肯定也比较耗时。
针对上面的问题2和3，我们需要采取两种方式来解决：即减少map数和增加map数；