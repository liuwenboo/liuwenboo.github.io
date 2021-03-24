[toc]

# HDFS客户端操作

## HDFS客户端环境准备

**文件准备**

首先将与集群上安装的hadoop相同版本解压至windows目录（英文目录）

解压时如果提示符号链接错误是因为权限问题，使用**管理员权限**cmd进入压缩文件目录，使用命令``start winrar x -y hadoop-2.10.1.tar.gz`

下载hadoop.dll，winutils.exe文件至/bin目录（这是windows依赖）

https://github.com/cdarlint/winutils/blob/master/hadoop-2.9.2/bin/

因为 Hadoop 主要基于 Linux 编写，这个 winutil.exe 主要用于模拟 Linux 下的目录环境，因此 Hadoop 放在 Windows 下运行的时候，需要这个辅助程序才能运行

在windows系统环境变量中添加hadoop路径

```
变量名  HADOOP_HOME
变量值  D:\hadoop\hadoop-2.10.1
```

在PATH中添加

```
%HADOOP_HOME%\bin
```

修改\etc\hadoop\hadoop-env.cmd中的JAVA_HOME变量（路径中含有Program Files，但是路径不能有空格，使用`set JAVA_HOME=C:\PROGRA~1\Java\jdk1.8.0_271`）

配置完hadoop后在cmd中使用命令`hadoop version`查看是否配置好（需要管理员权限运行）。

**配置客户端（idea）**

创建maven工程：File->New->Project->Maven，选择next，自己选择路径，GroupID，ArtifactID，点击Finish完成。

点击pom.xml，引入依赖，即添加如下代码：

```xml
<dependencies>
    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <version>RELEASE</version>
    </dependency>
    <dependency>
        <groupId>org.apache.logging.log4j</groupId>
        <artifactId>log4j-core</artifactId>
        <version>2.8.2</version>
    </dependency>
    <dependency>
        <groupId>org.apache.hadoop</groupId>
        <artifactId>hadoop-common</artifactId>
        <version>2.10.1</version>
    </dependency>
    <dependency>
        <groupId>org.apache.hadoop</groupId>
        <artifactId>hadoop-client</artifactId>
        <version>2.10.1</version>
    </dependency>
    <dependency>
        <groupId>org.apache.hadoop</groupId>
        <artifactId>hadoop-hdfs</artifactId>
        <version>2.10.1</version>
    </dependency>
    <dependency>
        <groupId>jdk.tools</groupId>
        <artifactId>jdk.tools</artifactId>
        <version>1.8</version>
        <scope>system</scope>
        <systemPath>${java.home}/../lib/tools.jar</systemPath>
    </dependency>
</dependencies>
```

然后maven会自动添加依赖，或点击右侧Maven面板，点击Reload All Maven Projects，需要等一段时间。（注意修改添加代码中的hadoop版本和jdk版本）

此时已经可以运行hadoop项目，但是会报log4j:WARN......

在建立的maven工程中/src/main/resources目录下建立文件 log4j.properties，添加如下代码即可：

```
log4j.rootLogger=INFO, stdout
log4j.appender.stdout=org.apache.log4j.ConsoleAppender
log4j.appender.stdout.layout=org.apache.log4j.PatternLayout
log4j.appender.stdout.layout.ConversionPattern=%d %p [%c] - %m%n
log4j.appender.logfile=org.apache.log4j.FileAppender
log4j.appender.logfile.File=target/spring.log
log4j.appender.logfile.layout=org.apache.log4j.PatternLayout
log4j.appender.logfile.layout.ConversionPattern=%d %p [%c] - %m%n
```

**编写测试文件**

在/src/main/java目录中新建包com.hdfs.learn，在包中新建java类HDFSClient

```java
package com.hdfs.learn;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;

import java.io.IOException;

public class HDFSClient {

