<h1 style="text-align:center; font-weight:bold;">并行处理与体系结构-实验报告4</h1>
<center>杨大荣 | 24S151173 | 电子信息</center>

---

## 0 实验环境
本次实验在T2 210的机房计算机上完成。同时完成4个节点（4台机器）的测试。

---

## 1 Problem 1: Using MPI and CUDA

### 1.1 设计实现

**问题分析** 原问题需要在多个节点间并行计算如下的问题，其中A和B均为2进制矩阵：
$$
C = (A \otimes B^m) \oplus (A \otimes B^{m-1}) \oplus (A \otimes B^{m-2}) \oplus \dots \oplus (A \otimes B^1) \oplus A,
$$

**串行方法（Serial）** cpu串行的方法在单个节点的单个进程上，按照上面的式子，依次完成m步串行的$A \otimes B^m$ 的迭代计算。其中每一步的$B^m$ 可以由上一步的$B^{m-1}$计算得到。过程中将每一步的结果逐步累加到结果矩阵C中。

**MPI+CUDA方法（Parallel）** 
**（1）MPI并行部分**。参考课件上矩阵向量乘法和Lab3的MPI方法，可以使用MPI对上述的A矩阵进行分块，分发到多个节点上计算，最后再将结果进行汇总。这一部分很常规，示例代码如下：
```cpp
    if (rank == 0) {
      int i;
      // 主进程 发送A矩阵
      for (i = 1; i < number_of_processes - 1; i++) {
          MPI_Send(h_A.data() + i * srow * n, srow * n, MPI_INT, i, 0, MPI_COMM_WORLD);
      }
      MPI_Send(h_A.data() + i * srow * n, srow_last * n, MPI_INT, number_of_processes - 1, 0, MPI_COMM_WORLD);
      // 主进程 cuda计算矩阵乘法
      std::copy(h_A.begin(), h_A.begin() + local_rows * n, h_localA.begin());
      gpuSetupAndCompute(local_rows, n, m, h_B.data(), h_localA.data(), h_localC.data(), number_of_thread_in_a_block, number_of_block_in_a_grid);
      // 主进程 收集所有结果
      h_C.resize(n * n);
      std::copy(h_localC.begin(), h_localC.end(), h_C.begin());
      for (i = 1; i < number_of_processes - 1; i++){
        MPI_Recv(h_C.data() + i * srow * n, srow * n, MPI_INT, i, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
      }
      MPI_Recv(h_C.data() + i * srow * n, srow_last * n, MPI_INT, number_of_processes - 1, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
    else {
      // 从进程 接收A矩阵
      MPI_Recv(h_localA.data(), local_rows * n, MPI_INT, 0, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
     // 从进程 cuda计算矩阵乘法
      gpuSetupAndCompute(local_rows, n, m, h_B.data(), h_localA.data(), h_localC.data(), number_of_thread_in_a_block, number_of_block_in_a_grid);
      // 从进程 发送局部计算结果
      MPI_Send(h_localC.data(), local_rows * n, MPI_INT, 0, 0, MPI_COMM_WORLD);
    }
```

**（2）CUDA并行部分**。基于目标式子，为了让步骤更加直观，我们将其等价的拆分成两个矩阵D、C迭代的形式。
- 对于矩阵D满足：$D_{k}=D_{k-1}\otimes B(k=1\dots m)$，其中$D_{0}=A$。
- 对于矩阵C满足：$C_{k}=C_{k-1}\oplus D_{k}(k=1\dots m)$，其中$C_{0}=A$。
- 最终结果输出为：$C_{m}$。

上述一共需要m步的迭代，而每次迭代需要依次计算$D_{k}$和$C_{k}$，其中$C_{k}$的计算需要等待$D_{k}$计算完成。基于此，cuda并行部分实现如下所示。
**cuda初始化和主循环逻辑**。主函数中采用了多次迭代的计算，调用m次cuda核函数完成m步串行迭代。而由于块数和线程数的大小受限制，每次$D_{k}$（或$C_{k}$）的计算只能load其中的一部分，要反复load直到算完整个$D_{k}$（或$C_{k}$）。其中每一轮迭代的计算中，$C_{k}$的计算依赖于$D_{k}$，需要进行一个同步。示例代码如下：
```cpp
  int x0, y0;
  int sqrt_in_blk = ceil(sqrt(number_of_thread_in_a_block));
  int sqrt_in_grid = ceil(sqrt(number_of_block_in_a_grid));
  int delta = sqrt_in_blk * sqrt_in_grid;
  dim3 blockSize(sqrt_in_blk, sqrt_in_blk);
  dim3 gridSize(sqrt_in_grid, sqrt_in_grid);
  // 迭代串行计算m次
  for(int round = 1; round <= m; round ++) {
    CHECK(cudaMemcpy(d_D_backup, d_D, na * n * sizeof(int), cudaMemcpyDeviceToDevice));
    x0 = 0;
    y0 = 0;
    // 反复load计算Dk
    while(x0 < n && y0 < n){
      gpuCompute_D<<<gridSize, blockSize>>>(na, n, d_B, d_D, d_D_backup, x0, y0);
      y0 += delta;
      if(y0 >= n) {
        y0 = 0;
        x0 += delta;
      }
    }
    cudaDeviceSynchronize();
    // 反复load计算Ck 等待Dk算好
    x0 = 0;
    y0 = 0;
    while(x0 < n && y0 < n){
      gpuCompute_C<<<gridSize, blockSize>>>(na, n, d_B, d_C, d_D, x0, y0);
      y0 += delta;
      if(y0 >= n) {
        y0 = 0;
        x0 += delta;
      }
    }
    cudaDeviceSynchronize();
  }
```

