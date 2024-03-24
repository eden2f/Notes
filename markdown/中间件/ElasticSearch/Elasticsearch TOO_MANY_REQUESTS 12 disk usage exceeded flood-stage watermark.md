# Elasticsearch TOO_MANY_REQUESTS/12/disk usage exceeded flood-stage watermark

## 异常信息
ES插入大量的数据时报错：TOO_MANY_REQUESTS/12/disk usage exceeded flood-stage watermark, index has read-only-allow-delete block
## 原因
“磁盘使用量超过了洪水位水印，请求过多”。具体来说，Elasticsearch 会在磁盘空间使用率超过某个阈值时，将集群状态设置为“洪水位”，并开始拒绝一些请求，以避免磁盘空间被耗尽，从而导致集群不可用。
当出现“TOO_MANY_REQUESTS”和“disk usage exceeded flood-stage watermark”错误时，意味着 Elasticsearch 集群的磁盘空间已经接近饱和，此时需要及时采取措施，以防止 Elasticsearch 集群因磁盘空间不足而崩溃。可以通过清理不必要的数据、增加磁盘容量等方式来缓解这个问题。
## 解决方法
```
PUT _all/_settings
{
  "index.blocks.read_only_allow_delete": null
}
```
报错问题解决；
紧急给Elasticsearch的硬盘扩容，扩容完毕后执行以下语句关闭索引的只读状态：
```
PUT _all/_settings
{
  "index.blocks.read_only_allow_delete": null
}
```
如果来不及给Elasticsearch硬盘扩容，可以先关闭磁盘分配保护，让最后仅有的5%的磁盘空间缓冲一点时间，然后再给硬盘扩容。
关闭磁盘分配保护
```
PUT _cluster/settings
{
  "transient": {
    "cluster.routing.allocation.disk.threshold_enabled": false
  }
}
```
关闭索引的只读状态
```
PUT _all/_settings
{
  "index.blocks.read_only_allow_delete": null
}
```
```
#! this request accesses system indices: [.apm-agent-configuration, .apm-custom-link, .async-search, .kibana_1, .kibana_7.12.0_001, .kibana_task_manager_1, .kibana_task_manager_7.12.0_001, .reporting-2019.12.01, .reporting-2020.11.08, .reporting-2020.12.06, .reporting-2021-11-07, .security-7, .tasks], but in a future major version, direct access to system indices will be prevented by default
{
  "acknowledged" : true
}
```
磁盘扩容完毕后，启用磁盘分配保护
```
PUT _cluster/settings
{
  "transient": {
    "cluster.routing.allocation.disk.threshold_enabled": true
  }
}
```
