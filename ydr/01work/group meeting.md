---
status: In Progress
---

---


delta demo 测试

（1）相似度和上一个页面比较


|             | Ideal | womv14 | ours |
| ----------- | ----- | ------ | ---- |
| guass, 600  | 9     | 16     | 31   |
| guass, 1200 | 18    | 31     | 108  |
| guass, 2000 | 29    | 42     | 67   |

反常：
1.相似度和上一个页面比较，导致差量累积（handle）
2.好像代码上==差量计算==还有问题

![[Pasted image 20250208105728.png]]



---

测试思路：
- local wear部分
	- fio trace提供到字节粒度
	- trace层级：基于基本的io-trace，在一个页内自由修改，修改写入量
	- 微观层级：反复写lba为0的页面，修改写入量
	- ideal：统计 **不同的写入数据** 作为ideal。

----


## Motivation

![[Pasted image 20241225162518.png]]

## Design

![[Pasted image 20250110112718.png]]



---

## group meeting

by Darong Yang

01.10

---

### Part 1: What I Do
- 修复问题，重放on-demand、womv和naive在homes和mail两个trace的测试
- 梳理Delta Page部分的逻辑，分析其收益和开销。对何时触发Delta Page进行建模分析。
- 实现了Base Pages + Delta Page的写入逻辑（还需要功能测试一下）

---

==1、Trace重放 (1/2)==

**mail负载：naive、womv、on-demand**

![[4.png]]

注：naive奇怪的原因，下来分析是发现没触发GC

---

==1、Trace重放 (2/3)==

![[Pasted image 20250110110056.png | 750]]

注：看起来womv14和on-demand都没触发gc？需要下来研究一下，是采集指标的问题？还是性能曲线的问题？

---

==1、Trace重放 (3/3)==

**home负载：naive、womv、on-demand**

![[4 1.png]]

<font color="#ff0000">！！！：高速跑home负载，on-demand和womv又寄掉了，Kernel NULL pointer，很快能解决</font>

---


==2、Delta Page的建模分析 (1/2)==

![[Pasted image 20250110112718.png]]

---

==2、Delta Page的建模分析 (2/2)==

![[Pasted image 20250110104429.png]]


---

### Part 2: Next to Do

- -> 解决上述提到的home负载的问题

==Delta Page代码实现ing==

- <font color="#7f7f7f">√ 实现了On Demand Base Pages</font>
- <font color="#7f7f7f">√ 大致实现了Base Pages + Delta Page的写入逻辑</font>
- -> 调试一下Delta Page的写入逻辑有无问题
- -> 实现Base Pages + Delta Page的读取逻辑
- -> 实现every N operations的reset逻辑

<font color="#00b0f0">coding工作预计在3天左右完成</font>

---

Thanks!

---


group meeting

by Darong Yang

01.03

---


What I Do
- 实现On Demand Base的部分
- 研究fio trace，将以前的trace转为fio格式进行重放。

（实现的WOMv、和本周的On Demand Base都没法过trace，naive的可以）


---


==On demand Base (Ours) ==
![[firefly.png |700]]


---

==（上次的结果）==

![[64pu-1xlat.jpg]]

---

==fio重放 naive==

![[bab5ddeea0edbed688ef07173373b7bd.png | 700]]

1.05 小时

---

==但io-trace的曲线有大问题==

![[e9d458789ff8a08c691deacd0dba9aa7.png |700]]

---

==跑trace发生fail的问题，诸如：==

![[820f9ed3ab66870252de0efb4aab147a.png | 700]]

---

Next to do
- 定位on demand base和womv跑fio失败的问题
- 继续做自己代码的实现
- 上次讨论的：页面内部的更新分布情况， trace的性能曲线，erase unit的曲线

---

Thanks

- 缩小trace
- 提前触发GC

---


group meeting

by Darong Yang

12.27

---

What I Do

- 降低性能屏蔽计算开销实验
- 小数据更新实验
- 再次梳理故事、动机
- 开始做自己的实现，还在coding


---

==降低性能屏蔽计算开销实验== (1/3) 

参数：**64 PU - 1x lat**

![[64pu-1xlat.jpg]]

---

==降低性能屏蔽计算开销实验== (2/3) 

参数：**4 PU - 1x lat**

![[4pu-1xlat.jpg]]

