# 数据湖相关疑难杂症

这里会介绍一些在湖频繁遇到的问题，并且给出一定的解决方案。这里介绍的很多指标，需要用户自己通过 `set enable_profile=true` 采集 SQL 的 profile 进行分析。

## HDFS 慢节点问题

在访问 HDFS 的文件时，如果发现 profile 中 `__MAX_OF_FSIOTime` 和 `__MIN_OF_FSIOTime` 的值有很大的差距，则意味着当前环境存在 HDFS 慢节点的情况。如下 Profile，就是典型的慢节点场景。

```bash
 - InputStream: 0
   - AppIOBytesRead: 22.72 GB
     - __MAX_OF_AppIOBytesRead: 187.99 MB
     - __MIN_OF_AppIOBytesRead: 64.00 KB
   - AppIOCounter: 964.862K (964862)
     - __MAX_OF_AppIOCounter: 7.795K (7795)
     - __MIN_OF_AppIOCounter: 1
   - AppIOTime: 1s372ms
     - __MAX_OF_AppIOTime: 4s358ms
     - __MIN_OF_AppIOTime: 1.539ms
   - FSBytesRead: 15.40 GB
     - __MAX_OF_FSBytesRead: 127.41 MB
     - __MIN_OF_FSBytesRead: 64.00 KB
   - FSIOCounter: 1.637K (1637)
     - __MAX_OF_FSIOCounter: 12
     - __MIN_OF_FSIOCounter: 1
   - FSIOTime: 9s357ms
     - __MAX_OF_FSIOTime: 60s335ms
     - __MIN_OF_FSIOTime: 1.536ms
```

这时候可以通过在 `be.conf` 中配置 `hdfs_client_enable_hedged_read=true` 开启 HDFS 的 [hedged read](https://issues.apache.org/jira/browse/HDFS-5776)。`be.conf` 提供了如下参数来更加精细化的调整：

| 参数名称                                 | 默认值 | 备注                                                         |
| ---------------------------------------- | ------ | ------------------------------------------------------------ |
| hdfs_client_enable_hedged_read           | false  | 是否开启 HDFS hedged read                                    |
| hdfs_client_hedged_read_threadpool_size  | 128    | 对应 `hdfs-site.xml` 中的 `dfs.client.hedged.read.threadpool.size` |
| hdfs_client_hedged_read_threshold_millis | 2500   | 对应  `hdfs-site.xml` 中的 `dfs.client.hedged.read.threshold.millis` |

如果在 profile 中观测到下面几个指标有值大于 0，则代表 hedged read 开启成功。

```bash
 - TotalHedgedReadOps
 - TotalHedgedReadOpsInCurThread
 - TotalHedgedReadOpsWin
```

关于 hedged read 的几个参数解释，可见 [https://hadoop.apache.org/docs/r2.8.3/hadoop-project-dist/hadoop-common/release/2.4.0/RELEASENOTES.2.4.0.html](https://hadoop.apache.org/docs/r2.8.3/hadoop-project-dist/hadoop-common/release/2.4.0/RELEASENOTES.2.4.0.html)。

**注意：Hedged read 并不是万金油，会显著增加 JVM 堆内存的消耗，在物理机内存比较小的情况下，不建议开启 hedged read。真正合理的解决办法是通过 [Data Cache](/data_source/data_cache.md) 对远端数据进行自动缓存，消除慢节点对查询的影响。**
