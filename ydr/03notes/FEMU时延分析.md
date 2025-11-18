起因是我改变nand.h里TLC的时延，发现ocssd 1.2的fio性能没有任何变化。我感觉应该是femu的ocssd1.2自身时延模拟的问题。我打算研究一下femu的时延模拟这个方面。

## femu论文
从 3.2 Accuracy 开始
### 时延实现
在femu中，数据传输和时延模拟相解耦。
- 数据传输：I/O请求到达时，femu会直接通过DMA方式完成读写的数据传输。
- 时延模拟：（1）传输数据的时候，femu会用计算出来的模拟时延$latency$来得到请求结束时间$T_{endio}$，并用$T_{endio}$标记这个I/O请求，将这个请求插入到按照$T_{endio}$作为优先级的endio队列中。（2）femu专门有一个线程用于处理endio队列，当队列中I/O请求所记录的完成时间$T_{endio}$大于等于现在时间$T_{now}$时，会不断从队列头部取出发送end-io中断。

### 时延模型
最核心的挑战在于计算每一个I/O请求的$T_{endio}$。
（1）基础的时延模型

每个平面只有一个页面寄存器，因此无法并行服务多个I/O，NAND页面读取、写入和传输的时间都是单值。
计算通道/平面的下一次空闲时间，考虑一个写请求到达通道#1和平面#1。
- 空闲通道的下一次空闲时间
$$ T_{freeC1} = T_{now} + T_{transfer}$$
- 空闲平面的下一次空闲时间
$$ T_{freeP1} = T_{freeC1} + T_{write}$$
- 本次写请求的结束时间
$$T_{endio}=T_{freeP1}$$
相同的通道和平面，而请求变为读请求。
- 空闲平面的下一次空闲时间
$$ T_{freeP1} = T_{now} + T_{read}$$
- 空闲通道的下一次空闲时间
$$ T_{freeC1} = T_{freeP1} + T_{transfer}$$
- 本次读请求的结束时间
$$T_{endio}=T_{freeC1}$$
注：论文此处的平面，在进一步并行化的现在，其语义应该是现在所说的芯片（即，Chips或Die）。
（2）高级的OC时延模型
OC指的是Open Channel SSD。作者只是讲了OC SSD的时延模拟所会遇到的一些问题和femu的效果，并没有讲述具体的实现。
- 不同1：OC的Plane使用双寄存器。
- 不同2：OC的页面延迟是不均匀的。（即，upper page和lower page的时延不同，映射不均匀）

## nvme.h
**有关结构体**
- `struct FemuCtrl`，整个femu的全局字段。
- `struct NvmeRequest`，每个请求的相关字段。
	- `int64_t stime`：请求的开始时间戳，表示何时开始处理该请求。
	- `int64_t reqlat`：请求的时延（request latency），用于记录该请求的延迟。**`int64_t gcrt`**: 全局完成时间戳（global completion runtime）。（这个字段在femu里没有用到？）
	- `int64_t expire_time`：请求的过期时间戳，用于时延模拟。（对应为论文中的$T_{endio}$）。
## nand.h 和 nand.c
通过宏定义定义了闪存颗粒的时延，如`#define TLC_LOWER_PAGE_WRITE_LATENCY_NS   (820500)`。
**有关结构体**
- `struct NandFlashTiming`，维护不同flash颗粒及其不同页面的读、写、擦除和传输时延。
**有关函数**
- `init_nand_flash_timing`
该函数在`init_nand_flash`时被调用，根据所定义的宏，完成上述结构体`struct NandFlashTiming`的填充。
- `get_page_read_latency`等函数
输入flash颗粒和页面类型，从上述结构体`struct NandFlashTiming`返回对应的时延。
## timing.c
**有关函数**
- 函数`set_latency`
该函数为全局的`FemuCtrl *n`设置时延有关的字段，如`n->upg_rd_lat_ns = ...`。
- 函数`advance_channel_timestmap`
该函数根据传入的当前时间`now`，通道编号`id`，和本次操作的类型`opcode`，以及当前通道的状态`n->chnl_locks[ch]`，来更新通道编号`id`的通道下一次可以使用的时间。目前这个函数直接返回了`now`。
但根据提供的剩下代码来看，其主要思路大致描述为：（1）获取当前操作的通道可用时间。如果当前的通道是可用状态（即，`now>=n->chnl_next_avail_time[ch]`），则直接可用（即，`start_data_xfer_ts=now`）；否则会等待到可用时间。（2）更新下次操作的通道可用时间。如果当前操作是`erase`操作，通道无需做页面传输，直接可用；否则为`read`或`write`操作时，还需要等待页面传输结束（即，`n->chnl_pg_xfer_lat_ns * 2`的时间）。
（疑惑：但是为什么要乘以2，这个不太能理解，可能是出于数据传输可靠性的考虑？）
- `advance_chip_timestamp`
思路类似于上述函数`advance_channel_timestmap`，根据操作的类型得到当前操作（读、写、擦除）的时延，然后和现在芯片的可用时间`n->chip_next_avail_time[lunid]`共同更新下一次芯片的可用时间。
注意：
- 函数`advance_channel_timestmap`和 `advance_chip_timestamp`仅在oc12.c和oc20.c被调用了，暂不清楚为什么，不知道其他SSD怎么模拟的时延。
- 这些通道和芯片时延的更新，都只是修改了全局结构体`struct FemuCtrl`的有关字段。

