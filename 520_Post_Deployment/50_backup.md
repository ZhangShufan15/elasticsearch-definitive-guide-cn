##集群备份

像任何一个其他存储数据的软件一样，定期例行备份数据十分重要。Elasticsearch的复制策略提供了运行时的高可用，它能够在零星的节点下线的情况下保持服务不间断。

集群复制在严重的集群故障时不能提供数据保护。因而，为了防止出现大的事故，你需要一个真正的集群备份——完整拷贝。

可以使用`snapshot` API备份集群。它可以获取集群的当前状态和数据，并保存到一个共享的存储中。备份过程是“智能”的。第一份备份是一个完全的数据拷贝，但是，所有后续的备份只存储上一次备份和当前时间之间的增量数据。当你不断的备份数据时，数据是增量的添加和删除的。也就意味着，后续的备份操作会因为传输的数据更少而更加快速。

为了使用备份功能，你必须首先创建一个存储用于保存数据。以下是几个你可以选择的存储类型：

* 共享问价系统，比如NAS
* Amazon S3
* HDFS（Hadoop分布式文件系统）
* Azure Cloud

###创建存储

我们来配置一个共享文件系统存储：

```javascript
PUT _snapshot/my_backup <1>
{
    "type": "fs", <2>
    "settings": {
        "location": "/mount/backups/my_backup" <3>
    }
}
```

* <1>这里是我们所用存储的名字，叫做`my_backup`
* <2>这里声明存储的类型是共享文件系统
* <3>最后，我们给定一个挂载好的驱动地址作为备份目的地

> 备注：共享文件系统路径必须能够在所有的集群节点可访问！

上述请求会创建存储并获取挂载点的元数据信息。这里还有一些其他的参数可以配置，这要看节点的性能、网络和存储位置：

`max_snapshot_bytes_per_sec`:

在备份数据到存储时，该参数控制备份处理的带宽。默认值是20mb/s。

`max_restore_bytes_per_sec`:

在恢复数据时，该参数控制恢复处理的带宽。默认值是20mb/s。

假设我们的网络速度比较快，并且能够承受额外的传输量，因此，我们提高默认值：

```javascript
POST _snapshot/my_backup/ <1>
{
    "type": "fs",
    "settings": {
        "location": "/mount/backups/my_backup",
        "max_snapshot_bytes_per_sec" : "50mb", <2>
        "max_restore_bytes_per_sec" : "50mb"
    }
}
```

* <1>注意这里使用了`POST`而不是 `PUT`。用于更改已经存在的存储。
* <2>设置新的配置值

###备份所有的索引

一个存储可以容纳多个备份。每一个备份包含一组索引（所有索引、部分索引、单个索引）。当创建备份的时候，你需要指明要备份的索引，并且指定一个唯一的备份名称。

我们从最基本的备份命令开始：

```javascript
PUT _snapshot/my_backup/snapshot_1
```

这将备份所有打开的索引到`my_backup`存储中，备份名为 `snapshot_1`文件。这个请求会立刻返回，备份操作将在后台执行。


> 备注：
> 通常情况下，你希望备份过程在后台执行，但有时候可能也会想等待脚本执行完成后再返回。这可以通过在请求中添加`wait_for_completion` 标记做到：
>
> ```javascript
> PUT _snapshot/my_backup/snapshot_1?wait_for_completion=true
> ```
>
> 这将阻塞请求，知道备份结束。对于大的备份可能需要很长的时间才能返回。

==== Snapshotting Particular Indices

The default behavior is to back up all open indices.((("indices", "snapshotting particular")))((("backing up your cluster", "snapshotting particular indices")))  But say you are using Marvel,
and don't really want to back up all the diagnostic `.marvel` indices.  You
just don't have enough space to back up everything.

In that case, you can specify which indices to back up when snapshotting your cluster:

[source,js]
----
PUT _snapshot/my_backup/snapshot_2
{
    "indices": "index_1,index_2"
}
----

This snapshot command will now back up only `index1` and `index2`.

==== Listing Information About Snapshots

Once you start accumulating snapshots in your repository, you may forget the details((("backing up your cluster", "listing information about snapshots")))
relating to each--particularly when the snapshots are named based on time
demarcations (for example, `backup_2014_10_28`).

To obtain information about a single snapshot, simply issue a `GET` reguest against
the repo and snapshot name:

[source,js]
----
GET _snapshot/my_backup/snapshot_2
----

This will return a small response with various pieces of information regarding
the snapshot:

[source,js]
----
{
   "snapshots": [
      {
         "snapshot": "snapshot_1",
         "indices": [
            ".marvel_2014_28_10",
            "index1",
            "index2"
         ],
         "state": "SUCCESS",
         "start_time": "2014-09-02T13:01:43.115Z",
         "start_time_in_millis": 1409662903115,
         "end_time": "2014-09-02T13:01:43.439Z",
         "end_time_in_millis": 1409662903439,
         "duration_in_millis": 324,
         "failures": [],
         "shards": {
            "total": 10,
            "failed": 0,
            "successful": 10
         }
      }
   ]
}
----

