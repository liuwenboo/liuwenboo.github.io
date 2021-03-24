[TOC]

# MapReduce概述

Hadoop：解决海量数据的存储和海量数据的计算

HDFS：负责存储

MapReduce：负责计算

MapReduce是一个**分布式运算程序的编程框架**，是用户开发”基于Hadoop的数据分析应用“的核心框架。
核心功能是将**用户编写的业务逻辑代码**和**自带默认组件**整合成一个完整的**分布式运算程序**，并发运行在一个Hadoop集群上。

**优点**：

1. 易于编程，简单的实现一些接口，就可以完成一个分布式程序
2. 良好的扩展性，简单的增加机器即可
3. 高容错性，其中一台机器挂了，把上面的计算任务转移到另外一个节点上运行即可，而且这个过程不需要人工参与，完全是由Hadoop内部完成的
4. 适合**PB级**以上海量数据的**离线**处理

**缺点**：

1. 不擅长实时计算
2. 不擅长流式计算，流式计算的输入数据是动态的，而MapReduce的输入数据集是静态的
3. 不擅长DAG（有向图）计算

**一个完整的MapReduce程序在分布式运行时有三类实例进程：**

1. MrAppMaster：负责整个程序的过程调度及状态协调
2. MapTask：负责Map阶段的整个数据处理流程
3. ReduceTask：负责Reduce阶段的整个数据处理流程

**常用的数据类型对应的Hadoop数据序列化类型：**

是Hadoop自身封装的序列化类型，序列化就是将内存中的对象写入到磁盘持久化

| Java类型   | HadoopWritable类型 |
| :--------- | :----------------- |
| boolean    | BooleanWritable    |
| byte       | ByteWritable       |
| int        | IntWritable        |
| float      | FloatWritable      |
| long       | LongWritable       |
| double     | DoubleWritable     |
| **String** | **Text**           |
| map        | MapWritable        |
| array      | ArrayWritable      |

## MapReduce编程规范

用户编写的程序分成三个部分：Mapper、Reducer和Driver。

1. Mapper阶段

   1. 用户自定义的Mapper要继承自己的父类
   2. Mapper的输入数据是KV对的形式（KV的类型可自定义）
   3. Mapper中的业务逻辑写在map()方法中
   4. Mapper的输出数据是KV对的形式（KV的类型可自定义）
   5. map()方法（MapTask进程）对每一个<K,V>调用一次

2. Recuder阶段

   1. 用户自定义的Reducer要继承自己的父类
   2. Reducer的输入数据类型对应Mapper的输出数据类型，也是KV对
   3. Reducer的业务逻辑写在reducer()方法中
   4. ReduceTask进程对每一组相同K的<K,V>组调用一次reduce()方法

3. Driver阶段

   相当于YARN集群的客户端，用于提交我们整个程序到YARN集群，提交的是封装了MapReduce程序相关运行参数的job对象
   
   1. 获取配置信息，获取job对象实例
   2. 制定本程序的jar包所在的本地路径
   3. 关联Mapper/Reducer业务类
   4. 指定Mapper输出数据的jv类型
   5. 指定最终输出的数据的kv类型
   6. 指定job的输入原始文件所在目录
   7. 指定job的输出结果所在目录
   8. 提交作业

## MapReduce测试案例

代码在idea中HDFS-0209项目下src/main/java目录下com.mr.wordcount包中

注：main方法右键 -> 编辑运行配置中，修改程序参数（输入参数 args[]），然后再运行

## 自定义程序在集群上运行

把程序打包，用maven打jar包，需要添加的打包插件依赖

编辑pom.xml：

