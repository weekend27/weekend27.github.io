## 什么是段合并
elasticsearch中每个索引都会创建一个到多个分片和零个到多个副本，这些分片或副本实质上都是lucene索引。lucene索引是基于多个索引段创建，索引文件中绝大部分数据都是只写一次，读多次，而只有用于保存文档删除信息的文件才会被多次更改。在某些时刻，当某种条件满足时，多个索引段会被拷贝合并到一个更大的索引段，而那些旧的索引段会被抛弃并从磁盘中删除，这操作叫做段合并（segment merging）。

## 为什么要进行段合并
- 索引段的个数越多，搜索性能越低且消耗内存更多。
- 索引段是不可变的，物理上你并不能从中删除信息（如果你碰巧从索引中删除了大量文档，但这些文档只是做了删除标记，物理上并没有被删除）而当段合并发送时，这些标记为删除的文档并没有被复制到新的索引段中。

## 段合并的好处
- 当多个索引段合并为一个的时候，会减少索引段的数量并提高搜索速度
- 同时也会减少索引的容量（文档数）

## 段合并的代价
IO操作代价，在速度较慢的系统中，段合并会显著影响性能。

## 如何进行段合并
利用curator进行段合并，如对xx_%Y.%m.%d类型的index，对2天前的index都进行强制合并，配置如下：
```
actions:
  1:
    action: forcemerge
    description: >-
      Perform a forceMerge on selected indices to 'max_num_segments' per shard.
    options:
      max_num_segments: 1
      delay:
      timeout_override: 10000000
      continue_if_exception: True
      disable_action: False
    filters:
    - filtertype: pattern
      kind: prefix
      value: xx_
      exclude:
    - filtertype: age
      source: name
      direction: older
      timestring: '%Y.%m.%d'
      unit: days
      unit_count: 2
      exclude:
```

如果出现以下错误：
```
Failed to complete action: forcemerge.  <class 'curator.exceptions.FailedExecution'>: Exception encountered.  Rerun with loglevel DEBUG and/or check Elasticsearch logs for more information. Exception: TransportError(403, u'cluster_block_exception', u'blocked by: [FORBIDDEN/12/index read-only / allow delete (api)];')
```
则可以通过修改配置，允许read-only的index被delete：
```
curl -XPUT -H "Content-Type: application/json" {host}:{port}/_all/_settings -d '{"index.blocks.read_only_allow_delete": null}'
```

forcemerge的过程比较漫长，需要等待。

## 参考资料
curator forcemerge：https://www.elastic.co/guide/en/elasticsearch/client/curator/current/forcemerge.html