
##动态修改配置项

Elasticsearch的很多配置项都是动态的，可以通过API进行修改。对于配置后需要强制节点（或者集群）重启的配置项，我们强烈建议不要动态修改。虽然，通过修改配置文件可以修改配置项，但是，我们建议需要时使用API去修改配置项。


`cluster-update`API存在两种运行模式：

`临时起效`

这个类型的配置项修改在集群重启之前起效，一旦发生了集群重启，所有的这类修改将被擦除。

`永久起效`

如果没有发生显式修改，这个类型的配置项将一直起效。哪怕，集群重启这些配置修改也将起效，并且会覆盖静态配置文件中的相同配置项。

下面的配置示例同时包含了临时配置项和永久配置项：

```Javascript
PUT /_cluster/settings
{
    "persistent" : {
        "discovery.zen.minimum_master_nodes" : 2 <1>
    },
    "transient" : {
        "indices.store.throttle.max_bytes_per_sec" : "50mb" <2>
    }
}
```

* <1> 永久起效，配置值在集群重启后仍然起效
* <2> 临时起效，在一次整个集群的完全重启后失效

完整的可动态修改的配置项列表，请查看[在线文档](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/cluster-update-settings.html)。