---

==降低性能屏蔽计算开销实验== (3/3)

参数：**64 PU - 16x lat**

![[4pu-16xlat.jpg]]


---

==文件增/删/重命名/修改时间戳== (1/2)
![[Pasted image 20241223205544.png | 700]]

---

==文件删/重命名/修改时间戳== (2/2)
![[Pasted image 20241223205408.png | 700]]

---

==故事、动机与实现== (1/4)

**故事的想法**

![[Pasted image 20241225162518.png | 700]]

---

==故事、动机与实现== (2/4)

**我们的方案**


![[Pasted image 20241225162619.png | 700]]

---

==故事、动机与实现== (3/4)

**WOMv的特性**

**讨论：** 假设一个LBA的更新次数是n，分析不同方案所需要申请的页面数。讨论三种方案，Non、WOMv(2,4)、WOMv(1,4)。
其有关信息如下：

| 指标   | Non | WOMv(2,4) | WOMv(1,4) |
| ---- | --- | --------- | --------- |
| 写入次数 | 1   | 5         | 15        |
| 放大系数 | 1   | 2         | 4         |

---

==故事、动机与实现== (4/4)

**选择合适的WOMv**

![[Pasted image 20241224202143.png | 700]]


---

Discussion
- [x] vmalloc -> kmalloc （没有提升）
- [x] 其他paper的性能（TODO）
- [x] 读写性能一致（TODO）

---

Next to Do

- 继续写自己的实现
- 调研其他paper的SSD性能
- 研究读写性能的问题

---

Thanks

---


vmalloc -> kmalloc

其他paper的性能

读写问题

抽离出：
- on-demand WOMv架构
	- base pages的on demand
	- delta page
- 针对小更新的dual programming
	- 如何解决local wear

---


group meeting

by Darong Yang

12.20

---

What I Do
- 实现womv的读逻辑，完整实现对比对象womv
- 跑读/写混合测试
- （改bug改的比较久，现在看起来没什么问题了，能过fio）

---


==naive读写混合测试==

![[Pasted image 20241218112706.png]]

---

==womv(2,4)不加编码的读写混合测试==

![[Pasted image 20241218122505.png]]

---


==womv(2,4)读写混合测试==

![[Pasted image 20241220175402.png]]

---

现有的问题
- 读/写的性能曲线居然是一样的？（研究一下）
- 编码的时延比预期的高很多，导致加上之后的性能比naive低24倍。（得想办法handle住编码的开销，从计算型存储的角度将这个开销屏蔽给硬件）

---

Next to Do

- 自己写一些测试负载，测试womv在小更新下，具有局部磨损的问题（可能需要统计页面的电压分布？）想说明的问题如下：
	- （寻找小更新的场景）
	- （case of study 想跑一下h2c-dedup的dedup负载）
	- （case of study  或者跑一下DB/FS的meta操作）
- 开始实现自己的代码逻辑，预计1-2周。
![[Pasted image 20241220180902.png]]

---


group meeting

by Darong Yang

12.13

---

What I Do
- 上个月改完femu的bug，这周用fio测了一下base的性能
- 把WOMv的写+GC逻辑实现了，（读有点麻烦，还差一点点）
- 对其进行fio的randwrite测试，并和naive的对比
- add：来之前，在小容量1GB SSD上的fio写跑过了。刚刚跑大容量16GB SSD的fio写没跑过。有点小问题。



---

==fio测naive性能==：**顺序读写，加校验**（校验空档可以gc）
![[Pasted image 20241208161649.png]]

---

==fio测naive性能==：**顺序读写，加校验**（校验空档可以gc）

![[Pasted image 20241208164227.png]]

---

==fio测naive性能==：**随机读写，不加校验**（凸显GC）
![[Pasted image 20241208170735.png]]

---

==fio测WOMv(2,4)性能==：**随机写，不加校验（凸显GC）**


![[Pasted image 20241213110847.png]]

---

==fio测naive性能==：**上述相同负载**

![[Pasted image 20241213111751.png]]

---

Next To Do
- 先围绕WOMV，完成更加完善的一些测试数据
	- 调整大容量的bug
	- 读写混合fio
	- 自己定制一些负载，能够体现说明问题
![[Pasted image 20241213120624.png]]

---

Thanks!

