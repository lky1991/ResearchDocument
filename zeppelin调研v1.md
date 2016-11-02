###zeppelin概述
 Zeppelin是一个Web notebook形式的交互式数据查询分析工具，可以在线用scala、sql等语言对数据进行查询分析并生成报表。

###支持接入的数据处理引擎
* Spark
* Hive
* Tajo

###支持的交互语言
* Scala
* Python
* SparkSQL
* Markdown 
* Shell

 如果在Zeppelin notebook中编写代码，默认会被解释为Scala语言。编写python代码时第一行要用%pyspark 标识，SparkSQL代码要用%sql 标识，Markdown代码要用%md标识，Shell代码要用%sh标识。

###支持添加外部依赖
在实际应用中，我们编写程序一般都是需要依赖外部的相关类库的，比如我们现在需要Spark读取Mysql里面的数据，这时我们需要添加Mysql相关的依赖，下面介绍两种方法来添加外部依赖：

* 通过指定依赖库的group ID、artifact ID以及version来指定具体的依赖

      %dep  ：加入外部依赖库的标识

      z.load("mysql:mysql-connector-java:5.1.35")  

* 通过指定jar的本地路径来加载外部依赖
    
     %dep

     z.load("/path/to.jar")

###支持数据的可视化
一些基本的图表（柱状图、折线图、饼状图）都被集成在Zeppelin中。数据分析的结果可以在Zeppelin中得到漂亮的展示。
###支持动态表格
Zeppelin可以在你的notebook中动态地创建一些输入格式。
      
     例如：%md
          Hello ${name=sun}

###支持协作
Notebook的URL可以在协作者间分享并且可以实时广播任何变化。

###支持发布
Zeppelin 提供了一个 URL 用来只展示结果，那个页面不包括 Zeppelin 的菜单和按钮。你可以非常容易的将其作为一个iframe集成到你的网站。 

 
####与Spark的集成 
Zeppelin 提供了内置的 Apache Spark 集成，你不必单独构建一个模块、插件或者库就可以在Spark shell下进行编程。zeppelin为Spark提供以下服务：

* 在notebook中自动引入SparkContext 和 SQLContext
* 可以从本地文件系统或maven库载入运行时依赖的jar包
* 可以取消job的执行和展示job的执行进度


###zeppelin与Hue对比
####相同点
 都可以进行交互式的数据查询、分析
#####不同点

* zeppelin 对Hadoop生态圈的集成比较少，目前仅仅对Spark有较好的支持；而Hue对Hadoop生态圈的集成比较多，相对比较成熟。
* zeppelin 只能通过在notebook中编写代码来实现数据的查询与分析，不支持手动式的交互。
* zeppelin相对于Hue在进行数据查询分析方面更灵活（可以通过编写代码来对数据做定制处理）以及对结果有更好的可视化支持。

