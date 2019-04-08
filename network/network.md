# network

## gp_max_packet_size

某些场景中，gp_max_packet_size对整体加载的吞吐没有太大的影响，主要影响的是加载性能的波动，默认值8KB，会有较大波动，以500ms的间隔加载，有多次时延超过1s的，调整到1KB或2KB性能波动会小很多。

有次扩容新加的机器有跨核心交换机，默认的8KB设置数据包太大，超过网卡MTU设置，传输过程中并网卡需要拆包有可能发生UDP乱序，交换机层面可能认为是异常的DoS攻击把包丢弃掉了，导致整个集群SQL HANG死，具体原因他们还在进一步的分析；http://info.stack8.com/blog/udp-fragmentation-why-should-you-avoid-it

             另外想咨询个问题，很多客户在实时的场景里也想用到AO表，但是AO表不能有唯一索引，即使表上有索引，排序字段如果是索引字段，也用不到索引，是否可以考虑加到产品特性里，如果确实比较难做，能不能从原理上给客户一个较为详细的解释。

## NIC driver buffer size

 the network performance of greenplum could enhance to a brand new level by just adjust a nic driver setting or os setting.  Kernel buffer size of net data is small like 512B, while many NIC driver has 8K~32K buffer, so there is mismatch.

The ring buffer size for NIC DMA can be queried with the ethtool command:

```
# replace enp0s3 with your actual network devname
$ ethtool -g enp0s3
Ring parameters for enp0s3:
Pre-set maximums:
RX:             4096
RX Mini:        0
RX Jumbo:       0
TX:             4096
Current hardware settings:
RX:             256
RX Mini:        0
RX Jumbo:       0
TX:             256
```

To increase it we could also use the ethtool command:

```
$ sudo ethtool -G enp0s3 rx 4096 tx 4096

$ ethtool -g enp0s3
Ring parameters for enp0s3:
Pre-set maximums:
RX:             4096
RX Mini:        0
RX Jumbo:       0
TX:             4096
Current hardware settings:
RX:             4096
RX Mini:        0
RX Jumbo:       0
TX:             4096
```

the max supported should be the RX&TX values in "Pre-set maximums", however I'm not sure. In the kernel source code (e.g. intel ixgbe driver) the max RX buffer size is just a hard coded value in the source code (IXGBE_MAX_RXD in ixgbe.h), not sth. reported by the hardware. It's possible that all ixgbe series NICs supports the same buffer size so it's hard coded; it's also possible that the value 4096 is the max value that can be safely supported. In general I think we can trust these max values reported by ethtool.