    public static void main(String[] args) throws IOException {

        Configuration conf = new Configuration();
        conf.set("fs.defaultFS","hdfs://hadoop001:9000");
        //1 获取hdfs客户端路径
        FileSystem fs = FileSystem.get(conf);

        //2 在hdfs上创建路径
        fs.mkdirs(new Path("/user/test"));

        //3 关闭资源
        fs.close();

        System.out.println("over");
    }
}
```

运行后即可在50070端口查看结果

**出现权限问题的错误**

两种方法

第一种，编辑configurations，添加VM Options，添加参数`-DHADOOP_USER_NAME=root`（root为所有者的名字）

这种方法较麻烦

第二种方法，更改hadoop代码

```java
package com.hdfs.learn;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;

import java.io.IOException;

public class HDFSClient {

    public static void main(String[] args) throws IOException ,Exception, URISyntaxException{

        Configuration conf = new Configuration();
        //conf.set("fs.defaultFS","hdfs://hadoop001:9000");
        //1 获取hdfs客户端路径
        //FileSystem fs = FileSystem.get(conf);
        FileSystem fs = FileSystem.get(new URI("hdfs://hadoop001:9000",conf,"root"));
        //第一个参数uri是访问的namenode的地址
        //注意要在windows的hosts中配置过hadoop001的主机地址
    
        //2 在hdfs上创建路径
        fs.mkdirs(new Path("/user/test"));

        //3 关闭资源
        fs.close();

        System.out.println("over");
    }
}
```

以上代码中抛出异常等部分均为ide自动生成。

## HDFS的API操作

**HDFS文件上传（测试参数优先级）**

```java
fs.copyFromLocalFile(new Path("E:/test.txt"), new Path("/user/test/test2.txt"));
```

在客户端中运行文件上传至阿里云hdfs服务器会出现问题，解决方法及代码见[文件上传错误](#使用idea上传文件成功但是大小为0)

**设置副本数**

在项目目录src/main/resoureces下创建hdfs-site.xml复制代码：

```
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

<configuration>
   <property>
              <name>dfs.replication</name>
              <value>1</value>
  </property>
</configuration>
```

跟安装目录etc/hadoop/hdfs-site.xml中的内容一样，只是仅保留了副本数的设置。

这说明，它的优先级高于集群上的hdfs-site.xml再高于默认值3

此时再在原测试代码中添加

```java
conf.set("dfs.replication", "2");
```

此时上传的文件副本数为2.

总结优先级排序：代码 > 资源库下的（项目的resources目录）> 集群上的hdfs-site.xml > 默认值

**HDFS文件下载**

```java
fs.copyToLocalFile(new Path("/user/test021422"), new Path("D:/hadoop/learn/test0215.txt"));
```

**HDFS文件夹删除**

```java
fs.delete(new Path("/user/test"), true);
	//如果删除的是文件，第二个参数设置什么都可以，如果要删除文件夹，则需要设置为true，否则会报异常,也就是控制是否递归删除
```

**HDFS文件名更改**

```java
fs.rename();
```

**HDFS文件详情查看**

查看文件名称、权限、长度、块信息

```java
RemoteIterator<LocatedFileStatus> listFiles = fs.listFiles(new Path("/user"), true);

while (listFiles.hasNext()) {
    LocatedFileStatus fileStatus = listFiles.next();
    System.out.println(fileStatus.getPath().getName()); //文件名称
    System.out.println(fileStatus.getPermission());     //文件权限
    System.out.println(fileStatus.getLen());            //文件长度

    BlockLocation[] blockLocations = fileStatus.getBlockLocations();    //文件块信息
    for (BlockLocation blockLocation : blockLocations) {
        String[] hosts = blockLocation.getHosts();

        for (String host : hosts) {
            System.out.println(host);
        }
    }
    System.out.println("-------------------------------");
}
```

**HDFS文件和文件夹判断**

```java
FileStatus[] fileStatuses = fs.listStatus(new Path("/user/root/input"));

