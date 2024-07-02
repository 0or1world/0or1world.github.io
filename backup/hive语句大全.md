hive数据库是hdfs上的文件夹，表也是文件夹，表里的数据是文件
hive建表
create table t_student(id string,name string,age int,classNo string)
row format delimited
fields terminated by ',';

创建外部表
create external table t_a(id string,name string)
row format delimited
fields terminated by ','
location '/.../...'
外部表和内部表的区别，drop时内部表的hdfs数据一同删除，外部表的hdfs上的数据不删除

create table t_b(id string,name string)
row format delimited
fields terminated by ','

create table t_a(id string,name string)
row format delimited
fields terminated by ','

--加载数据从hdfs上加载
load data inpath '/dataB/b.txt' into table t_a;
--加载数据从本地上加载
load data local inpath '/root/bb.txt' into table t_b;

t_a
+---------+-----------+--+
| t_a.id | t_a.name |
+---------+-----------+--+
| a | 1 |
| b | 2 |
| c | 4 |
| d | 5 |
+---------+-----------+--+

t_b
+---------+-----------+--+
| t_b.id | t_b.name |
+---------+-----------+--+
| a | xx |
| b | yy |
| d | zz |
| e | pp |
+---------+-----------+--+

--笛卡尔积
select a.,b.
from t_a inner join t_b

+---------+-----------+---------+-----------+--+
| t_a.id | t_a.name | t_b.id | t_b.name |
+---------+-----------+---------+-----------+--+
| a | 1 | a | xx |
| b | 2 | a | xx |
| c | 4 | a | xx |
| d | 5 | a | xx |
| a | 1 | b | yy |
| b | 2 | b | yy |
| c | 4 | b | yy |
| d | 5 | b | yy |
| a | 1 | d | zz |
| b | 2 | d | zz |
| c | 4 | d | zz |
| d | 5 | d | zz |
| a | 1 | e | pp |
| b | 2 | e | pp |
| c | 4 | e | pp |
| d | 5 | e | pp |
+---------+-----------+---------+-----------+--+

--内连接
select * from t_a join t_b where t_a.id = t_b.id;
select * from t_a join t_b on t_a.id = t_b.id;

--左外连接
select
a.,b.
from t_a a left join t_b b

select
a.,b.
from t_a a left join t_b b on a.id = b.id

+-------+---------+-------+---------+--+
| a.id | a.name | b.id | b.name |
+-------+---------+-------+---------+--+
| a | 1 | a | xx |
| b | 2 | b | yy |
| d | 5 | d | zz |
| NULL | NULL | e | pp |
+-------+---------+-------+---------+--+

--右外连接
select
a.,b.
from t_a a right outer join t_b b

select
a.,b.
from t_a a right outer join t_b b on a.id = b.id
+-------+---------+-------+---------+--+
| a.id | a.name | b.id | b.name |
+-------+---------+-------+---------+--+
| a | 1 | a | xx |
| b | 2 | b | yy |
| d | 5 | d | zz |
| NULL | NULL | e | pp |
+-------+---------+-------+---------+--+

--全外连接
select
a.,b.
from t_a a full outer join t_b b on a.id = b.id

+-------+---------+-------+---------+--+
| a.id | a.name | b.id | b.name |
+-------+---------+-------+---------+--+
| a | 1 | a | xx |
| b | 2 | b | yy |
| c | 4 | NULL | NULL |
| d | 5 | d | zz |
| NULL | NULL | e | pp |
+-------+---------+-------+---------+--+

--左半连接 相当于指定条件的内连接，但是只显示左边的表数据
select
a.*
from t_a a left semi join t_b b on a.id = b.id

+-------+---------+--+
| a.id | a.name |
+-------+---------+--+
| a | 1 |
| b | 2 |
| d | 5 |
+-------+---------+--+

--例如有这些数据
vi access.log.0804
192.168.33.3,http://www.sina.com/stu,2017-08-04 15:30:20
192.168.33.3,http://www.sina.com/teach,2017-08-04 15:35:20
192.168.33.4,http://www.sina.com/stu,2017-08-04 15:30:20
192.168.33.4,http://www.sina.com/job,2017-08-04 16:30:20
192.168.33.5,http://www.sina.com/job,2017-08-04 15:40:20

vi access.log.0805
192.168.33.3,http://www.sina.com/stu,2017-08-05 15:30:20
192.168.44.3,http://www.sina.com/teach,2017-08-05 15:35:20
192.168.33.44,http://www.sina.com/stu,2017-08-05 15:30:20
192.168.33.46,http://www.sina.com/job,2017-08-05 16:30:20
192.168.33.55,http://www.sina.com/job,2017-08-05 15:40:20

