---
status: In Progress
---
- [x] 都是naive，为什么一个会触发gc，一个不会触发gc（gc_active不为1）
 区别在于：gc的时候是否可以选择sec_to_update的line 
- [x] core有哪些过程？开销占比分别如何？其他对比对象的I/O路径呢？
预读 - 解压缩 - 差量压缩 - 编码（成功）
预读 - 解压缩 - 差量压缩 - 编码（失败） - 压缩
预读 - 1压缩 - 编码 | 1写入-> CORE-Lossless
预读 - 2压缩 - 编码 | 1写入-> CORE-Delta2
预读 - 3压缩 - 编码 | 1写入-> CORE-Delta1
naive: 1235ns
core-overall: 233416ns
core-read: 15548 ns
core-compress: 86192ns
all:537657ns 
pre:25118ns 
de:23603ns 
d:224727 
en:159208 
co:201699ns
2预读 - 2编码 | 2写入-> WOM-v(2,4)
4预读 - 4编码 | 4写入-> WOM-v(1,4)
- [x] 为什么改变nand的时延，读写带宽反而变化不大？
改变fio的`--size`命令，发现SSD的速度差别很大，why？
可能还和`--iodepth`选项有关，都改成32看看？
black_block 模式呢？
调试一下femu的timing model。
发现是femu timing model的问题

- [x] 为什么fio触发不了decompress
随机写，发生更新的概率太低了！
将逻辑地址改小一点。
fio的逻辑地址为5%时，没跑成功
fio的逻辑地址为10%时，可以跑成功
- [x] 现有的实现顺序写，触发垃圾回收就会kernel null pointer，较小的size的随机写，触发垃圾回收也会kernel null pointer
- [x] 如何实现随机读写性能的测试和采集？
sudo apt install dstat
sudo apt install sysstat
- [x] 改进一下non-cyclic的实现
使用等价的版本即可
- [x] 预估一下4kb压缩的时延


### 验证读/写逻辑

- fio的verify选项
- 自己写py脚本
- 文件系统挂载测试

## womv14

STEP 1: 实现one-to-many FTL，简称OTM FTL。OTM FTL的读/写满足：
- 其l2p表是1对多映射，一个lba会对应多个ppa
- 每次写入的时候，会将一个lba的页面数据scatter，写入到多个ppa中
- 每次读取的时候，会将多个ppa读取出来，然后gather到一个lba页面中
- 当触发垃圾回收的时候，应该是以一个较大的pack（多个ppa）的方式一起进行gc的“读-写”

4字节：u32
8字节：u64
放大：WOMV_WA
- [x] 将用户可见容量缩小为1/4，pblk_capacity
- [x] 将l2p
- [x] 修改pblk_trans_map_get和pblk_trans_map_set，支持单lba，多ppa的多级l2p表查询。及其调用函数。
- [x] 写的时候分配多个物理页，读的时候读多个物理页
- [x] 修改pblk_invalidate_range，支持多ppa失效
- [x] pblk->capacity和DEVICE_CAPACITY，前者是设备逻辑地址空间，后者是设备的实际地址空间
- [x] pblk_update_map、pblk_update_map_gc和pblk_update_map_dev等要适配pblk_trans_map_get/set的修改。
- [x] pblk_lookup_l2p_seq、pblk_lookup_l2p_rand
- [x] pblk_rb_copy_to_bio、read_rq_gc、pblk_prepare_resubmit
- [x] 写缓存：地址拓展。pblk_w_ctx，围绕pblk_w_ctx的全部函数。
- [x] 后台罗盘：多分配几个
- [x] pblk_make_rq对应的失效、读、写函数


struct ppa_addr ppa;
pblk_ppa_set_empty(&ppa);
pblk_trans_map_set(pblk, lba, ppa, round);

1.将ring buffer的一个lba，拆分为WOMV_WA个entry
==2.每个entry，分配一个ppa==

(PBLK_EXPOSED_PAGE_SIZE / WOMV_WA)