for (FileStatus fileStatus : fileStatuses) {
    if (fileStatus.isFile()){
        System.out.println("f:" + fileStatus.getPath().getName());  //文件
    }else {
        System.out.println("d:" + fileStatus.getPath().getName());  //文件夹
    }
}
```

**集群间数据拷贝**

1. scp实现两个远程主机间文件复制

   ```bash
   scp -r hello.txt root@hadoop002:/user/root/hello.txt	//push
   scp -r root@hadoop002:/user/root/hello.txt hello.txt 	//pull
   scp -r root@hadoop002:/user/root/hello.txt root@hadoop003:/user/root/hello.txt hello.txt
   	//通过本地主机中转实现两个远程主机的文件复制
   ```

2. 采用distcp命令实现**两个Hadoop集群之间**的递归数据复制

   ```bash
   hadoop distcp hdfs://hadoop002:9000/user/root/hello.txt hdfs://hadoop003:9000/user/root/hello.txt
   ```

## HDFS的I/O流操作

上面的API操作都是框架封装好的，如果想自己实现上述操作，可以采用IO流的方式实现数据的上传和下载。

**HDFS文件上传**

```java
@Test
public void putFileToHDFS() throws URISyntaxException, IOException, InterruptedException {

    // 1 获取fs对象
    Configuration conf = new Configuration();
    conf.set("dfs.client.use.datanode.hostname", "true");
    FileSystem fs = FileSystem.get(new URI("hdfs://hadoop001:9000"), conf, "root");

    // 2 获取输入流
    FileInputStream fis = new FileInputStream(new File("D:\\hadoop\\learn\\test0215.txt"));

    // 3 获取输出流
    FSDataOutputStream fos = fs.create(new Path("/user/test.txt"));

    // 4 流的对拷
    IOUtils.copyBytes(fis, fos, conf);

    // 5 关闭资源
    IOUtils.closeStream(fos);
    IOUtils.closeStream(fis);
    fs.close();

}
```

**HDFS文件下载**

```java
// 2 获取输入流
FSDataInputStream fis = fs.open(new Path("/user/test.txt"));

// 3 获取输出流
FileOutputStream fos = new FileOutputStream(new File("d:/hadoop/learn/test2"));
```

**定位文件读取**

**下载大文件的第一块**

只是流的对拷部分进行处理
设置一个缓冲区，然后进行循环控制下载下来多少数据：

```
// 2 获取输入流
FSDataInputStream fis = fs.open(new Path("/user/root/hadoop-2.10.1.tar.gz"));

// 3 获取输出流
FileOutputStream fos = new FileOutputStream(new File("d:/hadoop/learn/hadoop-2.10.1.tar.gz.part1"));

// 4 流的对拷(只拷贝128M)
byte[] buf = new byte[1024];
for (int i = 0; i < 1024 * 128; i++) {
    fis.read(buf);
    fos.write(buf);
}
```

**下载第二块**

假设这个文件总共就分成了两块，所以下载第二块的对拷部分不需要设置缓冲区控制，即从**第三步设置读取的起点**开始下载

```
// 2 获取输入流
FSDataInputStream fis = fs.open(new Path("/user/root/hadoop-2.10.1.tar.gz"));

// 3 设置指定读取的起点
fis.seek(1024 * 1024 * 128);

// 4 获取输出流
FileOutputStream fos = new FileOutputStream(new File("d:/hadoop/learn/hadoop-2.10.1.tar.gz.part2"));