vi access.log.0806
192.168.133.3,http://www.sina.com/register,2017-08-06 15:30:20
192.168.111.3,http://www.sina.com/register,2017-08-06 15:35:20
192.168.34.44,http://www.sina.com/pay,2017-08-06 15:30:20
192.168.33.46,http://www.sina.com/excersize,2017-08-06 16:30:20
192.168.33.55,http://www.sina.com/job,2017-08-06 15:40:20
192.168.33.46,http://www.sina.com/excersize,2017-08-06 16:30:20
192.168.33.25,http://www.sina.com/job,2017-08-06 15:40:20
192.168.33.36,http://www.sina.com/excersize,2017-08-06 16:30:20
192.168.33.55,http://www.sina.com/job,2017-08-06 15:40:20

--建分区表
create table t_access(ip string,url string,access_time string)
partitioned by (day string)
row format delimited
fields terminated by ','

--导入分区表数据
load data local inpath '/root/access.log.0806' into table t_access partition(day='2017-08-06');

--查看当前分区情况
show partitions t_access;

--求每条url访问的次数
select
url,count(1)
from t_access
group by url;

--求每条url访问的ip中，最大的ip是哪个
select
url,max(ip)
from t_access
group by url;

--求每个ip访问同一个页面的记录中，最晚的一条
select
ip,url,max(access_time)
from t_access
group by ip,url

--求PV
select
url,count(1)
from t_access
group by url

+--------------------------------+------+--+
| url | _c1 |
+--------------------------------+------+--+
| http://www.sina.com/excersize | 3 |
| http://www.sina.com/job | 7 |
| http://www.sina.com/pay | 1 |
| http://www.sina.com/register | 2 |
| http://www.sina.com/stu | 4 |
| http://www.sina.com/teach | 2 |
+--------------------------------+------+--+

--求UV
select
url,count(distinct(ip))
from t_access
group by url

+--------------------------------+-----+--+
| url | c1 |
+--------------------------------+-----+--+
| http://www.sina.com/excersize | 2 |
| http://www.sina.com/job | 5 |
| http://www.sina.com/pay | 1 |
| http://www.sina.com/register | 2 |
| http://www.sina.com/stu | 3 |
| http://www.sina.com/teach | 2 |
+--------------------------------+-----+--+

--求每条url访问的ip中，最大的ip是哪个
--PV
--UV
--用mapreduce实现

--求8月6号pv，uv
select
count(1),url
from t_access
where day = '2017-08-06'
group by url

--求8月4号以后，每天访问http://www.sina.com/job 的次数，以及访问者中ip最大的
select
count(1),max(ip),day
from t_access
where day > '2017-08-04'
and url = 'http://www.sina.com/job'
group by day

select
count(1),max(ip),day
from t_access
where url = 'http://www.sina.com/job'
group by day having day > '2017-08-04'

--求8月4号以后，每天每个页面访问总次数，以及页面最大的ip，并且上述查询结果中访问次数大于2的
select tmp.* from
(select
url,day,count(1) cnts,max(ip) ip
from t_access
group by day,url having day > '2017-08-04') tmp
where tmp.cnts > 2

--每天，pv排序
select
url,day,count(1) cnts
from t_access
group by day,url
order by cnts desc

--hive里的order by是全局排序
--sort by是在map里进行排序

select
tmp.*
from
(select
url,day,count(1) cnts
from t_access
group by day,url) tmp
distribute by tmp.day sort by tmp.cnts desc

--有如下数据
+-------------+---------------+--+
| t_sale.mid | t_sale.money |
+-------------+---------------+--+
| AA | 15.0 |
| AA | 20.0 |
| BB | 22.0 |
| CC | 44.0 |
+-------------+---------------+--+

--distribute by先把数据分发到各个reduce中，然后sort by在各个reduce中进行局部排序
select
mid,money
from t_sale
distribute by mid sort by money desc

+------+--------+--+
| mid | money |
+------+--------+--+
| AA | 15.0 |
| AA | 20.0 |
| BB | 22.0 |
| CC | 44.0 |
+------+--------+--+

--cluster by mid 等于 distribute by mid sort by mid
--cluster by后面不能跟desc，asc，默认的只能升序

--order by 是全排序，所有的数据会发送给一个reduceTask进行处理，在数据量大的时候，reduce就会超负荷
select
mid,money
from t_sale
order by money desc

--设置最大的reduce启动个数
set hive.exec.reducers.max=10;
--设置reduce的启动个数
set mapreduce.job.reduce=3

