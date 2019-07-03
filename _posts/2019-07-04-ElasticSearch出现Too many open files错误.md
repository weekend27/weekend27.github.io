### 问题描述
ES无法正常访问，error log如下：

```
java.nio.file.FileSystemException: /home/disk1/siem-es-data/siem-log-es/nodes/0/indices: Too many open files
at sun.nio.fs.UnixException.translateToIOException(UnixException.java:91)
at sun.nio.fs.UnixException.rethrowAsIOException(UnixException.java:102)
at sun.nio.fs.UnixException.rethrowAsIOException(UnixException.java:107)
at sun.nio.fs.UnixFileSystemProvider.newDirectoryStream(UnixFileSystemProvider.java:427)
at java.nio.file.Files.newDirectoryStream(Files.java:457)
at org.elasticsearch.env.NodeEnvironment.findAllIndices(NodeEnvironment.java:530)
at org.elasticsearch.gateway.local.state.meta.LocalGatewayMetaState.clusterChanged(LocalGatewayMetaState.java:245)
at org.elasticsearch.gateway.local.LocalGateway.clusterChanged(LocalGateway.java:215)
at org.elasticsearch.cluster.service.InternalClusterService$UpdateTask.run(InternalClusterService.java:467)
at org.elasticsearch.common.util.concurrent.PrioritizedEsThreadPoolExecutor$TieBreakingPrioritizedRunnable.runAndClean(PrioritizedEsThreadPoolExecutor.java:188)
at org.elasticsearch.common.util.concurrent.PrioritizedEsThreadPoolExecutor$TieBreakingPrioritizedRunnable.run(PrioritizedEsThreadPoolExecutor.java:158)
at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
at java.lang.Thread.run(Thread.java:748)
```

### 问题分析
打开的文件过多，一般来说是由于应用程序对资源使用不当造成，比如没有及时关闭Socket或数据库连接等。但也可能应用确实需要打开比较多的文件句柄，而系统本身的设置限制了这一数量。当前的情况，属于后者：**系统本身的设置限制了这个数量**。

### 问题解决

#### 方法一
1. 查看系统允许打开的最大文件数：cat /proc/sys/fs/file-max
2. 查看每个用户允许打开的最大文件数：ulimit -a，发现系统默认的是open files (-n) 1024，问题就出现在这里
3. 在系统文件/etc/security/limits.conf中修改这个数量限制，在文件中加入内容：
```
* soft nofile 65536 
* hard nofile 65536
```

#### 方法二
1. 使用命令 `ps -ef | grep java`  (java代表你程序，查看你程序进程) 查看你的进程ID，记录ID号，假设进程ID为12
2. 使用命令 `lsof -p 12 | wc -l`    查看当前进程id为12的 文件操作状况，执行该命令出现文件使用情况为 1052
3. 使用命令 `ulimit -a`  查看每个用户允许打开的最大文件数，发现系统默认的是open files (-n) 1024，问题就出现在这里
4. 使用命令 `ulimit -n 4096` 将open files (-n) 1024 设置成open files (-n) 4096，这样就增大了用户允许打开的最大文件数