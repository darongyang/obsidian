```widgets
type: countdown
date: 2025-01-08 07:59:59
to: ATC 2025 DDL
```

<center><a href="https://docs.qq.com/sheet/DU3VhalhWbkFjU1Jv?tab=m61vuahttps://docs.qq.com/sheet/DU3VhalhWbkFjU1Jv?tab=m61vua"> 课题组工作进度汇报 </a></center>

> **Index**
> paper: [[CORE paper]]
>1. Weekly & Monthly & Quarterly Report:  [[ydr/01work/work record#1. Weekly & Monthly & Quarterly Report]] 
> **==2. Tasks Notes==:** [[ydr/01work/work record#2. Tasks Notes]]
> **==3. TODO Items==:** [[ydr/01work/work record#3. TODO Items]]
> 4. Paper to Read: [[ydr/01work/work record#4. Paper to Read]]
> 5. Previous Progress: [[ydr/01work/work record#5. Previous Progress]]
>6.  Undergraduate Thesis: [[ydr/01work/work record#6. Undergraduate Thesis]]
---
## 1. Weekly & Monthly & Quarterly Report
> detail in doc @[[work report]]

### **1.1 九月份，第三季度计划**

完成论文初稿

1.完成论文新思路代码
2.完成论文实验
3.完成论文

1.写好小论文

1.完成论文初稿

1.论文争取赶上10.19的asplos
2.完成除了测试以前的论文内容
3.调通和实现代码，确定方案并做好测试

### **1.2 Part1 Progress in This Week**
1.os book的图形部分修改
2.继续完成capcity和lifetime的测试，完成performance测试，画图
3.修改完一版论文的design部分

1.做trace测试和分解测试（关于capcity和lifetime方面）
2.测试之余，一边重画论文4 Design的图，重写文字

1.完成base pages+delta page的代码

1.对何时触发Delta Page进行建模分析
2.实现加入Delta Page的写入逻辑，读取逻辑做了一半
3.调试on demand和womv重放homes和mail遇到的gc bug

1.全部实现了on demand的womv的代码
2.将以前的trace转为fio格式
3.完成on-demand womv14 naive三个在home mail负载下的重放，得到性能数据。
4.花了较多时间解决on-demand womv重放trace遇到的系列bug (Kernel NULL pointer/oom killer等)

1.网安考试复习
2.添加页面热度感知部分，重构故事和动机；和学长对论文
3.继续实现on demand coding逻辑

1.对womv进行读写混合fio测试
2.跑文件的增/删/重命名等小实验，数据更新在20%内
3.测试组会讨论的屏蔽计算时延方案：加SSD时延/降低并行数，确实能屏蔽计算时延
4.基于组会讨论，研究了fio的负载感知的用法，可以用其重放io trace，定制缓冲区内容。

1.上周把womv的写+gc逻辑完成，这周进一步对其进行性能有关测试，以性能曲线说明其存在的问题，可用于motivation
2.继续将womv的读逻辑实现了

1.用fio对naive盘进行读写等测试
2.实现对比对象WOMv

1.方班辩论赛
2.回顾LightNVM代码，正在实现WOMv和新思路的代码（还未完成


1.阅读方班论文
2.课程考试

1.对CORE的性能时延分解分析，进一步性能测试遇到问题
2.阅读学习femu的时延模型
3.课程任务

1.修改和完善3.4的challanges，构思写了contribution一小段（intro其他还没改）
2.课程任务

1.初步确定和讨论测试思路
2.重新写background和motivation，重新画了前面的所有图，补充了碎片化的motivation数据图

1.完善代码实现，完成周期+弹性编码的改进
2.改fio的bug无法解决，卡了两三天

1.写论文，完成了design&imple的初稿，图和文字

### **1.3 Part2 The difficulties Encountered**
1.和学长定位到性能问题在于写前读，并用iostat替代fio采集性能；
2.但写前读的开销会随写的并发数提高而突显，所有这个方向的工作都有此问题，我只能通过减少并发数来handle，但这会很怪

1.一开始在测试方法和fio上踩坑，和学长交流解决
2.我的代码在部分trace的replay（如mail）时，没跑通

1.womv14和on-demand都没触发gc，但性能曲线都有下降，需要分析原因

1.什么时候使用delta page需要建模分析
2.naive重放mail要1h，womv14重放mail需要5h

1.fio的上述用法可能会对后续测试有帮助，但还是不够灵活；
2.读/写的性能曲线是一致的，还没找到原因

1.组会提到的大容量SSD fio的write跑不过已经解决，是内存泄漏问题
2.读逻辑还没跑fio，可能会有些小问题

1.代码隔得有点久了，存在冷启动问题
2.两者都会带来多级映射表，实现稍微有点复杂

1.课内

1.修改nand的延迟，发现SSD性能的性能没有变化，在找原因
2.课程任务重

1.多个课程的pre、作业和实验等ddl
2.感觉我的challanges不solid

1.下周开始课程任务重了特别多，感觉没什么时间搞论文
2.十月底感觉要赶不上了

1.fio顺序写的bug无法解决

1.之前的问题，跑fio遇到kernel bug，在debug
2.现在实现的代码没搞完，测试方案没确定，学弟那边没法和我并行起来；只能让他先看些文字工作，我论文的design和imple部分

### **1.4 Part3 Plans in Next Week**
1.完成6 Evaluation部分的文字
2.讨论并补充2 Motivation的图，尽量完成Motivation

1.定位问题，跑完测试
2.论文的4 Design部分

1.代码收尾/功能测试/压力测试等工作
2.研究和学长讨论的若干测试方案

1.完成所有的代码
2.开始设计实验，进行相关测试

1.缕清什么时候使用delta page
2.争取这周内把delta page部分代码搞完

1.继续实现on demand coding逻辑
2.做3个动机需要的论据：页面内部的更新分布，性能开销，local wear


1.准备网安考试
2.实现自己改进版本的代码，预计1-2周

1.对WOMv做更丰富的实验：读写混合fio、自定义负载等，说明其存在的问题
2.开始做delta-only的新思路代码的实现

1.继续实现完WOMv和新思路代码
2.对WOMv做一些测试

1.实现完WOMv和新思路代码
2.os作业批改、考试出题

1.方班ppt
2.课程考试实验作业

1.找到femu模拟实验的问题，进一步性能测试
2.完善CORE分解测试，画图写文字

1.按照新的motivation继续完成代码、复现对象和做测试

1.基本上是课程任务
2.按照新的motivation继续完成代码、复现对象和做测试

1.确定测试方案
2.复现wom-v
3.测试一下性能曲线，模拟高性能压缩，测一下时延

1.改好bug，跑通fio
2.继续把该做的代码做完
3.确定一些实验方案，组会讨论一下


---

## 2. TODO Items

### **2.1 Urgent Tasks**
#ddl
- [x] 11.8 并行第三次实验
- [x] 11.9 组会汇报
- [x] 11.15 凸优化考试+作业3+实验2+实验3+阅读报告
- [x] 11.16 英语考试
- [x] 11.19 方班
- [x] 11.25 并行考试

### **2.2 Long-term Tasks**
- **make a summary** once in the morning/afternoon/evening
- read **a paper per day** fast, take quick notes, and conclusion
- **learn English daily,** such as 1. words, 2. grammar, 3. listening, 4. writing, and 5. reading

### **2.3 Flexible Tasks**
  #mark
- [x] learn about and install obsidian dataview
- [x] translate this page into English
- [x] compare the different ways to build blogs and take notes (such as typora, mi notes, notion, zotero, and obsidian)
- [x] get to know and use ipynb
- [x] install code pilot
- [x] get to know ZNS：https://github.com/sg20180546/ZNS-awesome-paper
- [x] get to know MySQL bench：https://zhuanlan.zhihu.com/p/705600890
- [x] Remap SSD阅读

### 2.4 Misc Tasks
  #paper
- [x] 改一个重编程的图
- [x] 完成3.4的challenge
- [x] 完成intro的三点contribution
- [x] 完成introduction
- [x] 解释清楚为什么couple的方案不行?
![[2024-10-11.png]]

  #paper
- [x] 思考womv该怎么实现
- [x] 测一下压缩的延时，变成单纯的截断的速度如何
- [x] 早上将现有代码搬运到lightnvm，实现无损压缩
- [x] 实现全页面压缩的CORE，完善压缩逻辑

 
- [x] 按照新梳理的思路写motivation
- [x] 结合back&moti思路思考一下测试该怎么做
- [x] 修改desgin & imple 的问题
- [x] 弄助教津贴


---

## 4. Paper to Read
Extremely-Compressed SSDs with I/O Behavior Prediction

---

## 5. Previous Progress
> details in doc @progress record
- 
- 24.09.10 18:00 finish giving a presentaion
- 24.09.09 10:00 finish making a presentation draft
- 24.09.07 18:00 update ppt (v3.0), handle some problems mentioned on group meeting, and listen to a report
- 24.09.07 12:00 update ppt (v2.0) deliver a presentation on group meeting
- 24.09.05 18:00 make a simple fragament test for csd
- 24.09.05 10:00 make a ppt (v1.0) for presentation and send to 
- 24.09.02 16:00 Read CASL work， and take notes
- 24.08.26 17:00 Killer review finished
- 24.08.20 21:00 Killer review bg & moti

---

## 6. Undergraduate Thesis

> details in doc @undergraduate thesis

---