--hive数据类型
--数字类型
tinyint 1byte -128到127
smallint 2byte -32768到32767 char
int
bigint
float
double

--日期类型
timestamp
date

--字符串类型
string
varchar
char

--混杂类型
boolean
binary 二进制

--复合类型
--数组

流浪地球,吴京:吴孟达:张飞:赵云,2019-09-11
我和我的祖国,葛优:黄渤:宋佳:吴京,2019-10-01

create table t_movie(movie_name string,actors array<string>,show_time date)
row format delimited fields terminated by ','
collection items terminated by ':';

array_contains

select

from t_movie
where array_contains(actors,'张飞')

size

select
movie_name,actors,show_time,size(actors) num
from t_movie

========================================

map类型
1,zhangsan,father:xiaoming#mother:chentianxing#brother:shaoshuai,28
2,xiaoming,father:xiaoyun#mother:xiaohong#sister:xiaoyue#wife:chentianxing,30

create table t_user(id int,name string,family map<string,string>,jage int)
row format delimited fields terminated by ','
collection items terminated by '#'
map keys terminated by ':';

map_keys
select
id,name,map_keys(family) as relation,jage
from t_user;

map_values
select
id,name,map_values(family) as relation,jage
from t_user;

--家庭成员数量
select
id,name,size(family),jage
from t_user

--有兄弟的人
select

from t_user
where array_contains(map_keys(family),'brother')

========================================
1,zhangsan,18:man:beijing
2,lisi,22:woman:shanghai

create table t_people(id int,name string,info struct<age:int,sex:string,addr:string>)
row format delimited fields terminated by ','
collection items terminated by ':';

select
id,name,info.age
from t_people;

========================================

192,168,33,66,zhangsan,male
192,168,33,77,lisi,male
192,168,43,101,wangwu,female

create table t_people_ip(ip_seg1 string,ip_seg2 string,ip_seg3 string,ip_seg4 string,name string,sex string)
row format delimited fields terminated by ',';

--字符串拼接函数
concat
select concat('abc','def')

concat_ws
select concat_ws('.',ip_seg1,ip_seg2,ip_seg3,ip_seg4),name,sex from t_people_ip

===================================================================

--求字符串长度
length
select lenth('jsdfijsdkfjkdsfjkdf');

========================================

select to_date('2019-09-11 16:55:11');

--把字符串转换成unix时间戳
select unix_timestamp('2019-09-11 11:55:11','yyyy-MM-dd HH:mm:ss');

--把unix时间戳转换成字符串
select from_unixtime(unix_timestamp(),'yyyy-MM-dd HH:mm:ss');

========================================

--数学运算函数
--四舍五入
select round(5.4)
--四舍五入保留2位小数
select round(5.1345,2)
--向上取整
select ceil(5.3)
--向下取整
select floor(5.3)
--取绝对值
select abs(-5.2)
--取最大值
select greatest(3,4,5,6,7)
--取最小值
select least(3,4,5,6,7)

==================================================

1,19,a,male
2,19,b,male
3,22,c,female
4,16,d,female
5,30,e,male
6,26,f,female

+---------------+----------------+-----------------+----------------+--+
| t_student.id | t_student.age | t_student.name | t_student.sex |
+---------------+----------------+-----------------+----------------+--+
| 1 | 19 | a | male |
| 2 | 19 | b | male |
| 5 | 30 | e | male |

| 3 | 22 | c | female |
| 4 | 16 | d | female |
| 6 | 26 | f | female |

+---------------+----------------+-----------------+----------------+--+

row_number() over()

select
id,age,name,sex,row_number() over(partition by sex order by age desc) rk
from t_student

select
tmp.*
from
(select
id,age,name,sex,row_number() over(partition by sex order by age desc) rk
from t_student) tmp
where tmp.rk = 1

select
id,age,name,sex,rank() over(partition by sex order by age desc) rk
from t_student

select
id,age,name,sex,dense_rank() over(partition by sex order by age desc) rk
from t_student

========================================

A,2015-01,5
A,2015-01,15
B,2015-01,5
A,2015-01,8
B,2015-01,25
A,2015-01,5
C,2015-01,10
C,2015-01,20
A,2015-02,4
A,2015-02,6
C,2015-02,30
C,2015-02,10
B,2015-02,10
B,2015-02,5
A,2015-03,14
A,2015-03,6
B,2015-03,20
B,2015-03,25
C,2015-03,10
C,2015-03,20

--求每个用户每个月的销售额和到当月位置的累计销售额
create table t_saller(name string,month string,amount int)
row format delimited fields terminated by ','

create table t_accumulate
as
select name,month,sum(amount) samount
from t_saller
group by name,month