注意：<mainClass>cn.mymaven.test.TestMain</mainClass>部分需要替换为自己工程主类

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>2.3.2</version>
            <configuration>
                <source>1.8</source>
                <target>1.8</target>
            </configuration>
        </plugin>
        <plugin>
            <artifactId>maven-assembly-plugin</artifactId>
            <configuration>
                <descriptorRefs>
                    <descriptorRef>jar-with-dependencies</descriptorRef>
                </descriptorRefs>
                <archive>
                    <manifest>
                        <mainClass>cn.mymaven.test.TestMain</mainClass>
                    </manifest>
                </archive>
            </configuration>
            <executions>
                <execution>
                    <id>make-assembly</id>
                    <phase>package</phase>
                    <goals>
                        <goal>single</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

然后将maven点击reload或update

在右侧maven工程部分，点击Lifecycle -> maven install，target目录下就会出现打包好的jar包。

将打包好的jar包传到集群，然后运行

```bash
hadoop jar wc.jar com.mr.wordcount.WordcountDriver /user/root/learn_mr/input /user/root/learn_mr/output
```

# Hadoop序列化

序列化：就是把内存中的对象（数组、集合等），转换成字节序列以便于存储到磁盘（持久化）和网络传输。

为什么不用Java的序列化：
Java的序列化是一个重量级序列化框架（Serializable），一个对象被序列化后，会附带很多额外的信息，不便于在网络中高效传输

Hadoop序列化特点：

1. 紧凑：高效使用存储空间
2. 快速：读写数据的额外开销小
3. 可扩展：随着通信协议的升级而可升级
4. 互操作：支持多语言的交互

## 自定义bean对象实现序列化接口

具体实现步骤：

1. 必须实现Writable接口

2. 反序列化时，需要反射调用空参构造函数，所以必须有空参构造

   ```java
   public FlowBean(){
       super();
   }
   ```

3. 重写序列化方法

   ```java
   @Override
   public void write(DataOutput out) throws IOException {
       out.writeLong(upFlow);
       out.writeLong(downFlow);
       out.writeLong(sumFlow);
   }
   ```

4. 重写反序列化方法

   ```java
   @Override
   public void readFields(DataItput in) throws IOException {
       upFlow = in.readLong();
       downFlow = in.readLong();
       sumFlow = in.readLong();
   }
   ```

5. 注意反序列化的顺序和序列化的顺序完全一致

6. 要想把结果显示在文件中，需要重写toString()，可用“\t”分开，方便后续用。

7. 如果需要将自定义的bean放在key中传输，则还需要实现Comparable接口，因为MapReduce框架中的Shuffle过程要求对key必须能排序

   ```java
   @Override
   public int compareTo(FlowBean o) {
       //倒序排列，从大到小
       return this.sumFlow > o.getSumFlow() ? -1 : 1;
   }
   ```

## 序列化案例实操

1. 需求统计每一个手机号耗费的总上行流量、下行流量、总流量

   输入数据格式：

   id  手机号码  网络ip  上行流量  下行流量  网络状态码

   期望输出数据格式

   手机号码  上行流量  下行流量  总流量

2. Map阶段

   1. 读取一行数据，切分字段
   2. 抽取手机号、上行流量、下行流量
   3. 一手机号为key，bean对象为value输出，即context.write(手机号, bean)；
   4. **bean对象要想能够传输，必须实现序列化接口**（要先写这一步，因为到Map那一步，定义输出类型的时候需要）

3. Reduce阶段

   1. 累加上行流量和下行流量得到总流量
   2. 写出

**代码**在idea中HDFS-0209项目下src/main/java目录下com.mr.flowsum包中

# MapReduce框架原理（重点）

## InputFormat数据输入

**切片与MapTask并行度决定机制**

MapTask的并行度决定Map阶段的任务处理并发度，进而影响到整个Job的处理速度。

数据块：Block是HDFS屋里上把数据分成一块一块。

数据切片：数据切片只是在逻辑上对输入进行分片，并不会在磁盘上将其切分成片进行存储。

每个数据切片交给一个MapTask去做任务，所以如果每个数据块在不同的DataNode上，且数据切片和Block大小不等时，会浪费大量的网络IO（不同节点上的数据要合成一个数据切片完成MapTask任务）