// 5 流的对拷(只拷贝128M)
IOUtils.copyBytes(fis, fos, conf);
```

# HDFS的数据流

## HDFS写数据流程

**写入过程剖析**

1. 创建客户端对象fs
2. （客户端 -> NameNode）向NameNode请求上传文件/user/root/hadoop-2.10.1.tar.gz
3. （NameNode -> 客户端）响应可以上传文件（如果文件已存在则报FileAlreadyExist）
4. （客户端 -> NameNode）请求上传第一个Block（0-128M），请返回DataNode
5. （NameNode -> 客户端）返回dn1, dn2. dn3节点（选取节点的主要影响因素：距离近、负载小），表示采用这三个节点存储数据
6. 客户端创建输出流FileOutputStream
7. （客户端 -> DataNode1,2,3）请求建立Block传输通道
8. （DataNode1,2,3 -> 客户端）dn1, dn2, dn3应答成功
9. （客户端 -> DataNode1,2,3）传输数据Packet
10. （客户端 -> NameNode）传输数据完成

其中DataNode的传输请求和传输数据为依次传输，客户端发给dn1，dn1发给dn2，dn2发给dn3，然后dn3再返回dn2，再原路返回。

**网络拓扑-节点距离计算**

节点距离：两个节点到达最近的共同祖先的距离总和

**机架感知（副本存储节点选择）**

[官网链接](http://hadoop.apache.org/docs/r2.10.1/hadoop-project-dist/hadoop-hdfs/HdfsDesign.html#Data_Replication)

2.7.2版本：
第一个副本在Client所在的节点上，如果客户端在集群外，随机选一个。
第二个副本和第一个副本位于相同机架，随机节点。
第三个副本位于不同机架，随机节点。

2.10.1版本：
第一个副本在Client所在的节点上，如果客户端在集群外，随机选一个。
第二个副本位于不同（远程）机架上的一个随机节点。
第三个副本与第二个副本在同一机架，位于另一随机节点。

有的老版本与2.10.1这种策略也一样，这种好处是安全性更高，缺点是距离变大导致IO代价增大。

## HDFS读数据流程

1. 创建客户端对象fs
2. （客户端 -> NameNode）请求下载文件/user/root/hadoop-2.10.1.tar.gz
3. （NameNode -> 客户端）返回木匾文件的元数据（也就是告诉客户端数据在哪个节点上）
4. 客户端创建输入流FileInputStream
5. （客户端 -> DataNode1,2,3）请求读数据blk_1（采取距离最近原则）
6. （DataNode1,2,3 -> 客户端）传输数据
7. （客户端 -> DataNode1,2,3）请求读数据blk_2（采取距离最近原则）
8. （DataNode1,2,3 -> 客户端）传输数据
9. 客户端再把读取到的数据块拼接到一起，得到文件

注：每一个Block的读取不是并行的

# NameNode和SecondaryNameNode

## NN和2NN工作机制

NameNode中的元数据是存储再哪里的？

元数据：描述数据的数据，对数据及信息资源的描述性信息。

首先元数据需要存放在**内存中**，但是一段断电数据丢失整个集群就无法工作了，因此产生**在磁盘中备份元数据的FsImage**（即镜像文件）。
这样带来的新问题就是内存中的元数据更新时如果同时更新磁盘中的FsImage会导致效率过低。因此引入**Edits文件**（只进行追加操作，效率很高），每当元数据有更新或者添加元数据时，修改内存中的元数据并追加到Edits中。这样，一旦NameNode节点断电，可以通过FsImage和Edits的合并，合成元数据。
但是如果长时间添加数据到Edits，会导致该文件过大，效率降低，且恢复元数据需要的时间过长。因此需要定期进行合并。为了效率，**引入一个新的节点SecondaryNameNode**，专门用于FsImage和Edits的合并。

**操作流程**

1. （NameNode）加载编辑日志和镜像文件到内存
2. （client -> NameNode）元数据的增删改请求
3. （NameNode）先记录操作日志、更新滚动日志
4. （NameNode）再对内存数据增删改

**引入SecondaryNameNode**

1. （SecondaryNameNode -> NameNode）请求是否需要CheckPoint（也就是询问是否需要合并编辑日志和元数据）

   CheckPoint触发条件：（这两个值都可以设置，只要有一个条件满足即执行）

   1. 定时时间到（1小时）
   2. Edits中的数据满了（一百万条）

2. （SecondaryNameNode -> NameNode）请求执行CheckPoint

3. （NameNode）滚动正在写的Edits：将正在进行的编辑日志文件更名备份并创建新的空编辑日志文件，后面的日志都卸载新建的文件中

4. （NameNode -> SecondaryNameNode）将元数据和备份下来的编辑日志文件拷贝到2NN

5. （SecondaryNameNode -> 内存）加载到内存并合并

6. 生成新的FsImage“FsImage.chkpoint”

7. 将“FsImage.chkpoint”拷贝到nn

8. 重命名为“FsImage”

## Fsimage和Edits解析

镜像文件和编辑日志文件在NN节点的tmp/dfs/name/current路径和2NN节点的tmp/dfs/namesecondary/current路径下

查看方式：

```
hdfs oiv -p XML（文件类型） -i fsimage_0000000000000000105（镜像文件） -o fsimage.xml（转换后输出路径）
cat fsimage.xml
```

```
hdfs oev -p XML（文件类型 -i edits_0000000000000002266-0000000000000002267（编辑日志） -o edits.xml（转换后输出路径）
cat edits.xml
```

**思考：**

在文件中可以看出，fsimage中没有记录块所对应DataNode，为什么？

在集群启动后，要求DataNode上报数据块信息，并间隔一段时间后再次上报，可以保证集群数据的可靠性。

NameNode如何确定下次开机启动的时候合并哪些Edits？

文件seen_txid中记录了最新的Edits编号。

## CheckPoint时间设置

[NN和2NN工作机制下操作流程中使用](#操作流程)

在hdfs-default.xml中可以查看到具体默认设置

```xml
<property>
	<name>dfs.namenode.checkpoint.period</name>
	<value>3600</value>
