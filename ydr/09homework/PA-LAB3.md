<h1 style="text-align:center; font-weight:bold;">并行处理与体系结构-实验报告3</h1>
<center>杨大荣 | 24S151173 | 电子信息</center>

---

## 0 实验环境
本次实验在T2 210的机房计算机上完成。同时完成单节点和多节点（两台机器）的测试。

---

## 1 Problem 1: Matrix Multiplication using MPI

### 1.1 设计实现
**问题分析** 原问题计算一个M$\times$N矩阵和一个N $\times$ K矩阵的乘法

**串行方法（Serial）** cpu串行的方法在单个节点的单个进程上依次计算结果矩阵C的每一个元素`C[i][j]`。

**MPI方法（MPI）** 基于MPI的矩阵乘法方法参考课件上矩阵向量乘法的MPI，对A矩阵进行分块，分发到多个进程计算，最后再将结果汇总。具体说来，主要逻辑是：
- 1）主进程（rank为0）负责所有的初始化、A矩阵的分块和`MPI_Send`分发操作；
- 2）其他从进程通过`MPI_Recv`接收所负责的A矩阵的数据
- 3）所有进程会进一步完成自己所负责的A子矩阵和B矩阵的计算任务
- 4）其他从进程将计算结果通过`MPI_Send`发送回主进程
- 5）主进程接收所有的结果，并和自己的结果汇总为最终的结果
上述步骤对应到代码的标注示意如下：
```c
    if (rank == 0) {
        // 1）分发A矩阵
        for (i = 1; i < numprocs - 1; i++)
            MPI_Send(A.data() + i * srow * k, srow * k, MPI_FLOAT, i, 0, MPI_COMM_WORLD);
        MPI_Send(A.data() + i * srow * k, srow_last * k, MPI_FLOAT, numprocs - 1, 0, MPI_COMM_WORLD);
        
        // 3）计算矩阵乘法
        std::copy(A.begin(), A.begin() + local_rows * k, localA.begin());
        MatrixMul(localA, B, localResult, local_rows, k ,n);

        // 5）收集所有结果
        mpiResult.resize(m * n);
        std::copy(localResult.begin(), localResult.end(), mpiResult.begin());
        for (i = 1; i < numprocs - 1; i++)
            MPI_Recv(mpiResult.data() + i * srow * n, srow * n, MPI_FLOAT, i, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
        MPI_Recv(mpiResult.data() + i * srow * n, srow_last * n, MPI_FLOAT, numprocs - 1, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
    } else {
        // 2）接收A矩阵
        MPI_Recv(localA.data(), local_rows * k, MPI_FLOAT, 0, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
        // 3）计算矩阵乘法
        MatrixMul(localA, B, localResult, local_rows, k ,n);
        // 4）发送局部计算结果
        MPI_Send(localResult.data(), local_rows * n, MPI_FLOAT, 0, 0, MPI_COMM_WORLD);
    }
```


### 1.2 测试结果

**测试方法**。
- 单机：在T2 210机房中的单台机器测试。选择m,n,k数据量为800,1200,2000。依次设置4、6、8进程数。
- 多机：在T2 210机房中的两台机器测试。数据量和单机相同。每台机器依次设置2、3、4、6进程数。

**测试结果**
- 单机测试结果如下表所示：

|            | 4           | 6           | 8                    |
| ---------- | ----------- | ----------- | -------------------- |
| **Serial** | 10865.00 ms | 10867.90 ms | not enough slots ... |
| **MPI**    | 3045.67 ms  | 2054.00 ms  | not enough slots ... |
- 多机测试结果如下表所示：

|            | 2 2         | 3 3         | 4 4         | 6 6        |
| ---------- | ----------- | ----------- | ----------- | ---------- |
| **Serial** | 10794.80 ms | 11123.40 ms | 11216.40 ms | 11043.5 ms |
| **MPI**    | 3031.89 ms  | 2066.50 ms  | 1597.99 ms  | 1210.56 ms |

**结果分析** 从测试结果中看到：
- 在单台机器和多台机器间，MPI方法的计算速度都比Serial要好
- 随着线程数的增加，MPI方法计算速度提升，但没遵循线性提升，因为多进程带来的通信开销
- 对比单机测试和多机测试，线程数相同的情况下（例如单机的4和多机的2 2），两者的计算速度大致相同。

---


## 2 Problem 2: Ring-based Allreduce
### 2.1 设计实现