以下几个函数修改了函数声明，添加了round，对应地方逻辑需要修改

修改l2p表的几个函数-从pblk_trans_map_set衍生的
pblk_update_map_dev
pblk_update_map_cache
pblk_update_map_gc

pblk_invalidate_range

pblk_trans_map_get
pblk_trans_map_set

TODO: 完善所有逻辑，检查落盘写是否正确。
read、gc、resubmit等逻辑

```
data (0) cur:1, left:32702, vsc:32702, s:32702, map:1/32768 (0)
// nrs, mem,  subm,   sync, l2p_update, ....
1024    0       0       0       0       0       NULL - 0/1023/0 - 0
```

- [x] 研究一个bug，
- [x] 添加打印l2p的逻辑进一步看是否正确，
- [x] gc的有效性检查要结合round
- [x] 将容量和并行单元改小一点，触发GC看看逻辑正确性
- [x] 将容量变大到16GB时，上述逻辑有问题，研究一下 -> 经典内存泄漏的bug
- [x] 启动的时候需要读，分析启动的读流程。定位问题发生在：pblk_read_check_seq。过早return了。
- [x] 为什么pblk_read_from_cache失败了。发现ppa传入错误。
- [x] 为什么cache怎么读都是相同的开头数据？：不是womv14的问题啊，debug了大半天，是fio脚本写错的原因
- [x] 对于所有的lba落盘都有:
```
pblk myblk: corrupted read LBA (0/1976)
```
![[Pasted image 20241217214035.png]]
- [x] 为什么seek是999？落盘lba是1009？fio脚本写错了
- [x] 为什么pblk_update_map_dev更新后是cache？好像是有的lba是cache内覆盖写
```
[  187.186487] pblk_update_map_dev, round:1
[  187.186487] ===========l2p table============
[  187.186488] [lba]: 78
[  187.186490] <ppa0> 968 (cache)
[  187.186491] <ppa1> 969 (cache)
[  187.186492] <ppa2> 970 (cache)
[  187.186493] <ppa3> 971 (cache)
```
- [x] 全部all in disk的情况好像没问题（不触发GC）
- [x] 测试临界区，half in disk，half in cache，好像也没有问题
- [x] 测试落盘读，有没有什么问题？
- [x] 还得测试混合情况，这是最麻烦的，好像是没问题

STEP 2: 实现WOMv FTL。WOMv FTL的读写满足：
- [x] 基于上述的OTM FTL
- [x] W：每次需要落盘写时，会将scatter的数据进行WOMv编码，写入到一个page中。
- [x] U：每次需要落盘写时，会获取旧的物理页，在旧的物理页面上写。如果有效，则继续用这个页面写（该line可能已经被关闭）；如果无效则重新分配一个新的页面（在活动的line上分配）。
- [x] 原来什么时候需要失效旧的物理页面？
	- [x] 原来在线写更新l2p表为cacheline时，如果旧地址是设备地址，会进行失效
	- [x] 原来进行rb驱逐的时候，如果条目已经失效，则会将分配给它的物理页失效
	- [x] 原来gc迁移是否要进行失效呢？答案是不需要的，因为失效页面的标记是为gc服务的。当已经进行GC了，失效页面就不必要了。
- [x] 新的逻辑？
	- [x] 在线写更新l2p表为cacheline时，如果旧地址是设备地址，不会将其失效，而是将其保存在w_ctx中
	- [x] 对该物理地址所在的line进行sec_to_update的引用，落盘写完的时候再释放，允许GC；如果这个line已经处于GC，则直接丢弃，重新分配一个。如何保证最后是用gc的地址，还是更新的地址呢？看谁先准入缓存：（1）更新比gc先进缓存。在gc写入缓存前，但凡发生了更新，都会把gc迁移的放弃掉。（2）gc比更新先进缓存。
	- [x] 什么时候要inc？（1）第一个entry握住ppa时inc？（2）后续的entry传递该ppa时，是否
	- [x] 这个页面写完了要atomic_dec一次，==如果判断满了不写这个页面了也要atomic_dec一次==；
	- [x] 进行rb驱逐的时候，如果条目已经失效，不将物理页失效
	- [x] 物理页失效的唯一时机交给：沿用该物理页进行全womv编码时，发生失败