</property>
```

即3600秒，1小时执行一次

```xml
<property>
	<name>dfs.namenode.checkpoint.txns</name>
	<value>1000000</value>
    <description>操作动作次数</description>
</property>

<property>
	<name>dfs.namenode.checkpoint.check.period</name>
	<value>60</value>
    <description>1分钟检查一次操作次数</description>
</property>
```

## NameNode故障处理

（偏运维方向）

方法一：将2NN中的数据都拷贝到NN中，然后再启动NameNode：`sbin/hadoop-daemon.sh start namenode`。

方法二：使用 -importCheckpoint 选项启动NameNode守护进程，从而将2NN中数据拷贝到NN目录中。
如果2NN不和NN在一个主机节点上，需要将2NN存储数据的目录拷贝到NN存储数据的平级目录，并删除in_use.lock文件
然后导入检查点数据（等待一会ctrl+c结束掉）：`hdfs namenode -importCheckpoint`
然后就可以启动NN：`sbin/hadoop-daemon.sh start namenode`。

## 集群安全模式

**安全模式**

1. **NameNode启动**

   **NameNode启动时**，首先将镜像文件载入内存，并执行编辑日志中的各项操作。一旦在内存中成功建立文件系统元数据的映像，则创建一个新的fsimage文件和一个空的编辑日志。此时，NameNode开始监听DataNode请求。**这个过程期间，NameNode一直运行在安全模式，即NameNode的文件系统对于客户端来说是只读的。**

2. **DataNode启动**

   **系统中的数据块的位置并不是由NameNode维护的，而是在所有数据节点启动的时候动态上传给NameNode的，并以块列表的形式存储在DataNode中。**在系统的正常操作期间，NameNode会在内存中保留所有块位置的映射信息。**在安全模式下**，各个DataNode会向NameNode发送最新的块列表信息，NameNode了解到足够多的块位置信息之后，即可高效运行文件系统。

3. **安全模式退出判断**

   如果满足**“最小副本条件”**，NameNode会在30秒后退出安全模式。
   最小副本条件：在整个文件系统中99.9%的块满足最小副本级别（默认值：dfs.replication.min = 1）
   也就是假如有1000个文件块，最多可以有一个文件找不到副本。
   **在启动一个刚刚格式化的HDFS集群时，因为系统中还没有任何块，所以NameNode不会进入安全模式。**

**基本语法**

集群处于安全模式，不能执行重要操作（写操作）。集群启动完成后，自动退出安全模式。

查看安全模式状态：

```shell
hdfs dfsadmin -safemode get
```

进入安全模式状态：

```shell
hdfs dfsadmin -safemode enter
```

离开安全模式状态：

```shell
hdfs dfsadmin -safemode leave
```

等待安全模状态：（等待安全模式退出后再执行后续操作，相当于阻塞）

```shell
hdfs dfsadmin -safemode wait
```

## NameNode多目录配置

NN的本地目录可以配置成多个，且**每个目录存放内容相同**，增加了可靠性

注：配置需要关闭HDFS服务器，然后删掉/tmp和/log，然后格式化，再启动

具体配置，在hdfs.site.xml中增加内容：

```xml
<property>
    <name>dfs.namenode.name.dir</name>
    <value>file:///${hadoop.tmp.dir}/dfs/name1,file:///${hadoop.tmp.dir}/dfs/name2</value>