**问题分析** 需要利用点对点通信的MPI接口，实现一个基于Ring方式的集合通信Allreduce的接口，至少要实现max和sum两种计算。

**Ring-Based的方法** 核心思路如课件的下图所示：
- 将每个节点的数据按节点数量p均分。
- 每个进程每次发送总数据量m的1/p给下一个进程。
- 重复上述步骤执行2*(p-1)次。
- 其中前(p-1)次，每个进程接收到数据的同时要进行规约，产生最终规约子数据。后(p-1)次，仅需要传递这个数据到各个进程即可。

![[Pasted image 20241114162853.png]]
在具体实现中，一些实现的关键点有：
- 计算好每个进程需要发送数据的起始地址和大小、每个进程需要接收数据的起始地址和大小
- 通过课件中MPI异步发送的方式避免无效等待，每个进程接收到所需要的数据后就开始计算（题中的sum或max运算）
```c
void RING_Allreduce(float *sendbuf, float *recvbuf, int count, MPI_Datatype datatype, MPI_Op op, MPI_Comm comm) {
	... // 初始化工作等
    for (int step = 0; step < (numprocs - 1) * 2; ++step) {
        // 确定进程本次要发送的start end
        int send_size = count / numprocs;
        int send_start = ((rank - step + numprocs * 2) % numprocs) * send_size; // 
        int send_end = send_start + send_size;
        if(send_end + send_size > count){
            send_end = count - 1;
            send_size = send_end - send_start + 1;
        }
        
        // 确定进程本次要接收的start end
        int recv_size = count / numprocs;
        int recv_start = ((rank - (step + 1) + numprocs * 2) % numprocs) * recv_size;
        int recv_end = recv_start + recv_size;
        if(recv_end + recv_size > count){
            recv_end = count - 1;
            recv_size = recv_end - recv_start + 1;
        }

        // 将start end范围数据发给下一个进程
        if(step==0)
            MPI_Isend(&sendbuf[send_start], send_size, datatype, send_to, step, comm, &send_request);
        else
            MPI_Isend(&recvbuf[send_start], send_size, datatype, send_to, step, comm, &send_request);

        // 接收上一个进程传来的数据
        MPI_Irecv(&recvbuf[recv_start], recv_size, datatype, recv_from, step, comm, &recv_request);
        MPI_Wait(&recv_request, &recv_status);

        // 前p-1对接收的数据完成规约
        if(step < numprocs - 1) {
            if(op == MPI_SUM){
                for (int i = 0; i < recv_size; i++){
                    index = recv_start + i;
                    float sendData = sendbuf[index];
                    recvbuf[index] += sendbuf[index];
                }
            }
            if(op == MPI_MAX) {
                for (int i = 0; i < recv_size; i++){
                    index = recv_start + i;
                    float recvData = recvbuf[index];
                    float sendData = sendbuf[index];
                    recvbuf[index] = recvData > sendData ? recvData : sendData;
                }
            }
        }
        MPI_Wait(&send_request, &send_status);
    }
}
```


### 2.2 测试结果

**测试方法**。
- 单机：在T2 210机房中的单台机器测试。选择数据量为100000000，操作为sum。依次设置4、6、8进程数。
- 多机：在T2 210机房中的两台机器测试。选择数据量为10000000（比单机少一个数量级），操作为sum。每台机器依次设置2、3、4、6进程数。

**测试结果**

- 单机测试结果如下表所示：

|          | 4          | 6          | 8                    |
| -------- | ---------- | ---------- | -------------------- |
| **MPI**  | 440.156 ms | 758.87 ms  | not enough slots ... |
| **Ring** | 415.104 ms | 573.748 ms | not enough slots ... |
- 多机测试结果如下表所示：

|          | 2 2        | 3 3        | 4 4        | 6 6        |
| -------- | ---------- | ---------- | ---------- | ---------- |
| **MPI**  | 5125.92 ms | 4242.72 ms | 4810.53 ms | 4769.06 ms |
| **Ring** | 768.078 ms | 766.507 ms | 700.436 ms | 832.372 ms |

**结果分析** 从测试结果中看到：
- 在单台机器上，Ring方法的计算速度仅略好于MPI原始方法（1.32x）
- 在多台机器上，Ring方法的计算速度显著好于MPI原始方法（5.54x）
- 随着线程数的增加，MPI原始方法和Ring方法的计算速度波动，没有明显的提升。考虑为多进程带来的通信开销应该大于规约的计算开销。