决定机制：

1. 一个Job的Map阶段并行度由客户端在提交Job时的切片数决定
2. 每一个Split切片分配一个MapTask并行实例处理
3. 默认情况下，切片大小 = BlockSize
4. 切片时不考虑数据集整体，而是逐个针对每一个文件单独切片

**Job提交流程源码详解**

```java
waitForCompletion();
submit();
// 1 建立连接
	conncect();
	// 1 创建调教Job的代理
	// 2 判断是本地yarn还是远程
// 2 提交job
	// 1 创建给集群调教数据的Stag路径
	// 2 获取jobid，并创建Job路径
	// 3 拷贝jar包到集群
	// 4 计算切片，生成切片规划文件
	// 5 向Stag路径写XML配置文件
	// 6 提交Job，返回提交状态
```

**FileInputFormat切片源码详解**

1. 程序先找到你数据存储的目录

2. 开始遍历处理（规划切片）目录下的每一个文件

3. 遍历第一个文件ss.txt

   1. 获取文件大小fs。sizeOf(test.txt)

   2. 计算切片大小

      computeSliteSizse(Math.max(minSize, Math.min(maxSize, blocksize))) = blocksize = 128M（YARN上128M，本地模式的是32M）

   3. 默认情况下，切片大小 = blocksize

   4. 开始切，每次切片时，都要判断切完上下的部分是否大于块的1.1倍，不大于1.1倍就划分一块切片

   5. 将切片信息写到一个切片规划文件中

   6. 整个切片的核心过程在getSplit()方法中完成

   7. InputSplit只记录了切片的元数据信息，比如起始位置、长度以及所在的节点列表等

4. 提交切片规划文件到YARN上，YARN上的MrAppMaster就可以根据切片规划文件计算开启MapTask个数

**FileInputFormat切片机制**

1. 简单地按照文件的内容长度进行切片
2. 切片大小，默认等于Block大小（YARN上128M，本地模式的是32M）
3. 切片时不考虑数据集整体，而是逐个针对每一个文件单独切片

**获取切片信息的API**

```java
String name = inputSplit.getPath().getName();	//获取切片的文件名称
FileSplit inputSplit = (FileSplit) context.getInputSplit();	//根据文件类型获取切片信息
```

**CombineTextInputFormat切片机制**

框架默认的TextInputFormat切片机制是对任务按文件规划切片，不管文件多小，都会是一个单独的切片，都会交给一个MapTask，这样如果有大量小文件，就会产生大量MapTask，处理效率极其低下。

CombineTextInputFormat用于小文件过多的场景，将多个小文件从逻辑上规划到一个切片中，这样多个小文件就可以交个一个MapTask处理

虚拟存储切片最大值设置

```java
CombineTextInputFormat.setMaxInputSplitSize(job, 4194304);	//4M
```

**切片过程**

1. 虚拟存储过程
   1. 如果一个文件小于setMaxInputSplitSize，则划分一块
   2. 如果一个文件暂未分块部分大于setMaxInputSplitSize且小于2*setMaxInputSplitSize，则均匀分在两块中
2. 切片
   1. 判断虚拟存储的文件大小是否大于setMaxInputSplitSize值，大于等于则单独形成一个切片
   2. 如果不大于则跟下一个虚拟存储文件进行合并，共同形成一个切片，如果合并后的再加上下一个文件还是小于setMaxInputSplitSize值，则再加上下一个文件，共同形成一个切片

**实现过程**

驱动类中添加代码如下：

```java
// 如果不设置 InputFormat，它默认用的是 TextInputFormat.class
 job.setInputFormatClass(CombineTextInputFormat.class); 
//虚拟存储切片最大值设置 4m
 CombineTextInputFormat.setMaxInputSplitSize(job, 4*1024*1024); 
```

**FileInputFormat实现类**

