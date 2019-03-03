# big-data-knowledge
📖大数据相关知识集锦

* [hdfs](#hdfs)
* [yarn](#yarn)
* [hive](#hive)
* [mapreduce](#mapreduce)
* [spark](#spark)
* [hbase](#hbase)
* [zookeeper](#zk)
* [kafka](#kafka)
* [nsq](#nsq)

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

* HDFS的读流程和写流程

    读过程

    ![read-hdfs](./imgs/read-hdfs.jpg)

    写过程

    ![write-hdfs](./imgs/write-hdfs.jpg)

* HDFS通过CRC校验来保证数据的正确性
 
* proquet列式存储

    * [深入分析Parquet列式存储格式](http://www.infoq.com/cn/articles/in-depth-analysis-of-parquet-column-storage-format)
    * [Dremel made simple with Parquet](https://blog.twitter.com/engineering/en_us/a/2013/dremel-made-simple-with-parquet.html)
    * [Dremel: Interactive Analysis of Web-Scale Datasets](http://static.googleusercontent.com/media/research.google.com/zh-CN//pubs/archive/36632.pdf)

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

* YARN是hadoop的集群资源管理系统，YARN被引入Hadoop 2，最初是为了改善MapReduce的实现，但它具有足够的通用性，也可以用于其他的分布式计算模式，例如Spark，那么MapReduce1和YARN的区别是啥呢？

    MapReduce1中，有两类守护进程控制者作业的执行过程：一个`jobtracker`及一个或多个`tasktracker`。jobtracker通过调度tasktracker上运行的任务来协调所有运行在系统上的作业。tasktracker在运行任务的同时将运行进度报告发送给jobtracker，jobtracker由此记录每项作业任务的整体进度情况。如果其中一个任务失败，jobtracker可以在另一个tasktracker节点上重新调度该任务。

    MapReduce1中，jobtracker同时负责作业调度(将任务与tasktracker匹配)和任务进度监控(跟踪任务、重启失败或迟缓的任务；记录任务流水，如维护计数器的计数)。相比之下，YARN中，这些职责是由不同的实体担负的：资源管理器和application master(每个 MapReduce 作业一个)。jobtracker也负责存储已完成作业的作业历史。在YARN中，与之等价的角色是时间轴服务器，它主要用于存储应用历史。

    YARN中与tasktracker等价的角色是节点管理器。
    
    | MapReduce1 | YARN |
    | ---------- | ---- |
    | Jobtracker | 资源管理器、application master、时间轴服务器|
    | Tasktracker| 节点管理器 |
    | Slot | 容器 |
    
* YARN中存在三种调度方法

	* FIFO
	* 容器调度器
	* 公平调度器

<h3 id="hive">hive</h3>

* 数据仓库(DW/Data Warehouse)分层原则(每家公司都有自己的规范)

	* dim：维度层，一般用于存储属性信息，多用于联表查询
	* dwd/ods(data warehouse detail)：事实明细层，存储事实表的明细粒度数据，比较底层的数据，源数据清洗得来，例如埋点后捞出来的数据
	* dwa(data warehouse aggregation)：事实聚合层，存储事实表聚合粒度数据，按需求联合查询得到的聚合表
	* app(application)：应用层，存储直接供给应用的数据

	![dw架构图](./imgs/dw.png)

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
		
		![hive-join](./imgs/hive-join.png)

* hive sql的优化

	[Hive SQL的优化](http://lxw1234.com/archives/2015/06/317.htm)

* hive函数总结

    [hive函数总结](https://www.cnblogs.com/yejibigdata/p/6380744.html)
    
* hive的text存储格式和parquet存储格式

	text是行式存储，多用于手动load数据进入hive表，例如`pandas.Dateframe.tocsv()`
	
	parquet是列式存储，在一列有很多相同数值(例如NULL和常数)这样的时候，稀疏存储能省很多空间，同时列式存储在select的时候不用遍历每行，直接遍历列就行
	
* hive中的压缩设置

	* hive.exec.compress.intermediate：默认该值为false，设置为true为激活中间数据压缩功能。HiveQL语句最终会被编译成Hadoop的Mapreduce job，开启Hive的中间数据压缩功能，就是在MapReduce的shuffle阶段对mapper产生的中间结果数据压缩。在这个阶段，优先选择一个低CPU开销的算法。
	* mapred.map.output.compression.codec：该参数是具体的压缩算法的配置参数，SnappyCodec比较适合在这种场景中编解码器，该算法会带来很好的压缩性能和较低的CPU开销。
	* hive.exec.compress.output：用户可以对最终生成的Hive表的数据通常也需要压缩。该参数控制这一功能的激活与禁用，设置为true来声明将结果文件进行压缩。
	* mapred.output.compression.codec：将hive.exec.compress.output参数设置成true后，然后选择一个合适的编解码器，如选择SnappyCodec。

		```
		set hive.exec.compress.intermediate=true;
		set mapred.map.output.compression.codec=org.apache.hadoop.io.compress.SnappyCodec;
		set hive.exec.compress.output=true;
		set mapred.output.compression.codec=org.apache.hadoop.io.compress.SnappyCodec;
		```
	
* Hive中文件格式可以在`create table`的时候指明，默认是采用textfile的格式，也可以指定为orc，parquet等

	```
	create table if not exists...
	
	sotred as orc/parquet
	```

* hive可以通过load local data将本地文件load到hdfs上，但是parquet的文件不能这样，需要先用pandas的df.to\_parquet()，才可以推上去(该方法新增于0.21.0版本)

	```
	import pandas as pd
	df = pd.DataFrame(data={'col1': [1, 2]})
	df.to_parquet('df.parquet.snappy', compression='snappy')
	pd.read_parquet('df.parquet.snappy')
	...
		col1
	0	1
	1	2
	```

* hive命令后面的选项

    * hive -f：使用-f选项可以运行指定文件中的命令，`hive -f script.q`指代我们运行脚本文件`script.q`
    * hive -S：无论是在交互式还是非交互式模式下，Hive都会把操作运行时的信息打印输出到标准错误输出，使用-S可以强制不显示这些信息
    * hive -e：使用-e选项可以在行内嵌入命令，例如`hive -e 'select * from table'`

* hive中查看函数使用方法的工具函数

    `describe function length`来查看length的用法

* hive从0.14.0版开始允许使用`INSERT INTO TABLE...VALUES`语句来插入一小撮以文字形式指明的记录，它并不是直接插入到data file，而是将数据放入暂存目录，由hive底层的同步进程周期性拷贝过去

* hive支持多表插入

    ```
    FROM source
    INSERT OVERWRITE TABLE target1
    select col1
    INSERT OVERWRITE TABLE target2
    select col2;
    ```

* hive中使用order by的时候会对数据进行全排列，同时只会使用一个reducer worker，我们可以用sort by和distribute by来进行代替，因为这个时候我们可以手动设置多个reducer worker，方法如下：

    ```
    set mapred.reduce.tasks=2;
    ```

* 和 Hadoop Streaming 类似，TRANSFORM、MAP和REDUCE子句可以在Hive中调用外部脚本或程序，如下所示:

    ```
    ADD FILE /Users/map.py;
    
    select
        transform(year, temperature, quality)
    using
        'python map.py'
    as
        year, temperature
    from
        record
    ```

    ```
    from (
        from record2
        map year, temperature, quality
        using 'python is_good_quality.py'
        as year, temperature)map_output
    reduce year, temperature
    using 'python max_temperature_reduce.py'
    as year, temperature;
    ```

    上面的`map`和`reduce`关键字都可以用`transform`来替换

* [Hive中实现Group By后，取Top K条记录](https://www.coder4.com/archives/4059)，这篇博客用的是UDF，python的话也可以使用transform调用map.py来实现相似的功能

* [Hive 数据倾斜解决方案（调优）](https://blog.csdn.net/s646575997/article/details/51510661)

* hive join 优化 -- 小表join大表

    * 小、大表join
        
        在小表和大表进行join的时候，将**小表放在前面**，效率会高，hive会将小表缓存

    * mapjoin

        使用mapjoin将小表放入内存，在map端和大表逐一匹配，从而省去reduce

        ```
        select
            /*+mapjoin(b)*/ a.a1,
            a.a2,
            b.b2
        from
            tablea a
        join
            tableb b
        on
            a.a1 = b.b1
        ```
    
        在0.7版本后，也可以用配置来自动化

        ```
        set hive.auto.convert.join=true;
        ```

* hive 获取分组 topk 

    hive 不能像 mysql 一样用局部变量和嵌套子查询来做，但是 hive 提供了 `rank`，`row_number`，`dense_rank` 三个函数
    
    [Hive分组取Top N数据](https://blog.csdn.net/WYpersist/article/details/80318305)

    ```
    select
        name,
        subject,
        score,
        rank() over (partition by name order by score desc) as rank
    from
        table
    group by
        name,
        subject,
        score
    ```

* hive 行列转换

    [Hive--行转列（Lateral View explode()）和列转行（collect_set() 去重）](http://www.voidcn.com/article/p-kvqbqneb-bbk.html)
    
    * 行转列

        ```
        select
            col1,
            col2,
            name
        from
            game.game_test
        lateral view explode(split(col3, ',')) col3 as name
        ```
    
    * 列转行

        ```
        select
            col1,
            col2,
            concat_ws(',', collect_set(col3)) as col3
        from
            game.game_test
        group by
            col1,
            col2
        ``` 

* hive 随机取样

    ```
    select
        *
    from
        table
    distribute by rand()
    sort by rand()
    ```

<h3 id="mapreduce">mapreduce</h3>

* MapReduce简介
    
    MapReduce是一个编程模型，也是一个处理和生成超大数据集的算法模型的相关实现。用户首先创建一个Map函数处理一个基于k/v pair的数据集合，输出中间的基于k/v pair的数据集合；然后再创建一个Reduce函数用来合并所有的具有相同中间key值的中间value值，MapReduce架构的程序能够在大量的普通配置的计算机上实现并行化处理，可以用于处理TB级别的数据

    ![MapReduce](./imgs/mapreduce.png)

	* 用户程序首先调用的MapReduce库将输入文件分成M个数据片段，每个数据片段的大小从16MB到512MB(可以通过可选的参数来控制每个数据片段的大小)。然后用户程序在机群中创建大量的程序副本。
	* 这些程序副本中的有一个特殊的程序 - master。副本中其它的程序都是worker程序，由master分配任务。有M个Map任务和R个Reduce任务将被分配，master将一个Map任务或Reduce任务分配给一个空闲的worker。
	* 被分配了map任务的worker程序读取相关的输入数据片段，从输入的数据片段中解析出k/v pair，然后把k/v pair传递给用户自定义的Map函数，由Map函数生成并输出的中间k/v pair，并缓存在内存中。
	* 缓存中的k/v pair 通过分区函数分成R个区域，之后周期性的写入到本地磁盘上。缓存的k/v pair在本地磁盘上的存储位置将被回传给master，由master负责把这些存储位置再传送给Reduce worker。
	* 当Reduce worker程序接收到master程序发来的数据存储位置信息后，使用RPC从Map worker所在主机的磁盘上读取这些缓存数据。当Reduce worker读取了所有的中间数据后，通过key进行排序后使得具有相同key值的数据聚合在一起。由于许多不同的key值会映射到相同的Reduce任务上，因此必须排序。如果中间数据太大无法在内存中完成排序，那么就要在外部进行排序。
	* Reduce worker程序遍历排序后的中间数据，对于每一个唯一的中间key值，Reduce worker程序将这个key值和它相关的中间 value 值的集合传递给用户自定义的 Reduce 函数。Reduce 函数的输出被追加到所属分区的输出文件
	* 当所有的 Map 和 Reduce 任务都完成之后，master 唤醒用户程序。在这个时候，在用户程序里的对 MapReduce 调用才返回。

* MapReduce的shuffle过程

    * [MapReduce shuffle过程详解](https://blog.csdn.net/u014374284/article/details/49205885) 这篇博客讲的还阔以，但是有两个地方有问题，一是key通过hash取模获得partition是在进入kvbuffer之后，二是reduce worker从map worker copy数据不是通过http，而是通过rpc

    * [Hadoop深入学习：MapReduce的Shuffle过程详解](http://flyingdutchman.iteye.com/blog/1879642)

    * 总的来说，shuffle阶段可以分为map端的partition阶段，sort阶段，以及reduce端的copy阶段和merge阶段
    
    * reduce端的merge不是一次性完成的，比如，如果有50个map输出，而合并因子是10（10为默认值，由mapreduce.task.io.sort.factor属性设置），合并将进行5趟，每趟将10个文件合并成一个文件，因此最后有5个中间文件，然后，将这5个文件作为reduce的输入，从而省去了一次磁盘的往返过程

        ![reduce-merge](./imgs/reduce-merge.jpg)

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
			* stream.map.output.field.separator：指定 map task 输出记录中 key 所使用的分隔符，默认是使用 \t
			* stream.num.map.output.key.fields：指定map task输出记录中key所占的域数目
			* map.output.key.field.separator：指定 partition 阶段对 map 输出使用哪种分隔符
			* num.key.fields.for.partition：指定对key分出来的前几部分做partition，而非整个key，需要配合 -partitioner org.apache.hadoop.mapred.lib.KeyFieldBasedPartitioner 一同使用，修改默认的 hashPartition
			* mapred.compress.map.output：map的输出是否压缩
			* mapred.map.output.compression.codec：map的输出压缩方式
			* mapred.output.compress：reduce的输出是否压缩
			* mapred.output.compression.codec：reduce的输出压缩方式
    		
* mapreduce中的combine阶段，众所周知，mapreduce中有map和reduce两个阶段，其实还有一个用户可以选择的combine阶段，对map出来的数据进行预聚合，减少传递给reduce worker的数据量，加快处理速度，例如，求出某个key的最大值，就可以在map worker中取对应的key的最大值，不用将所有的数据都丢给reduce worker，combiner函数在map 排序后的输出上运行

* MapReduce框架在记录到达reducer之前按key对记录排序，但key所对应的值并没有排序。甚至在不同的执行轮次中，这些值的排序也不固定，因为它们来自不同的map任务且这些map任务在不同轮次中完成时间各不相同。一般来说，大多数MapReduce程序会避免让reduce函数依赖于值的排序。但是，有时也需要通过特定的方法对key进行排序和分组等以实现对值的排序，例如统计每年的最高气温就很适合

    ```
    hadoop jar path.jar \
        -D stream.num.map.output.key.fields=2 \
        -D mapreduce.partition.keypartitioner.options=-k1,1 \
        -D mapreduce.job.output.key.comparator.class=org.apache.hadoop.mapred.lib.KeyFieldBasedComparator \
        -D mapreduce.partition.keycomparator.options="-k1n -k2nr" \
        -files map.py,reduce.py
        -input input/all
        -output output
        -mapper "python map.py"
        -partitioner org.apache.hadoop.mapred.lib.KeyFieldBasedPartitioner \
        -reducer "python reduce.py"
    ```

    设置`stream.num.map.output.key.fields`为2，等于说，value是空，但是在分区的时候，只用key来分区，确保了一致性，设置keycomparator，按照第一列升序，第二列降序来排序，实现既定功能，reduce的时候只需要取出每一年的第一条记录就行

* MapReduce中常见的join方法

    * reduce side join

        reduce side join是一种最简单的join方法，在map阶段同时读取两个文件file1和file2，为了区分两种来源的key/value数据对，然后对每条数据打一个tag，比如：tag=0表示来自文件File1，tag=2表示来自文件File2。即：map阶段的主要任务是对不同文件中的数据打标签。在reduce阶段，reduce函数获取key相同的来自File1和File2文件的value list， 然后对于同一个key，对File1和File2中的数据进行join（笛卡尔乘积）。即：reduce阶段进行实际的连接操作

    * map side join

        之所以存在reduce side join，是因为在map阶段不能获取所有需要的join字段，即：同一个key对应的字段可能位于不同map中。Reduce side join是非常低效的，因为shuffle阶段要进行大量的数据传输。Map side join是针对以下场景进行的优化：两个待连接表中，有一个表非常大，而另一个表非常小，以至于小表可以直接存放到内存中。这样，我们可以将小表复制多份，让每个map task内存中存在一份（比如存放到hash table中），然后只扫描大表：对于大表中的每一条记录key/value，在hash table中查找是否有相同的key的记录，如果有，则连接后输出即可

    * SemiJoin

        SemiJoin，也叫半连接，是从分布式数据库中借鉴过来的方法。它的产生动机是：对于reduce side join，跨机器的数据传输量非常大，这成了join操作的一个瓶颈，如果能够在map端过滤掉不会参加join操作的数据，则可以大大节省网络IO。
实现方法很简单：选取一个小表，假设是File1，将其参与join的key抽取出来，保存到文件File3中，File3文件一般很小，可以放到内存中。在map阶段，使用DistributedCache将File3复制到各个TaskTracker上，然后将File2中不在File3中的key对应的记录过滤掉，剩下的reduce阶段的工作与reduce side join相同

    * reduce side join + BloomFilter

        在某些情况下，SemiJoin抽取出来的小表的key集合在内存中仍然存放不下，这时候可以使用BloomFiler以节省空间。
BloomFilter最常见的作用是：判断某个元素是否在一个集合里面。它最重要的两个方法是：add() 和contains()。最大的特点是不会存在false negative，即：如果contains()返回false，则该元素一定不在集合中，但会存在一定的true negative，即：如果contains()返回true，则该元素可能在集合中。因而可将小表中的key保存到BloomFilter中，在map阶段过滤大表，可能有一些不在小表中的记录没有过滤掉（但是在小表中的记录一定不会过滤掉），这没关系，只不过增加了少量的网络IO而已

<h3 id="spark">spark</h3>

* spark-cluster的工作模式

    ![spark-cluster](./imgs/spark-cluster.jpg)

* RDD的三种生成方式

    * 从内存中的对象集合生成
    * 从本地文件或hdfs中读取出
    * 从RDD转换而来

* RDD支持两种类型的操作，转化操作(transformation)和行动操作(action)，转化操作会由一个RDD生成一个新的RDD，行动操作会对RDD计算出一个结果，转化操作和行动操作的区别在于Spark计算RDD的方式不同，虽然你可以在任何时候定义新的RDD，但Spark只会惰性计算这些RDD，它们只有第一次在一个行动操作中用到时，才会真正计算

* 如果在多个行动中重用同一个操作，可以使用`RDD.persist()`或`RDD.cache()`让Spark把这个RDD缓存下来，提高效率

* 创建RDD最简单的方式就是把程序中一个已有的集合传给SparkContext的`parallelize()`

    `lines = sc.parallelize(['a', 'b', 'c'])`

* 向Spark传递函数的时候需要小心，python会在你不经意的时候把函数所在的对象也序列化传递出去，当你传递的对象是某个对象的成员，或者包含了对某个对象中一个字段的引用时(例如self.field)，Spark就会把整个对象发送到工作节点上

	```python
	class SearchFunctions(object):
		def __init__(self, query):
			self.query = query
		def isMatch(self, s):
			return self.query in s
		def getMatchesFunctionReference(self, rdd):
			# 问题: 在"self.isMatch"中引用了整个self
			return rdd.filter(self.isMatch)
		def getMatchesMemberReference(self, rdd):
			# 问题: 在"self.query"中引用了整个self
			return rdd.filter(lambda x: self.query in x)
	```
	
	替代方法是存储为局部变量，然后传递局部变量
	
	```python
	class WordFunctions(object):
		...
		def getMatchesMemberReference(self, rdd):
			query = self.query
			return rdd.filter(lambda x: query in x)
	```

* spark中collect函数可以打印出rdd中所有的数值，但是需要保证内存装的下，collectAsMap方法和collect类似，用于pair RDD，最终返回Map类型的结果

    ```python
    rdd = sc.parallelize([(1, 2), (1, 3), (3, 3)])
    rdd.collectAsMap()

    # {1: 3, 3: 3}
    ```

    RDD中同一个key中存有多个value，后面的会覆盖前面的，最终得到的结果就是key唯一

* spark的分区操作

    spark能够对数据集在节点间的分区进行控制，在分布式程序中，通信的代价是很大的，因此控制数据分布以获得最少的网络传输可以极大地提升整体性能。分区并不是对所有的应用都有好处的--比如，如果给定RDD只需要被扫描一次，我们完全没必要预先进行分区处理。类似`join()`，`cogroup()`，`reduceByKey()`等操作，分区很有好处

    python中分区例子

    ```python
    rdd.partitionBy(100)
    ```

* Spark的共享变量类型：广播和累加器

    * 广播，可以高效的让程序向所有工作节点发送一个较大的只读值，以供一个或多个Spark操作使用

        ```python
        broadcast_var = sc.broadcast(T)

        在工作节点可以通过broadcast_var.value来获取广播变量
        ```
    
    * 累加器，可以在不同的工作节点写累加器，然后在驱动器程序中调用

        ```python
        # 在Python中累加空行

        file = sc.textFile(inputfile)
        blankLines = sc.accumulator(0)
        
        def extractCallSigns(line):
            global blankLines

            if line == '':
                blankLines += 1

            return line.split(' ')

        callSigns = file.flatMap(extractCallSigns)
        callSigns.saveAsTextFile(outputDir)
        print 'Blank lines: %d' % blankLines.value
        ```

* 基于分区进行操作

    * mapPartitions(f)，f的参数是各分区的迭代器，return一个迭代器

        ```python
        rdd = sc.parallelize([1, 2, 3, 4], 4)
        def f(units): yield sum(units)
        rdd.mapPartitions(f).collect()
        # [3, 7]
        ```

    * mapPartitionsWithIndex(f), f的参数是partition的idx和迭代器，return一个迭代器

        ```python
        rdd = sc.parallelize([1, 2, 3, 4], 4)
        def f(idx, units): yield idx
        rdd.mapPartitionsWithIndex(f).sum()
        # 6
        ```

    * foreachPartition(f)，f的参数是一个迭代器

        ```python
        rdd = sc.parallelize([1, 2, 3, 4], 4)
        def f(units):
            for u in units:
                print u
        rdd.foreachPartition(f)
        ```

* spark应用提交到集群上的方法: spark-submit --py-files \*.py --master yarn-client python\_file.py

* 在yarn-client模式或者独立模式下的spark应用，可以在驱动器ip下的4040端口查看spark任务的状态，DAG等信息，很有用

* spark性能优化

    * 利用分区提高并行度
    * 当Spark需要通过网络传输数据，或是将数据溢写到磁盘上，Spark需要把数据序列化为二进制文件，可以采用Kryo的第三方序列化库，能够获得更短的序列化时间和更高的压缩比
    * 使用persist或者cache方法缓存分区，避免重复计算
    * 设置executor节点的cores和memory

* spark sql可以直接通过hivesql访问hive表格的数据，需要把hive\_site.xml放到spark的conf文件夹中，也可以直接访问hdfs的parquet，orc文件，然后注册临时表

    ```python
    from pyspark.sql import HiveContext

    hiveCtx = HiveContext(sc)
    rows = hiveCtx.sql(hive_sql)

    data = hiveCtx.read.parquet(path of parquet in hdfs)
    data.registerTempTable('table') # 作为临时表
    hiveCtx.sql("select * from table")
    ```

* spark sql允许用户自定义函数(UDF)，可以将自定义函数类似hive中的count函数一样用于sql中

    ```python
    hiveCtx.registerFunction('strLen', lambda x: len(x), IntegerType())
    df = hiveCtx.sql("select strLen('name') from table")
    ```

    如果要返回一个list，可以使用types里的StructField和StructType来自定义

    ```python
    schema = StructType([StructType('name', StringType(), True), StructType('age', IntegerType(), True)])
    udf_func = hiveCtx.registerFunction('udf_func', lambda x: (x, 1), schema)
    ```

* spark的rdd和spark.sql的df的横行merge和纵向merge方法

    * rdd
        * 横向 map / mapPartitions
        * 纵向 union
    * df
        * 横向 crossJoin(select alias) / join
        * 纵向 union

* 和Spark基于RDD的概念很相似，Spark Streaming使用离散化流作为抽象表示，叫做DStream，Spark Streaming 会把每个interval收到的数据放入DStream

* Dstream的转化操作可以分为有状态和无状态两种，有状态的可以创建window，处理多个interval的数据

* [Spark Streaming 简介](http://bigdataer.net/?p=244)
    
* [spark 将dataframe数据写入hive表](https://blog.csdn.net/zgc625238677/article/details/53928320)，基本思路是先将df注册为本地table，再从本地table insert到hive表中

<h3 id="hbase">hbase</h3>

* hbase是一个在HDFS上开发的面向列的分布式数据库，如果需要实时地随机访问超大规模数据集，就可以使用HBase这一Hadoop应用

* hbase也是一个master-slave的存储模型，它用一个master节点协调管理一个或多个regionserver从属机。hbase主控机(master)负责启动一个全新的安装，把区域分配给注册的regionserver，恢复regionserver的故障，master的负载很轻。regionsever负责零个或多个的区域管理以及响应客户端的读写请求。regionserver还负责区域的划分并通知HBase master有了新的子域

    ![hbase-master-slave](./imgs/hbase-master-slave.jpg)

* [HBase深入浅出](https://www.ibm.com/developerworks/cn/analytics/library/ba-cn-bigdata-hbase/index.html)

<h3 id="zk">zookeeper</h3>

* [ZooKeeper简介](https://juejin.im/post/5b970f1c5188255c865e00e7?utm_source=gold_browser_extension)

* ZooKeeper维护着一个树形层次结构，树中的节点被称为znode。znode可以用于存储数据，并且有一个与之相关联的ACL(AccessControlLists)。ZooKeeper被设计用来实现协调服务(这类服务通常使用小数据文件)，而不是用于大容量数据存储，因此一个znode能存储的数据被限制在1MB以内

* ZooKeeper可以用来实现分布式锁，分布式锁能够在一组进程之间提供互斥机制，使得在任何时刻只有一个进程可以持有锁。分布式锁可以用于在大型分布式系统中实现领导者选举，在任何时间点，持有锁的那个进程就是系统的领导者。为了使用ZooKeeper来实现分布式锁服务，我们使用顺序znode来为那些竞争锁的进程强制排序。思路很简单：首先指定一个作为锁的znode，通常用它来描述被锁定的实体，称为\/leader，然后希望获得锁的客户端创建一些短暂顺序znode，作为锁znode的子节点。在任何时间点，顺序号最小的客户端将持有锁。例如，有两个客户端差不多同时创建znode，分别为/leader/lock-1和/leader/lock-2，那么创建/leader/lock-1的客户端将会持有锁，因为znode顺序号最小，只有前一个znode释放了锁，后一个才能获得锁

* 为什么最好使用奇数台服务器构成ZooKeeper集群

    我们知道在ZooKeeper中Leader选举算法采用了Zab(ZooKeeper Atomic Broadcast 原子广播)协议。Zab核心思想是当多数Server写成功，则任务数据写成功

    * 如果有3个Server，则最多允许1个Server挂掉
    * 如果有4个Server，则同样最多允许1个Server挂掉
    
    既然3个或者4个Server，同样最多允许1个Server挂掉，那么它们的可靠性是一样的，所以选择奇数个ZooKeeper Server即可

* Zab 协议核心：所有的事务请求必须一个全局唯一的服务器（Leader）来协调处理，集群其余的服务器称为 follower 服务器。Leader 服务器负责将一个客户端请求转化为事务提议（Proposal），并将该 proposal 分发给集群所有的 follower 服务器。之后 Leader 服务器需要等待所有的 follower 服务器的反馈，一旦超过了半数的 follower 服务器进行了正确反馈后，那么 Leader 服务器就会再次向所有的 follower 服务器分发 commit 消息，要求其将前一个 proposal 进行提交。因为半数 follower 服务器 ack 之后，写操作就 commit 了，因此 zookeeper 不能保持随时一致性，只能保证最终一致性

	![zab 协议](./imgs/zab.jpg)

* ZooKeeper 领导人选举

	领导人选举分为第一次投票和变更投票两个阶段
	
	第一次投票。无论哪种导致进行Leader选举，集群的所有机器都处于试图选举出一个Leader的状态，即LOOKING状态，LOOKING机器会向所有其他机器发送消息，该消息称为投票。投票中包含了SID（服务器的唯一标识）和ZXID（事务ID），(SID, ZXID)形式来标识一次投票信息。假定Zookeeper由5台机器组成，SID分别为1、2、3、4、5，ZXID分别为9、9、9、8、8，并且此时SID为2的机器是Leader机器，某一时刻，1、2所在机器出现故障，因此集群开始进行Leader选举。在第一次投票时，每台机器都会将自己作为投票对象，于是SID为3、4、5的机器投票情况分别为(3, 9)，(4, 8)， (5, 8)。
	
	变更投票。每台机器发出投票后，也会收到其他机器的投票，每台机器会根据一定规则来处理收到的其他机器的投票，并以此来决定是否需要变更自己的投票，这个规则也是整个Leader选举算法的核心所在，其中术语描述如下
	
	vote\_sid：接收到的投票中所推举Leader服务器的SID。
	
	vote\_zxid：接收到的投票中所推举Leader服务器的ZXID。
	
	self\_sid：当前服务器自己的SID。
	
	self\_zxid：当前服务器自己的ZXID。

    事务 id 是一个64位的整数，前32位代表 leader 选择的轮次，每重新选举一次 leader，自增1，同时将后32位清零，后32位代表本轮内的事务顺序，一条事务到来的时候，自增1
	
	每次对收到的投票的处理，都是对(vote\_sid, vote\_zxid)和(self\_sid, self\_zxid)对比的过程。

	规则一：如果vote\_zxid大于self\_zxid，就认可当前收到的投票，并再次将该投票发送出去。

	规则二：如果vote\_zxid小于self\_zxid，那么坚持自己的投票，不做任何变更。

	规则三：如果vote\_zxid等于self\_zxid，那么就对比两者的SID，如果vote\_sid大于self\_sid，那么就认可当前收到的投票，并再次将该投票发送出去。

	规则四：如果vote\_zxid等于self\_zxid，并且vote\_sid小于self\_sid，那么坚持自己的投票，不做任何变更。

	结合上面规则，给出下面的集群变更过程
	
	![zk_leader_election](./imgs/zk_leader_election.jpg)
	
	由上面规则可知，通常那台服务器上的数据越新（ZXID会越大），其成为Leader的可能性越大，也就越能够保证数据的恢复。如果ZXID相同，则SID越大机会越大
	
* Zab 协议恢复模式的保证

	* 我们绝不能遗忘已经被deliver的消息，若一条消息在一台机器上被deliver，那么该消息必须将在每台机器上deliver
	* 我们必须丢弃已经被skip的消息，比如 leader 发出了一个提议，但是还没有 commit 就挂了，这样恢复的时候，会重新 commit，但是其他 server 是没有 commit 这条指令的，这样就会造成不一致，zk 的 zxid 的前32位可以避免这种情况发生，因为重新选举之后，前32位自增加一，这样，当收到比自己前32位小的时候 zxid 的时候，直接丢弃即可

* [zk 系列文章](https://www.cnblogs.com/sunddenly/p/4138580.html)

<h3 id="kafka">kafka</h3>

* Kafka专为分布式高吞吐量系统而设计，是一个分布式发布 - 订阅消息系统和一个强大的队列，可以处理大量的数据，并使您能够将消息从一个端点传递到另一个端点。 Kafka适合离线和在线消息消费。 Kafka消息保留在磁盘上，并在群集内复制以防止数据丢失。 Kafka构建在ZooKeeper同步服务之上。 它与Apache Storm和Spark非常好地集成，用于实时流式数据分析

* kafka的优点

    * 可靠性 - Kafka是分布式，分区，复制和容错的
    * 可扩展性 - Kafka消息传递系统轻松缩放，无需停机
    * 耐用性 - Kafka使用"分布式提交日志"，这意味着消息会尽可能快地保留在磁盘上，因此它是持久的
    * 性能 - Kafka对于发布和订阅消息都有高吞吐量。即使存储了许多TB的消息，它也保持稳定的性能。Kafka非常快，并保证零停机和零数据丢失

* kafka架构图

    ![kafka架构](./imgs/kafka.jpg)

* kafka中一个topic可以由一个consumer\_group访问，group中的每个consumer负责一部分partition，如果consumer和kafka的连接经常中断，那么会频繁触发kafka的rebalance，这样就会在consumer端积压数据，导致数据流不下去

* [Kafka 工作原理](https://www.jianshu.com/p/6cbe28a44543)

* [Kafka 复制机制](https://colobu.com/2017/11/02/kafka-replication/)

* [Kafka vs Nsq](https://zhuanlan.zhihu.com/p/46421050)

* Kafka 需要 Zookeeper 做两件事情

    * 将 broker 的一些元信息存储进 Zookeeper
    * 使用 Zookeeper 实现领导人选举

* 每个 Partition 物理上对应一个文件夹，里面存放很多 Segment，这样删除数据的时候直接删除最早的 Segment 就行，提高了效率

* Kafka 在 Partition 之间的无序的，在 Partition 内部是有序的

* Kafka 的 Producer 默认是异步发送数据的，其实也是 GC(group commit) 的概念，可以显示调用 flush 来立即发送

* Kafka 的 Producer 具有 Retry 机制，发送也是异步的，有可能出现 1 在 Retry 的时候，2 已经发送成功了，这样即使发送到一个 Partition，顺序也乱了，如果非常在意这种情况的话，可以将 max.in.flight.requests.per.connection 设置为 1，同样的，这样比较影响性能

* 在 Producer 发送消息的时候，如果不显示指定 key，消息路由 Partitioner 会采用轮询的方式，将消息负载均衡的打到每个 Partition 中，如果有非常在意消费顺序的消息需要发送的时候，就可以显示指定 key，这样 Producer 就可以将该部分数据写入一个 Partition 中，最简单的实现就是对 key 做 hash % Partition 数目

* Kafka Rebalance 的两种方法

    * 自治式 Rebalance：每个 Consumer 决定自己是否需要 Rebalance

        * Consumer 启动时将其 ID 注册到 Consumer Group 下，在 ZK 上的路径为 /consumers/[consumer group]/ids/[consumer id]
        * 在 /consumers/[consumer group]/ids 上注册 Watch
        * 在 /brokers/ids 上注册Watch
        * 强制自己在 Consumer Group 内启动 Rebalance 流程

        特点：
        
        * 任何 Broker 或者 Consumer 的增减都会触发所有的 Consumer 的 Rebalance
        * 每个 Consumer 分别单独通过 ZK 判断哪些 Broker 和 Consumer 宕机了，那么不同 Consumer 在同一时刻从 ZK 上看到的 View 可能就不同，这是由 ZK 的特性决定的，这就会造成不正确的 Rebalance 尝试
        * 所有的 Consumer 都不知道其他的 Consumer 是否 Rebalance 是否成功，这可能会导致 Kafka 工作在一个不正确的状态
    
    * 集中式 Rebalance：基于 Coordinator(协调者) 的 Rebalance
        
        * 从 ZK 读取所有的 Topic 以及是否有新的 Topic 被创建
        * 监听 Topic 的变化以及 Partition 的变化
        * 接收 Consumer 的注册，为每一个 Group 选择一个 Leader
        * Leader 通过 SyncGroup 将 Rebalance 分配方案发给 Coordinator
        * 其他 Member 通过 SyncGroup 从 Coordinator 获取各自的分配结果

* Kafka 采用 Partition 级别的复制来实现 HA，ISR 是 Kafka 中经典的高可用机制

    * Leader 会维护一个与其基本保持同步的 Replica 列表，该列表称为 ISR（in-sync Replica）
    * 如果一个 Follower 比 Leader 落后太多，或者超过一定时间未发起数据复制请求，则 Leader 将其从 ISR 中移除
    * 当 ISR 中所有 Replica 都向 Leader 发送 ACK 时，Leader 即 Commit

* 当 ISR 中的机器全部宕机，Kafka有两种处理方法

    * 等待 ISR 中任一 Replica 恢复，并选它为 Leader
        
        * 等待时间较长，降低可用性
        * 或 ISR 中的所有 Replica 都无法恢复或数据丢失，则该 Partition 将用不可用

    * 选择第一个恢复的 Replica 为新的 Leader，无论它是否在 ISR 中

        * 并未包含所有已被之前 Leader Commit 过的消息，因此会造成数据丢失
        * 可用性较高

    CAP 无法同时满足，默认采用第二种，保证可用性
            
* kafka-zookeeper

    ![kafka-zookeeper](./imgs/zk-kafka.png)

* 个人觉得讲的不错的博客

    [Kafak 设计解析](http://www.jasongj.com/2015/03/10/KafkaColumn1/)

* Producer 在发布消息到某个 Partition 时，先通过 ZK 找到该 Partition 的 Leader，然后无论该 Topic 的 Partition Factor 为多少，Producer 只将消息发给该 Partition 的 Leader

<h3 id="nsq">nsq</h3>

* nsq的三大核心组件

	* nsqlookupd是守护进程负责管理拓扑信息。客户端通过查询nsqlookupd来发现指定话题(topic)的生产者，并且 nsqd 节点广播话题（topic）和通道（channel）信息。简单的说nsqlookupd就是中心管理服务，它使用tcp(默认端口4160)管理nsqd服务，使用http(默认端口4161)管理nsqadmin服务。同时为客户端提供查询功能

		总的来说，nsqlookupd具有一下功能或特性

        * 唯一性，在一个Nsq服务中只有一个nsqlookupd服务。当然也可以在集群中部署多个nsqlookupd，但它们之间是没有关联的
        * 去中心化，即使nsqlookupd崩溃，也会不影响正在运行的nsqd服务
        * 充当nsqd和naqadmin信息交互的中间件
        * 提供一个http查询服务，给客户端定时更新nsqd的地址目录

	* nsqadmin是一套 WEB UI，用来汇集集群的实时统计，并执行不同的管理任务
		
		* 提供一个对topic和channel统一管理的操作界面以及各种实时监控数据的展示，界面设计的很简洁，操作也很简单
		* 展示所有message的数量
		* 能够在后台创建topic和channel
		* nsqadmin的所有功能都必须依赖于nsqlookupd，nsqadmin只是向nsqlookupd传递用户操作并展示来自nsqlookupd的数据

	* nsqd是一个守护进程，负责接收、排队、投递消息给客户端，真正干活的就是这个服务，它主要负责message的收发，队列的维护。nsqd会默认监听一个tcp端口(4150)和一个http端口(4151)以及一个可选的https端口

		总的来说，nsqd 具有以下功能或特性
		
		* 对订阅了同一个topic，同一个channel的消费者使用负载均衡策略（不是轮询）
		* 只要channel存在，即使没有该channel的消费者，也会将生产者的message缓存到队列中（注意消息的过期处理）
		* 保证队列中的message至少会被消费一次，即使nsqd退出，也会将队列中的消息暂存磁盘上(结束进程等意外情况除外)
		* 限定内存占用，能够配置nsqd中每个channel队列在内存中缓存的message数量，一旦超出，message将被缓存到磁盘中
		* topic，channel一旦建立，将会一直存在，要及时在管理台或者用代码清除无效的topic和channel，避免资源的浪费

    * [nsq 的源码分析](https://github.com/mickey0524/nsq-analysis)
			