</property>
```

## 小文件存档

1. HDFS存储小文件弊端

   每个块的元数据存储在NN的内存中，不管原本文件有多大，它的元数据都是固定的大小，**大量的小文件会耗尽NN中的大部分内存**，因此HDFS存储小文件会非常低效

2. 解决存储小文件办法**之一**

   HDFS存档文件或HAR文件，是一个更高效的文件存档工具，它将文件存入HDFS块，在减少NN内存使用的同事，允许对文件进行透明的访问。具体来说，HDFS存档文件对内还是一个一个独立文件，对NN而言却是一个整体，减少了NN的内存。

3. 案例实操

   1. 需要启动YARN进程

      ```bash
      start-yarn.sh
      ```

   2. 归档文件

      把 /user/root/input 目录里面的所有文件归档成一个叫input.har的归档文件，并把归档后文件存储到 /user/root/output 路径下

      ```bash
      hadoop archive -archiveName input.har -p /user/root/input /user/root/output
      ```

      注：输出目录是原本不存在的

   3. 查看归档文件

      ```bash
   hadoop fs -ls -R har:///user/root/output/input.har
      ```
   
      har: 是一种协议
   
   4. 解归档文件
   
      ```bash
      hdfs dfs -mkdir -p /user/root/input2
      hadoop fs -cp har:///user/root/output/input.har/* /user/root/input2
      ```

## 回收站案例

默认是关闭回收站功能的

开启回收站功能**参数说明**：

1. 默认值 fs.trash.interval=0，0表示禁用回收站；**其他值表示设置文件的存活时间**，单位为分钟
2. 默认值 fs.trash.checkpoint.interval=0，检查回收站的间隔时间。**如果该值为0，则该值设置和 fs.trash.interval的参数值相等**
3. 要求 fs.trash.checkpoint.interval<=fs.trash.interval

**启用回收站**

修改core-site.xml，配置垃圾回收时间为60分钟。

```xml
<property>
    <name>fs.trash.interval</name>
    <value>60</value>
</property>
```

查看回收站：路径为 /user/root/.Trash/...

**修改访问垃圾回收站用户名称**

默认用户名称是dr.who，修改为root用户

修改core-site.xml

```xml
<property>
    <name>hadoop.http.staticuser.user</name>
    <value>root</value>