---
Advice Record
::: block
  #paper
- [x] not zones -> other
- [x] QAT
- [x] Hot cold -> focus on reprogramming / uniform + xxx / 
- [x] trace -> block trace, not app 
- [x] brain picture
::: 

---

## Previous Meeting

---

group meeting

by Darong Yang

12.06

---

What I Do
- 回顾lightNVM代码，正在实现WOMv，争取下周做完

---

Next To Do
- 跑起WOMv，两个新对象：on-Demand的，delta的
- 测试WOMv的性能开销
- 测试局部修改WOMv的页面浪费

---

group meeting

by Darong Yang

11.27

---

What I do
- 总结上次讨论的思路，和实现方向
- 这两天刚回顾LightNVM代码，正在在实现
	- （1）WOMv
	- （2）新思路的代码

---
**之前的思路：**
![[Pasted image 20241129100711.png]]

---

![[Pasted image 20241129104743.png]]

---

![[Pasted image 20241129104807.png]]

---
Next to do
- 把WOMv、新思路的代码实现
- 完成WOMv代码可能会做点实验

---

What I do

1.阅读femu的时延模型，这两天解决了性能的Bug
踩坑，但有作用。
![[1729821673711.png]]

2.这几天画分解测试的图
不同CORE方案、以及和ideal的对比

---

| 通道  | 芯片  | 带宽        |
| --- | --- | --------- |
| 8   | 8   | 122MiB/s  |
| 4   | 8   | 66.9MiB/s |
| 4   | 4   | 35.2MiB/s |
| 1   | 4   | 9493KiB/s |
| 1   | 1   | 2434KiB/s |


---

Next to do

1.调查时延，插时延（《压缩速度》 -> 5us），测性能->性能曲线


---

group meeting

by Darong Yang

10.18

---

what I do
1. 研究性能测试怎么测
2. core-delta的时延分段测试时延。
3. 现在的测试思路想基于naive加时延：| 计算部分（同步执行） | 落盘部分（异步执行） |
4. 发现改时延对性能毫无影响，还不知道为什么？

---

Next to do
1. 继续研究performance的测试
2. 功能的分解测试

---



--- 

group meeting

by Darong Yang

09.27

---

what I do
1. 上周：完成design和implementation
2. 本周：做且<u>还在做</u>代码实现（cycle循环、弹性编码、两种pages等）
3. 卡bug：耽误三天没搞定，暂时放弃了，CORE的fio无法跑顺序写，可以跑随机写
4. 想添加一个分解测试，其他测试方案现在还在思考中
5. 因为卡bug、课程任务多起来导致进度很慢

---

TODO Next Step

1. 看时间是否改bug
2. 继续做好自己的代码优化、复现对比对象WOM-v
3. 分解测试绘制、找一些新的trace->委派xiaobo zheng
4. 确定测试方案


---



END

THANKS!

---

  #paper
- [x] 图的draft：思考横纵坐标
- [x] 



---
 09.13

what I do
1. make a overview graph 
2. rewrite the design and make it more clear 

---
what I do (2/3)
::: block <!-- element align="left"  -->
 
- **New Title:** CORE: Efficient <u>Co</u>mpression-Based Page <u>Re</u>programming Approach on High-Density Flash Memory
- **New Overview Graph:** 

:::

![[paper-overview.png|550]]


---
what I do (3/3)
::: block <!-- element align="left"  -->

<font color="#0070c0">2. rewrite the design and make it more clear </font>

**Design**

3.2 Dual-dimensional Reprogramming
- Two dimensional of reprogramming
- Traditional Methods: $(i)$ *one-to-many strategy*, $(ii)$ *coupled metadata strategy*
- Our Space Management: $(i)$ *one-to-one strategy*, $(ii)$ *decoupled metadata strategy*

---

TODO Next Step 
::: block <!-- element align="left"  -->

**Motivation**

2.4 the data classification and the fragmentation problem in CSD

adjust and distinguish between background and motivation


---

TODO Next Step (2/2)

::: block <!-- element align="left"  -->

**Design**

3.1 Overview 


3.2 Dual-dimensional Reprogramming 
- Reprogram Workflow 
- ...

3.3 ... more need to do 

<font color="#00b0f0">Finish the paper before the Evluation</font>


---

END

THANKS!

---



