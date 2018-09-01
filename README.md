# big-data-knowledge
📖大数据相关知识集锦

* [hdfs](#hdfs)
* [yarn](#yarn)
* [hive](#hive)
* [mapreduce](#mapreduce)

<h3 id="hdfs">hdfs</h3>

* HDFS简介

	HDFS是Hadoop Distributed File System的简写
	
	* HDFS具有高容错性和高吞吐性的特点
	* HDFS目前是 append only，暂时不支持随机 write 的操作
	* HDFS适合用于存储以及批量操作大规模的数据集(PB级别)
	* 不适合实时访问，具有高延迟性，例如新建了一张hive表，需要过一会才能看到
	* [Hadoop HDFS 教程（一）介绍](https://www.jianshu.com/p/8969eb90a59d)
	* [Hadoop HDFS（二）结构解析和名词解释](https://www.jianshu.com/p/86a70ac1f5f9)

* HDFS存在一个单点问题，即全Hadoop系统只有一个NameNode，如果NameNode挂了怎么办

    * 将hadoop元数据写入到本地文件系统的同时，再实时同步到一个远程挂载的网络文件系统
    * 运行一个secondaryNameNode
        * 元数据持久化到磁盘，在fsimage中存放元信息，在edits中存放对元信息的操作的文件
        * 定时到NameNode中去获取edit logs，并更新到fsimage
        * 一旦它有了新的fsimage文件，它将其拷贝回NameNode中
        * NameNode在下次重启时会使用这个新的fsimage文件，从而减少重启的时间

* HDFS中的块为什么这么大？

    HDFS的块比磁盘的块大，其目的是为了最小化寻址开销。如果块足够大，从磁盘传输数据的时间会明显大于定位这个块开始位置所需的时间。因而，传输一个由多个块组成的大文件的时间取决于磁盘传输速率

<h3 id="yarn">yarn</h3>

* yarn简介

	yarn是hadoop内部的资源管理系统
	
	* 资源管理(10k的机器数)
		* CPU，Memory...
		* 资源利用 & 共享
	* 调度/监控分布式jobs
	* 统一的接口管理
		* MapReduce
		* Spark
		* Flink

<h3 id="hive">hive</h3>

* 数据仓库(DW/Data Warehouse)分层原则(每家公司都有自己的规范)

	* dim：维度层，一般用于存储属性信息，多用于联表查询
	* dwd/ods(data warehouse detail)：事实明细层，存储事实表的明细粒度数据，比较底层的数据，源数据清洗得来，例如埋点后捞出来的数据
	* dwa(data warehouse aggregation)：事实聚合层，存储事实表聚合粒度数据，按需求联合查询得到的聚合表
	* app(application)：应用层，存储直接供给应用的数据

	![dw架构图](/imgs/dw.png)

* hive的join操作，只支持等值匹配，不支持like模糊匹配，如果非要使用like，需要使用笛卡尔积，这个效率太低，不如放到内存中匹配，下面是笛卡尔积的写法

    ```sql
    SELECT table1.brand, SUM(table2.sold) 
    FROM table1, table2
    WHERE table2.product LIKE concat('%', table1.brand, '%') 
    GROUP BY table1.brand;
    ```

* hive表分为内部表和外部表，内部表drop的时候会将hdfs上的数据**一起删除**，外部表drop的时候**不会删除**hdfs上的数据

* 创建hive表语句栗子

	```sql
	create external table table_name (
		uid bigint comment '用户id',
		name string
	) comment '用户表'
	PARTITIONED BY (`date` string)
	ROW FORMAT DELIMITED
		FIELDS TERMINATED BY `\t` // 指定每行中字段分隔符为\t
		LINES TERMINATED BY `\n` // 指定行分隔符
		COLLECTION ITEMS TERMINATED BY `,` // 指定集合中元素之间的分隔符
		MAP KEYS TERMINATED BY `:` // 指定数据中Map类型的Key与Value之间的分隔符
	LOCATION
		'hdfs://XXX'
	```
	
	external指代这张表是否为外部表

* 向hive表中加载数据

	* 建表时直接指定

		如果你的数据已经在hdfs上存在，已经为结构化的数据，并且数据所在的hdfs路径不需要维护，那么直接在create的时候指定location字段为hdfs路径即可
	
	* 从本地文件系统或者hdfs的一个目录中加载，使用 LOAD DATA命令加载数据

		```sql
		load data local inpath XXX overwrite into table partition(day = '20180808') # load 本地文件
		
		load data inpath XXX overwrite into table partition(day = '20180808') # load hdfs文件
		```
		
	* 从一个select查询中load 数据

		```sql
		insert overwrite table table_name partition(day = '20180808')
		
		select
			*
		from
			table
		where
			date = '20180808'
		```

* hive中join的原理和机制

	笼统的说，hive中的join可以分为common join(reduce阶段完成join)和map join(map阶段完成join)
	
	* map阶段
	
		读取源表的数据，map输出时候以join on条件中的列为key，如果Join有多个关联键，则以这些关联键的组合作为key。map输出的value为join之后所关心的(select或者where中需要用到的)列，同时在value中还会包含表的Tag信息，用于标明此value对应哪个表；
		
	* shuffle阶段

		根据key的值进行hash,并将key/value按照hash值推送至不同的reduce中，这样确保两个表中相同的key位于同一个reduce中
		
	* reduce阶段
		
		根据key数值完成join操作，期间通过tag来识别不同表中的数据
		
	* 例子

		```sql
		SELECT 
			a.id,
			a.dept,
			b.age 
		FROM
			a join b 
		ON
			a.id = b.id;
		```
		
		![hive-join](/imgs/hive-join.png)

* hive sql的优化

	[Hive SQL的优化](http://lxw1234.com/archives/2015/06/317.htm)

* hive函数总结

    [hive函数总结](https://www.cnblogs.com/yejibigdata/p/6380744.html)
    
* hive的text存储格式和parquet存储格式

	text是行式存储，多用于手动load数据进入hive表，例如`pandas.Dateframe.tocsv()`
	
	parquet是列式存储，在一列有很多相同数值(例如NULL和常数)这样的时候，稀疏存储能省很多空间，同时列式存储在select的时候不用遍历每行，直接遍历列就行

<h3 id="mapreduce">mapreduce</h3>

* MapReduce简介
    
    MapReduce是一个编程模型，也是一个处理和生成超大数据集的算法模型的相关实现。用户首先创建一个Map函数处理一个基于k/v pair的数据集合，输出中间的基于k/v pair的数据集合；然后再创建一个Reduce函数用来合并所有的具有相同中间key值的中间value值，MapReduce架构的程序能够在大量的普通配置的计算机上实现并行化处理，可以用于处理TB级别的数据

    ![MapReduce](/imgs/mapreduce.png)

	* 用户程序首先调用的MapReduce库将输入文件分成M个数据片段，每个数据片段的大小从16MB到512MB(可以通过可选的参数来控制每个数据片段的大小)。然后用户程序在机群中创建大量的程序副本。
	* 这些程序副本中的有一个特殊的程序 - master。副本中其它的程序都是worker程序，由master分配任务。有M个Map任务和R个Reduce任务将被分配，master将一个Map任务或Reduce任务分配给一个空闲的worker。
	* 被分配了map任务的worker程序读取相关的输入数据片段，从输入的数据片段中解析出k/v pair，然后把k/v pair传递给用户自定义的Map函数，由Map函数生成并输出的中间k/v pair，并缓存在内存中。
	* 缓存中的k/v pair 通过分区函数分成R个区域，之后周期性的写入到本地磁盘上。缓存的k/v pair在本地磁盘上的存储位置将被回传给master，由master负责把这些存储位置再传送给Reduce worker。
	* 当Reduce worker程序接收到master程序发来的数据存储位置信息后，使用RPC从Map worker所在主机的磁盘上读取这些缓存数据。当Reduce worker读取了所有的中间数据后，通过key进行排序后使得具有相同key值的数据聚合在一起。由于许多不同的key值会映射到相同的Reduce任务上，因此必须排序。如果中间数据太大无法在内存中完成排序，那么就要在外部进行排序。
	* Reduce worker程序遍历排序后的中间数据，对于每一个唯一的中间key值，Reduce worker程序将这个key值和它相关的中间 value 值的集合传递给用户自定义的 Reduce 函数。Reduce 函数的输出被追加到所属分区的输出文件
	* 当所有的 Map 和 Reduce 任务都完成之后，master 唤醒用户程序。在这个时候，在用户程序里的对 MapReduce 调用才返回。

* MapReduce的shuffle过程

    * [MapReduce shuffle过程详解](https://blog.csdn.net/u014374284/article/details/49205885) 这篇博客讲的还阔以，但是有两个地方有问题，一是key通过hash取模获得partition是在进入kvbuffer之后，二是reduce worker从map worker copy数据不是通过http，二是通过rpc

    * [Hadoop深入学习：MapReduce的Shuffle过程详解](http://flyingdutchman.iteye.com/blog/1879642)

* mr的inputfile可以写多个，可以在map.py中通过数据格式来区分不同的文件，也可以通过环境变量来得到hdfs上文件的绝对路径

    [在mr streaming中获取文件名](https://blog.csdn.net/bitcarmanlee/article/details/51735053)

* Hadoop Streaming

    ![Hadoop Streaming 计算过程](./imgs/hadoop_streaming.jpg)
    
    python编写mapreduce就是使用了Hadoop Streaming的特点
    
    * Streaming的优点：
    	* 开发效率高
    		* 只需按照一定的格式从标准输入读取数据、向标准输出写数据就行
    		* 容易单机调试: cat input | mapper | sort | reducer > output
    	* 程序运行效率高
			* 对于CPU密集的计算，有些语言如C/C++编写的程序可能比用Java效率高一些
		* 便于平台进行资源控制
			* Streaming框架中通过limit等方式可以灵活地限制应用程序使用的内存资源
	* Streaming的局限
		* Streaming默认只能处理文本数据
		* 两次数据拷贝和解析（分割），带来一定的开销
	
	* Streaming的开发要点：
		* input：指定输入文件的HDFS路径，支持使用*通配符和指定多个文件或目录，可多次使用
		* output：指定输出文件的HDFS路径，路径必须不存在，且具备创建该目录的权限，只能使用一次
		* mapper：用户自己写的mapper程序
		* reduer：用户自己写的reduce程序
		* file：打包文件到提交的作业中
			* map和reduce的执行文件，如run.sh
			* map和reduce要用输入的文件，如配置文件
			* 还有-cacheFile, -cacheArchive分别用于向计算节点分发HDFS文件和HDFS压缩文件
		* jobconf：提交作业的一些配置属性，常见配置：
			* mapred.map.tasks：map task数目
			* mapred.reduce.tasks：reduce task数目
			* stream.num.map.output.key.fields：指定map task输出记录中key所占的域数目
			* num.key.fields.for.partition：指定对key分出来的前几部分做partition，而非整个key
			* mapred.compress.map.output：map的输出是否压缩
			* mapred.map.output.compression.codec：map的输出压缩方式
			* mapred.output.compress：reduce的输出是否压缩
			* mapred.output.compression.codec：reduce的输出压缩方式
    		
* mapreduce中的combine阶段，众所周知，mapreduce中有map和reduce两个阶段，其实还有一个用户可以选择的combine阶段，对map出来的数据进行预聚合，减少传递给reduce worker的数据量，加快处理速度，例如，求出某个key的最大值，就可以在map worker中取对应的key的最大值，不用将所有的数据都丢给reduce worker