+--------------------+---------------------+-----------------------+--+
| t_accumulate.name | t_accumulate.month | t_accumulate.samount |
+--------------------+---------------------+-----------------------+--+
| A | 2015-01 | 33 |
| A | 2015-02 | 10 |
| A | 2015-03 | 20 |
| B | 2015-01 | 30 |
| B | 2015-02 | 15 |
| B | 2015-03 | 45 |
| C | 2015-01 | 30 |
| C | 2015-02 | 40 |
| C | 2015-03 | 30 |
+--------------------+---------------------+-----------------------+--+
--最前面一行
select
name,month,samount,sum(samount) over(partition by name order by month rows between unbounded preceding and current row) accumlateAmount
from
t_accumulate

| A | 2015-01 | 33 |
| A | 2015-02 | 10 |
| A | 2015-03 | 20 |
| A | 2015-04 | 30 |
| A | 2015-05 | 15 |
| A | 2015-06 | 45 |
| A | 2015-07 | 30 |
| A | 2015-08 | 40 |
| A | 2015-09 | 30 |

select
name,month,samount,sum(samount) over(partition by name order by month rows between 2 preceding and 1 following ) accumlateAmount
from
t_accumulate

preceding |
当前行 | 窗口长度
following |

min() over() ,max() over() , avg() over()

========================================

1,zhangsan,化学:物理:数学:语文
2,lisi,化学:数学:生物:生理:卫生
3,wangwu,化学:语文:英语:体育:生物

create table t_stu_subject(id int,name string,subjects array<string>)
row format delimited fields terminated by ','
collection items terminated by ':'

explode()

select explode(subjects) from t_stu_subject;

select distinct tmp.subs from (select explode(subjects) subs from t_stu_subject) tmp

========================================

lateral view连接函数

select
id,name,sub
from
t_stu_subject lateral view explode(subjects) tmp as sub

+-----+-----------+------+--+
| id | name | sub |
+-----+-----------+------+--+
| 1 | zhangsan | 化学 |
| 1 | zhangsan | 物理 |
| 1 | zhangsan | 数学 |
| 1 | zhangsan | 语文 |
| 2 | lisi | 化学 |
| 2 | lisi | 数学 |
| 2 | lisi | 生物 |
| 2 | lisi | 生理 |
| 2 | lisi | 卫生 |
| 3 | wangwu | 化学 |
| 3 | wangwu | 语文 |
| 3 | wangwu | 英语 |
| 3 | wangwu | 体育 |
| 3 | wangwu | 生物 |
+-----+-----------+------+--+

========================================

wordcount

words

create table words(line string)

line
hello world hi tom and jack
hello chentianxing qiaoyuan and shaoshuai
hello hello hi tom and shaoshuai
chentianxing love saoshuai
hello love what is love how love

split()

select
tmp.word,count(1) cnts
from
(select
explode(split(line,' ')) word
from words) tmp
group by tmp.word order by cnts desc

========================================

--炸map
select
id,name,key,value
from
t_user
lateral view explode(family) tmp as key,value

========================================

--有web系统，每天产生下列的日志文件

2017-09-15号的数据：
192.168.33.6,hunter,2017-09-15 10:30:20,/a
192.168.33.7,hunter,2017-09-15 10:30:26,/b
192.168.33.6,jack,2017-09-15 10:30:27,/a
192.168.33.8,tom,2017-09-15 10:30:28,/b
192.168.33.9,rose,2017-09-15 10:30:30,/b
192.168.33.10,julia,2017-09-15 10:30:40,/c

2017-09-16号的数据：
192.168.33.16,hunter,2017-09-16 10:30:20,/a
192.168.33.18,jerry,2017-09-16 10:30:30,/b
192.168.33.26,jack,2017-09-16 10:30:40,/a
192.168.33.18,polo,2017-09-16 10:30:50,/b
192.168.33.39,nissan,2017-09-16 10:30:53,/b
192.168.33.39,nissan,2017-09-16 10:30:55,/a
192.168.33.39,nissan,2017-09-16 10:30:58,/c
192.168.33.20,ford,2017-09-16 10:30:54,/c

2017-09-17号的数据：
192.168.33.46,hunter,2017-09-17 10:30:21,/a
192.168.43.18,jerry,2017-09-17 10:30:22,/b
192.168.43.26,tom,2017-09-17 10:30:23,/a
192.168.53.18,bmw,2017-09-17 10:30:24,/b
192.168.63.39,benz,2017-09-17 10:30:25,/b
192.168.33.25,haval,2017-09-17 10:30:30,/c
192.168.33.10,julia,2017-09-17 10:30:40,/c