思考：在运行MapReduce程序时，输入的文件格式包括：基于行的日志文件、二进制格式文件、数据库表等。那么，针对不同的数据类型，MapReduce是如何读取这些数据的呢？

FileInputFormat常见的接口实现类包括：TextInputFormat（处理文本）、KeyValueTextInputFormat（处理KV对）、NLineInputFormat（按行处理）、CombineTextInputFormat（处理小文件）和自定义InputFormat等。

1. **TextInputFormat**（按文件的大小切片，如果都是小文件则每个文件一块）

   是默认的FileInputFormat实现类。按行读取每条记录。

   键key是存储该行在整个文件中的**起始字节偏移量**， LongWritable类型。
   值value是这行的内容，不包括任何行终止符（换行符和回车符），Text类型。

2. **KeyValueTextInputFormat**（按文件的大小切片，如果都是小文件则每个文件一块）

   每一行均为一条记录，被分隔符分割为key，value。可以通过在驱动类中设置

   ```java
   conf.set(KeyValueLineRecordReader.KEY_VALUE_SEPERATOR,"\t");
   ```

   来设定分隔符。默认分隔符为tab ( \t ) 。

   还要设置输入格式：

   ```java
   job.setInputFormatClass(KeyValueTextInputFormat.class); 
   ```

   **案例代码：**输出文件中每一行第一个单词的出现次数，idea中HDFS-0209项目下src/main/java目录下com.mr.kv包中。

   **注**：代码中一定记得加上设置输入格式的部分，不然默认的和需要的会发生冲突（前面提过），会报找不到Jar包的错误。

3. **NLineInputFormat**（按多少行切片）

   代表每个map进程处理的InputSplit不再按Block块去划分，而是按NLineInputFormat指定的**行数N**来划分。即输入文件的总行数/N=切片数，如果不整除，切片数=商+1。

   Key是偏移量，LongWritable
   Value是内容，Text

   在驱动类中设置：

   ```java
   NLineInputFormat.setNumLinesPerSplit(job, 3);	// N=3
   job.setInputFormatClass(NLineInputFormat.class);
   ```

4. **CombineTextInputFormat**（按设置的最大值切片）

5. **自定义InputFormat**（按文件的大小切片，如果都是小文件则每个文件一块）

   步骤如下：

   1. 自定义一个类继承FileInputFormat

      1. 重写isSplitable()方法，返回false不可切割
      2. 重写createRecordReader()，创建自定义的RecordReader对象，并初始化

   2. 改写RecordReader，实现一次读取一个完整文件封装为KV

      1. 采用IO流一次读取一个文件输出到value中，因为设置了不可切片，最终把所有文件都封装到了value中
      2. 获取文件路径信息+名称，并设置key

   3. 在输出时使用SequenceFileOutFormat输出合并文件（这个是文件类型，其中文件名称为Key，文件内容为Value）
      设置Driver

      ```java
      // 1 设置输入的inputFormat
      job.setInputFormatClass(WholeFileInputformat.class);
      // 2 设置输出端outputFormat
      job.setOutputFormatClass(SequenceFileOutputFormat.class);
      ```

      **案例代码：**将几个小文件合并成一个SequenceFile，idea中HDFS-0209项目下src/main/java目录下com.mr.inputformat包中。

## MapReduce工作流程

分片决定MapTask个数，分区决定ReduceTask个数

## Shuffle机制 

Map方法之后，Reduce方法之前的数据处理过程称之为Shuffle（洗白），包括分区，排序，合并，压缩等流程。

**Partitioner分区**

默认分区是根据key的hashCode对ReduceTasks个数取模得到的，用户没法控制哪个key存储到哪个分区

```java
job.setNumReduceTasks(int tasks);	//设置分区个数，默认为1
```










# Hadoop数据压缩





# Yarn资源调度器（ 面试重点）





# Hadoop企业优化





# MapReduce扩展案例





# 常见错误