---
status: In Progress
---
 ```widgets
type: countdown
date: 2024-10-19 19:59:59
to: ASPLOS 2025 DDL
```

```widgets
type: quote
quote: Don’t let the current state limit your imagination
author: Devin Yang
```


#paper
- [x] 改成inspired by，allocating them on demand

## Title
CORE: Efficient <u>Co</u>mpression-Based Page <u>Re</u>programming Approach on High-Density Flash Memory

例如，写入 100 B 对象可能需要写入 4 KB 的 Flash 页，放大 40× 写入的字节数，并迅速磨损 Flash 设备。
Kangaroo: Caching Billions of Tiny Objects on Flash

## Introduction

曾经，压缩作为高开销的存在，一直谨言慎行。如今压缩专用硬件，例如IAA，QAT等加速方案的出现，使得曾经是大瓶颈的压缩，现在变得触手可得，其时延可以几乎不计。


我们的贡献：
- 我们观察到现有的内联压缩和页面编程存在各自的问题和结合的机遇，提出将两者相结合的关键见解
- 结合不是直接的，我们提出了高密度闪存上基于压缩的重编程方案：（1）二维重编程方案；（2）弹性编码方案；（3）新方案与压缩的结合。基于此，我们实现了高密度闪存上的无开销、无碎片的混合编程架构。
- 结果：我们发现xxx


background & motivation
  #paper
- [x] 前面background和motivation需要定义重编程的概念。
- [x]  前面backgound和motivation要讲不同编码的开销等

## Background
1.讲QLC-SSD及NAND Flash
2.讲重编程及如何实现
3.讲数据冗余的特性


### 2.1 High-density Flash Memory

caption：黑色的粒子代表注入的电子，蓝色的箭头代表注入的方向。逐步施加脉冲编程电压会不断提高cell的电压状态。

**编程特性**。图(ref)a展示了NAND闪存的编程原理。NAND闪存在编程的时候，会在Cell的Control Gate施加编程电压Vpp，将电子捕获在trapping layer。通过这个方式，存储单元的电压Vth会提高并保持在一定阈值从而能够表示相应的比特。图(ref)b展示了NAND闪存上的Incremental Step Pulse Programming (ISPP)方案。在电子的注入trapping layer的过程中，编程电压Vpp通常采用渐进递增的脉冲信号，每次会将Vpp提高dVpp，每次的脉冲时长为Tpulse，每次的验证时长为Tverify。通过ISPP方案，一个存储单元的电压Vth是单向递增的，并且能够从擦除状态逐步的到达最高电压Vmax。在高密度闪存上，一个存储单元的Vth(从擦除状态到Vmax)会被划分为多个电压区间。划分的区间数越多，存储单元的所能表示数据的密度就越大。将存储单元的电压按照上述的方式提升到指定的区间即可表示相应比特位的数据。NAND闪存以页面作为一次编程操作的基本单位。==编程的单位：page==   ==在高密度闪存上，多级电压特性，简单介绍，能够引出寿命问题即可==


**Programming Characteristics**. Figure \ref{fig:flash-pe}(a) illustrates the programming principle of NAND flash memory. When programming a NAND flash cell, the programming voltage ($V_{pp}$) is applied to the control gate of the cell, trapping electrons in the trapping layer\cite{liu2017qlc}. Through this way, the threshold voltage ($V_{th}$) of the cell will increase and be maintained at a certain level, thereby representing the corresponding bits. 

Figure \ref{fig:flash-pe}(b) illustrates the incremental step pulse programming (ISPP) scheme on NAND flash memory. During the process of injecting electrons into the trapping layer, the $V_{pp}$ typically uses progressively increasing pulse signals, with the $V_{pp}$ being increased by $\varDelta V_{pp}$ each time. Each pulse duration is $T_{pulse}$, and each verification duration is $T_{verify}$. Through the ISPP scheme, the $V_{th}$ of the cell increases unidirectionally and can gradually reach the maximum voltage ($V_{max}$) from the erased state. 

In high-density flash memory, the $V_{th}$ of a cell (ranging from the erased state to $V_{max}$) is divided into multiple voltage intervals. The more intervals, the higher the data density that the cell can represent. By increasing the $V_{th}$ to a specific interval, the cell will represent corresponding bits. NAND flash uses a page as the basic unit for each programming operation.

总结 1：在高密度闪存，cell的Vth划分为多个区间，通过Vth的状态来表示不同数据。

**Summary 1:** In high-density flash memory, the $V_{th}$ of a cell is divided into multiple intervals, representing different data by the state of the $V_{th}$.

 **擦除特性**。正如前面所述，在一次编程过程中，闪存存储单元的Vth是单向递增的。如果一个被编程过后的存储单元（例如Vth为V7），想表示更低的电压等级（例如V‘th为V4，但V4小于当前的V7，V'th表示新的电压状态）的数据（即意味着电压状态发生逆向的冲突），只有将这个存储单元进行擦除，让其Vth回到最低状态（即擦除状态），才能够重新将其编程到上述更低的电压状态（即V4）来表示新比特位。注意，如果新的比特位的电压状态不会发生冲突（例如V’th为V10），理论上无需将存储单元进行擦除。当存储单元的Vth达到Vmax，存储单元将无法再表示任何数据，只有执行擦除操作让才能够再次被编程(cite)。
每次编程的粒度为page，每次发生更新，上述cell的电压冲突是几乎无法避免。页面必须经过擦除才能重新编程。因此管理介质的程序通常采用异地更新的方式：当对旧的页面进行更新操作的时候，管理程序将原有的页面标记为无效页面，并重新分配一个空闲NAND页面来进行数据的重新写入。
擦除时延高于编程时延，擦除操作通常不会立马执行。通常等到空闲空间不足时（例如小于一定的阈值），管理程序会通过垃圾回收的方式完成擦除：在介质上，NAND闪存最小的擦除单位是块（页面的集合，大于页面的粒度）。管理程序会搬迁旧块中有效的页面到新块中，然后发起一次对旧块的擦除操作。在管理程序中，虽然最小擦除单位是快，出于高效，实际每次GC都会以擦除单元（若干块的组合）为基本单位进行。