- [x] R：相应的读的时候，要将读到的数据进行解码，然后再gatter到一个page中。

==当发生页面更新时，print出就地更新数据的大小==

## delta womv

目标：实现Firefly FTL，简称Firefly FTL。Firefly FTL满足：
- 将更新拆解为大更新和小更新，在l2p表分别记录lba的大更新数和小更新数
- 动态多级l2p，1个lba最多可能会对应最多5个ppa
- 基本的结构：base块+差量块
- base块采用动态编码等级，依据l2p的大更新数，决定编码方案
- 差量块也采用动态编码等级，核心在于小数据的多维编程
- 上述的编码方案会记录在l2p表中
- 第一次写入，写base块
- 后续写入，计算和base块的差量，聚合该差量，写入到预留页中
- 如果更新的差量太大，会根据大更新数进行，全WOMv编码（on-demand），根据历史lba更新次数，动态决定页面的编码方式。

**STEP 1：** 实现Elastic WOMv，记录lba的历史`global_update`次数，根据该字段选择不同的编码方式：WOMv(1,4)，或WOMv(2,4)，或不编码。（需要确定临界？）
- [x] 记录`global_update`次数。
- [x] 选择最优的编码方案，阈值如下讨论。约定编码等级：Non < WOMv(2,4)  < WOMv(1,4)。在所需要申请页面数相似的情况下，应该优先选择等级更低的编码维持更低的开销。
- [x] 记录所选择的编码方案在l2p表中。
**更具体实现：**
- [x] 修改和原来`DATA_PER_PAGE`有关的逻辑，即从single变为elastic
	- [x] 写逻辑修改
	- [x] 读逻辑修改
	- [x] 其他有关逻辑
- [x] 修改和原来`WOMV_WA`有关的逻辑，即从single变为elastic
	- [x] 写逻辑修改
	- [x] 读逻辑修改
	- [x] 其他有关逻辑
- [x] 注意释放program_type数组。
- [x] `./start.sh`的时候会不定时会产生Kernel NULL的bug（后续不会遇到了）
- [x] 发生bio截断读取的时候，会产生kernel BUG at bio
- [x] 重放mail.iolog的时候，会产生ppa为empty的bug
	- [x] 为entry维护一个后向指针
	- [x] entry从ring buffer中驱逐出的时候，看一下后续的entry获取到了自己的ppa没（也就是ppa是否为空），如果为空则传递给后续的entry
	- [x] firefly和womv同时都需要修改
- [x] 解决firefly遇到的内存耗尽的bug
- [x] 还要看触发gc后，是否会有bug（看起来没有）
- [x] 研究naive跑mail.iolog时，IO曲线的问题
	- [x] 触发GC了吗？
	- [x] or 为什么没有触发GC？
- [x] 梳理STEP2要做的点，缕清实现思路
pblk_trans_cnt_get
pblk_trans_cnt_reset
pblk_trans_cnt_inc

**讨论：** 假设一个LBA的更新次数是n，分析不同方案所需要申请的页面数。讨论三种方案，Non、WOMv(2,4)、WOMv(1,4)。
其有关信息如下：

| 指标   | Non | WOMv(2,4) | WOMv(1,4) |
| ---- | --- | --------- | --------- |
| 写入次数 | 1   | 5         | 15        |
| 放大系数 | 1   | 2         | 4         |


```python
def womv24(n):
    return 2 * (n // 5 + 1)

def non(n):
    return 1 * (n // 1 + 1)

def womv14(n):
    return 4 * (n // 15 + 1)
```

![[Pasted image 20241224202143.png]]


