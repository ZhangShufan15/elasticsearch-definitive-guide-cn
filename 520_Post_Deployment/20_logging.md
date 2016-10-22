##日志

Elasticsearch输出一系列的日志，默认存储在`ES_HOME/logs`目录中。Elasticsearch的默认日志级别是`INFO`。该日志级别提供适度数量的运行信息，同时被设计的更加轻量级，从而你的日志不会变的很大。

当需要调试问题的时候，尤其是遇到节点发现问题时（因为它总是依赖于繁琐的网络配置），把日志级别修改成`DEBUG`会有很大的帮助。

你可以修改`logging.yml`文件，然后重启所有节点来修改日志级别，但是这样一来无趣，而来会带来不必要的宕机。此时，你可以通过`cluster-settings` API修改日志级别，修改的方法我们在上一节刚刚学过。

你需要，在你感兴趣的logger前加上`logger.` （注：得到配置项名称）。我们看下如何打开`discovery`模块的日志。

```javascript
PUT /_cluster/settings
{
    "transient" : {
        "logger.discovery" : "DEBUG"
    }
}
```

设置生效之后，Elasticsearch就会开始输出`discovery`模块`DEBUG`级别的日志。


> 提示：避免使用`TRACE`级别的日志。这个级别的日志十分冗长，这时候的日志意义不大。(注：日志级别顺序：OFF->FATAL->ERROR->WARN->INFO->DEBUG->TRACE->ALL)

##慢日志

还有另外一种类型的日志，叫做慢日志。慢日志的目的是：捕获查询和索引响应时间超过某个特定阈值的请求。这对捕获速度特别慢的用户查询十分有帮助。

默认情况下，慢日志是没有打开的。你可以通过定义操作类型（query, fetch, or index）、日志级别（(`WARN`, `DEBUG`等）、响应时间阈值来开启慢日志。

这是一个索引级的配置示例，它对单独的索引进行设置：

```javascript
PUT /my_index/_settings
{
    "index.search.slowlog.threshold.query.warn" : "10s", <1>
    "index.search.slowlog.threshold.fetch.debug": "500ms", <2>
    "index.indexing.slowlog.threshold.index.info": "5s" <3>
}
```

* <1> 输出`WARN`级别的日志，当查询时间超过10s
* <2> 输出`DEBUG`级别的日志，当查询时间超过500ms
* <3> 输出`INFO`级别日志，当索引时间超过5s

你也可以在`elasticsearch.yml` 文件中定义慢查询阈值。对于没有单独设置慢查询日志的索引，会继承`elasticsearch.yml` 文件中的配置值。

一旦设置了慢查询阈值，你就可以像修改其他配置项一样切换慢查询日志级别：

```javascript
PUT /_cluster/settings
{
    "transient" : {
        "logger.index.search.slowlog" : "DEBUG", <1>
        "logger.index.indexing.slowlog" : "WARN" <2>
    }
}
```

* <1> 设置搜索慢查询日志为`DEBUG`级别
* <2> 设置索引慢查询日志为`WARN`级别