## oc12.c 和 ftl.c
**有关函数**
- 函数`oc12_advance_status`
这个函数是oc12的时延模型函数。按照上述时延模型，根据操作类型设置了`req->expire_time`。
- 函数 `ssd_advance_status`
这个函数是bbssd的时延模型函数。

## nvme-io.c
**有关函数**
- 函数`cmp_pri`、`get_pri`和`set_pri`
从这几个函数可以看出，在队列的维护中，以`expire_time`字段作为请求的优先级。结束时间越大，优先级越高。
nvme处理的poller`nvme_poller`：
- 函数`nvme_process_sq_io`
该函数是请求队列的处理函数。
**Step1:** 首先会从NVMe的请求队列取出I/O请求。
**Step2:** 然后初始化一个`NvmeRequest *req`，这其中包括这个`req`的`cmd_opcode`、`stime`和`expire_time`等字段，后两个时间都为现在的时间`qemu_clock_get_ns`。
**Step3:** 接着执行数据的传输：`nvme_io_cmd`->`n->ext_ops.io_cmd`->`nvme_rw`/`oc12_io_cmd`/`zns_io_cmd` -> `backend_rw`。不同的SSD分别实现了不同的数据传输的`io_cmd`（例如，没有FTL的`OCSSD`会在`oc12_io_cmd`里完成timing model，而`nvme_rw`则不会完成timing model，`BBSSD`会在自己的FTL里完成timing model）。
**Step4:** 将取出的req插入到femu的请求队列`n->to_ftl`中。
```c
int rc = femu_ring_enqueue(n->to_ftl[index_poller], (void *)&req, 1);
```
- 函数`nvme_process_cq_cpl`
该函数是完成队列的处理函数。
该函数中有几个重要的数据结构：`(FemuCtrl *)n->to_ftl[index_poller]`和`(FemuCtrl *)n->to_poller[index_poller]`、`n->cq[req->sq->sqid]`和`n->pq[index_poller]`。
**Step1:** 从环形缓冲区中取出请求，并插入到优先队列中。
`to_ftl`和`to_poller`是I/O请求的环形缓冲区，插入到优先队列前的一个过渡缓冲区。如果是黑盒模式`BBSSD`，则使用`to_poller`环形缓冲区；否则使用`to_ftl`环形缓冲区。
`n->pq[index_poller]`是I/O请求的优先队列，按照`expire_time`作为优先级。
**Step2:**  从优先队列依次取出请求，比较`req->expire_time`和现在的时间`now`，如果`now > req->expire_time`，则将`req`插入到完成队列`n->cq[req->sq->sqid]`中。如果超时太久（超过20000ns)，则进行记录。
**Step3:** 标记对应的完成队列要进行中断`n->should_isr[req->sq->sqid] = true;`，并发送中断`nvme_isr_notify_io`。

**为什么有两种缓冲区？**
只有在`BBSSD`中同时使用到了上述两种环形缓冲区，即只有`BBSSD`使用了`to_poller`环形缓冲区。在函数`ftl_thread`中有：
```c
static void *ftl_thread(void *arg){
	while ...
		// 从to_ftl中取出req
		rc = femu_ring_dequeue(ssd->to_ftl[i], (void *)&req, 1);
		// 处理操作，执行黑盒SSD的FTL维护，并计算该操作的时延
		switch (req->cmd.opcode) {
            case NVME_CMD_WRITE:
                lat = ssd_write(ssd, req); 
            ...
        }
        // 处理完成后，获得Tenio，插入到to_poller队列
        rc = femu_ring_enqueue(ssd->to_poller[i], (void *)&req, 1);
}
```

我觉得这是因为，只有`BBSSD`会有FTL表。首先，对于所有SSD，I/O请求都先会插入到`to_ftl`缓冲区中。对于白盒SSD，无需对这些请求进行额外的FTL操作。但对于`BBSSD`则需要将这些位于`to_ftl`缓冲区的取出，进行FTL操作后，再放入新的`to_poller`缓冲区中。最后这些已经完成的I/O请求才会被进一步进行真正的时延模拟。

## 原因分析
- ocssd的时延更新在结构体`static struct NandFlashTiming nand_flash_timing`中，该结构体是静态的啊。。。
- 原来该结构体声明在了nand.h中，而更新接口在nand.c中，获取接口在nand.h中
- 导致不同的.c文件都有一个静态的成员`nand_flash_timing`，其他.c文件调用nand.c的接口只改变了nand.c中独有的成员，调用nand.h接口查询的是自己独有的成员，结果查询结果全是0。。。
## Reference
- <a href="https://haslab.org/2021/05/03/femu-nvme.html"> FEMU/nvme源码分析 </a>
- <a href="https://www.usenix.org/system/files/conference/fast18/fast18-li.pdf"> Cheap, Accurate, Scalable and Extensible Flash Emulator </a>