**STEP2：** Focus on 更新的特性，We present：**1.** 实现 (Base + Delta) WOMv；**2.** Furthermore，同时对于Base和Delta都采用Elastic的方法。(上述lba编程类型的变换类似于如下的状态转换机)
![[Pasted image 20250107204740.png | 600]]
- [x] 如果页面更新的比例小于==70%==，则计算新旧页的Delta，并将Delta存入Delta Page。如果Delta Page写满了，则保持Base Pages不变，重新分配一个新的Delta Page。
- [x] 否则直接对Base更新，增加其`global_update`次数。如果Base Pages写满了，则重新分配新的Base Pages。
- [x] 对于Base的更新，按照上述的on demand方法，自动选择最优编码方案。Goal：维持更低的开销。
- [x] 对于Delta的更新，按照elastic方法，贪心选择编码方案。Goal：尽可能在一个页面内存下Delta。
**更具体的实现**
- [x] 讨论x的阈值，什么时候存delta好，什么时候写base好？
![[Pasted image 20250107205247.png]]
- [x] 由于现在的实现是将lba的多个pages拆开，单独的发往ring buffer，单独的实现落盘。<u>因此在确定发往ring buffer前，要做好每一个lba的上述处理</u>。最后决定往ring buffer里写Base Pages，还是写Delta Page。如果是写Delta Page，需要计算出与Base Pages的新Delta，将其存在w_ctx中。
- [x] 同步修改和重构所有和pblk_lba_program_type有关的逻辑
- [x] 将写前读，添加到写请求一到来的时候（发往ring buffer前）。
	- [x] 如果在cache中，就从cache中拷贝。如果在disk中，就进行disk读。
	- [x] 恢复出每个lba的原始数据，统计新修改和原来的差值
- [x] 对新的页面和旧的页面进行差量压缩得到delta
- [x] ==如果开机晾的比较久，这个时候start的时候进行挂载，就会出现空指针错误然后kill掉==
	- [x] 可能和pblk_submit_read中not all in cache的没有处理empty页的有关
- [x] firefly和womv在跑homes的时候会寄掉
- [x] 如果delta rate小于0.7，则对该lba只往ring buffer写delta页，否则只往ring buffer写base页
- [x] 需要将每个lba从disk中读取到的原始未解码的数据存下来
- [x] 写入ring buffer的时候，存在w_ctx中
- [x] 仔细处理pblk_write_to_cache函数中的释放逻辑
- [x] `__pblk_rb_write_entry`这个将数据写入缓冲区的函数需要做到delta和base的感知
- [x] 进一步完成：gc写和用户写的做到delta和base感知的两个逻辑
	- [x] 调用上述函数前需要保证delta entry的delta_len被填充
- [x] 完成依据ring buffer中的base pages和delta page，实现lba写入的落盘逻辑
	- [x] 在pblk_submit_io_set里打印出w_ctx的信息
	- [x] 识别base pages，实现base pages的落盘逻辑：按照原来的依据lba_wa的方式对旧页面编码。过w_ctx直接进行使用，避免重复disk读
	- [x] 识别delta page，实现delta page的落盘逻辑：按照elastic的方式对旧的页面进行编码
	- [x] 不管base pages和delta page，两者的编码框架都是一致的，旧页面编码失效之后再申请新页面进行编码
	- [x] opt：如果delta_len为0，可以不发送I/O请求
- [x] ==实现对应的读取逻辑==
	- [x] 实现带有delta page的l2p表查询与截断，缓存命中读取逻辑
		- [x] pblk_lookup_l2p_seq：如果有差量page，额外多判断一个差量page在不在cache中。base + delta全部都在cache中，才算all in cache。返回bio的截断index。
		- [x] pblk_rb_copy_to_bio：从rb中读取base或delta时，会将其分别进行gatter/xor处理，复原出bio数据
		- [x] pblk_read_ppalist_rq：为调用pblk_lookup_l2p_seq，如果是all in cache，利用pblk_rb_copy_to_bio，实现从rb中读取所有的base和delta
		- [x] pblk_read_rq，参考pblk_read_ppalist_rq
		- [x] 
	- [x] l2p表的ppa读取，分base获取和delta获取
	- [x] 按照program type读取数据
	- [x] 如果是只有BASE，直接拼接
	- [x] 如果是还有DELTA，还额外需要解压缩