--统计日活跃用户

--统计每日新增用户

--统计历史用户

--创建日志表
create table t_web_log(ip string,uname string,access_time string,url string)
partitioned by (day string)
row format delimited fields terminated by ',';

--创建日活跃用户表
create table t_user_active_day
like
t_web_log;

--统计日活跃用户
insert into table t_user_active_day partition(day='2017-09-15')
select tmp.ip,tmp.uname,tmp.access_time,tmp.url
from
(select
ip,uname,access_time,url,row_number() over(partition by uname order by access_time) rn
from
t_web_log where day='2017-09-15') tmp
where rn <2

--创建历史用户表
create table t_user_history(uname string)

--创建日新增用户表
create table t_user_new_day like t_user_active_day;

--统计日新增用户
insert into table t_user_new_day partition(day='2017-09-15')
select
tua.ip,tua.uname,tua.access_time,tua.url
from t_user_active_day tua
left join t_user_history tuh on tua.uname = tuh.uname
where tua.day = '2017-09-15' and tuh.uname IS NULL

--记录历史用户
insert into table t_user_history
select
uname
from t_user_new_day
where day='2017-09-15'

insert into table t_user_active_day partition(day='2017-09-16')
select tmp.ip,tmp.uname,tmp.access_time,tmp.url
from
(select
ip,uname,access_time,url,row_number() over(partition by uname order by access_time) rn
from
t_web_log where day='2017-09-16') tmp
where rn <2

insert into table t_user_new_day partition(day='2017-09-16')
select
tua.ip,tua.uname,tua.access_time,tua.url
from t_user_active_day tua
left join t_user_history tuh on tua.uname = tuh.uname
where tua.day = '2017-09-16' and tuh.uname IS NULL

insert into table t_user_history
select
uname
from t_user_new_day
where day='2017-09-16'

--桶表
--创建桶表，按id分成三个桶
create table t1(id int,name string,age int)
clustered by(id) into 3 buckets
row format delimited fields terminated by ',';

--桶表插入数据的方式
create table t1_1(id int,name string,age int)
row format delimited fields terminated by ',';

set hive.enforce.bucketing=true;
set mapreduce.job.reduces=3;

insert into t1 select * from t1_1;

========================================
hive自定义函数
package com.bwie.hive.udf;

import com.alibaba.fastjson.JSONObject;
import org.apache.hadoop.hive.ql.exec.UDF;

public class JsonUDF extends UDF {

public String evaluate(String json,String colum){
    JSONObject jsonObject = JSONObject.parseObject(json);
    String value = jsonObject.getString(colum);
    return value;
}
}

打包上传
add jar /root/hivetest-1.0-SNAPSHOT.jar;
create temporary function getjson as 'com.bwie.hive.udf.JsonUDF';

select
getjson(json,'movie') as movie,getjson(json,'rate') as rate,from_unixtime(CAST(getjson(json,'timeStamp') as BIGINT),'yyyy-MM-dd HH:mm:ss') as showtime,getjson(json,'uid') as uid
from t_ex_rat limit 10;

--创建永久函数
方法一、
在hive-site.xml里添加jar包
<property>
<name>hive.aux.jars.path</name>
<value>file:///usr/local/apache-hive-1.2.2-bin/lib/hivetest-1.0-SNAPSHOT.jar</value>
</property>
疑似jar包要放到lib下才能生效
create function getjson AS 'com.bwie.hive.udf.JsonUDF'

方法二、
create function getjson as 'com.bwie.hive.udf.JsonUDF' using jar 'hdfs://hdp1703/lib/hivetest-1.0-SNAPSHOT.jar'

drop function getjson

========================================

create table t_employee(id int,name string,age int)
partitioned by(state string,city string)
row format delimited fields terminated by ','

create table t_employee_orgin(id int,name string,age int,state string,city string)
row format delimited fields terminated by ',';

load data local inpath '/root/employ.txt' into table t_employee_orgin;

在严格模式下，多个分区一定有一个分区是固定的
insert into t_employee
partition(state='china',city)
select id,name,age,city
from t_employee_orgin
where state='china'

--非严格模式
set hive.exec.dynamic.partition.mode=nostrict;
insert into t_employee
partition(state,city)
select id,name,age,state,city
from t_employee_orgin

========================================

select to_date('2019-09-20 17:30:00')

unix_timestamp('2019/09/20 17:30:00','yyyy/MM/dd HH:mm:ss')

select from_unixtime(unix_timestamp('2019/09/20 17:30:00','yyyy/MM/dd HH:mm:ss'),'yyyy-MM-dd HH:mm:ss')