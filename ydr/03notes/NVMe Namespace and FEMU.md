经过femu时延分析，成功解决了改变介质时延而ocssd性能没有发生变化的bug。然而这次遇到的问题是改变SSD内部并行数（如channel数和chip数），ocssd的性能没有发生变化。定位到具体的原因在于`psl[i] = ns->start_block + (ppa << lbads);`，和NVMe namespace有关。
## 基本概念
**概念**。SATA SSD的一个闪存空间只对应一个逻辑空间。在NVMe中，一个闪存空间可以对应多个逻辑空间。每一个逻辑空间就是一个namespace，简称为ns。
**相关字段**。64字节NVMe命令的第4-7字节（从0开始计数）指定了要访问的ns。每个ns都有id`ns_id`和名称`ns_name`。如果将闪存空间划分为两个ns，大小分别为M个page和N个page，则其逻辑地址分别就为0~M-1和0~N-1。主机读写SSD需要同时指定ns和逻辑地址，否则会发生错误。识别命名空间数据结构包含报告命名空间大小、容量和利用率的相关字段：
- 命名空间大小 (NSZE) 字段定义了逻辑块 (LBA 0 到 n-1) 中命名空间的总大小。
- 命名空间容量 (NCAP) 字段定义了在任何时间点可以分配的最大逻辑块数。
- 命名空间利用率 (NUSE) 字段定义当前在命名空间中分配的逻辑块数。
**表现形式**。ns是由主机创建和管理的，在默认情况下，NVMe SSD 上的命名空间大小等于制造商确定的 LBA 大小。每个创建好的ns，表现为`/dev/`下的一个独立的块设备，在主机看来是两个独立的块设备（如/dev/nvme0n1 表示正在查看控制器#0的ns#1）。每个NS是独立的，可以用完全不同的参数配置。例如逻辑块大小等。
**共享与私有**。一个命名空间可以附加到两个或多个控制器，称为共享命名空间。相反，一个命名空间只能附加到一个控制器，称为私有命名空间。两者都由主机决定。
![[ns-controller.png]]

## FEMU有关代码
nvme.h中：
```c
typedef struct NvmeLBAF {
    uint16_t    ms; 
    uint8_t     lbads; 
    uint8_t     rp; 
} NvmeLBAF;
```
`NvmeLBAF`定义了NVMe Namespace的LBA格式，每个命名空间支持多个LBA格式。
- `ms`：元数据大小，字节数
- `lbads`：逻辑块的大小
- `rp`：相对性能（Relative Performance），用于描述不同 LBA 格式之间的性能差异。（有点怪，不确定）

```c
typedef struct NvmeRangeType {
    uint8_t     type;
    uint8_t     attributes;
    uint8_t     rsvd2[14];
    uint64_t    slba;
    uint64_t    nlb;
    uint8_t     guid[16];
    uint8_t     rsvd48[16];
} NvmeRangeType;
```
`NvmeRangeType`结构体定义了 NVMe 中的一个范围类型，用于描述命名空间的逻辑块范围。通常用于 `Dataset Management` 命令，用来指定哪些 LBA 是有效或无效的。
- `type`和`attributes`：逻辑块范围的相关属性。
-  `slba`：起始 LBA（逻辑块地址）。
- `nlb`：指定范围的块数。
- `guid`：全局唯一标识符 (GUID)。
- 其他字段是保留或未来扩展字段。

```c
typedef struct NvmeIdNs {
    uint64_t    nsze;
    uint64_t    ncap;
    uint64_t    nuse;
    uint8_t     nsfeat;
    uint8_t     nlbaf;
    uint8_t     flbas;
    uint8_t     mc;
    uint8_t     dpc;
    uint8_t     dps;
    uint8_t     nmic;
    uint8_t     rescap;
    uint8_t     fpi;
    uint8_t     dlfeat;
    uint16_t    nawun;
    uint16_t    nawupf;
    uint16_t    nacwu;
    uint16_t    nabsn;
    uint16_t    nabo;
    uint16_t    nabspf;
    uint16_t    noiob;
    uint8_t     nvmcap[16];
    uint16_t    npwg;
    uint16_t    npwa;
    uint16_t    npdg;
    uint16_t    npda;
    uint16_t    nows;
    uint8_t     rsvd74[30];
    uint8_t     nguid[16];
    uint64_t    eui64;
    NvmeLBAF    lbaf[16];
    uint8_t     rsvd192[192];
    uint8_t     vs[3712];
} NvmeIdNs;
```
`NvmeIdNs`结构体定义了 NVMe 标识命名空间时所使用的命名空间识别结构，通常用于 NVMe 的 `Identify Namespace` 命令返回信息。
- `nsze`：命名空间的总大小（以 LBA 为单位）。
- `ncap`：命名空间的最大容量。
- `nuse`：命名空间的已用空间。
- `nsfeat`：命名空间的特性标识，指示命名空间支持的特性。
- `nlbaf`：支持的 LBA 格式数量。
- `flbas`：默认的 LBA 格式选择。
- `lbaf[]`：LBA 格式数组，最多支持 16 种格式。
- 其他字段如 `nguid`（全局命名空间唯一标识符）、`eui64`（扩展唯一标识符）等是命名空间的标识信息。

