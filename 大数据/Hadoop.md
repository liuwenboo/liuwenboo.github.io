[toc]

# 基础操作

## 运行

#### 三种命令方式的区别

hadoop fs：使用面最广，可以操作任何文件系统。

hadoop dfs与hdfs dfs：只能操作HDFS文件系统相关（包括与Local FS间的操作），前者已经Deprecated，一般使用后者。

dfs是fs的实现类，hadoop fs “包含” hdfs dfs。

#### 常用命令

**运行命令**：

```
hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.10.1.jar wordcount user/root/input/ user/root/output
```

注意：输出文件目录必须没有，它会自己创建。

**查看输出结果：**

```
hadoop fs -ls (output / wcoutput 等)	//文件夹中有_SUCCESS就算运行成功了，运行结果在另一个文件中
hadoop fs -cat output/part-r-00000
hdfs dfs -cat output/part-r-00000
hdfs dfs -cat output/p*
```

**输出hadoop上的文件：**

```
hadoop fs –rm [文件地址]
```

**直接在hdfs集群上创建或删除文件夹：**

```
hdfs dfs -mkdir <-p>(创建多级目录) /user/test
hdfs dfs -rm (或-rmdir) /user/test	//删除文件或文件夹
```

**将本地文件夹存储至hadoop：**

```
hadoop fs –put [本地目录] [hadoop目录]
hdfs dfs -put [本地目录] [hadoop目录]
```

**将hadoop上某个文件down至本地已有目录下：**

```
hadoop fs -get [文件目录] [本地目录]
```

**在hadoop指定目录内创建新目录：**

```
hadoop fs –mkdir /user/t
```

**在hadoop指定目录下新建一个空文件：**

```
hadoop fs -touchz /user/new.txt
```

**将正在运行的hadoop作业kill掉：**

```
hadoop job –kill [job-id]
```

**将hadoop指定目录下所有内容保存为一个文件，同时down至本地：**

```
hadoop dfs –getmerge /user /home/t
```

**-help：输出这个命令参数**

如 `hadoop fs -help rm`

**-moveFromLocal：从本地剪切粘贴到HDFS**

```
hadoop fs -moveFromLocal ./test.txt（本地） /user/test/（hdfs）
```

是将本地文件**剪切**至hdfs系统

**-copyFromLocal：从本地复制粘贴到HDFS**

```
hadoop fs -copyFromLocal ./test.txt（本地） /user/test/（hdfs）
```

是将本地文件**复制**至hdfs系统

**-copyToLocal：从HDFS复制粘贴到本地**

```
hadoop fs -copyToLocal /user/test/（hdfs）./test.txt（本地） 
```

是将hdfs系统的文件**复制**至本地

**-cp：从HDFS的一个路径拷贝到HDFS的另一个路径**

**-mv：在HDFS目录中移动文件**

**-get：等同于copyToLocal，就是从HDFS下载文件到本地**

**-put：等同于copyFromLocal**

**-getmerge：合并下载多个文件**

比如HDFS的目录 /aaa/下有多个文件：log.1，log.2，log.3，...

```
hadoop fs -getmerge /aaa/* ./test.txt
```

**-appendToFile：追加一个文件到已经存在的文件末尾**

```
hadoop fs -appendToFile ./test2.ext /user/test/test.txt
```

**-chgrp、-chmod、-chown：与linux中的用法一样**

分别是修改组，修改权限，修改所有者和所有者组

**-setrep：设置HDFS中文件的副本数量**

```
hadoop fs -setrep 10 /user/test/test.txt
```

如果副本数量小于集群上服务器的数量，则会随机一部分服务器上没有存储备份数据

如果副本数量大于集群上服务器的数量，则所有服务器上都会存有备份，并且再增加服务器后，会自动存储上备份数据

**-du：统计文件夹的大小信息**

```
hadoop fs -du /
hadoop fs -du -h /    (把较大的文件单位换算成可读的M，K等)
hadoop fs -fu -h -s /   （查看当前 / 这个文件夹总的大小）
```

## 完全分布式

**rsync** 远程同步工具：速度比scp快，因为它只对差异文件做更新，而scp是把所有文件都复制过去

**xsync：**

xsync是对rsync脚本的二次封装，所以需要先下载rsync命令`yum install -y rsync`

先进入用户主目录：

```bash
mkdir bin
cd bin/
touch xsync
vim xsync
```

编写：

```bash
#!/bin/sh
# 获取输入参数个数，如果没有参数，直接退出
pcount=$#
if((pcount==0)); then
        echo no args;
        exit;
fi
# 获取文件名称
p1=$1
fname=`basename $p1`
echo fname=$fname
# 获取上级目录到绝对路径
pdir=`cd -P $(dirname $p1); pwd`
echo pdir=$pdir
# 获取当前用户名称
user=`whoami`
# 循环
for((host=001; host<=004; host++)); do
        echo $pdir/$fname $user@slave$host:$pdir
        echo ==================slave$host==================
        rsync -rvl $pdir/$fname $user@slave$host:$pdir
done
#Note:这里的slave对应自己主机名，需要做相应修改。另外，for循环中的host的边界值
```

然后修改脚本xsync具有执行权限：

```bash
chmod 777 xsync
```

调用脚本形式：xsync文件名称

```bash
xsync /home/test
```

如果需要输入密码是因为没有配置ssh免密登录

注：如果将 xsync 放在用户主目录下仍然不能实现全局使用，可以将 xsync 移动到 /usr/local/bin 目录下

# 要点

### 为什么不能一直格式化namenode

在$HADOOP_HOME/tmp/dfs目录下的name和data文件夹中的VERSION文件中可以看到nn和dn的集群ID（clusterID），必须要一样才能保持通讯，格式化namenode有可能会改变ID号，导致无法通讯，或者datanode与namenode总有一个是挂的，即幽灵情况。

**解决办法**

在格式化namenode之前，先结束jps中的进程，然后删除datanode里面的信息（默认在/tmp）和/log。