**计算矩阵Dk**。矩阵$D_{k}$的计算逻辑较为简单，依据上一次所存储的$D_{k-1}$的值和矩阵B进行一次乘法操作。计算从上一次结束的地方(x0,y0)的偏移开始。
```cpp
__global__ void gpuCompute_D(int n_A, int n, int* B, int* D, int* D_backup, int x0, int y0) {
  // D Index that cuda core handle
  int x = blockIdx.x * blockDim.x + threadIdx.x + x0;
  int y = blockIdx.y * blockDim.y + threadIdx.y + y0;
  int D_sum = 0;
  // 确保cuda core合法
  if(x < n_A && y < n) {
    // 计算D(k)矩阵 利用D(k-1)和B
    D_sum = 0;
    for (int k = 0; k<n; k++) {
      D_sum = (D_sum + D_backup[x * n + k] * B[k * n +y]) % 2;
    }
    // 更新D
    D[x * n + y] = D_sum;
  }
}
```

**计算矩阵Ck**。矩阵$C_{k}$的计算逻辑较为简单，依据本轮所计算得到的$D_{k}$的值和自身的旧值$C_{k-1}$进行一次加法操作即可。计算从上一次结束的地方(x0,y0)的偏移开始。
```cpp
__global__ void gpuCompute_C(int n_A, int n, int* B, int* C, int* D, int x0, int y0) {
  // C/D Index that cuda core handle
  int x = blockIdx.x * blockDim.x + threadIdx.x + x0;
  int y = blockIdx.y * blockDim.y + threadIdx.y + y0;
  int C_sum = 0;
  // 确保cuda core合法
  if(x < n_A && y < n) {
    // 计算C(k)矩阵 利用C(k-1)和D(k)
    C_sum = (C[x * n + y] + D[x * n + y]) % 2;
    C[x * n + y] = C_sum;
  }
}
```

### 1.2 测试结果

**测试方法**。
在T2 210机房中的四台机器中测试。每台机器最大设置6个进程数。修改makefile依次测试了总进程数为4、8和16的情况，测试结果如下表所示：

**测试结果**
- 总进程数为4：

|              | test_large_1 | test_large_2 | test_large_3 |
| ------------ | ------------ | ------------ | ------------ |
| **Serial**   | 0.19259 s    | 0.76895 s    | 1.49595 s    |
| **Parallel** | 1.91196 s    | 16.5762 s    | 15.6089 s    |
| **Rate**     | 9.9270x      | 21.556x      | 10.4341x     |
- 总进程数为8：

|              | test_large_1 | test_large_2 | test_large_3 |
| ------------ | ------------ | ------------ | ------------ |
| **Serial**   | 0.25458 s    | 0.913269 s   | 1.49595 s    |
| **Parallel** | 2.01807 s    | 13.8752 s    | 15.1929 s    |
| **Rate**     | 7.92692x     | 15.1929x     | 11.8388x     |
- 总进程数为16：

|              | test_large_1 | test_large_2 | test_large_3 |
| ------------ | ------------ | ------------ | ------------ |
| **Serial**   | 0.35085 s    | 0.72965 s    | 1.40589 s    |
| **Parallel** | 2.05173 s    | 14.5783 s    | 15.5702 s    |
| **Rate**     | 5.8479x      | 19.9798x     | 11.0749x     |

**结果分析** 从测试结果中看到：
- 使用mpi和cuda的并行方法，总会比cpu的串行方法快，但受限于并行的单元数，加速比在5-20x
- 对比不同数据集，在test_large_2上的优化效果最好（达到20x），在test_large_1上的优化效果最小（5-10x）

---