```c
typedef struct NvmeNamespace {
    struct FemuCtrl *ctrl;
    NvmeIdNs        id_ns;
    NvmeRangeType   lba_range[64];
    unsigned long   *util;
    unsigned long   *uncorrectable;
    uint32_t        id;
    uint64_t        size; /* Coperd: for ZNS, FIXME */
    uint64_t        ns_blks;
    uint64_t        start_block;
    uint64_t        meta_start_offset;
    uint64_t        tbl_dsk_start_offset;
    uint32_t        tbl_entries;
    uint64_t        *tbl;
    Oc12Bbt   **bbtbl;
    /* Coperd: OC20 */
    struct {
        uint64_t begin;
        uint64_t predef;
        uint64_t data;
        uint64_t meta;
    } blk;
    void *state;
} NvmeNamespace;
```
`NvmeNamespace`结构体定义了 NVMe 的命名空间（Namespace），是一个逻辑分区，包含了该命名空间的标识符、逻辑分区范围、各种属性和状态。

和命令空间LBA格式有关的宏：
```c
// 从 flbas 字段的第 4 位提取标志位，用于判断 LBA 格式是否包含扩展信息。
#define NVME_ID_NS_FLBAS_EXTENDED(flbas)    ((flbas >> 4) & 0x1)

// 该宏提取 flbas 字段的低 4 位（bit 0-3），表示命名空间的默认 LBA 格式索引。
#define NVME_ID_NS_FLBAS_INDEX(flbas)       ((flbas & 0xf))

// 这两个宏用于获取命名空间 (Namespace) 中逻辑块寻址格式（LBA Format）的块大小信息
#define NVME_ID_NS_LBADS(ns) \
    ((ns)->id_ns.lbaf[NVME_ID_NS_FLBAS_INDEX((ns)->id_ns.flbas)].lbads)
#define NVME_ID_NS_LBADS_BYTES(ns) (1 << NVME_ID_NS_LBADS(ns))

// 获取命名空间的元数据大小 先获取默认地址格式 然后获取ms
#define NVME_ID_NS_MS(ns) \
    le16_to_cpu(((ns)->id_ns.lbaf[NVME_ID_NS_FLBAS_INDEX((ns)->id_ns.flbas)].ms))

// 获取lba_index对应的地址格式类型的块大小和元数据大小
#define NVME_ID_NS_LBAF_DS(ns, lba_index) (ns->id_ns.lbaf[lba_index].lbads)
#define NVME_ID_NS_LBAF_MS(ns, lba_index) (ns->id_ns.lbaf[lba_index].ms)
```

其他一些和ns有关的宏分析：
```c
// 该宏从 nsfeat 字段中提取最低位（bit 0）来确定命名空间是否启用了稀疏分配
#define NVME_ID_NS_NSFEAT_THIN(nsfeat)      ((nsfeat & 0x1))

// 从 mc 字段的第 1 位提取标志位，判断元数据是否独立存储
#define NVME_ID_NS_MC_SEPARATE(mc)          ((mc >> 1) & 0x1)

// 该宏从 mc 字段的第 0 位提取标志位，判断元数据是否嵌入在用户数据中。
#define NVME_ID_NS_MC_EXTENDED(mc)          ((mc & 0x1))

// 该宏从 dpc 字段的第 4 位提取标志位，判断是否支持最后 8 字节的保护信息。其余的宏类似，也是与保护信息相关。
#define NVME_ID_NS_DPC_LAST_EIGHT(dpc)      ((dpc >> 4) & 0x1)
#define NVME_ID_NS_DPC_FIRST_EIGHT(dpc)     ((dpc >> 3) & 0x1)
#define NVME_ID_NS_DPC_TYPE_3(dpc)          ((dpc >> 2) & 0x1)
#define NVME_ID_NS_DPC_TYPE_2(dpc)          ((dpc >> 1) & 0x1)
#define NVME_ID_NS_DPC_TYPE_1(dpc)          ((dpc & 0x1))
#define NVME_ID_NS_DPC_TYPE_MASK            0x7
```

oc12.c中（`oc12_read`和`oc12_write`）有关代码：
```c
// 获取该namespace默认的lba格式
const uint8_t lbaid = NVME_ID_NS_FLBAS_INDEX(ns->id_ns.flbas);
// 获取该lba格式对应的块大小和元数据大小
const uint8_t lbads = NVME_ID_NS_LBAF_DS(ns, lbaid);
const uint16_t ms = NVME_ID_NS_LBAF_MS(ns, lbaid);
uint64_t data_size = nlb << lbads;
uint64_t meta_size = nlb * ms;

// 获取物理地址ppa对应的实际存储的内存地址（单位字节）
psl[i] = ns->start_block + (ppa << lbads)
```

## 原因分析
- 为了实现传输数据，ocssd12将`req->slba`的数组从ocssd的ppa变成了内存地址
- 而时延模拟在传输数据结束后。在时延模拟时使用了修改后的`req->slba`（已经被改成了内存地址），导致了错误。正确的做法是需要将`req->slba`先还原为ocssd的ppa，再进行时延模拟。
```c
uint64_t * ppa_ocssd = (uint64_t *)g_malloc0(sizeof(uint64_t) * nlb);

for (i = 0; i < nlb; i++) {
	...
    ppa_ocssd[i] = psl[i]; // 寄存ppa
    psl[i] = ns->start_block + (ppa << lbads);
}
...
// 传输数据
backend_rw(n->mbe, &req->qsg, psl, req->is_write);

// 恢复ppa
for (i = 0; i < nlb; i++) {
    psl[i] = ppa_ocssd[i];
}
// 时延模拟
oc12_advance_status(n, ns, cmd, req);
```
## 参考资料
- 《深入浅出SSD：固态存储核心技术、原理与实战》
- <a href="https://nvmexpress.org/resource/nvme-namespaces/"> NVMe Namespaces | NVM EXPRESS</a>
-  <a href="https://www.ibm.com/docs/en/linux-on-systems?topic=nvme-namespaces"> NVMe Namespaces | IBM</a>