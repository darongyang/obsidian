<h1 style="text-align:center; font-weight:bold;">并行处理与体系结构-实验报告1</h1>
<center>杨大荣 | 24S151173 | 电子信息</center>

---

## 0 实验环境
本次实验在个人计算机完成，cpu信息如下：
```
Architecture:          x86_64
CPU op-mode(s):        32-bit, 64-bit
Address sizes:         48 bits physical, 48 bits virtual
Byte Order:            Little Endian
CPU(s):                16
On-line CPU(s) list:   0-15
Vendor ID:             AuthenticAMD
Model name:            AMD Ryzen 7 4800U with Radeon Graphics
CPU family:            23
Model:                 96
Thread(s) per core:    2
Core(s) per socket:    8
Socket(s):             1
Stepping:              1
BogoMIPS:              3593.12

Caches (sum of all):
L1d:                   256 KiB (8 instances)
L1i:                   256 KiB (8 instances)
L2:                    4 MiB (8 instances)
L3:                    4 MiB (1 instance)
```

---

## 1 Problem 1: Matrix Multiplication
### 1.1 设计实现
**问题分析** 原问题计算两个N$\times$N矩阵的乘法。

**串行方法（Serial）** cpu串行的方法在单个节点的单个进程上依次计算结果矩阵C的每一个元素`C[i][j]`。

**MPI方法（MPI）** 基于MPI的矩阵乘法方法参考课件上矩阵向量乘法的MPI，对A矩阵进行分块，分发到多个进程计算，最后再将结果汇总。具体说来，主要逻辑是：1）主进程（）代码的示意如下：
```c
    #pragma omp parallel for private(j,k,sum) num_threads(thread_count)
    for (i = 0; i < N; i++) {
        for (j = 0; j < N; j++) {
            sum = 0;
            for (k=0; k < N; k++) {
                C[i][j] += A[i][k]*B[k][j];
            }
            C[i][j] = sum;
        }
    }
```

**两层循环展开（Double）** 实现的Double方法对两层循环展开，即并行计算结果矩阵中的所有元素（元素间并行）。代码示意如下：
```c
    #pragma omp parallel for private(k,sum) collapse(2) num_threads(thread_count)
    for (i = 0; i < N; i++) {
        for (j = 0; j < N; j++) {
            sum = 0;
            for (k=0; k < N; k++) {
                C[i][j] += A[i][k]*B[k][j];
            }
            C[i][j] = sum;
        }
    }
```


### 1.2 测试结果
![[lab1-task1 2.png]]
**测试方法** 基于上述三种方案，在N=500，N=1000的矩阵下进行测试，分别采用了1、4、16线程进行并行。
**测试结果** 从测试结果中看到：（1）在单线程下，Double和Single方法的计算时间均高于Serial方法；其原因是由于没有并行效果，还额外引入了线程创建和竞争的开销。（2）在多线程（如4、16线程）下，Double和Single方法的计算时间均相对于Serial方法接近关于threads的线性优化，并行的优化效果超过了上述的开销。（3）在各种条件下，Double方法均不如Single方法。其原因可能是核心数较少，Double的并行单元翻倍，并行单元间的竞争过于激烈。

---


## 2 Problem 2: Histogram
### 2.1 设计实现
**问题分析** 统计生成的N个浮点数，落入到不同区间的数量。

**串行方法（Serial）** 原来的Serial方法依次分析每一个数。

**并行方法（Parallel）** 将N个数的循环进行展开，即并行处理这N个数。其中，每个运行线程对各自处理的数，会维护自己统计的结果，记录在sub_bins中。统计任务完成后，每个线程会将自己的统计结果sub_bins互斥加到全局的总体结果bins中。代码示意如下：
```c
    #pragma omp parallel num_threads(num_threads)
    {
        int * sub_bins;
        sub_bins = (int *)calloc(num_bins, sizeof(int)); 
        int idx;
        #pragma omp for
        for (i = 0; i < n; i++) {
            int val = (int)array[i];
            if (val == num_bins) {
                idx = num_bins - 1;
            } else {
                idx = val % num_bins;
            }
            sub_bins[idx]++;
        }
        #pragma omp critical
        {
            for (i = 0; i < num_bins; i++) {
                bins[i] += sub_bins[i];

            }
        }
    }
```


### 2.2 测试结果
![[lab1-task2 1.png]]
**测试方法** 基于上述两种方案，在N=100 million，N=500 million的数据下进行测试，分别采用了1、4、16线程进行并行。
**测试结果** 测试结果可以看出：（1）在单线程条件下，Parallel方法的时间同样是高于Serial方法；（2）在多线程条件下，Parallel方法的时间相对于Serial方法接近关于threads的线性优化。