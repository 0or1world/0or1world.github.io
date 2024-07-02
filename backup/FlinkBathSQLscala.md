**读文本**
```
val env = ExecutionEnvironment.getExecutionEnvironment

    val tableENV = TableEnvironment.getTableEnvironment(env)
    //读取文本数据
    val unit = env.readTextFile("J:\\idea2019\\works\\sjmyexam03\\ru.txt")
    //隐式转换
    import org.apache.flink.api.scala._
    import org.apache.flink.table.api.scala._
    //使用map将读取到的数据切割放入元组(将数据放入DataSet)
    val unit1:DataSet[(String,String,String)] = unit.map(
      x => {
        val strings = x.split(",")

        (strings(0), strings(1), strings(2))

      }

    )
  //注册表                   //表名   //DataSet //字段名  
    tableENV.registerDataSet("stu",unit1,'ids,'name,'jk)
                              //sqlQuery
    val table = tableENV.sqlQuery(" select ids,name,jk from stu where ids = '1'")

    tableENV.toDataSet[Row](table).print()


  }
```
**读数组**
```
     val env = ExecutionEnvironment.getExecutionEnvironment

    val tableENV = TableEnvironment.getTableEnvironment(env)

    import org.apache.flink.api.scala._
    import org.apache.flink.table.api.scala._

    val tuples = List(
      ("u001", "alex", "jk"),
      ("u002", "lucy", "lolita"),
      ("u003", "rose", "jk")
    )

    val tuple3 = List(
      ("u001", "女"),
      ("u002", "女"),
      ("u003", "男")
    )



      //转换数据
    val unit = env.fromCollection(tuples)

    val value = env.fromCollection(tuple3)
    //注册表
    tableENV.registerDataSet("pro",unit,'ids,'name,'likes)

    tableENV.registerDataSet("my",value,'ids,'sex)
    //查询
    val table = tableENV.sqlQuery("select * from pro p left join my j on p.ids=j.ids ")


    tableENV.toDataSet[Row](table).print()