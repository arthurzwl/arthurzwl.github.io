---
title: Hadoop文件读取问题定位分析
date: 2019-01-31 18:06:56
categories:
  - 大数据
tags:
  - hadoop
---
## 问题描述

后台定时任务并发读取hdfs文件时，出现报错，无法正常读取hdfs上的文件内容。
具体错误内容如下：

``` java
java.io.IOException: Filesystem closed
        at org.apache.hadoop.hdfs.DFSClient.checkOpen(DFSClient.java:883)
        at org.apache.hadoop.hdfs.DFSClient.getFileInfo(DFSClient.java:2114)
        at org.apache.hadoop.hdfs.DistributedFileSystem$21.doCall(DistributedFileSystem.java:1300)
        at org.apache.hadoop.hdfs.DistributedFileSystem$21.doCall(DistributedFileSystem.java:1296)
        at org.apache.hadoop.fs.FileSystemLinkResolver.resolve(FileSystemLinkResolver.java:81)
        at org.apache.hadoop.hdfs.DistributedFileSystem.getFileStatus(DistributedFileSystem.java:1312)
        at org.apache.hadoop.fs.FilterFileSystem.getFileStatus(FilterFileSystem.java:416)
        at org.apache.hadoop.fs.viewfs.ChRootedFileSystem.getFileStatus(ChRootedFileSystem.java:224)
        at org.apache.hadoop.fs.viewfs.ViewFileSystem.getFileStatus(ViewFileSystem.java:379)
        at org.apache.hadoop.hdfs.FederatedDFSFileSystem.getFileStatus(FederatedDFSFileSystem.java:721)
        at org.apache.hadoop.fs.FileSystem.exists(FileSystem.java:1426)
```

## 问题定位

通过日志内容，我们可以发现，在我们调用FileSystem.exists方法时，底层调用的DFSClient.checkOpen方法抛出了IOException异常，FileSystem closed，可以大概确认是FileSystem异常关闭导致的问题。FileSystem 实现了Closeable接口，根据我们日常的最佳编程实践，一般在申请IO资源后，IO操作结束后都会调用close方法释放资源。代码里我们也确实使用JDK7 try-with-resources释放了FileSystem，难道这里有问题，或者是我们获取FileSystem的方法有问题吗？带着这些问题我继续查看了FileSystem的源码。

``` java
  public static FileSystem get(URI uri, Configuration conf) throws IOException {
    String scheme = uri.getScheme();
    String authority = uri.getAuthority();

    if (scheme == null && authority == null) { // use default FS
      return get(conf);
    }

    if (scheme != null && authority == null) { // no authority
      URI defaultUri = getDefaultUri(conf);
      if (scheme.equals(defaultUri.getScheme()) // if scheme matches default
          && defaultUri.getAuthority() != null) { // & default has authority
        return get(defaultUri, conf); // return default
      }
    }

    String disableCacheName = String.format("fs.%s.impl.disable.cache", scheme);
    if (conf.getBoolean(disableCacheName, false)) {
      return createFileSystemWithConfigurationService(uri, conf);
    }

    return CACHE.get(uri, conf);
  }
```

我们使用的是静态方法FileSystem.get(uri, conf) 获取的FileSystem实例。通过源码查看，我们可以发现这个静态方法并不是每次都穿件一个全新实例，而是默认去缓存里获取实例。

进一步查看FileSystem$Cache的源码，我们可以看到缓存使用了HashMapp<Key, FileSystem>存储了每次创建的FileSystem实例，并使用同步代码块保证线程安全问题。Key是通过uri和conf创建的，而在我们的代码里uri和conf在访问相同集群时是一样。这样就导致了在并发访问同一集群时，多个线程共享使用的同一个FileSystem实例，如果一个线程关闭了资源，另一个线程没有执行完，那么势必会导致抛出IOException Filesystem closed异常。

``` java
  // 只截取了部分源码
  static class Cache {
  
    private final Map<Key, FileSystem> map = new HashMap<Key, FileSystem>();
  
    FileSystem get(URI uri, Configuration conf) throws IOException{
      Key key = new Key(uri, conf);
      return getInternal(uri, conf, key);
    }

    private FileSystem getInternal(URI uri, Configuration conf, Key key) throws IOException{
      FileSystem fs;
      synchronized (this) {
        fs = map.get(key);
      }
      if (fs != null) {
        return fs;
      }

      if (conf.getBoolean("dfs.client.filesystem.leak.debug", false)) {
        try {
          throw new IllegalThreadStateException("New file system instance is created");
        } catch (IllegalThreadStateException e) {
          LOG.error("Create a new file system for key: " + key + " cache size: " + map.size(), e);
        }
      }

      fs = createFileSystemWithConfigurationService(uri, conf);
      synchronized (this) { // refetch the lock again
        FileSystem oldfs = map.get(key);
        if (oldfs != null) { // a file system is created while lock is releasing
          fs.close(); // close the new file system
          return oldfs;  // return the old file system
        }

        // now insert the new file system into the map
        if (map.isEmpty()
                && !ShutdownHookManager.get().isShutdownInProgress()) {
          ShutdownHookManager.get().addShutdownHook(clientFinalizer, SHUTDOWN_HOOK_PRIORITY);
        }
        fs.key = key;
        map.put(key, fs);
        if (conf.getBoolean("fs.automatic.close", true)) {
          toAutoClose.add(key);
        }
        return fs;
      }
    }
```

## 问题解决

发现问题以后，我们如何解决呢？
通过阅读代码和网上搜索，目前大概有三种解决方案：

1. 代码里使用FileSystem最后不调用close方法
2. 设置 fs.hdfs.impl.disable.cache 为true，关闭缓存
3. 使用 FileSystem.newInstance 方法，每次获取新的实例

关于第三种方法，在网上搜索的过程中，我发现[HADOOP-4655](https://issues.apache.org/jira/browse/HADOOP-4655)以前有人也在类似的使用场景下遇到了同样的问题，他们最终的解决方案是修改了API，增加了newInstance静态方法。

方案选择上，基于我们的使用场景，低频次的离线后台任务，对时间不是很敏感，并不需要长时间保持连接，我选择了方案三。