</property>
```

**通过程序删除的文件不会经过回收站**

需要调用 moveToTrash() 才进入回收站

```java
Trash trash = New Trash(conf);
trashx.moveToTrash(path);
```

**恢复回收站数据**

```bash
hadoop fs -mv /user/root/.Trash/Current/test.txt /user/root/input
```

**清空回收站**

```bash
hadoop fs -expunge
```

## 快照管理

快照相当于对目录做一个备份，**并不会立即复制所有文件**，而是指向同一个文件。当写入发生时，才会产生新文件。

```bash
hdfs dfsadmin -allowSnapshot <路径>		//开启指定目录的快照功能
hdfs dfsadmin -disallowSnapshot <路径>	//禁用指定目录的快照功能，默认是禁用
hdfs dfs -createSnapshot <路径>			//对目录创建快照
hdfs dfs -createSnapshot <路径> <名称>	   //指定名称创建快照
hdfs dfs -renameSnapshot <路径> <旧名称> <新名称>	   //重命名快照
hdfs lsSnapshottableDir					//列出当前用户所有可快照目录
hdfs snapshotDiff <路径1> <路径2>			//比较两个快照目录的不同之处
hdfs dfs -deleteSnapshot <path> <snapshotName>	//删除快照
```











# DataNode

## DataNode工作机制

NameNode中保存元数据

DataNode中保存**数据、数据长度、校验和、时间戳**、等

1. DN启动后向NN注册，告诉NN保存了哪些数据
2. NN将信息保存到元数据，然后给DN应答，注册成功
3. 以后每周期（1h）上报所有块信息，为了可靠性
4. 心跳每3秒一次，心跳返回结果带有NN给该DN的命令
5. 超过10分钟30秒没有收到某个DN的心跳，则认为该节点不可用

**数据完整性**

crc校验，等校验方式

对原始数据重新crc计算，然后和传输过来的crc校验位比较。看是否一致

**掉线时限参数设置**

1. DN进程死亡或者网络故障造成DN无法与NN通信
2. NN不会立即把该节点判定为死亡，要经历一段时间，这段时间暂称作超时时长
3. HDFS默认的超时时长为10分钟+30秒
4. 计算公式为：TimeOut = 2 * dfs.namenode.heartbeat.recheck-interval + 10 * dfs.heartbeat.interval。这两个参数的默认值分别为5分钟和3秒（这两个参数在hdfs-default.xml中）

## 服役新数据节点

1. 环境准备

   1. 在一个DN主机（比如hadoop004）上再克隆一台hadoop005主机

   2. 修改IP地址和主机名称

      ```
      vim /etc/udev/rules.d/70-persistent-net.rules
      将第一段删除，复制第二段中的硬件地址，将NAME改成eth0
      vim /etc/sysconfig/network-scripts/ifcfg-eth0
      将硬件地址替换成前面复制的
      vim /etc/sysconfig/network
      修改主机名称
      reboot
      ```

   3. 删除原来HDFS文件系统留存的文件（/tmp和/log）

   4. `source /etc/profile`

2. 服役新数据节点准备

   直接启动DN`sbin/haddop-daemon.sh start datanode`

## 退役旧数据节点

**添加白名单**

前面那种状态比较危险，只要知道了NN的信息，谁都可以连到集群上。因此设置白名单，添加到白名单的主机节点，都允许访问NN，不在白名单的主机节点，都会被退出。

配置步骤：

1. 在NN的 ./etc/hadoop 目录下创建 dfs.hosts 文件添加如下主机名称

   ```
   hadoop001
   hadoop002
   hadoop003
   hadoop004
   ```

2. 在NN的 hdfs-site.xml 配置文件中增加 dfs.hosts 属性

   ```xml
   <property>
       <name>dfs.hosts</name>
       <value>/usr/hadoop/hadoop-2.10.1/etc/hadoop/dfs.hosts</value>
   </property>
   ```

3. 配置文件分发

   ```bash
   xsync hdfs-site.xml
   ```

4. 刷新NN（目的是让NN重新读一下hdfs-site.xml文件）

   ```bash
   hdfs dfsadmin -refreshNodes
   ```

   此时hadoop005主机就下线了（在web浏览器50070端口可以查看）

5. 更新ResourceManager节点（可以不做）

   刷新的是资源

   ```bash
   yarn rmadmin -refreshNodes
   ```

6. 如果有节点退出后，数据不均衡，可以用命令实现集群的再平衡

   ```bash
   sbin/start-balancer.sh
   ```

**黑名单退役**

在黑名单上的主机都会被强制退出

配置步骤：

1. 在NN的 ./etc/hadoop 目录下创建 dfs.hosts.exclude 文件，添加要退役主机的名称

   ```
   hadoop 005
   ```

2. 在NN的 hdfs-site.xml 配置文件中增加 dfs.hosts.exclude 属性

   ```xml
   <property>
       <name>dfs.hosts.exclude</name>
       <value>/usr/hadoop/hadoop-2.10.1/etc/hadoop/dfs.hosts.exclude</value>
   </property>
   ```

3. 分发 hdfs-site.xml

   ```bash
   xsync hdfs-site.xml
   ```

4. 刷新NN，刷新ResourceManager

   ```bash
   hdfs dfsadmin -refreshNodes
   yarn rmadmin -refreshNodes
   ```

5. 检查Web浏览器，退役节点的状态为 decommission in progress（退役中），说明数据节点正在复制块到其他节点中

6. 等待节点状态为decommissioned（所有块已经复制完成），停止该节点及节点资源管理器。**注意：**如果副本数是3，服役的节点小于等于3，是不能退役成功的，需要修改副本数后才能退役

   ```bash
   sbin/hadoop-daemon.sh stop datanode
   sbin/yarn-daemon.sh stop nodemanager
   ```

7. 如果有节点退出后，数据不均衡，可以用命令实现集群的再平衡

   ```bash
   sbin/start-balancer.sh
   ```

**注：**白名单上的节点和黑名单上的不能重复，不然会崩溃

## DataNode多目录配置

DN也可以配置成多个目录，**每个目录存储的数据不一样**。即：数据不是副本。

注：配置需要关闭HDFS服务器，然后删掉/tmp和/log，然后格式化，再启动.

具体配置：

hdfs-site.xml

```xml
<property>
    <name>dfs.datanode.data.dir</name>
    <value>file:///${hadoop.tmp.dir}/dfs/data1,file:///${hadoop.tmp.dir}/dfs/data2</value>
