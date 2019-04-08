# 压缩

Greenplum 有很多地方采用了压缩。不同的地方使用不同的压缩算法。

## Workfile

5.x 可以使用 zlib，6 之后可以使用 zstd。速度和内存消耗都比较低。

    set gp_workfile_compression = on

从 6 开始，workfile 使用了 PG 的 BufFile API。做了大量的重构。

5.x 之前：

    set gp_workfile_compress_algorithm ='zlib'
    gpconfig -c gp_workfile_compress_algorithm  -v zlib

zlib handler 需要 32k 内存。如果有10w个，就是3.2GB内存小猴。

这一块还需要更仔细的分析，zstd 对内存的需求量更大了.

## dispatch

使用 zstd 默认对plan进行压缩，然后 dispatch。

对于很大的 SQL， dispatch 的代价非常高，特别是带有很多分区表的SQL，
光`debug_query`就非常耗内存和网络。

## AO 数据

过去使用zlib，现在使用 zstd。 AOCO 如果文件数过多，也会造成其对应的
handler 占用内存过多。 现在有望可以降到 1/32 分之一。

## Toast 数据