- [x] 一些致命的bug
	- [x] 如何缩小GC触发的时机？：rl->high和rl->rsv_blocks：rl->high = (rl->total_blocks / 100) * 95;
	- [x] 先确定gc是走到哪一步寄掉的
	- [x] 在gc_read的时候，出现了meta_lba为空的情况
	- [x] 在gc_write的时候，pblk_submit_io_set是怎么处理gc写入到rb的页面的
	- [x] GC触发的时候，那个神秘的错误出现了，ssh连不进去femu
- [x] 做一点调试
	- [x] 写入逻辑
		- [x] Base Pages Read: Base Pages的读取调试，是否有旧页面？
		- [x] Delta Get: 差量的计算调试
		- [x] Type Update: LBA的状态更新，旧状态&新状态
		- [x] RB Send: 写入rb采用了哪种方式？
		- [x] W_CTX Get: 落盘拿到的相关context
		- [x] Disk Send: 是否申请新的Page? 写Delta 还是Base
	- [x] 读取逻辑
		- [x] All Cache: all in cache读
		- [x] Partial Cache: not all in cache cache读 + disk读
	- [x] ==pblk_submit_write的bio alloc好像有点不太对，alloc多了一定页数出来？==

**STEP3：** 实现reset的逻辑。实现不同Base的invoke逻辑（忘了这是啥了）。


调试FLUSH：在`pblk_make_rq`函数中加入
```c
        if (!(bio->bi_opf & REQ_PREFLUSH))
            printk("gucao <%s> FLUSH\n", __func__);
```
调试bio_secs：在`pblk_write_to_cache`加入
```c
printk("gucao bio_secs: %d\n", bio_secs);
```


```c
* u32 *program_type; 
*? struct base_pages *lba_base_pages;
void **old_page_array;
void **new_page_array;
void **delta_page_array;
* u32 *delta_len;
* void * read_page;
* void * coding_page;
* void *data;
* u32 *lba_empty;
```




==测性能前要解决的问题：1.coding_page没释放的问题；2.femu读写性能一致的问题（理论应该读低于写很多）；3.delta_pages如果读过了要从写缓存传递到落盘写中。4.是否需要进一步缩小Trace。==

==GC越晚触发，GC落差越大，50%最好。==

![[Pasted image 20250220160402.png]]

![[Pasted image 20250220163702.png]]

![[Pasted image 20250220165019.png]]



naive-16x 加写前读：1w -> 2.8k
naive-1x 加写前读：120w -> 30s


## 6 Evaluation

我正在进行计算机存储系统领域的学术英语写作，写计算机顶会，有一些英语写作问题需要咨询你。主题是QLC NAND闪存。

回答问题：
Does Tetris outperform whole WOM-v in terms of performance and capacity across different workloads? (§6.2)
Does Tetris offer comparable or superior lifetime efficiency compared to whole WOM-v? (§6.3)
What are the critical techniques that enable Tetris to achieve these improvements? (§6.4)

### 6.1 测试环境

打算采用的trace：homes、mail、web、ssd-trace。
trace需要重新缩减。
测性能和测空间/寿命区分开
测空间/寿命可以用py脚本
测性能可以自己构造fio iolog或者用别的方式，待定

SSD给16GB，8 $\times$ 8的并行
内存dram给16GB吧

对比对象：
naive，womv24，womv14，ideal，Tetris



### 6.2 Efficiency Evaluation

### 6.4 Lifetime Optimization Assessment

![[6a02351b8468c34967d0e6e72b9a252.png]]


![[Pasted image 20250226235222.png]]

### 6.5 Tetris Features Analysis
**base on demand的分解** 

**delta on demand的分解**

**Cycle的编码方式**