</property>
```





# 出现问题

**使用idea上传文件成功但是大小为0**

既然是伪分布式集群，所以文件中的所有配置都要留内网ip，方便namenode与datanode相通信。

用xshell直接连接阿里云然后使用`hdfs dfs -put`命令上传文件时，由于机器本身就在内网中，所以不会出现问题，但是在主机windows环境下，idea中，无法直接连接hadoop集群所以导致错误。

客户端通过代码连接时，需要指定namenode通过hostname去连接datanode，按第一行来说，hostname要留内网ip。所以直接通过外网ip是没有办法连接hadoop集群的所以会导致报这种错。

```
File /hdfsapi/test/a.txt could only be replicated to 0 nodes instead of minReplication (=1).
```

所以需要对代码进行修改，增加代码：

```java
		conf.set("dfs.client.use.datanode.hostname","true");
                //指定通过hostname去连接datanode
```

加完之后整段代码如下：

```java
@Test
    public void testCopyFromLocalFile() throws URISyntaxException, IOException, InterruptedException {

        // 1 获取fs对象
        Configuration conf = new Configuration();
        conf.set("dfs.client.use.datanode.hostname", "true");
        FileSystem fs = FileSystem.get(new URI("hdfs://hadoop001:9000"), conf, "root");

        // 2 执行上传API
        fs.copyFromLocalFile(new Path("D:\\hadoop\\learn\\test0214"), new Path("/user/test021422"));

        // 3 关闭资源
        fs.close();

    }
```

**idea提示WARN log4j.logger.org.apache.hadoop.util.NativeCodeLoader=ERROR**

大概是因为没有自己编译，直接把官网压缩包直接解压使用的原因

解决办法：

1. 编译一遍然后把lib/native 目录下的文件替换为编译后的文件

2. 在log4j.property文件中添加如下代码抑制错误：

   ```
   log4j.logger.org.apache.hadoop.util.NativeCodeLoader=ERROR
   ```

   



