# general

## 磁盘空间不足，如果增加空间

原来的空间预留不足，添加新磁盘后，修改了数据目录。怎么搞？

### 方法一： 停止数据库，手动mv文件到目标目录，然后修改 gp_filespace_entry 表。在 6.0 中这个表没有了，修改 gp_segment_configuration.

### 方法二：借助 gprecoverseg -i 实现，先把所有 mirror 宕下来，用 gprecoverseg 生成配置文件，修改这个配置文件的实录目录，删除旧的 mirror 目录，然后执行 gprecoverseg -i，将mirror加到修改后的数据库目录中。对primary 再做一次相同的操作。