**Erasing Characteristics.** As previously mentioned, during a single programming, the $V_{th}$ of a flash cell increases unidirectionally. If a programmed cell (e.g., with $V_{th}$ at $V_7$) needs to represent data at a lower voltage level (e.g., with  $V_{th}^{'}$ at $V_4$, where $V_4$ is less than the current $V_7$, and $V_{th}^{'}$ represents the new $V_{th}$), indicating a reverse voltage conflict, the cell must first be erased to return its $V_{th}$ to the lowest state (i.e., the erased state). Only then can it be reprogrammed to the lower voltage state (i.e., $V_{4}$ ) to represent the new bit value. Note that if the voltage state of new bits does not conflict with the current one (e.g.,  $V_{th}^{'}$ at $V_{10}$), there is theoretically no need for an erase operation. Once the  $V_{th}$ of the cell reaches  $V_{max}$, the cell can no longer represent any new bits and must undergo an erase operation before it can be programmed to represent new bits again **(cite)**.

Due to the characteristics that programming operates at the granularity of pages, the aforementioned voltage conflicts in the cells during updates are nearly unavoidable at the same time. Therefore, pages must be erased before they can be programmed again. To manage this, NAND flash controller typically employs an out-of-place update strategy: upon updating an old page, the controller marks the original page as invalid and allocates a new free NAND page to rewrite the data.

The erase latency is higher than the programming latency, and erase operations are not performed immediately\cite{cho2024aero}. Instead, they are delayed until free space is insufficient (e.g., below a certain threshold). NAND flash controller initiates a garbage collection (GC) to perform erase operations: In NAND flash, the basic erase unit is a block (a collection of pages). During GC, valid pages from an old block are migrated to a new block, followed by an erase operation on the old block. Although the basic erase unit is a block at the hardware level, for efficiency, the controller typically uses a larger erase unit known as a super block (consisting of multiple blocks) at the firmware level.


**总结2**：对cell的编程时电压是逐渐单向递增的，想表示更低的电压状态必须通过擦除来回到初始状态。通常会采用异地更新的方式，延后擦除操作进行集中擦除。

**Summary 2:** With gradually and unidirectionally increasing of $V_{th}$, the cell must return to the erase state to represent a lower state. Typically, the pages use out-of-place updates, delaying the erase operation to perform it in blocks.


**寿命挑战**。对Vth空间的划分的级别不是越多越好，会使得cell对vth的偏移更加敏感。每个P/E Cycle都会磨损图(ref)中的tunnel oxide，使得cell能束缚电子的能力下降，带来Vth的偏移。在相同的磨损条件下，更细粒度的Vth区间更容易发生数据错误。因此在保证数据不出错的条件下，更细粒度的Vth划分会让cell的tunnel oxide所能承受的P/E Cycles下降。图(ref)表明这种下降是急剧的，从SLC cells到QLC cells，骤降了几个数量级(cite)。在5Xnm下，闪存的P/E Cycles从 约11000下降到了约800。管理程序往往通过以下方式来应对寿命急剧下降的挑战：（1）优化纠删码的纠错能力来提高P/E Cycle上限，（2）尽可能降低擦除次数来延缓达到这个上限(cite)。

\noindent \textbf{Lifespan Challenge.} The multi-level division of the $V_{th}$ space is not necessarily better, as it makes cells more sensitive to $V_{th}$ shifts\underline{\textbf{(cite)}}. Each P/E cycle wears down the tunnel oxide (as shown in Figure \ref{fig:flash-pe}), reducing the cell's ability to trap electrons, which causes $V_{th}$ shifts. 
Therefore, to ensure data reliability, the multi-level division of $V_{th}$ reduces the number of P/E cycles that the cell's tunnel oxide can endure. Figure \ref{fig:lifetimeline} shows that this decline is drastic, with the P/E cycles of flash memory dropping by several orders of magnitude from SLC cells to QLC cells across different manufacturing processes \underline{\textbf{(cite)}}. In response to this significant reduction in lifespan, it is crucial, beyond optimizing error correction code capabilities to increase the P/E cycle limit, for SSD controllers and upper-level storage systems to minimize the total number of erase operations on high-density flash memory to delay reaching this limit \underline{\textbf{(cite)}}.

Under the same wear conditions, finer granularity in $V_{th}$ intervals leads to a higher possibility of data errors. Therefore, under the premise of data integrity, finer $V_{th}$ divisions reduce the number of Program/Erase (P/E) cycles that the cell's tunnel oxide can endure. As shown in Figure \ref{fig:lifetimeline}, this reduction is drastic, decreasing by several orders of magnitude from SLC cells to QLC cells \underline{\textbf{(cite)}}. In 5Xnm technology, the P/E cycles of flash memory drop from approximately 11,000 to around 800. To address the challenges posed by the rapid decline in lifespan, NAND flash controller typically adopts the following strategies: $(i)$ optimizing the error correction capabilities of error correction codes to raise the P/E cycle limit, and $(ii)$ minimizing the number of erase operations to delay reaching this limit \underline{\textbf{(cite)}}.

![[lifespan.png]]


总结3：存储密度越大，存储单元的寿命就越急剧地下降。

 **Summary 3:** The higher the density, the more severe the lifespan challenge of the NAND flash cells becomes.

### 2.2 Page Reprogramming

![[reprogramming.png]]
图片描述：NAND Flash页面上的多次编程方案。局部编程每次对闪存的一部分区域施加$V_{pp}$，以字节级别粒度进行编程。页面重编程则按照“读-编码-编程”的方式对整个页面重新施加$V_{pp}$，仍然按照页面级别粒度进行编程。编码将高比特位的编码数据用于表示低比特位的有效数据，以进行避免电压冲突的周期迭代。
caption: Multiple Programming Schemes. Partial programming applies $V_{pp}$ to a specific region of the flash memory, with programming performed at the *byte-level* granularity. Page reprogramming follows a "read-encode-program" process, applying $V_{pp}$ to the entire page again, still operating at the *page-level* granularity. Encoding uses the bits supported by the cell to represent valid low-bit data, in order to carry out multiple cycles.

  #paper
- [x] 看情况决定是否要补充局部编程和页面重编程的对比图 

**局部编程**。先前的工作发现闪存在SLC模式下能够很容易实现对页面的局部编程。局部编程每次只编程页面的一部分，通过多次编程填满页面。其编程粒度是字节，而不是传统的page。然而，将页面拆解进行编程的代价是，对页面中新的区域进行编程会对已编程区域带来干扰而引发数据错误。因此局部编程只能工作于SLC模式下。局部编程可以在SLC上实现同一个页面的多次编程（类似于页面就地更新的效果）。但这不适用于高密度闪存，且并非真正意义的就地更新。

**Partial Programming.** Previous work has found that partial programming can be easily performed on NAND flash memory in SLC mode. It can program only a portion of a page at a time, filling the page over multiple programming. The granularity of partial programming is at the *byte* level, rather than the traditional *page* level. However, the issue of dividing a page for partial programming is that programming new regions of the page can interfere with already programmed regions, leading to data errors. As a result, partial programming can only be used in SLC mode. Partial programming allows multiple programming on the same page in SLC mode (similar to the in-place page updates). However, this method is not applicable to high-density flash memory with finer granularity in $V_{th}$  and it does not represent true in-place updates.

![[part.png]]




**存储单元的WOM-v编码**。 正如前面所述，NAND闪存上密度的增加来源于对存储单元的Vth的空间的多级划分。Vth上的2n个电压状态最多能够带来n bits数据表示的能力（例如QLC Cell，4 bits数据表示得益于16个电压状态）。理论上，cell在到达vmax前的每一个电压状态均能够用于数据表示。一种充分利用这个特点的方式是WOM-v code(cite)。
图(ref)展示了在一个QLC cell上将4比特的编码用于表示2比特有效数据的WOM-v code。在QLC的15次升压过程中，2比特的有效数据被循环表示了5次。这种方式的思想是将n bits来表示更少比特位的数据（即数据编码）。这形成Vth的状态到有效比特位的多对一的映射。基于此，能够建立起一次Vth递增过程的多个数据周期。这样的好处在于，能够充分借助不同数据周期来完成数据的更新，避免电压冲突。
最终，一个cell所能够存储的实际数据量大于原本的n bits（如图xxx中，cell最终能够存储15 bits，大于原来的4 bits）。当闪存的密度越大，WOM-v编码带来的优化效果越好。


**WOM-v Code on Cell**. As previously mentioned, the increase in NAND flash density comes from the multi-level division of the $V_{th}$ space of the cells. The ${2}^{n}$ voltage states on $V_{th}$ can represent up to $n$ bits of data (e.g., a QLC cell can represent 4 bits of data due to its 16 voltage states).
Theoretically, each voltage state of a cell from the erased state to $V_{max}$ can be used to represent data. 
One way to fully utilize this feature is through the WOM-v code \underline{\textbf{(cite)}}. Figure \ref{fig:pg-re}(c) illustrates the WOM-v code used in a QLC cell, where 4-bit encoding represents 2 bits of valid data. During the 15 voltage steps of the $V_{th}$, the 2 bits of valid data are cyclically represented 5 times. The key idea behind this approach is to use *n* bits to represent fewer bits of data (i.e., data encoding). This creates a many-to-one mapping from $V_{th}$ states to valid bit values. Based on this, multiple data cycles can be established within a single $V_{th}$ increment process. The benefit is that it leverages different data cycles to perform data updates, thereby avoiding voltage conflicts and fully utilize the $V_{th}$. Ultimately, the actual amount of bits that a cell can store exceeds the original n bits. For example, as shown in Figure \ref{fig:pg-re}(c), the QLC cell can eventually store 15 bits, which exceeds the initial capacity of 4 bits. As the density of flash memory increases, the optimization effect brought by WOM-v code becomes more significant.


![[wom-v.png]]

**页面重编程**。基于上述的编码方式，对于一次页面更新，一个页面中所有的cell的电压都不会发生冲突。当页面更新满足$\forall cell \in page, V_{th}^{''}(cell) \geqslant V_{th}^{'}(cell)$(其中$V_{th}^{''}$表示更新后的$V_{th}$，$V_{th}^{'}$表示更新前的$V_{th}$)，控制器能够对这个闪存页面进行一次没有电压冲突的页面重编程。页面重编程充分利用了高密度闪存上$V_{th}$的多级划分的特性。页面重编程的编程粒度保持为一个页面，不会遇到局部编程的干扰错误问题。如图xxx(b)所示，页面重编程会读取旧页面的值，然后基于旧页面进行重新编码，最后以页面为粒度完成无电压冲突的页面重编程。值得注意的是，即使页面重编程每次都需要编程一整个页面，页面的实际修改量可能小于page的大小。经过页面重编程，未发生数据变化的cell，$V_{th}$编程并维持在到原区间，即满足$V_{th}^{''}(cell) = V_{th}^{'}(cell)$；而对于发生数据变化的cell，$V_{th}$编程并维持在到新区间，满足$V_{th}^{''}(cell) \geqslant V_{th}^{'}(cell)$，其中$V_{th}^{''}$表示更新后的$V_{th}$，$V_{th}^{'}$表示更新前的$V_{th}$。

**Reprogramming of Page**. Based on the aforementioned encoding method on cell, during a page update, no voltage conflicts will occur across all cells of the page. When a page update satisfies the condition $\forall \text{cell} \in \text{page}, V_{th}^{''}(\text{cell}) \geqslant V_{th}^{'}(\text{cell})$ (where $V_{th}^{''}$ represents the updated $V_{th}$ and $V_{th}^{'}$ represents the original $V_{th}$), the controller can perform a page reprogramming without voltage conflicts. Page reprogramming takes full advantage of the multi-level division of $V_{th}$ in high-density flash memory. The programming granularity remains at the *page* level, avoiding the interference and errors associated with partial programming. As shown in Figure xxx(b), page reprogramming involves reading the old page, encoding based on the old page, and performing the reprogramming at the page granularity without voltage conflicts.

Note that even though page reprogramming requires programming the entire page, the actual amount of modified data may be smaller than the page size. After page reprogramming, cells whose data remains unchanged will have their $V_{th}$ programmed and remain in their original interval, i.e., satisfying $V_{th}^{''}(\text{cell}) = V_{th}^{'}(\text{cell})$, where $V_{th}^{''}$ represents the updated $V_{th}$ and $V_{th}^{'}$ represents the original $V_{th}$. Conversely, for cells with altered data, the $V_{th}$ is adjusted and settles into a new interval, satisfying $V_{th}^{''}(\text{cell}) \geqslant V_{th}^{'}(\text{cell})$. Page reprogramming has been widely utilized in previous work~\cite{cui2023fast, jaffer2022improving}.

**Reprogramming of Page**.  Based on the aforementioned encoding method on cell, during a page update, no voltage conflicts will occur across all cells of the page. A non-conflicting page update is defined as Equation \ref{eq:reprogram}. Theoretically, a page update without voltage conflicts can enable the reprogramming of the page. Page reprogramming has been utilized in previous work~\cite{cui2023fast, jaffer2022improving}. Page reprogramming allows for the specific partial modification of a page by rewriting the entire page, provided that it does not violate the condition defined by Equation \ref{eq:reprogram}.

总结4：(i) 对于存储单元。在高密度闪存上，WOM-v利用Vth的多级划分且单向递增的特性对存储单元进行编码。(ii) 对于页面。经过编码的页面，能够在不带来电压冲突的前提下实现页面的重编程。

**Summary 4**:  $(i)$ For Cells. In high-density flash memory, WOM-v leverages the multi-level division of $V_{th}$ and its unidirectional increasing characteristic to encode storage cells.  $(ii)$ For Pages.  Encoded pages can be reprogrammed without causing voltage conflicts.




## Motivation

高密度闪存的寿命急剧下降。基于现实数据特性和独特的介质特性，这个问题能够得到很好的缓解。然而现有的工作还没有很好的同时把控好这两点。


1.讲现有SSD压缩方式的问题
- 页面利用率低
	- 多级电压未利用
	- 碎片化
2.讲WOM-v方式面临的问题
- 空间开销大 -> 一个逻辑页面映射多个物理页面
- 局部更新 -> 只能达到预期的xxx
3.机遇与挑战（思考一下这一章怎么写）
- 传统的compress只用考虑横向，无需考虑纵向覆盖影响
- womv的重编程空间放大，保持页面对齐，无需考虑页面内部的布局设计
- 不同的womv有着不同的空间放大，不同的更新次数
![[paper-problem.png]]

- [x] rewrite and divide the background and motivation
- [x] rewrite the introduction

早就有，制造商藏着，有技术缺陷，模拟
  #paper
- [x] 学习参考PR-SSD及其相关的工作
https://scholar.google.com/scholar?cites=14491331437910871887&as_sdt=2005&sciodt=0,5&hl=en



### 3.1 Page Redundancy

==放motivation==

- 先前的工作已经对SSD上的数据做了深入的分析。SSD上的数据具备较好的压缩性，在经典的trace上具备较好的更新特性，更新的数据具备相似的特性。

描述：在图（a）中，图例表示不同更新次数的区间，横轴表示落入不同区间内页面数量的占比。在图（b）中，压缩率通过压缩后的数据大小除以压缩前的数据大小（即页面大小）来表示。
In Figure (a), the legend represents the intervals of different update counts, and the horizontal axis indicates the proportion of pages falling within each interval. In Figure (b), the compression ratio is represented by the size of the compressed data divided by the size of the uncompressed data (i.e., the page size).

SSD中存储的数据以page为单位来进行读写。在现实负载中，这些以较粗粒度（即page）组织的数据具备较大程度的冗余特性。在高密度闪存中，为了应对L2P表占用内存的增加问题，通常会采用更大的页面（如64KiB）。在更大的页面中，数据冗余程度会进一步增大。

In SSDs, data is read and written in units of pages. Data organized at this relatively coarse granularity (i.e., *page* level) exhibits a significant degree of redundancy in real-world workloads. In high-density flash memory, larger pages (e.g., 64 KiB) are typically used to address the issue of increased memory consumption by the L2P table. In these larger pages, the degree of data redundancy further increases.

可压缩性。现实的数据呈现很好的可压缩性。图 xxx(b)展示了不同种常见类型的数据在SSD的4KB大小的页面上的压缩率分布情况。这些页面的压缩率围绕总体压缩率呈现着正态分布。大多数数据集能达到50%以下的压缩率，在一些压缩性好的数据集下（例如Database）能够达到20%的压缩率。

**Compressibility.** Real-world data organized in pages on SSDs exhibits a good compressibility distribution. Figure \ref{fig:data-feature}(b) illustrates the compression rate distribution of several typical types of data on 4KB SSD pages. The compression ratios of these pages exhibit a normal distribution around the overall compression ratio. Most datasets achieve a compression ratio below 50%, and for some highly compressible datasets (e.g., Database), the compression ratio can reach as low as 20%.

更新与相似性。在高密度闪存上，由于粗粒度的页面，页面数据往往会发生相似性较高的页面更新。首先，在随机写入场景中，数据的热更新占比很高。图 4a展示了典型的现实世界负载下在SSD的4KB大小的页面上发生数据的更新操作的频数的占比分布。在这些负载中，超过一半的页面的会发生频繁的数据热更新。近年来，SSD上冷热数据分类概念被逐渐提出（cite ZNS FDP）。在热数据存放的区域，数据更新会更加集中和频繁。

\noindent \textbf{Update and Similarity.} In high-density flash memory, due to the coarse granularity of pages, page data often experiences updates with high similarity. Firstly, in random write scenarios, a significant proportion of data undergoes frequent hot updates\cite{zhang2016reducing}. Figure \ref{fig:data-feature}(a) shows the distribution of update frequencies on 4KB SSD pages under typical real-world workloads\underline{\textbf{(cite)}}. In these workloads, more than half of the pages experience frequent hot updates. In recent years, the concept of hot and cold data classification on SSDs has been gradually introduced (cite ZNS FDP). In regions where hot data is stored, data updates are more concentrated and frequent.

在高密度闪存上，由于粗粒度的页面，页面数据往往会发生相似性较高的页面更新。近年来，SSD上冷热数据分类概念被逐渐提出（cite ZNS FDP）。在热数据存放的区域，数据更新会更加集中和频繁。

第二，小数据的热更新会带来较大的写放大。这是由于“读-修改-写”的更新模式和SSD上的异地更新特点导致的。在数据库和文件系统中，元数据的修改仅占4KB页面的8%左右（写入放大为12.5x）(cite)。在基于B+ tree的存储系统中，记录的更新通常只占4KB页面的2.5%左右（写入放大为40x）(cite)。在去重系统中，50%去重率带来的元数据写放大能够达到20x(cite)。在高密度闪存的大页面（e.g., 16KB）的趋势下，小数据的更新带来的写放大问题会进一步凸显。

Secondly, page data often experiences updates with high similarity with updates of small data. Hot updates of small data lead to significant write amplification. This is due to the "read-modify-write" update mode and the out-of-place update strategy on SSDs. For example, in database and file system scenarios, metadata modifications typically occupy only 8\% of a 4KB page (resulting in a write amplification of 12.5x) \cite{li2015much,zhang2016reducing}. In $B^+$-tree-based storage systems, record updates usually occupy only about 2.5\% of a 4KB page (resulting in a write amplification of 40x) \cite{qiao2022closing,huang2023breathing}. In deduplication systems, a 50\% deduplication rate can lead to metadata write amplification reaching 20x \cite{wen2024eliminating}. With the trend toward larger pages in high-density flash memory, the issue of write amplification caused by page similarity becomes even more pronounced.

![[redundabcy.png]]

总结 5：在高密度闪存中，粗粒度的页面管理带来了较大程度的页面冗余。页面内部存在数据冗余的同时，页面间的数据也存在相似性。高密度闪存上大页面化的趋势，进一步加剧这种页面冗余。

In high-density flash memory, coarse-grained page management results in a significant degree of page redundancy. %% While there is data redundancy within pages, there is also similarity between pages.  %%The trend toward larger pages in high-density flash memory further exacerbates this page redundancy.

### 3.2 Limitations of Existing Inline Compression

==这里疯狂引用CSD有关的工作。==

![[utilization.png]]

描述：图中展示了SSD内部页面级压缩在平均情况和最差情况对页面的利用率（即x轴）。测试采用了经典的可压缩数据集。MIX表示混合负载。Basic表示页面的基本大小，并以此作为基准。Ideal是4KB页面在QLC介质下基于Vth的复用所能够达到的最大数据表示能力。

Page Utilization Rate in Inline Compression. The figure shows the page utilization (i.e., the x-axis) in SSDs under page-level compression in both average and worst-case scenarios. We use typical compressible datasets in Figure xxx. MIX represents mixed workloads. Basic refers to the baseline page size, which serves as the reference. Ideal represents the maximum data representation capacity of a 4KB page on QLC media, based on $V_{th}$ reuse.

内联压缩。基于上述的页面冗余特性，数据压缩被广泛的运用于SSD上。在页面内部冗余方面，得益于压缩专用计算单元，支持内部无损压缩的固态硬盘已经商用。这些工作（疯狂cite）提出了通过对闪存转换层（FTL）进行改造来将无损压缩内联到固态硬盘中(cite)。通过这种方式，一个物理页面通常可以存放多个逻辑页面，从而减小SSD上的写入流量。在页面间相似性方面，早期的工作delta-FTL提出在页面更新时仅保留和维护更新前后页面数据的差量。后续的工作借助SLC上的部分编程特性，进一步提出将无损压缩和差量压缩相结合来实现页面内的追加更新。通过这种方式，页面内部的热数据可以被分离，从而减少不必要的重复写入。

\textbf{Inline Compression.} Based on the aforementioned page redundancy characteristics, data compression is widely utilized in SSDs. In terms of internal page redundancy, SSDs that support internal lossless compression have been commercialized, thanks to dedicated hardware computation engines for compression. These works\cite{park2011zftl} proposed modifying the logical-to-physical table (L2P table) to integrate lossless compression within the SSDs. Through this method, a physical page can accommodate multiple logical pages, thereby reducing the writes on SSDs. In terms of inter-page similarity, delta-FTL\cite{wu2012delta} proposed maintaining only the differences between pre-update and post-update page during updates. Subsequent work\cite{zhang2016reducing} leveraged the partial programming characteristics on pages in SLC mode to further combine lossless compression and delta compression, enabling in-page append updates. Through this method, hot data within a page can be separated, thereby reducing unnecessary repeated writes to the remaining data.


**局限1：电压的低效利用**。 内联压缩相对于其他层级（如文件系统、块设备层）的压缩具备介质特性感知的优势。然而，现有的工作忽视了这一点。基于传统的数据表示方式，现有的内联压缩在高密度闪存的Vth的多级利用上是低效的。如图xxx所示，以一个QLC页面为例，在最理想的情况下，这些方案可以达到100%的页面利用（即图五中的Basic，页面的实际大小）。然而这和理论上一个QLC页面所能够达到的最大利用率375%还有很大的差距（即图五中的Ideal，页面实际能够表示的数据的大小）。

**Limitation 1: Inefficient Utilization of $V_{th}$.**  Inline compression has the advantage of being media-aware compared to compression at other layers, such as the file system or block device layers. However, this has been overlooked in existing work. Based on traditional data representation methods, current inline compression schemes are inefficient in utilizing the multi-level $V_{th}$ characteristics of high-density flash memory. As shown in Figure xxx, using a QLC page as an example, these schemes can achieve 100% page utilization under optimal conditions (i.e., the actual page size shown as "Basic" in Figure 5). However, this is still far from the theoretical maximum utilization of 375% that a QLC page can achieve (i.e., the actual data size that can be represented, shown as "Ideal" in Figure 5).


**局限性2：页面内部碎片化**。无论是内联无损压缩还是内联差量压缩，都需要在一个页面内放入多份变长的数据。随着剩余空间的不断减少，页面内所能够继续放入的页面大小随之减少。这会不可避免的带来页面内部空间的碎片化问题。如图xxx所示，对不同负载进行页面级的无损压缩，在平均情况下，页面大概有20%的未利用空间（这和负载的压缩比有关）。在最坏情况下，页面能够达到80%以上的未利用空间。碎片化问题导致内联压缩对于页面的实际利用率小于前面讨论的Basic，进一步拉大了和Ideal的差距。


**Limitation 2: Intra-page Fragmentation.** Both inline lossless compression and inline delta compression require placing multiple variable-length data segments within a single page. As the remaining space decreases, the size of the data segment that can be placed within the page also shrinks. This inevitably leads to intra-page fragmentation. As shown in Figure xxx, for typical workloads undergoing page-level lossless compression, there is roughly 20% unused space on average within a page (depending on the compression ratio of the workload). In the worst case, the unused space can exceed 80%. The fragmentation issue causes the actual page utilization of inline compression to fall below the previously discussed *Basic* level, further widening the gap from the *Ideal* scenario.

动机2：如图6(a)所示，在高密度闪存上，基于传统的数据表示，现有的内联压缩在$V_{th}$的利用上是低效的，使得页面的利用率远低于最大理论值。此外页面内部碎片化问题进一步拉大了这一差距。


**Motivation 2:** As shown in Figure 6(a), on high-density flash memory, based on traditional data representation, existing inline compression works are inefficient in utilizing multi-level  $V_{th}$, resulting in page utilization rates far below the maximum theoretical value. Additionally, the issue of intra-page fragmentation further widens this gap.


![[challenge1.png]]
![[challenge2.png]]`



### 3.3 Shortcomings of WOM-v Page Reprogramming

理论分析。假设一个cell最大能够存储$n$ bits，而通过WOM-v编码后，其用于表示$n^{'}$ bits数据。考虑一个cell在一个P/E Cycle内的数据表示能力，$k$代表cell的数据周期数，$total$代表cell能够表示总的有效数据比特数，$overhead$代表编码的空间开销。由于不同数据周期（cite）具备共享编码，k个数据周期会带来k-1个共享编码。容易得到数据周期k，总比特数total和空间开销overhead满足式子xxx的条件。整理后得到式子xxx。

**Theoretical Analysis.** Assume that a cell can store a maximum of $n$ bits, and after applying WOM-v encoding, it is used to represent $n^{'}$ bits of data. Considering the data representation capacity of a cell within a single P/E cycle, let $k$ represent the number of data cycles for the cell, $total$ represent the total number of valid data bits that the cell can store, and $overhead$ represent the encoding overhead. Since different data cycles share encoding (cite), the $k$ data cycles result in $k-1$ shared encodings. It can be easily deduced that for $k$ , $total$ and $overhead$ of a cell satisfy the conditions described by Equation xxx. After simplification, Equation xxx is obtained.

$$\left\{ \begin{array}{l} 2^{n'} \cdot k - (k-1) \le 2^{n} \\ \text{total} = n' \cdot k \\ \text{overhead} = \frac{n}{n'} \\ n \in \mathbb{Z}^+, \, n' \in \mathbb{Z}^+, \, 1 \le n' < n \end{array} \right.$$

$$\left\{ \begin{array}{l} k = \left\lfloor \frac{2^{n} - 1}{2^{n'} - 1} \right\rfloor \\ \text{total} = n' \cdot \left\lfloor \frac{2^{n} - 1}{2^{n'} - 1} \right\rfloor \\ \text{overhead} = \frac{n}{n'} \\ n \in \mathbb{Z}^+, \, n' \in \mathbb{Z}^+, \, 1 \leq n' < n \end{array} \right.$$

**I/O 放大**。从式子xxx中容易得到，当$n^{'}=1$时，total能够取得最大值$2^n-1$。在nb/cell的高密度闪存上，为了实现页面的最大化利用（即total取得最大值），读写开销(即overhead)会变为原来的n倍。例如，在QLC闪存上，需要将原来的4比特表示1比特的有效数据，overhead变为原来的4倍。这意味着每对一个来自host的逻辑页进行读写操作，实际上都要对4个QLC的物理页进行读写操作。这会让QLC闪存在空间上退化为SLC，且在性能上还远不如SLC（一个QLC页面的编程/读取时延远低于一个SLC页面，且WOM-v还需要对四个QLC页面进行编程/读取）。

**I/O Amplification.** From Equation xxx, it is clear that when $n^{'}=1$, the *total* can reach its maximum value of $2^n-1$. In $n$b/cell high-density flash memory, in order to maximize page utilization (i.e., when the *total* reaches its maximum), the read/programming overhead (i.e., *overhead* in Equation xxx) increases by a factor of $n$. For example, in QLC flash memory, where 4 bits are used to represent 1 valid bit, the overhead becomes 4 times the original. This means that for every read/programming operation on a logical page from the host, 4 physical QLC pages must actually be read/programmed. As a result, QLC flash memory effectively degrades to SLC in terms of space, but with significantly worse read/programming performance (since the read/programming latency of a QLC page is much higher than that of an SLC page, and WOM-v requires read/programming across four QLC pages).

公式的推论
- k的特性。始终有$k \ge 2$。随着n'的减少而增大。
- total的特性。当$n^{'}=1$时，total取得最大值$total_{max}=2^n-1$；当$n'=n-1$时，total取得最小值$total_{min} \approx 2(n-1)$

==公式的规范性==

**局部磨损**。正如前面所说，基于页面的重编程，页面能够对局部内容进行更新。然而，更新操作往往具有就地更新的局部特性（例如，文件系统的元数据倾向于就地覆写某些字段的旧值）。对这些页面直接采用WOM-v编码进行重编程时，会导致页面不同区域的$V_{th}$提升不均衡，带来页面的局部磨损（为了直观和便于理解，我们将$V_{th}$的提升类比并称为$V_{th}$的磨损）。如图xxx(b)所示，逻辑页的局部区域发生频繁更新导致其电压过早达到$V_{max}$，而其他区域电压还远没达到$V_{max}$。当磨损区域达到最大电压$V_{max}$后，再对该区域发生更新操作时，页面的重编程将失败。在高密度闪存上，随着页面的变大，局部磨损带来的浪费会更加严重。

**Local Wear.** As mentioned previously, pages can update local content based on page reprogramming. However, updates always exhibit in-place overwriting characteristics (e.g., file system metadata tends to overwrite old values in specific fields). When directly reprogramming these pages using WOM-v encoding, it leads to uneven $V_{th}$ increases across different regions of the page, causing local wear (For intuitive and easy understanding, we analogize the $V_{th}$ increase to $V_{th}$ wear). As shown in Figure xxx(b), frequent updates to specific regions of a logical page result in the $V_{th}$ in those areas reaching $V_{max}$ prematurely, while other regions are still far from reaching $V_{max}$. Once the worn-out regions reach the $V_{max}$, any further updates to those areas will cause the page reprogramming to fail. In high-density flash memory, as the page size increases, the waste caused by local wear becomes more severe.

动机3：对页面直接使用WOM-v编码会造成不可避免的I/O放大，进而带来存储密度的退化和性能严重下降的问题。此外，由于更新的空间局部性，页面会发生局部磨损，导致页面的$V_{th}$没有被充分利用。

**Motivation 3:** As shown in Figure 7(b), directly applying WOM-v Code to pages leads to inevitable I/O amplification, which results in storage density degradation and severe performance loss. Additionally, due to the spatial locality of updates, pages experience local wear, preventing the full utilization of the $V_{th}$ levels.

描述：不同方案的页面存储。红色箭头体现了页面存储宏观到微观的过程。x轴代表不同的cell，而纵轴代表$V_{th}$。不同的颜色的柱形代表不同的数据周期。弧形箭头表示数据周期的迭代。

**Page Storage in Different Schemes.** The red arrows represent the page storage from a macroscopic to a microscopic view. The x-axis represents the cells in pages, while the y-axis represents $V_{th}$. The different colored bars represent different data cycles, and the curved arrows indicate the iteration of data cycles.



### 3.4 Opportunities and Challenges

![[compare.png]]

==如何说服：两个问题==

机遇1：数据重编程有助于内联压缩，解决了页面内部碎片化和最大限度利用Vth。

**Opportunity 1:** *Page reprogramming facilitates inline compression, solving intra-page fragmentation and maximizing $V_{th}$ utilization*

解决页面内部碎片化。现有内联压缩产生碎片化的原因是：随着数据的逐渐放入，页面的可用空间会不断的减少，无法放入新的数据而产生碎片。然而，页面重编程改变了这一现状。基于页面重编程，对于放入的任何新数据，在其视角下，页面的可用空间保持为整个页面，不会因可用空间的减少带来碎片化。如图xxx(c)所示，对于新输入的数据e，即使横向上的剩余空间已经无法存放该数据，但利用页面重编程可以对旧的无效数据进行适当的覆写，实现无碎片化的迭代存储。基于这种方式，只要输入数据的大小小于页面的大小，都可以对其进行无碎片的数据存放。

**Solving Intra-page Fragmentation.** The cause of intra-page fragmentation in existing inline compression schemes is that as data is gradually stored in page, the available page space continuously decreases, leaving insufficient space for new data and causing fragmentation. However, page reprogramming changes this situation. Based on page reprogramming, for any newly inserted data, the available page space from its perspective remains the entire page, preventing fragmentation due to reduced available space. As shown in Figure xxx(c), for new incoming data *e*, even if the remaining space horizontally is insufficient to store this data, page reprogramming can appropriately overwrite old invalid data, enabling fragmentation-free iterative storage. Using this approach, as long as the size of the input data is smaller than the page size, it can be stored without fragmentation.

充分利用$V_{th}$。在高密度闪存上，基于页面重编程，内联压缩能够充分发挥页面的每一级的$V_{th}$，将页面的利用率最大化（趋近于图xxx的$Ideal$）。

**Maximizing $V_{th}$ Utilization.** In high-density flash memory, based on page reprogramming, inline compression can fully utilize each level of the page’s $V_{th}$, maximizing page utilization (approaching the "Ideal" shown in Figure xxx).

同时能让压缩发挥闪存最大的数据表示能力。

机遇2：内联压缩有助于数据重编程

**Opportunity 2:** *Inline compression facilitates page reprogramming.*

屏蔽开销。直接对页面做WOM-v编码会导致不可避免的I/O放大。其中，大量的冗余数据也被相应放大。 我们观察到，在式子xxx中，对于$n$或$n^{'}$，总的数据表示total呈现指数级增长，而开销仅是线性级增长。 当n值较小时，线性的开销可以通过编码前压缩进行很好的屏蔽。 例如，QLC上的WOMv编码，编码的开销为原来的1.3倍到4倍。这能很容易地被章节xxx所提及的页面压缩率所屏蔽。和直接对页面做WOM-v编码对比，通过编码前压缩，能够避免对无效的冗余数据进行编码，让WOM-v code聚焦于有效数据。 

**Mitigating Overhead.** Directly applying WOM-v encoding to a page results in inevitable write amplification. However, based on page redundancy, pre-encoding compression can mitigate the space overhead effectively. We observe from Equation xxx that for $n$ or $n^{'}$, the total data representation (i.e., *total*) grows exponentially, while the overhead increases only linearly. For instance, WOM-v code on QLC incurs an overhead of 1.3x to 4x, and on PLC, the overhead ranges from 1.25x to 5x. As discussed in Section xxx, pages exhibit redundancy, with lossless compression rates within a page reaching 50%, or even below 20%, and inter-page differential compression rates as low as 8% or less. Compared to directly encoding a page with WOM-v code, pre-encoding compression avoid unnecessary encoding of redundant data allowing WOM-v code to focus on the valid data that requires updating. This approach easily masks the linear space overhead in exchange for exponential growth in data representation.

**Mitigating Overhead.** Directly applying WOM-v code to a page results in inevitable I/O amplification, with a large amount of redundant data is also amplified. We observe from Equation (\ref{eq:womv2}) that for $n$ or $n^{'}$, the total data representation grows exponentially, while the overhead increases only linearly. When the value of $n$ is small, the linear overhead can be effectively mitigated through pre-encoding compression. For example, WOM-v encoding on QLC incurs an overhead of 1.3x to 4x, which can easily be masked by the page compression ratios mentioned in Section \ref{sg:pg-redun}. Compared to directly encoding the entire page with WOM-v, pre-encoding compression avoids encoding unnecessary redundant data, allowing WOM-v to focus solely on valid data.

解决局部磨损。页面局部磨损的原因在于：在粗粒度页面内，（1）不同区域的数据冷热不均衡，且（2）其页内存储位置固定，从而导致某些区域被过度磨损。然而，内联压缩能够充分解决这个问题。内联压缩（如无损压缩，差量压缩）生成小于页面的有效数据（即，没有冗余数据）。然后这些小于页面的数据可以通过重编程改变其页内的存储位置。最后通过电压均衡提升的方式，能够避免页面的局部磨损。此外差量压缩还能对页面内部小的热数据进行提取，并做冷热分离，进一步优化局部磨损。如图xxx（c）所示，这些压缩数据能够以循环迭代的方式存储，使得页面上所有存储单元的电压能够以一个相同的电压区间（对应不同的数据周期）进行步进式的递增。

**Solving Local Wear.** Local wear in a page is caused by: (1) uneven distribution of hot and cold data across different regions of a coarse-grained page, and (2) fixed storage locations within the page, leading to excessive wear in certain areas. However, inline compression can effectively address this issue. Inline compression (e.g., lossless compression, delta compression) generates data smaller than a full page (i.e., without redundant data). This smaller data can then use page reprogramming to change its storage location within the page, and by evenly increasing the $V_{th}$ across the page, local wear is avoided. Additionally, delta compression can extract small hot data segments within the page and separate hot and cold data, further optimizing the local wear. As shown in Figure xxx(c), these compressed data can be stored in a cyclic iterative way, allowing the voltage of all storage cells within the page to increase incrementally within the same voltage range (corresponding to different data cycles).

%% 打破WOM-v编码对页的低效方式，让WOM-v编码聚焦更新的数据。 %%
![[paper-fail.png]]

描述：页面存储多份有效数据。输入页面的第n个有效数据，表示为$D_n$；其元数据表示为$M_n$。

Page stores multiple valid data entries. The _n_-th valid data segment of an input page is denoted as $D_n$, and its corresponding metadata is denoted as $M_n$.




内联压缩和页面重编程相结合能带来双方问题的解决。然而，这个结合并不是一蹴而就，需要考虑页面重编程的特性现有的方法忽略了这些因素，导致他们并不适用于页面重编程。我们观察到，页面的重编程空间具有如下的特点：（1）写入数据放大；（2）数据纵向覆盖。



**传统空间策略。** 在==一个页面==中，传统的方法采用的是一对多的策略: 倾向于在一个页面中存放多份有效数据以利用满所有横向的空间。这些一对多的策略可能包括：（1）存放多份独立的数据（即$data_{A}$, $data_{B}$，...，$data_N$）；（2）存一份复合数据的多个子数据（即构成$data_{A}$的$data_{A0}$, $data_{A1}$，...，$data_{An}$，其中$data_A=data_{A0}+data_{A1}+...+data_{An}$）。

**Traditional Space Strategy.** In the implementation of page reprogramming, traditional methods typically employ a  *one-to-many strategy* : multiple valid data segments are stored within a single reprogramming space to fully utilize the available horizontal space. These  *one-to-many strategies*  may include: $(i)$ storing multiple sets of independent data (e.g., $data_{A}$, $data_{B}$, ..., $data_{N}$); $(ii)$ storing multiple subsets of a single complex data (e.g., $data_{A0}$, $data_{A1}$, ..., $data_{An}$, where $data_{A} = data_{A0} + data_{A1} + ... + data_{An}$).

**传统数据管理。** 数据管理上，这些一对多策略为每个数据段引入一个耦合的元数据。我们记作$Mix(D,M)$，其中$D$表示数据，$M$表示元数据。==如图xxx所示==，传统的一对多策略采用链式结构对多份有效数据进行管理: 首个有效数据$Mix(D_{0},M_{0})$存在于重编程空间的头部，其中$M_{0}$包含下一个数据$Mix(D_1,M_1)$的位置。依次类推，通过这种方式串起所有数据。

**Traditional Data Management**. In terms of data management, these *one-to-many strategies* introduce *coupled metadata strategy* for each data segment, denoted as $Mix(D, M)$, where $D$ represents the data and $M$ represents the metadata. As shown in Figure 6, the *one-to-many strategy* manages data in a chained structure: the first valid data segment, $Mix(D_{0}, M_{0})$, is stored at the beginning of the reprogramming space, where $M_{0}$ contains the location of the next data segment, $Mix(D_1, M_1)$. This process continues, chaining all data segments in this manner.

出自灵活性考虑，我们不讨论数组的管理方式。与传统一次编程到位不同，页面重编程允许动态的重编程新数据，数组管理不适合这种新场景。

压缩和页面重编程相结合能带来双方问题的解决。但是这个结合并没有想象的那么直接。会面临xxx的问题。然而现有的管理方式不能很好的适用。
- 传统方法的空间布局
- 传统方法的数据管理
![[tra-failure1.png]]

![[tra-failure2.png]]

**挑战：面向小数据的高效页面重编程**。
- 压缩带来小数据。
- 结合的关键在于：如何实现一个面向小数据的高效的页面重编程管理。传统的管理方式并不适用（原因是没有充分考虑以下的因素）。
- 考虑：1.数据覆写。2.空间放大。如何充分利用受限的空间。数据压缩后具有不同的长度。不同的编码开销不同和更新次数。

页面横向的空间不断减少。在有限的页面空间内，原本的压缩数据本来能够放入。但经过空间放大（例如QLC上的4倍放大），这个数据可能就无法继续存入。这会进一步加剧了内联压缩中的碎片化问题，带来更大的页面浪费。传统的内联压缩的数据管理方式并不能直接适用于页面重编程。

The horizontal space of the page continues to diminish. While the originally compressed data could fit within the limited page space, space amplification (e.g., 4x amplification on QLC) may prevent the data from being stored. This exacerbates the fragmentation issue in inline compression, leading to greater page waste. Traditional data management methods for inline compression are not directly applicable to page reprogramming.


除此之外，如何将新的数据管理策略运用到内联压缩，如何实现引入页面重编程的混合编程管理也是CORE所需要关注的问题。


**Failure.**  However, these \textit{one-to-many strategies}, which overlook vertical overwrites, are ineffective on page reprogramming. As shown in Figure \ref{fig:tradition-fail}, vertical data overwrites inevitably destroy multiple valid data segments, resulting in data loss or recovery failure. Write amplification exacerbates metadata amplification and page fragmentation problems. 


**挑战：基于页面重编程，传统方法并不奏效，结合内联压缩的挑战在于如何实现高效的小数据更新策略**。

挑战：页面重编程带来空间放大和纵向数据覆盖，使其无法直接和现有的内联压缩相结合。

**Challenge**: Page reprogramming introduces space amplification and vertical data overwrite in a page, making it incompatible with existing inline compression techniques.


## Design
### 4.1 Overview
![[paper-overview-1.png]]
图7的caption：图中的红色的箭头代表数据的在一个页面内的重编程的循环迭代。HD-Flash是高密度闪存的缩写。L2P表的左边字段是物理地址，右边字段是逻辑地址。

 The red arrows in the figure represent the cyclic cover of reprogramming within a single page. *HD-Flash* is an abbreviation for high-density flash memory. In the L2P table, the left field represents the physical address *ppa*, while the right field represents the logical address *lba*.
 
核心思想。CORE的核心思想是对单个页面的数据内容进行压缩和弹性编码（数据感知），并通过二维重编程来完成高密度闪存页面数据的就地更新（介质感知）。CORE能够在平衡开销的前提下，尽可能减少擦除次数，延长高密度闪存的寿命。

**Core Idea.** The core idea of CORE is to perform compression and *elastic encoding* (data-aware) on the data content of a single page, while utilizing *two-dimensional reprogramming* to achieve in-place updates of high-density flash memory pages (media-aware). CORE aims to minimize erase cycles as much as possible, thereby extending the lifespan of high-density flash memory, while maintaining a balance in overhead.

架构。我们在图xx展示了CORE的设计概览。CORE作用于host和高密度闪存之间，对来自host的逻辑页面进行处理，将数据在高密度闪存的页面上进行更高效的组织和管理。当主机发送对应的写请求时，CORE会根据数据的特性首先尝试对旧的ppa进行最大化复用，迫不得已再申请新的物理地址。相关的核心组件及其工作流包括：
- Compression and reprogramming pages。CORE会对页面的数据进行压缩，以屏蔽编码带来的空间开销。不同的压缩策略（即无损压缩、差量压缩）会对应着CORE中不同的重编程页面。
- Elastic Encoding。CORE根据压缩后的数据的长度进行弹性的编码，在保证数据能够放入闪存页面的前提下，最大限度发挥高密度闪存的数据表示能力。
- Dual-Dimensional Reprogramming。数据经过压缩和编码处理后，CORE充分利用多级电压的特性，通过二维重编程来实现高密度闪存上的页面就地更新，来减少高密度闪存上的擦除次数。
- 混合编程和L2P表。在CORE的具体实现中，我们对混合编程架构进行了进一步的阐述。

**Architecture.** We highlight the CORE design overview in Figure xxx. CORE operates between the host and high-density flash memory, processing logical pages from the host and organizing and managing data more efficiently on the high-density flash memory pages. Specifically, when the host issues a corresponding write request, CORE attempts to reuse the old physical page. The key components and workflow include:
- Compression and Reprogramming Pages. CORE compresses the page data to mitigate the space overhead caused by encoding. Different compression strategies (i.e., lossless compression, delta compression) correspond to different reprogramming pages in CORE ().
- Elastic Encoding. CORE applies the *elastic encoding strategy* based on the length of the compressed data, ensuring that the data fits within the flash memory pages while maximizing the data representation capability of high-density flash memory ().
- Dual-Dimensional Reprogramming. After compression and encoding, CORE fully leverages the characteristics of multi-level voltages to perform in-place page updates on high-density flash memory through *dual-dimensional reprogramming*, thereby reducing the number of erase cycles ().
- Hybrid Programming and L2P Table. In CORE’s implementation, we further elaborate on the hybrid programming architecture ().





### 4.2 Reprograming Space Layout
![[paper-dual.png]]

伴随如章节xxx提到的传统方法在页面重编程上的失败，CORE回答了页面重编程下的数据管理的方式。

With the failure of traditional data management method mentioned in Section \ref{sg:op-cha}, CORE answer the question that how to achieve the efficient data management approach on page reprogramming.

**重编程的两个维度。** CORE希望通过引入编码前数据压缩的方式来优化页面的重编程。如果处理后（压缩与编码）数据的大小小于闪存页面的空间，数据可以通过重编程的方式实现横向的追加。这意味着，在CORE方案中，高密度闪存的页面重编程具备两个维度: 通过重编程，页面的数据能够进行横向的追加和纵向的覆盖。

**Two Dimensions of Reprogramming.** CORE aims to optimize page reprogramming by introducing data compression before encoding (e.g., WOM-v code). If the size of the processed data (after compression and encoding) is smaller than the flash page capacity, data can be appended horizontally through reprogramming. This indicates that, in the CORE scheme, page reprogramming in high-density flash memory has two dimensions: horizontal appending and vertical overwriting of page data.

**我们的空间策略。** 一对多的策略渴望在一个重编程空间中存放更多份有效数据。CORE采用了相反的思想。CORE采用一对一的策略来应对上述的问题。在一个重编程空间中，CORE只存储一份且独立（即不由子数据构成）的数据。 这样的好处在于：能够实现灵活的纵向迭代，而无需考虑对其他数据的影响。单个数据的迭代能够彻底的应对元数据放大的问题和页面内部的的碎片化问题。

**Our Space Strategy.** The *one-to-many strategy* aims to store multiple valid data segments within a single reprogramming space. In contrast, CORE adopts an opposite approach. To address the aforementioned issues, CORE adopts a *one-to-one strategy*. In a reprogramming space, CORE stores only one independent data segment (i.e., not composed of sub-sets. e.g., just $data_{A}$). The benefit of this approach is that it enables flexible vertical overwrite without affecting other data. The iteration of a single data segment effectively addresses the problems of metadata amplification and internal page fragmentation.


![[non-cyclic.png]]
  #paper
- [x] 需要解释为什么传统的不行！
- 无法解决碎片化，数据能看到的空间被缩减，空间放大加剧碎片化
- 为了方便解析，且因元数据较小，故元数据采用level-1
- 电压均衡性问题，元数据和数据的特性不一致，比如考虑元数据和数据的编码是否一致
- 弹性编码能否实现？能，但右边会偏高
- 很低效

**非循环数据管理**。基于一对一策略，CORE将耦合的元数据管理进行了改造。思想很简单。如图xx所示，这种新的方式沿用了“链式”结构，但不同在于其中只有一份数据是有效的（即图xxx中的D5）。在这种管理方式中，每个周期的元数据通过固定的起始位置和链式结构进行定位。一个有效数据会被多个元数据索引。当数据无法放入当前周期的空间后，CORE会重头开始一个新的周期。然而，这种方法是低效的。每一个周期内，这些元数据呈现零散分布，而无法被覆盖（否则会破坏这个周期的链式结构）。虽然在一对一策略下，这种方法能应对数据覆盖的问题，但其并没有很好的解决章节xxx所提到的空间放大和碎片化的问题。

**Non-cyclic Data Management.** Based on a one-to-one strategy, the CORE modifies the coupled metadata strategy. The optimization is simple. As illustrated in Figure xx, this new approach retains the chained structure, but the key difference is that only one data segment is valid (i.e., D5 in Figure xxx). In this data management, metadata for each cycle is located through a fixed start position (i.e., the head of the reprogramming space)  and a chained structure. A valid data segment can be indexed by multiple metadata entries. Once data can no longer fit into the current cycle's space, the CORE initiates a new cycle from the beginning. 
However, this method is inefficient. Within each cycle, the metadata is scattered and cannot be overwritten (as doing so would disrupt the chain structure of the cycle). Although this approach addresses the issue of data overwriting under the one-to-one strategy, it does not effectively resolve the issues of space amplification and fragmentation mentioned in Section xxx.


在这种管理方式中，元数据通过高密度闪存上多级电压的最大值$V_{max}$定位。




**循环迭代数据管理。** CORE引入分离式的元数据策略。 借助CORE重编程，数据的位置会进行周期性的改变。CORE引入元数据对其进行管理，记录一些相关数据，如其起始位置和原始长度。每次数据更新同时会带来元数据的更新。CORE借助高密度闪存的电压特性很好的实现了这一点。CORE将数据和元数据进行解耦管理，并对高密度闪存页面采用了隐式的划分。我们记作$Dec(D,M)$，其中$D$表示数据，$M$表示元数据。CORE将数据和元数据布局在页面的两侧，对它们采用不同的重编程方式（后面会进一步详细阐述）。 这是为了便于CORE实现元数据定位，进而实现页面数据的定位。由于采用隐式的划分，当元数据和数据发生不可避免的碰撞冲突时，高密度闪存页面的编程达到上限。




**Our Data Management.** CORE uses a *decoupled metadata strategy*. With  reprogramming of CORE, the locations of data are periodically altered. CORE also introduces metadata to manage these changes by recording the relevant information, such as start position *start* and the length of original data *len*. Each data update also necessitates a corresponding metadata update. CORE leverages the voltage characteristic of high-density flash memory to achieve this effectively. CORE manages data and metadata separately, utilizing an implicit partitioning of high-density flash memory pages. This is denoted as $Dec(D, M)$, where $D$ represents data and $M$ represents metadata. CORE arranges data and metadata on opposite sides of the reprogramming page and uses different reprogramming methods for each (which will be further elaborated later). This approach facilitates metadata location, and subsequently, data location within the page. Due to the implicit partitioning, when inevitable collisions between metadata and data occur, the programming limit of high-density flash memory pages is reached.

### 4.3 Dual-dimensional Reprogramming

   #paper
- [x] 讲一个naive一点的方法：uncycle的重编程方式。这个方法会产生比较大的碎片化问题，len < 周期的剩余的空间才行
- [x] 而cycle的方式，只需要满足len < 周期的空间的size。

CORE对数据和元数据采用了相反的重编程方式，这充分利用了高密度闪存的特性。
CORE adopts opposite reprogramming approaches for data and metadata, fully leveraging the characteristics of high-density flash memory.

**元数据重编程工作流。** CORE在元数据的重编程上采用了纵向优先策略。元数据具有定长的数据特性。借助这个特点和高密度闪存的电压特性，CORE采用了纵向优先策略对元数据进行重编程。如图x所示，CORE以元数据的大小的窗口为一个滑动窗口，从高密度闪存页面的一侧开始记录元数据。每次发生元数据更新时，CORE会采用纵向提高当前窗口电压的方式对元数据直接进行窗口的原地重编程，覆盖旧的元数据。进行若干次重编程后，当前窗口的电压会达到$V_{max}$ 。CORE会将这个包含最高电压$V_{max}$的滑动窗口判定为失效，并向着另一个方向滑动一个滑动窗口大小的距离。如果该滑动窗口的电压能够满足元数据重编程的条件，则将这个滑动窗口作为新的有效窗口，然后对其重新采用纵向优先策略。CORE会对元数据重复上述重编程的过程，直到不再满足元数据重编程的条件。注意，这个新的滑动窗口未必需要是未编程过的，可能是建立在失效的旧数据上。

**Metadata Reprogramming.** CORE adopts a *vertical-first strategy* on metadata reprogramming. Metadata exhibits a fixed-length property. Leveraging this feature and the voltage characteristic of high-density flash memory, CORE employs a *vertical-first strategy* to reprogram metadata. As shown in Figure 8, CORE treats metadata size as a sliding window, beginning from one side of the high-density flash memory page to store metadata. Upon each metadata update, CORE performs in-place reprogramming of current window by incrementally increasing the voltage of the window vertically, thereby overwriting the old metadata. After several reprogramming cycles, the voltage of the current window will reach $V_{max}$. CORE then regards this window with the maximum voltage $V_{max}$ as invalid window and shifts by the size of one sliding window towards the opposite direction. If the voltage of this new window meets the conditions for metadata reprogramming, it is designated as the new valid window, and the *vertical-first strategy* is reapplied. CORE repeats this reprogramming process for metadata until the conditions for further reprogramming are no longer met. Note that the new sliding window may not necessarily be unprogrammed; it could be located over invalidated old data. 

**好处。** 对元数据采用纵向优先策略的好处在于能够借助高密度闪存的纵向覆盖特性来及时覆盖无效的元数据，充分利用元数据长度固定的特点和高密度闪存的最大编程变压$V_{max}$实现CORE的一对一策略的元数据定位。


**Benefit.** The benefit of the *vertical-first strategy* for metadata is that it takes advantage of the vertical overwrite characteristic of high-density flash memory to overwrite invalid metadata promptly, fully utilizing the fixed-length nature of metadata and the maximum programming voltage $V_{max}$ of high-density flash memory to achieve CORE's metadata localization of *one-to-one strategy*. 

**数据重编程工作流。** CORE在数据的重编程上采用了横向优先策略。纵向优先策略适用于元数据，但并不适用于数据。它的缺点是会导致页面的局部优先达到最大电压，带来有效编程空间的减少，导致页面无法放入大的数据而提前停止了重编程，带来内部碎片化的问题。由于元数据的长度固定且占用空间较小，该策略不会带来过于严重的上述问题。然而，数据具有变长且大小较大的特点，该策略无法奏效。为了避免上述问题，CORE对数据采用了横向优先策略。如图xx所示，每次数据更新，CORE会采用横向追加的方式对数据进行异地的重编程。当本次周期横向追加的空间已经无法容纳完整数据时，CORE会拆分数据并通过循环迭代的覆盖方式来实现无碎片化的重编程。注意，当元数据记录的结束位置end小于开始位置start，意味着该数据正处于循环迭代。元数据每次发生数据更新，CORE会对数据重复上述重编程过程，直到和元数据发生不可避免的冲突。

**Data Reprogramming.** CORE adopts a *horizontal-first strategy* on data reprogramming. While the *vertical-first strategy* is well-suited for metadata, it is not appropriate for data. Its downside is that it may cause certain sections of the page to reach $V_{max}$ prematurely, reducing the available reprogramming space, and leading to early termination of reprogramming, thereby exacerbating internal fragmentation. Due to the fixed length and smaller size of metadata, this issue is almost ignored. However, since data can vary in length and is typically larger, the *vertical-first strategy* is ineffective. To avoid these issues, CORE adopts a *horizontal-first strategy* for data. As shown in Figure 8, CORE performs out-of-place reprogramming for data by appending new data horizontally upon each update. When the available space can no longer accommodate the all new data in this cycle, CORE splits the data and applies a cyclic cover to achieve fragmentation-free reprogramming. If the end position of data (computed by *start* and *len* in medata) is smaller than the start position *start*, it means the data is currently undergoing a cyclic cover reprogramming. Upon each data update, CORE repeats the reprogramming process for data until an inevitable conflict with metadata arises. 


好处。横向优先策略的好处在于能够实现页面内部的电压均衡，并通过循环迭代的方式完全解决页面内部的碎片化问题。

**Benefits.** The benefit of the *horizontal-first strategy* is that it allows for voltage balancing within the page and completely resolves internal fragmentation through the iterative overwriting process.


  #paper
- [x] cover和overwrite、iteration的一致性
- [x] 前面要强调，数据和元数据都要进行解码

编码和解码过程。基于上述的元数据和数据的重编程策略，CORE的二维重编程的工作流可以描述如下：
（1）编码流程。重编程空间收到待存储的数据（输入重编程空间的数据已经完成了压缩和编码），标记为$data_{A|new}$，其中$A$表示数据类型，$new$表示数据版本。首先，CORE会解析旧的元数据。CORE从存储元数据一侧，以元数据大小为窗口进行滑动，如果当前窗口存在$V_{max}$则继续滑动，直到第一个满足不存在$V_{max}$的窗口。CORE对该窗口进行解码，恢复出旧的元数据$metadata_{A|old}$ 。然后，CORE完成数据和元数据的更新。CORE根据$metadata_{A|old}$得到的旧数据$data_{A|old}$位置信息，对新数据$data_{A|new}$采用*horizontal-first strategy*进行更新。在确定新数据$data_{A|new}$的位置后，CORE对元数据采用*vertical-first strategy*进行更新。当页面重编程的数据都确定后，CORE对页面提交一次重编程。
（2）解码流程。解码流程的步骤类似。CORE会通过窗口滑动来定位页面的元数据并进行解码。根据恢复出的元数据，CORE会对数据进行定位进行解码，恢复出原有的数据。

**Encoding and Decoding Workflow.** Based on the aforementioned reprogramming strategies for metadata and data, the workflow of dual-dimensional reprogramming in CORE can be described as follows:
- Encoding Workflow.  When the reprogramming space receives the data to be stored (note that the input data to the reprogramming space has already been compressed and encoded), it is denoted as $data_{A|new}$, where $A$ represents the data type and $new$ represents the data version. Firstly, CORE parses the old metadata. Starting from the metadata storage side, CORE slides across the windows, sized according to the metadata, and continues sliding if the current window contains $V_{max}$, until it finds the first window where $V_{max}$ is not present. CORE decodes this window to retrieve the old metadata $metadata_{A|old}$. Secondly, CORE completes the reprogramming of both the new data and its metadata. Using the location information from the old metadata $metadata_{A|old}$, CORE updates the new data $data_{A|new}$ by applying the _horizontal-first strategy_. After determining the location of the new data $data_{A|new}$, CORE updates the metadata using the _vertical-first strategy_. After all the data for page reprogramming is determined, the CORE commits a reprogramming of the page.
- Decoding Workflow. The decoding process follows similar steps. CORE locates the metadata on the page by sliding through the windows and decoding them. Based on the recovered metadata, CORE locates and decodes the data, restoring the original data.


**页面的重编程极限。** 数据和元数据同时更新时，会先确保元数据能够成功写入，然后再写入数据。以下几种情况可能会达到页面的编程极限：（1）重编程空间的电压已经达到最后一层，无法再满足元数据的重编程；（2）元数据能够成功重编程，但页面中剩余的有效的重编程空间大小已经小于数据的大小。

Page Reprogramming Limits.  When both data and metadata are updated simultaneously, CORE first ensures that the metadata is successfully written before writing the data. Several conditions may lead to the page reaching its reprogramming limit: $(i)$ the voltage in the reprogramming space has reached the final level, making it impossible to further reprogram the metadata; $(ii)$ the metadata is successfully reprogrammed, but the remaining valid reprogramming space on the page is insufficient to accommodate the new data.

### 4.4 Encoding Strategies for Reprogramming

  #paper
- [x] 讨论elastic code和single code中的电压规律性（左高右低）和不规律性，对应不同的元数据滑动方案
- [x] 画一下弹性编码的示意图

![[paper-elastic-code-1.png]]

  #paper
- [x] Elastic Encoding Strategy这个图需要将图标替换一下，换成更清晰的圈号1，和论文对标。

章节3.2和3.3回答了CORE的二维重编程的布局和重编程策略。它们基于的一个前提，这些输入的数据已经被压缩和编码完成。对输入数据的编码方式直接影响到页面二维重编程的次数。在本节，我们进一步介绍CORE的编码实现方式。

Sections 3.2 and 3.3 addressed the layout and reprogramming strategies of dual-dimensional reprogramming in CORE. They are based on the premise that the input data has already been compressed and encoded. The encoding method applied to the input data directly impacts the number of two-dimensional reprogramming cycles a page can undergo. In this section, we further introduce the encoding strategies in CORE.

**单一编码策略。** 按照空间放大的程度的从高到低，WOM-v Code分为level-3，level-2，level-1三个等级，依次带来4x,2x,接近1.3x的放大。单一编码策略在一个页面内只使用同一种编码方式。每个周期的电压始终以相同的距离步进提升。这种设计很简单，只需要给页面维护一个固定的编码等级字段即可（如，存放在页面的OOB）。

**Single Encoding Strategy.** Based on the degree of spatial amplification from highest to lowest, WOM-v Code is divided into three levels: level-3, level-2, and level-1, corresponding to amplification factors of 4x, 2x, and approximately 1.3x, respectively. The *single encoding strategy* employs only one type of encoding within a page. During each cycle, the voltage is always incrementally increased by the same step. This method is simple, as it only requires maintaining a fixed encoding level field (2 bits) for the page (e.g., stored it in the page's OOB). 

**不足。** 然而，这种方式并不高效。基于前面的讨论我们知道以下事实：（1）同一个页面的输入的数据会因压缩程度不同而具备不同的长度。（2）空间放大程度越大，编码的重编程次数也会越多（即，level-3，level-2, level-1理论上分别对应15，5，2次）。由于数据和编码的上述特性，单一编码策略会让二维重编程出现两种可能的问题：（1）空间瓶颈。对于较长的数据（即超过剩余重编程空间的1/4），只采用level-3编码可能会让数据无法存入页面而提前终止页面的重编程，导致很多电压状态没有利用。注意，随着重编程次数的增加，剩余重编程空间是逐渐减少的。（2）电压瓶颈。对于较小的数据，单一地采用level-1编码又会极大地浪费电压状态（即一次步进7个电压状态），重编程次数很低。

**Limitations.** However, this approach is not efficient. Based on the previous discussion, we know the following facts: $(i)$ The input data for the same page may vary in length due to different compression levels. $(ii)$ The higher the degree of spatial amplification, the more reprogramming cycles are available (i.e., level-3, level-2, and level-1 theoretically correspond to 15, 5, and 2 reprogramming cycles, respectively). Due to the aforementioned characteristics of data and encoding, the *single encoding strategy* can lead to two potential issues in *dual-dimensional reprogramming*: $(i)$ Space Bottleneck. For example, for larger data (i.e., data exceeding 1/4 of the remaining reprogramming space), using level-3 encoding alone may cause the data to fail to fit within the page, prematurely ending the page reprogramming process and leaving many voltage states unused. Note that as reprogramming cycles increase, the remaining reprogramming space gradually decreases. $(ii)$ Voltage Bottleneck. For smaller data, using level-1 encoding alone would result in significant waste of voltage states (i.e., a single step increases by 7 voltage states), leading to a very low number of reprogramming cycles.

 **弹性编码策略。** 为了解决上述问题，CORE针对变长数据提出了弹性编码策略。其核心的思想是允许一个页面根据数据或空间的变化情况灵活地使用不同等级的编码。具体来说，CORE采用了重编程次数优先的贪心方法来选择编码方式（即，level-3 $>$ level-2 $>$ level-1，其中$>$表示优先程度）。当更高优先级的编码无法让数据放入剩余的重编程空间中时，CORE会依次尝试更低优先级的编码方式，直到能将数据放入剩余的重编程空间。与单一编码策略不同，弹性编码策略在每次重编程时，都会对编码等级字段进行热更新。因此CORE将该字段（2比特）管理在重编程元数据中。图x展示了QLC闪存页面上的弹性编码策略的一个示意图。当（1）data1输入页面时，CORE对其使用level-3编码并存入页面。当（2）data2输入页面时，level-3编码已经无法让该数据存入重编程空间，CORE对其使用电压跨度更大的level-2编码。data2会挨着data1追加存储，并完成一次无碎片化的周期迭代。（3）data3又可以使用level-3编码，会在原来的data1电压上继续小幅度升压。（4）data4无法用level-3和level-2编码存储到剩余重编程空间，CORE被迫对其采用level-1编码。这会带来最大的电压跨度，并使得剩余重编程空间缩小。（5）data5和（6）data6会继续在缩小后的剩余的重编程空间中分别采用一次level-2和level-1编码。这个过程会持续，直到达到页面的编程极限。和单一编码策略不同，弹性编码策略的电压呈现不均匀分布。注意上述编码策略是针对变长数据的讨论。


 **Elastic Encoding Strategy.** To address the issues mentioned above, CORE introduces an *elastic encoding strategy* for variable-length data. The core idea is to allow a page to flexibly use different levels of encoding based on the changing conditions of the data or available space. Specifically, CORE employs a reprogramming-cycle-first greedy method to select the encoding scheme (i.e., level-3 > level-2 > level-1, where ">" denotes higher  priority). When a higher-priority encoding can no longer fit the data into the remaining reprogramming space, CORE will sequentially attempt lower-priority encoding schemes until the data can fit into the available space. Unlike the *single encoding strategy*, the *elastic encoding strategy* updates the encoding level field *level* dynamically during each reprogramming cycle. Therefore, CORE manages this field (2 bits) within the reprogramming metadata. Figure X illustrates a diagram of the elastic encoding strategy on a QLC flash page: (1) When $data_1$ is written to the page, CORE uses level-3 encoding and stores it on the page. (2) When $data_2$ is written, level-3 encoding can no longer fit the data into the remaining reprogramming space, so CORE switches to the level-2 encoding with higher voltage range. Then, $data_2$ is stored adjacent to $data_1$, completing a fragmentation-free reprogramming cycle. (3) $data_3$ can be encoded using level-3 again, which results in a small voltage increment on top of the previous data1 voltage. (4) $data_4$ cannot be stored using level-3 or level-2 encoding due to the remaining reprogramming space. CORE is forced to use level-1 encoding, resulting in the largest voltage step and reducing the available reprogramming space. (5) (6) $data_5$ and $data_6$ continue using level-2 and level-1 encoding, respectively, in the reduced remaining reprogramming space. (7) This process continues until the page reaches its programming limit. Unlike the single encoding strategy, the voltage distribution in the elastic encoding strategy is non-uniform. Note that the above encoding strategy discussion applies specifically to variable-length data.


图9的caption：纵坐标表示QLC的16个电压等级，横坐标表示闪存页面的存储单元，序号代表数据输入该闪存页面的顺序。

 元数据的编码类型与开销。为了简单起见，CORE对元数据采用固定的编码方式。考虑到其大小较小，CORE默认对元数据采用level-3的编码，来提供最大的重编程次数。CORE完整的元数据包括起始位置、结束位置和编码等级。在4KiB的闪存页上，上述字段分别为12比特、12比特和2比特。因此CORE的元数据大小仅为26比特，通过编码后进一步放大为13字节，这是可接受的。

Metadata Encoding Type and Overhead. For simplicity, CORE uses a fixed-level encoding for metadata. Given its small size, CORE defaults to level-3 encoding for metadata to maximize the number of reprogramming cycles. The complete metadata in CORE includes the start position *start*, the length of original data *len*, and encoding level *level*. For a 4KiB QLC NAND flash page, these fields require 12 bits, 12 bits, and 2 bits, respectively. Therefore, the total size of metadata in CORE is only 26 bits, which, after encoding, expands to 13 bytes—an acceptable overhead.

### 4.5 CORE Pages with Compression

![[Pasted image 20240921213641.png]]

图10 caption：带有$lba_A$和$lba_B$所在的方框集合代表从主机发送来的逻辑页面，其中标号（即，$1,2,...$）代表逻辑页面到达的先后顺序。浅色的虚线箭头代表已经无效的数据，深色的虚线箭头代表此时正有效的数据。箭头附近记录对页面进行的操作。页面$ppa_A$和$ppa_B$分别代表对应的物理页面。页面中灰色的斜线填充代表失效的数据，灰色实心填充框代表未经过编码的数据。

The boxes containing $lba_A$​ and $lba_B$​ represent the logical pages sent from the host, with numbers (i.e., 1, 2, ...) indicating the order in which the logical pages arrive. Pages $ppa_A$ and $ppa_B$​ correspond to their respective physical pages. The pale grey dashed arrows signify invalid data, while the dark black dashed arrows represent currently valid data. Operations performed on the pages are noted near the arrows. 

CORE根据页面中的编码空间的占比设计了3种页面，即全编码页、部分编码页和不编码页。不同类型的页面可以用于不同的压缩场景。在本小节，CORE主要聚焦于能够完成重编程的全编码页和部分编码页，及其它们和压缩的结合。

CORE designs three types of pages based on the proportion of coding space within the page: fully encoded pages, partially encoded pages, and non-encoded pages. Different types of pages can be used in different compression scenarios. CORE primarily focuses on fully encoded and partially encoded pages that support reprogramming, as well as their integration with compression.



CORE的全编码页。全编码页将高密度闪存页面的全部空间都用作编码区域。这在先前的工作中（例如WOM-v）已经被引入：将一个逻辑页编码为多个全编码页。然而，不同的是，CORE聚焦于单个全编码页，并为其引入了页面内的二维重编程来实现和与无损压缩完美结合。

无损压缩和重编程。如图xx(a)所示，对于主机发来的地址为$lba_A$的逻辑页面，CORE为其分配一个地址为$ppa_A$的全编码页。每次上层对逻辑地址$lba_A$发起新的写操作（即页面的更新）时，CORE都会对该逻辑页面的全部数据进行无损压缩操作，然后再将压缩后的数据编码存储到地址为$ppa_A$的全编码页中。通过二维重编程，旧的页面数据会被最新的页面数据通过无碎片化的循环迭代方式进行覆盖。根据页面数据的压缩性，CORE会通过弹性编码策略选用不同程度的编码。例如，当压缩后数据的大小小于50%（一个常见的压缩比），则可以对其使用level-2的编码。至于页面的读取，CORE会对页面进行二维重编程空间的解码恢复出原先的压缩数据，进一步进行数据解压缩完成原数据的恢复。

**Fully Encoded Pages.** Fully encoded pages utilize the entire space of a high-density flash memory page as an encoding region. This concept has been introduced in previous works (e.g., WOM-v), where one logical page is encoded into multiple fully encoded pages. However, CORE differs by focusing on a single *fully encoded page* and introducing in-page *two-dimensional reprogramming* to achieve seamless integration with lossless compression. 

**Lossless Compression and Reprogramming.** As illustrated in Figure xx(a), for a logical page with a host address of $lba_A$, CORE allocates a *fully encoded page* with a physical address of $ppa_A$. Each time a new write operation is initiated for the logical address $lba_A$ (i.e., an update to the page), CORE performs a lossless compression on the entire data of the logical page before encoding the compressed data into the *fully encoded page* at $ppa_A$. Through two-dimensional reprogramming, the old page data is overwritten by the new page data in a fragmentation-free cyclic cover. Depending on the compressibility of the page data, CORE employs a *elastic encoding strategy* to apply varying levels of encoding. For example, when the compressed data size is less than 50% (a typical compression ratio), level-2 encoding can be applied. As for page reading, CORE decodes the two-dimensional reprogramming space to recover the previously compressed data, and then further decompresses the data to restore the original content.

部分编码页。以往的方法以页面为单位，进行全部编码（即，全编码页），或者不进行编码（即，不编码页）。基于页面重编程前提下，CORE提出了一种高密度闪存上灵活的页面布局：部分编码页。部分编码将页面划分为编码区和非编码区，能够分别对应存储页面相关的热数据和冷数据。基于部分编码页的思想，CORE能够很容易实现高密度闪存上高效的页内差量压缩%% （在CORE前，这只能借助部分编程特性在SLC上实现） %%。

差量压缩与重编程。如图xx所示，对于上层先后发来的若干地址为$lba_B$逻辑页面时，CORE会首份数据（如图中的page 1）作为基准数据，压缩并存储在对应的地址为$ppa_B$的物理页面中。该基准数据作为冷数据，不进行编码，CORE以此为分界来划分编非编码区和编码区（即，剩下的空间）。划分的结果（对于4KB页面，该字段大小为12比特）记录在页面的OOB中。后续的逻辑页面会分别和这个基准数据进行差量压缩得到对应的差量（如图中的$delta_1$和$delta_2$）。这些差量会被编码和存储到页面的编码区域中。同样的，通过二维重编程，旧的差量会被新的差量通过无碎片的循环迭代进行覆盖。在相似性较好的情况下，这些差量可以采用*level-1*的编码。注意，每次差量计算都需要解码和解压缩基准页面。至于页面的读取，CORE需要根据OOB记录的区域，分别对基准数据解压缩，以及对差量数据解码和解压缩，然后根据恢复出的基准数据和差量数据来重构页面的原始数据。


**Partially Encoded Pages.** Previous approaches either encoded an entire page (i.e., *fully encoded pages*) or did not apply any encoding (i.e., *non-encoded pages*). Based on the premise of page reprogramming, CORE proposes a flexible page layout on high-density flash memory: *partially encoded pages*. These pages are divided into a non-encoded region and an encoded region, which can store cold and hot data respectively related to the page. With the concept of page reprogramming and *partially encoded pages*, CORE easily enables efficient intra-page delta compression on high-density flash memory.

**Delta Compression and Reprogramming.** As shown in Figure xx, when the host consecutively sends several logical pages with addresses $lba_B$, CORE takes the first page (as depicted in the figure as page 1) as the baseline page, compresses and stores it in the corresponding physical page with address $ppa_B$. This baseline page, considered cold data, is not encoded, and CORE uses this to demarcate the non-encoded and encoded regions (i.e., the remaining space). The division position (for a 4KB page, this field is 12 bits) is recorded in the OOB of a page. Subsequent logical pages undergo delta compression against the baseline data to obtain the corresponding deltas (e.g., $delta_1$ and $delta_2$ in the figure). These deltas are encoded and stored in the encoded region of the page. Similarly, through *two-dimensional reprogramming*, old deltas are overwritten by new deltas via fragmentation-free cyclic cover. When the similarity between pages is high, these deltas can be encoded using _level-1_ encoding. Note that each delta calculation requires decoding and decompressing the baseline page. As for page reads, based on the region recorded in the OOB, CORE decompresses the baseline data and decodes and decompresses the delta data. The original data of the page is then reconstructed using the restored baseline data and deltas.


不编码页。其中不编码页就是传统SSD上的页面，不对数据进行编码，因而无法支持页面的重编程。当页面的可压缩性太差而无法支持全编码页和部分编码页时，CORE会用不编码页来存储该页面。

Non-encoded pages. Non-encoded pages refer to the traditional pages on SSDs where no data encoding is applied, making them unable to support page reprogramming. When the compressibility of a page is too low to allow for fully encoded or partially encoded pages, CORE utilizes non-encoded pages to store the data.


<font color="#00b0f0">基于二维编程和CORE Pages，CORE能够实现高密度闪存上高效的数据压缩方案，降低碎片化，同时充分利用闪存电压。</font>

<font color="#00b0f0">如果偏离较大。就会重新申请一个。</font>


  #paper
- [x] Case1: 在第一中，空间加入剩余空间管理，减少碎片化问题
- [x] 添加拓展性讨论，衍生到PLC等等。

## Implementation

我们在LightNVM和FEMU-OCSSD结合的SSD模拟方式上实现了CORE。现有的黑盒SSD模拟的FTL仅负责时延的模拟，其存储的数据会和时延模拟进行解耦，直接传输到指定的内存区域，无法很好实现对页面数据的操作（如压缩或编码等）。Open Channel SSD的模拟方式（即LightNVM和FEMU-OCSSD）将闪存操作暴露给上层，可以很好地避免上述的问题，这也是前面工作所采用的模拟方式。此外，我们修复并使用了FEMU-OCSSD的OOB，来为CORE的页面提供OOB字段的支持。

We implemented CORE using a combination of LightNVM and FEMU-OCSSD for SSD simulation. In the existing Black-Box SSD simulators, the FTL primarily simulates latency, and the stored data is decoupled from the latency simulation, being directly transferred to a designated memory area. This approach makes it difficult to perform operations on page data, such as compression or encoding. The Open Channel SSD simulation method (i.e., LightNVM and FEMU-OCSSD), which exposes flash operations to the upper layers, effectively avoids these issues. This simulation method is also used in previous works (such as WOM-v). Additionally, we fixed and utilized FEMU-OCSSD OOB area to provide OOB field support for the pages in CORE.

### 5.1 Data padding and Byte Alignment
**数据不对齐问题。** 对于level-1编码，数据单元大小是3 bits，而编码单元大小是4 bits。在CORE的编码过程中，数据的长度（如1字节，即8比特）会和level-1编码的数据单元大小（3比特）发生无法对齐（即，无法整除3比特）的问题。注意，由于输入数据的大小会保持字节对齐，数据不对齐问题不会发生在level-2和level-3的编码中。

**Data Misalignment Issue.** In level-1 encoding, the data unit size is 3 bits, while the encoding unit size is 4 bits. During the encoding process in CORE, the length of the data (e.g., 1 byte, or 8 bits) cannot be exactly divided by the 3-bit data unit size of level-1 encoding, leading to a misalignment issue. Note that since the input data size remains byte-aligned, this misalignment issue does not occur with level-2 or level-3 encoding. 

**数据填充与编码字节对齐。** CORE通过数据填充来解决数据不对齐问题。 对于level-1编码，当发生数据的字节数和3不对齐问题的时候，如果剩余1字节不对齐，CORE会将填充4个比特位的0凑够12比特数据，最后生成2字节的level-1编码数据；如果最后剩余2字节不对齐，CORE会填充2比特的0凑够18比特数据，最后生产3字节的level-1编码数据。注意，出自实际运算的考虑，CORE保证了编码后的数据的字节对齐。

余1字节：填充4比特位，1/2字节
余2字节：填充2比特位，1/4字节
额外填充一个字节，

**Data Padding.** CORE resolves data misalignment issues through data padding. For level-1 encoding, when the number of data bytes is not divisible by 3, CORE applies the following padding strategies: $(i)$ If there is 1 byte remaining that causes misalignment, CORE pads 4 bits of zeros (instead of 1 bit, to maintain byte-alignment of the encoded data) to complete 12 bits of data, resulting in 2 bytes of level-1 encoded data. $(ii)$ If there are 2 bytes remaining, CORE pads 2 bits of zeros to complete 18 bits of data, resulting in 3 bytes of level-1 encoded data. Note that, for practical computational reasons, CORE ensures that the encoded data remains byte-aligned after the encoding process.

### 5.2 Hybrid Programming Architecture
  #paper
- [x] 思考是否添加并行的支持 
- [x] 思考是否需要拆分implementation和design

基于CORE的设计，CORE中会存在两种页面编程方式：（1）页面的重编程。CORE再次原有的旧页面，对其进行二维重编程。（2）异地编程。CORE会和传统的SSD FTL一样申请新页面进行写入。CORE对一个通用的混合编程架构的具体实现进行了阐述，这是以往的重编程工作中所忽视的。

Based on the design of CORE, there are two types of page programming in CORE: $(i)$ Page Reprogramming. CORE performs two-dimensional reprogramming on the existing old page. $(ii)$ Out-of-Place Programming. Similar to traditional FTL on SSD, CORE allocates a new page for a new programming. CORE elaborates on the implementation of a universal hybrid programming architecture, which has been overlooked in previous reprogramming works.


![[paper-hybrid.png]]

图11的caption：页面同时存在重编程和异地编程。CORE首先会从写缓存和L2P表中获取到旧页面的物理地址。然后对旧的页面进行读取、判断和GC检查，最终确定本次写请求的页面编程方式。

物理地址的获取。如图xx所示，CORE的写缓存会为输入的写请求$REQ(lba_A)$构造一个缓存结构体（编号为$id_n$），其物理地址字段*ppa*初始为空。在异地编程中，该字段只需简单的填写新申请到的页面地址。而在重编程中，该字段需要填写逻辑地址$lba_A$对应的旧页面的物理地址$ppa_{A|old}$。CORE会检索L2P表，如果缓存未命中，所检索到的地址就是所需要的物理地址，将其填写入字段ppa中；否则，该地址是cache line，CORE会根据cache line进一步检索到$lba_A$对应的旧的缓存结构（编号为$id_m$）。CORE将记录在旧的缓存结构（编号为$id_m$）中但未刷回的物理地址传递给新的缓存结构（编码为$id_n$），填写到对应的字段*ppa*中。该地址后续会进一步被用于编程判断和GC检查。

**Retrieving Physical Addresses.** As illustrated in Figure xx, CORE constructs a cache context (numbered as $id_n$) for the incoming write request $REQ(lba_A)$, with its physical address field _ppa_ initially set to empty. In the case of out-of-place programming, the new allocated page address can be simply filled into this field. However, for reprogramming, the field must be populated with the physical address $ppa_{A|old}$ corresponding to the logical address $lba_A$. CORE will search the L2P table; if the cache is not hit, the retrieved address is the required physical address and is filled into the field _ppa_. Otherwise, if the address corresponds to a cache line, CORE will further search for the old cache context (identified by $id_m$) corresponding to $lba_A$. CORE will then pass the physical address recorded in the old cache context (identified by $id_m$) that has not yet been flushed back to the new cache context (identified by $id_n$), filling it into the corresponding _ppa_ field. This address will subsequently be used for programming types decision and garbage collection check.

混合编程处理。正如前面所说，CORE包含重编程和异地编程两种页面编程方式。如图xx所示，Hybrid programming handler会根据检索到的物理地址，对页面发起一次读取请求，将对应页面加载到SSD的DRAM中进行处理。Hybrid programming handler会按照重编程的逻辑（如CORE中的二维重编程）对页面进行处理，并根据重编程对应的结束条件判断页面是否达到编程上限。如果该页面达到编程上限，CORE重新申请一个页面进行异地编程，同时将前面提及的物理地址字段*ppa*覆盖为新页面的物理地址。否则，字段*ppa*继续沿用旧页面的物理地址。在缓存结构体从缓存中淘汰时，L2P表的cache line会被替换为对应的新物理地址。

**Hybrid Programming Handler.** As previously mentioned, CORE incorporates two types of page programming: reprogramming and out-of-place programming. As illustrated in Figure xx, the hybrid programming handler initiates a read request for the page based on the retrieved physical address, loading the corresponding page into the DRAM for processing. The hybrid programming handler processes the page in DRAM according to the reprogramming logic (e.g., the two-dimensional reprogramming in CORE) and determines if the page has reached its programming limit based on the corresponding termination conditions. If the page has reached its programming limit, CORE will request a new page for out-of-place programming, simultaneously overwriting the previously mentioned physical address field _ppa_ with the physical address of the new page. Otherwise, the _ppa_ field  continues to use the physical address of the old page.  When the cache context is evicted from the cache, the cache line in L2P table will be replaced with the corresponding new physical address.

GC冲突。在混合编程中，有两个场景会同时对编程过后的有效页面同时进行操作：（1）垃圾回收。FTL会对选中的擦除单元中的有效页面进行迁移操作。（2）重编程。重编程会再次编程原来的有效页面。例如这样的场景，对选中旧的物理地址$ppa_{A|old}$进行重编程的过程的中途，这时候恰好触发GC逻辑且选中了该页面所在的擦除单元。用户写入和GC是解耦的，重编程无法感知这一过程，会将数据继续写入旧的页面。然而，这个旧的页面会紧接着被GC标记为失效并进行擦除，从而导致数据错误。注意，SSD的FTL是以一个较大的超级块（如LightNVM中的*line*）为单元完成一次GC逻辑。这意味着发生GC冲突的概率是很大的。

**GC Conflict.** In hybrid programming, two processes may simultaneously operate on the valid pages that have already been programmed: $(i)$ Garbage Collection, where the FTL migrates valid pages from selected erase units, and $(ii)$ Reprogramming, which involves reprogramming the original valid pages. For example, during the reprogramming process of the selected old physical address ​$ppa_{A|old}$, if the GC is triggered and selects the erase unit containing that page, a conflict arises. User writes and GC operations are isolated, so the reprogramming process is unaware of this conflict and continues to reprogram data to the old page. However, this old page is subsequently marked as invalid by GC and erased, leading to data corruption. Note that the FTL executes GC logic at the level of a large superblock (e.g., the _line_ in LightNVM), which increases the probability of GC conflicts.

  #paper
- [x] GC冲突补充更多的情况
	- [x] 1.页面检查失败需要：失效 旧的有效页面
	- [x] 2.页面检查成功需要：重编程 旧的有效页面
	- [x] 3.GC的时候会：搬迁 旧的有效页面
- [x] 是否存在更朴素的解决办法
	- [x] 例如直接在失效/重编程的时候，再次获取旧的有效页面地址
- [x] 原因是：获取$get$ - 处理$handle$ - 执行$submit$，过程中$handle$会和GC线程$gc$并行。加锁的方式代价过大，因此我们采用在$submit$前采用再次获取的检查机制。

GC 冲突的解决。传统的out-of-place编程不会遇到上述问题，因为垃圾回收永远不会选中新分配的页面，不会带来页面的冲突。因此我们只需要针对重编程进行处理。如图xx所示，当页面进行重编程时，CORE会对重编程的页面进行GC的检查。在CORE的具体实现中，我们采取了简单的重编程退让策略。在对地址为$ppa_{A|old}$的页面重编程过程中，最后向设备发起编程操作时，GC checker会对$ppa_{A|old}$的所在擦除单元的状态进行检查。如果该擦除单元的状态已经进入了GC状态，CORE会放弃对这个页面的重编程（即使没达到页面编程上限），并为逻辑地址$lba_A$重新申请一个地址为$ppa_{A|new}$的页面进行out-of-place编程。

**Solution.** Traditional out-of-place programming does not encounter the aforementioned issues, as GC never selects newly allocated pages of out-of-place programming, preventing conflicts consequently. Therefore, we only need to address reprogramming process. As illustrated in Figure xx, before the page at address $ppa_{A|old}$ being reprogrammed, CORE performs a GC check. GC checker inspects the state of the erase unit containing $ppa_{A|old}$​. If the erase unit is already in GC state, CORE will take straightforward reprogramming fallback strategy and forgo reprogramming this page (even if the page programming limit has not been reached). Then CORE will request a new page at address $ppa_{A|new}$​ for out-of-place programming for the logical address $lba_A$​ to avoid the conflict.



%% 与并行的兼容。 %%

  #paper
- [x] code14 ssd性能测试出内核crash, NULL的bug？ -> 定位一下，顺便熟悉一下开发环境
- [x] make a scheme for test


## Evaluation
#### 6.0 测试思路
**回顾motivation**
- 传统的CSD没能充分利用闪存特性，且碎片化程度大 -> <4096
- WOM-v编码空间开销大，且容易局部磨损 -> > 4096 <=> overhead -> ==5 次== , 4096 10字节 -> 50字节
- 挑战：传统的数据管理不奏效，WOM-v编码dilemma
**优势体现**
- 和传统的CSD比页面利用率，比对于寿命的优化情况
- 和WOM-v编码比性能/空间开销，比较局部磨损的负载下相对于WOM-v的优化效果
- 针对上述所设计的设计点的效果：分解测试

我们的实验旨在回答以下的一些问题：
- CORE相对于==传统的压缩方案（改）==是否在高密度闪存上更加高效?
- CORE相对于全数据WOM-v编码方案的开销是否更小？
- 一些核心设计如何影响CORE？

Our experiments aim to answer the following questions:
- Can CORE be more efficient compared to traditional compression schemes on high-density flash memory 
- Can CORE incur lower overhead compared to the full data WOM-v encoding scheme?
- How do the core design decisions impact CORE?

#### 6.1 Experimental Setup

#### 6.2 Lifetime Optimization
- 和谁比：ideal(4096 4)，两种CORE，传统的CSD，naive (4096)，WOM-v(4096 4 2 1)
- 怎么比：**实验1.** 针对单个页面测试，调参分析。
	- **子实验1：** 页面利用率测试，不同长度的数据输入，看页面的利用率（和压缩无关），直接用elastic，对比CORE，traditional，partital，ideal，naive
	- **子实验2：** 模拟页面更新场景：3种常见压缩率，不同更新字节
- 怎么比：**实验2.** 重放trace，做一下比较擦除页面数等等。

#### 6.3 Measuring Performance
- 和谁比：两种CORE，naive，full data wom-v
- 怎么比：借用前人方法，理论分析。1.battle一下QAT速度（真实设备，不对称压缩解压），按照时延路径做表格理论对比分析；==（简单提一下）==。
- 怎么比：2.QAT模拟（压缩->截断+delay），fio做写入性能分析。

#### 6.4 Evaluating CORE features
- 和谁比：CORE自身方案的对比
- 怎么比：针对单个页面做分析，比较不同技术点的效果
- such as metadata amplification in *decoupled metadata strategy*,  cyclic cover in *dual-dimensional reprogramming* , and *elastic encoding strategy*, two CORE pages.

**elastic code stratgy的分解测试**

**目的：** 作为一个补充的小测试，主要想分解测试elastic code的效果

**测试指标：** 
**（1）数据利用率**
1. 页面的最大数据表征能力$capacity$。推导一个通用的公式：对于x bit cell，最大数据表示能力有多少？（如，对于QLC-SSD页面的最大利用率，利用level-3）
2. 页面的数据表示$store$。记录页面能够表示多少的数据。
3. 页面的数据利用率。$rate=\frac{store}{capacity}\times100\%$

**实验设置**
1. 针对单个页面分析。
2. 模拟单页面实现CORE方案的*single encoding strategy*和*elastic encoding strategy*。
3. 调整输入页面的数据分布特性，即长度序列，定长，变长。
	1. 定长。$(i)$ 严格定长：自己调参构造数据控制；$(ii)$灵活定长：lzdatagen生成指定相同压缩率数据。
	2. 变长。$(i)$严格变长：自己调参构造数据控制；$(ii)$灵活变长：lzdatagen生成指定不同压缩率；$(iii)$使用真实可压缩数据集。
4. 比较三种level的single方案和弹性方案（共4个对比对象）的空间利用率。

  #paper
- [x] 参考WOM-v，调研一下哪些trace能使用？
- [x] elastic code的分解测试
	- [x] 推导页面最大数据表示能力
- [x] 性能曲线的测试

我们自己有两个方案CORE-lossless、CORE-delta

核心
测试方案参考：WOM-v，SLC-Append，delta-FTL，CSAL等等
主要体现：1.能够提升多少寿命；2.性能有多少提升，相对于WOM-v
1. 页面可更新次数
2. 跑真实负载，lba分布，调参，与womv比
3. 搞性能曲线，与womv比，想办法handle住compression的时延（好难啊，fio是不固定的，只能直接截断模拟时延假装压缩了）
4. 测每个页面的利用率，构造一个利用率指标出来。看看能不能记录和画出电压分布与利用图。
5. **实验三：** 看看能不能对功能做分解测试（针对单个页面）
	1. elastic code的分解测试
	2. 两种编程方案的数据特性阈值测试


- [x] 实现弹性和循环编程
	- [x] 拓展metadata
	- [x] 使用新的metadata搞通原来的逻辑
	- [x] 实现循环编程
	- [x] 根据参数动态调整全部变量信息：set_encoding
	- [x] 实现弹性编码
		- [x] 处理code type变量的内容
	- [x] 考虑不编码的逻辑


## Other Section
#### discussion
正如章节2.3提及，一个启发的工作将其用于减缓cell的损伤，但其在高密度闪存上仍然不奏效。

碎片化问题。一些工作是怎么解决的呢？

后续的工作进一步讨论了SSD和无损压缩相结合带来的页面内部的碎片化问题及优化方案。

Subsequent work \cite{zuck2014compression, kim2015exploiting} further discussed the issues of internal fragmentation within SSD pages caused by the integration of lossless compression and proposed optimization solutions. 

<font color="#00b0f0">与传统CSD的讨论。传统的CSD更倾向于在同一个页面中放入更多的数据，以增大CSD的逻辑空间。然而这种方式会带来较大的碎片化问题。同时一对多的方式带来了较为复杂的映射表管理结构。CORE则通过循环迭代的方式解决了碎片化的问题，同时无需对映射表进行过多修改，只需简单修改映射表的更新逻辑。对于压缩性差的数据，CSD和CORE均无法很好的发挥效果。</font>

与WOM-v的讨论。

与SLCxxx的讨论。


## Reference
![[ydr-qlc.pdf]]