# OLTP

## shared buffer and index replication

backend+bgwriter进程的写入量与mirror的写入量基本上是一致的，索引写放大的问题，在目前版本调大shared
buffer就可以解决，对于OLTP类的场景，可以建议把shared_buffer调大一些。
