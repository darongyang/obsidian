## NVMe的优势
SATA+ACHI
PCIe+NVMe
NVMe和AHCI对比
SATA接口的SSD读性能最大不会超过600MB/s。
SSD时代，性能瓶颈转移到协议上。为SSD诞生，但可以用于低时延的新介质上。
NVMe协议是一个命令层和应用层层次的高级协议，一般跑在PCIe接口上。PCIe则是事务层、数据链路层和物理层的协议。
优势
- 优化约20%的延时（100纳秒到75纳秒）
- 高IOPS。IOPS=队列深度/延时。传统SATA队列深度32。而PCIe队列深度能达到128-256。AHCI队列数量为1。NVMe队列数量最大为64K。
## NVMe命令
Admin命令：队列管理、配置等
I/O命令：读、写等数据命令

## NVMe的队列和门铃
- SQ和CQ，在主机内存中
- DB，在SSD控制器中
基于NVMe和PCIe，SSD直接和RC相连，而RC和CPU相连。
基于队列和门铃的SSD数据传输流程
- 主机将I/O请求插入SQ
- 主机写SQ尾部的DB，通知SSD从SQ中取指
- SSD从SQ取出命令并执行
- SSD往CQ写入执行结果
- SSD发送中断通知完成
- 主机收到中断，处理CQ，查看完成状态
- 主机写CQ头部的DB，回复SSD
NVMe寻址
SSD主动负责从主机指定内存获取待写入的数据 或 将待读取的数据写入到主机指定内存。
主机告诉SSD指定内存的方式：PRP和SGL。
- PRP
将主机内存看作若干物理页，一个PRP对应一个物理页空间。PRP包括页面基地址和页内偏移量。PRP页内偏移量后两位是0，因此只能保持4字节对齐。
若干个物理页需要PRP链表，链表内的PRP的页内偏移量必须是0，而且不允许有相同的PRP。
每个NVMe命令都有两个PRP字段，PRP1和PRP2。这两个字段可能指向数据，也可能指向PRP链表。
Admin命令只能用PRP。
I/O命令可以PRP，还可以SGL。
- SGL
究竟是PRP还是SGL？看NVMe命令的`CDW0[15:14]`字段。