For a complete listing of all snapshots in a repository, use the `_all` placeholder
instead of a snapshot name:

[source,js]
----
GET _snapshot/my_backup/_all
----

==== Deleting Snapshots

Finally, we need a command to delete old snapshots that ((("backing up your cluster", "deleting old snapshots")))are no longer useful.
This is simply a `DELETE` HTTP call to the repo/snapshot name:

[source,js]
----
DELETE _snapshot/my_backup/snapshot_2
----

It is important to use the API to delete snapshots, and not some other mechanism
(such as deleting by hand, or using automated cleanup tools on S3).  Because snapshots are
incremental, it is possible that many snapshots are relying on old segments.
The `delete` API understands what data is still in use by more recent snapshots,
and will delete only unused segments.

If you do a manual file delete, however, you are at risk of seriously corrupting
your backups because you are deleting data that is still in use.


==== Monitoring Snapshot Progress

The `wait_for_completion` flag provides a rudimentary form of monitoring, but
really isn't sufficient when snapshotting or restoring even moderately sized clusters.

Two other APIs will give you more-detailed status about the
state of the snapshotting.  First you can execute a `GET` to the snapshot ID,
just as we did earlier get information about a particular snapshot:

[source,js]
----
GET _snapshot/my_backup/snapshot_3
----

If the snapshot is still in progress when you call this, you'll see information
about when it was started, how long it has been running, and so forth.  Note, however,
that this API uses the same threadpool as the snapshot mechanism.  If you are
snapshotting very large shards, the time between status updates can be quite large,
since the API is competing for the same threadpool resources.

A better option is to poll the `_status` API:

[source,js]
----
GET _snapshot/my_backup/snapshot_3/_status
----

The `_status` API returns immediately and gives a much more verbose output of
statistics:

[source,js]
----
{
   "snapshots": [
      {
         "snapshot": "snapshot_3",
         "repository": "my_backup",
         "state": "IN_PROGRESS", <1>
         "shards_stats": {
            "initializing": 0,
            "started": 1, <2>
            "finalizing": 0,
            "done": 4,
            "failed": 0,
            "total": 5
         },
         "stats": {
            "number_of_files": 5,
            "processed_files": 5,
            "total_size_in_bytes": 1792,
            "processed_size_in_bytes": 1792,
            "start_time_in_millis": 1409663054859,
            "time_in_millis": 64
         },
         "indices": {
            "index_3": {
               "shards_stats": {
                  "initializing": 0,
                  "started": 0,
                  "finalizing": 0,
                  "done": 5,
                  "failed": 0,
                  "total": 5
               },
               "stats": {
                  "number_of_files": 5,
                  "processed_files": 5,
                  "total_size_in_bytes": 1792,
                  "processed_size_in_bytes": 1792,
                  "start_time_in_millis": 1409663054859,
                  "time_in_millis": 64
               },
               "shards": {
                  "0": {
                     "stage": "DONE",
                     "stats": {
                        "number_of_files": 1,
                        "processed_files": 1,
                        "total_size_in_bytes": 514,
                        "processed_size_in_bytes": 514,
                        "start_time_in_millis": 1409663054862,
                        "time_in_millis": 22
                     }
                  },
                  ...
----
<1> A snapshot that is currently running will show `IN_PROGRESS` as its status.
<2> This particular snapshot has one shard still transferring (the other four have already completed).

The response includes the overall status of the snapshot, but also drills down into
per-index and per-shard statistics.  This gives you an incredibly detailed view
of how the snapshot is progressing.  Shards can be in various states of completion:

`INITIALIZING`::
    The shard is checking with the cluster state to see whether it can
be snapshotted.  This is usually very fast.

`STARTED`::
    Data is being transferred to the repository.
    
`FINALIZING`::
    Data transfer is complete; the shard is now sending snapshot metadata.
    
`DONE`::
    Snapshot complete!
    
`FAILED`::
    An error was encountered during the snapshot process, and this shard/index/snapshot
could not be completed.  Check your logs for more information.


==== Canceling a Snapshot

Finally, you may want to cancel a snapshot or restore.((("backing up your cluster", "canceling a snapshot")))  Since these are long-running
processes, a typo or mistake when executing the operation could take a long time to
resolve--and use up valuable resources at the same time.

To cancel a snapshot, simply delete the snapshot while it is in progress:

[source,js]
----
DELETE _snapshot/my_backup/snapshot_3
----

This will halt the snapshot process. Then proceed to delete the half-completed
snapshot from the repository.


