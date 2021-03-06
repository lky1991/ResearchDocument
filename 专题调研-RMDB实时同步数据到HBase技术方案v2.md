#专题调研-RMDB实时同步数据到HBase技术方案
##调研目的
	
	由于目前很多企业的数据都存储在Oracle中，但关系型数据库存在的天然局限性，导致无法大量应用到大数据平台的业务需求中，例如对大量数据的导出和批量处理工作。因此为了能够满足更多的业务需求，需要提供一套从RMDB中同步到HBase的技术方案。
##解决思路
* 分析Oracle是否具有同MySql一致的binlog日志文件
* 如果存在类似的日志文件，是否可以进行解析，如何解析，并提供Demo
* 对于日志的解析工作如何完成同步工作
* 提供完整的解决方案

##解决方案
###Oracle日志说明
根据研究发现，Oracle存在与Mysql类似的日志文件，但分为：在线重做日志、归档日志两种。日志从产生到归档整个流程见下图
![oracle日志归档流程图](oracle日志归档流程图.png)

* 当Oracle数据库存在对于对数据变更的操作时，会执行1操作，将操作日志写入在线重做日志中
* 当在线重做日志达到 归档条件（在线重做日志的空间配额写满、手工或者自动进行日志切换）后执行4过程，将日志写入到归档日志中
	* 当在线重做日志已经达到设定的最大空间，且还未结束归档操作时，对于新增写入的日志，则通过2操作，写入到日志缓冲区中
	* 当在线重做日志归档结束后，执行3操作，将缓冲区的数据全量刷入到在线重做日志中

####在线重做日志
	在线重做日志，又称联机重做日志，Oracle会将执行的insert,update, delete,create,drop操作的信息（select语句除外，除非select语句产生锁定、select在子查询中或者数据库开启审计时）以一种Oracle规定的方式记录到在线重做日志中，主要用于数据的审计和恢复。如果数据库处于归档模式，在线重做日志达到 归档条件（在线重做日志的空间配额写满、手工或者自动进行日志切换）后，会将其写入归档日志中。

select语句会产生日志的情况如下：

* select操作产生锁定时，比如select ....for update
* select在子查询中，比如：create table as select...., insert into a select等会产生日志
*  数据库开启审计

    
####归档日志
	所谓归档日志，就是指在线重做日志在满足归档条件（在线重做日志的空间配额写满、手工或者自动进行日志切换）时，Oracle将在线重做日志以文件形式保存到硬盘进行持久化，便于以后的恢复和查询。前提条件是数据库要处于归档模式。
###Oracle日志解析
	对于Oracle日志的解析，根据研究发现，在线日志和归档日志的日志内容是一致的，因此只需要一套解析程序即可。下面主要对解析程序进行核心说明：

####Logminer工具
	LogMiner 是Oracle公司从产品8i以后提供的一个实际非常有用的分析工具，使用该工具可以轻松获得Oracle 重做日志文件（归档日志文件）中的具体内容，通过该工具可以获取到的信息如下：

* TIMESTAMP： 数据改变发生的时间
* COMMIT_TIMESTAMP： 数据改变提交的时间
* SEG_OWNER ：段的所有者名称
* SEG_NAME ：数据发生改变的段名称
* SEG_TYPE ：数据发生改变的段类型
* SEG_TYPE_NAME： 数据发生改变的段类型名称
* TABLE_SPACE 变化段的表空间
* ROW_ID： 特定数据变化行的ID
* SESSION_INFO ：数据发生变化时用户进程信息，如login_username=WACOS client_info= * * * * OS_username=oracle Machine_name=ITDB OS_terminal= OS_process_id=2782 OS_program * name=sqlplus@ITDB (TNS V1-V3)
* **OPERATION**： 重做记录中记录的操作(如insert，update,drop,ddl)
* **SQL_REDO** ：可以为重做记录重做指定行变化的SQL语句(使数据发生改变的sql语句)
* SQL_UNDO 可以为重做记录回退或恢复指定行变化的SQL语句

通过上面的分析发现，我们可以通过Logminer工具对日志进行挖掘，来获取使数据发生该变的sql语句（SQL_REDO的信息），因此，对于Oracle日志的解析我们可以通过Logminer来实现。
####在线重做日志解析
	由于数据库中数据发生改变时，Oracle会实时记录数据库的数据更新到重做日志中，并且每一个改变都对应唯一的scn（序列改变号），因此我们可以根据scn号来解析导致数据库数据发生改变的sql语句，每一次解析完就更新当前解析到的scn号。
####归档日志解析
	处于归档模式的数据库，当在线重做日志的空间配额写满或者手工、自动进行日志切换时，会产生归档日志，归档日志不会出现覆盖，一旦产生就会永久存在，并且日志文件的编号是递增的。因此我们可以根据日志文件的编号来加载日志文件进行sql语句的解析，每一次解析完就更新最近一次解析的日志的编号。

###Oracle同步完整解决方案
####整体技术架构
	采用流式数据处理技术架构，主要原因是在整个同步过程中，核心部分在于对在线重做日志的日志采集，该日志存在周期性写入磁盘的机制，因此为了更加高效的同步数据，采取实时获取的方案。
![Oracle同步机制](Oracle同步机制.png)

* 首先定制化开发Flume
	* 在Flume中加入对Oracle日志的解析机制，将日志中的日志内容解析出可以理解的SQL语句
* 首次全量同步Oracle数据
	* 首次同步，采用从归档日志中读取所有操作记录的SQL语句，在归档日志中存储了Oracle中所有数据操作的记录
* 实时同步Oracle数据
	* 实时同步，直接从在线重做日志中读取，由于在线重做日志中记录的是目前对于Oracle进行操作的记录内容，具有更高的实时性
* 数据写入Kafka和HBase
	* 对于采集到的日志解析成SQL语句后写入Kafka
	* 由Spark Streaming对SQL进行翻译工作，并将翻译后的数据写入HBase
   