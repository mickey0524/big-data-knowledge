# big-data-knowledge
📖大数据相关知识集锦

* HDFS简介

	HDFS是Hadoop Distributed File System的简写
	
	* HDFS具有高容错性和高吞吐性的特点
	* HDFS目前是 append only，暂时不支持随机 write 的操作
	* HDFS适合用于存储以及批量操作大规模的数据集(PB级别)
	* 不适合实时访问，具有高延迟性，例如新建了一张hive表，需要过一会才能看到
	* [Hadoop HDFS 教程（一）介绍](https://www.jianshu.com/p/8969eb90a59d)
	* [Hadoop HDFS（二）结构解析和名词解释](https://www.jianshu.com/p/86a70ac1f5f9)

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

* 数据仓库(DW/Data Warehouse)分层原则(每家公司都有自己的规范)

	* dim：维度层，一般用于存储属性信息，多用于联表查询
	* dwd/ods(data warehouse detail)：事实明细层，存储事实表的明细粒度数据，比较底层的数据，源数据清洗得来，例如埋点后捞出来的数据
	* dwa(data warehouse aggregation)：事实聚合层，存储事实表聚合粒度数据，按需求联合查询得到的聚合表
	* app(application)：应用层，存储直接供给应用的数据

	![dw架构图](/imgs/dw.png)

* hive的join操作不支持like模糊匹配，如果非要使用like，需要使用笛卡尔积，这个效率太低，不如放到内存中匹配，下面是笛卡尔积的写法

    ```sql
    SELECT table1.brand, SUM(table2.sold) 
    FROM table1, table2
    WHERE table2.product LIKE concat('%', table1.brand, '%') 
    GROUP BY table1.brand;
    ```

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