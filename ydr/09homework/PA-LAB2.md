<h1 style="text-align:center; font-weight:bold;">并行处理与体系结构-实验报告2</h1>
<center>杨大荣 | 24S151173 | 电子信息</center>

---

## 0 实验环境
本次实验在T2 210的机房计算机上完成，cpu和gpu的相关信息如下：

| cpu                       | gpu                   | cuda         |
| ------------------------- | --------------------- | ------------ |
| Intel(R) Core(TM) i7-8700 | NVIDIA GeForce GT 730 | Version 11.4 |


---

## 1 Problem 1: Matrix Vector Multiplication
### 1.1 设计实现

**问题分析** 原问题计算一个M$\times$N矩阵和一个N $\times$ 1向量的乘法。

**CPU方法（cpu）** 原来的cpu串行方法会依次计算结果向量V的每一个元素`V[i]`。

**GPU全局内存方法（gpu-gm）** 使用CUDA编程，让一个线程计算结果向量的一个元素。直接通过分配的全局内存`d_matrix`和`d_vector`进行访存操作，并将结果写回全局内存`d_result`中。
```c
__global__ void gpuMVectorMultiplyGM(float* d_matrix, float* d_vector, float* d_result, int Row, int Col) {
    int idx = blockIdx.x * blockDim.x + threadIdx.x;
    if (idx < Row) {
        float sum = 0.0;
        for (int j = 0; j < Col; ++j) {
            sum += d_matrix[idx * Col + j] * d_vector[j];
        }
        d_result[idx] = sum;
    }
}
```

**GPU共享内存方法（gpu-sm）** **1）动机分析**。对于一个块，块内线程计算的时候，列向量是可以复用的，可以将其加载到共享内存中进行加速访存。**2）具体实现**。块内的线程每次搬运大小为`TILE_WIDTH`的列向量区间，然后计算这个区间的中间结果并累加，计算完整个列向量后将总结果写回全局内存。**3）两次同步）**。其中为了保证数据正确性，需要两次同步。第一次同步是为了确保所有线程都将本次数据搬迁到位；第二次同步是为了确保所有线程都完成本次计算，方可进行下一轮的数据搬迁。**4）边界处理**。值得注意的是，矩阵大小和列向量大小可能不会和块内线程数保持对齐，导致有的块中实际计算线程（满足`idx<Row`）小于预期的`TILE_WIDTH`。这个时候如果只用计算线程搬迁列向量无法每次搬迁`TILE_WIDTH`的大小导致数据错误。因此注意：在搬迁内存时，块内所有`TILE_WIDTH`个线程都可以搬迁列向量；但在计算时，只满足`idx<Row`的线程才可以进行计算。
```c
__global__ void gpuMVectorMultiplySM(float* d_matrix, float* d_vector, float* d_result, int Row, int Col) {
    __shared__ float sub_vec[TILE_WIDTH];
    int idx = blockIdx.x * blockDim.x + threadIdx.x;
    float sum = 0.0;
    // 每次滑动一个TILE_WIDTH
    for (int m = 0; m < Col; m += TILE_WIDTH) {
        // 并行内存搬运 边界检查
        if(m + threadIdx.x < Col)
           sub_vec[threadIdx.x] = d_vector[m + threadIdx.x];
        // 所有warp同步一下
        __syncthreads();
        // 线程边界检查
        if (idx < Row) 
            // 所有的线程开始计算本区间的值
            for (int n = 0; n < TILE_WIDTH; ++n) 
                if (m + n < Col) 
                    sum += sub_vec[n] * d_matrix[idx * Col + m + n];
        // 所有warp都算完了才能进入下一个
        __syncthreads();
    }

    // 把sum写回全局内存
    if (idx < Row) 
        d_result[idx] = sum;
}
```


### 1.2 测试结果

|            | 1k x 1k  | 10k x 10k  |
| ---------- | -------- | ---------- |
| **cpu**    | 4.277 ms | 423.102 ms |
| **gpu-gm** | 1.673 ms | 129.521 ms |
| **gpu-sm** | 1.432 ms | 121.858 ms |

**测试方法** 分别在1000 $\times$ 1000和10000 $\times$ 10000的矩阵规模，测试了上述三种方法，结果如上表所示。
**测试结果** 从测试结果中看到：
- 全局内存方法gpu-gm能够在上述两种矩阵规模，分别是cpu速度的2.55x和3.27x
- 共享内存方法gpu-sm比gpu-gm方法略好，但速度相当。考虑原因是：1.列向量的复用效果有限；2.频繁的同步以及未考虑合并访存和bank conflict等问题，弱化了共享内存的优化效果。

---


## 2 Problem 2: Matrix Transpose
### 2.1 设计实现

**问题分析** 将一个M$\times$N的矩阵进行转置。这个问题不涉及计算，只涉及访存优化。

**CPU方法（cpu）** 原来的cpu串行方法会依次转置结果矩阵T的每一个元素`T[i][j]`。

**GPU全局内存方法（gpu-gm）** CUDA编程，让一个线程转置结果矩阵的一个元素。直接通过分配的全局内存`input`进行访存操作，并将结果写回全局内存`result`中。
```c
__global__ void gpuTransGM(float* input, float* result,
     int row, int col) {
    int x = blockIdx.x * blockDim.x + threadIdx.x;
    int y = blockIdx.y * blockDim.y + threadIdx.y;
    if (x < col && y < row) {
        result[x * row + y] = input[y * col + x];
    }
}
```


**GPU共享内存方法（gpu-sm）** **1）动机分析**。上述全局内存的方法无法同时实现读合并访存和写合并访存（如上述的实现中是读合并访存，但不是写合并访存）。我们可以共享内存充当缓冲区，同时实现全局内存的读合并访存和写合并访存。**2）具体实现**。每个块完成转置任务时，声明一个共享内存share。块内线程读取全局内存input中其负责的转置部分到共享内存share中，全局内存input遵循按行读（即，读合并访问），共享内存share是按列写（缓冲区非合并，没关系）。进行同步，将整个share都转置/搬运完成后。将共享内存share的结果按列读取（缓冲区非合并，但没关系），并按行写入（即，写合并访问）到全局内存input中。**3）bank conflict**。通过对共享内存添加一个padding列来重排内存布局来避免共享内存中的bank冲突。
```c
// GPU shared mem 矩阵转置
__global__ void gpuTransSM(float* input, float* result,
     int row, int col) {
    __shared__ float share[TILE_WIDTH][TILE_WIDTH + 1]; 
    int x = blockIdx.x * TILE_WIDTH + threadIdx.x;
    int y = blockIdx.y * TILE_WIDTH + threadIdx.y;
  
    if (x < col && y < row) 
        // 从全局内存读取的时候按x优先
        share[threadIdx.y][threadIdx.x] = input[y * col + x];

    // 所有warps全部读取到shared mem完成
    __syncthreads();

    x = blockIdx.y * TILE_WIDTH + threadIdx.x;
    y = blockIdx.x * TILE_WIDTH + threadIdx.y;


    if (x < row && y < col) 
        // 往全局内存写的时候按x优先
        result[y * row + x] = share[threadIdx.x][threadIdx.y];

}
```

### 2.2 测试结果

|            | 1k x 1k  | 10k x 10k  |
| ---------- | -------- | ---------- |
| **cpu**    | 6.341 ms | 582.822 ms |
| **gpu-gm** | 1.356 ms | 78.132 ms  |
| **gpu-sm** | 0.636 ms | 42.920 ms  |

**测试方法** 基于上述三种方案，在分别在1000 $\times$ 1000和10000 $\times$ 10000的矩阵规模完成了测试，结果如上表所示。
**测试结果** 测试结果可以看出：
- 全局内存的方法gpu-gm在两种矩阵规模下，速度分别是cpu的4.68x和7.46x。
- 共享内存的方法gpu-sm分别是gpu-gm的2.13x和1.82x

---


## 3 Problem 3: Parallel Convolution
### 3.1 设计实现
**问题分析** 计算一个M$\times$N的矩阵和一个K$\times$K的卷积核的卷积。经过调研，卷积核的大小通常是较小的奇数，K常取值为3和5。

**CPU方法（cpu）** 不考虑卷积核的翻转、矩阵的padding等，实现cpu版本的卷积操作如下。原来的cpu串行方法会依次计算卷积结果矩阵的每一个元素。
```c
void cpuConv(const std::vector<int> &image, const std::vector<int> &kernel, std::vector<int> &output, int Row, int Col, int K) {
    for(int k=0; k<=Row - K; k++)  //特征平面的行 列平移 行卷积
    {
        for(int r=0;r<=Col - K; r++) //特征平面的列 行平移 列卷积
        {
            int sum = 0;
            //单次卷积 点对点相乘 然后相加
            for(int i=0;i<K;i++) //卷积的行
                for(int j=0;j<K;j++) //卷积的列
                    sum +=  image[(i+k)*Col + j+r]*kernel[i*K+j];
            output[k * (Col - K + 1) + r] = sum;
        }
    }
    return ;
}
```

**GPU全局内存方法（gpu-gm）** 基于上述cpu方法进行CUDA编程，让一个线程计算卷积结果矩阵的一个元素。直接通过分配的全局内存`image`和`kernel`进行访存操作，并将结果写回全局内存`output`中。
```c
__global__ void gpuConvGM(int *image, int *kernel, int *output, int Row, int Col, int K) {
    int idx = blockIdx.x * blockDim.x + threadIdx.x;
    int idy = blockIdx.y * blockDim.y + threadIdx.y;

    if( (idx >= 0 && idx <= Row - K) &&  (idy >= 0 && idy <= Col - K)) {
        int sum = 0;
        for (int m = 0; m < K; ++m)
            for (int n = 0; n < K; ++n) 
                sum +=  image[(idx + m) * Col + idy + n] * kernel[m * K + n];
        output[idx * (Col - K + 1) + idy] = sum;
    }
}
```


**GPU共享内存方法（gpu-sm）** **1）动机分析**。对于一个块，块内线程计算的时候，卷积核是可以复用的，可以将其加载到共享内存中进行加速访存。**2）具体实现**。定义一个共享内存share，在计算前所有线程先将卷积核加载到共享内存share中。考虑到卷积核较小，这里假定所静态声明的共享内存可以存入卷积核，而不对卷积核进行分块操作。
```c
__global__ void gpuConvSM(int *image, int *kernel, int *output, int Row, int Col, int K) {
    __shared__ int shared[TILE_WIDTH][TILE_WIDTH];
    int idx = blockIdx.x * blockDim.x + threadIdx.x;
    int idy = blockIdx.y * blockDim.y + threadIdx.y;

    // 每个线程负责搬运一下本次的kernel
    if(threadIdx.x < K && threadIdx.y < K)
        shared[threadIdx.y][threadIdx.x] = kernel[threadIdx.y * K + threadIdx.x];
        
    __syncthreads();

    if( (idx >= 0 && idx <= Row - K) &&  (idy >= 0 && idy <= Col - K)) {
        int sum = 0;
        for (int m = 0; m < K; ++m) 
            for (int n = 0; n < K; ++n) 
                sum += image[(idx + m) * Col + idy + n] * shared[m][n];

        output[idx * (Col - K + 1) + idy] = sum;
    }

}
```


### 3.2 测试结果

|                        | 1k x 1k, 3 | 10k x 10k, 3 |
| ---------------------- | ---------- | ------------ |
| **cpu**                | 44.34 ms   | 4312.41 ms   |
| **gpu-gm (blk:32x32)** | 11.83 ms   | 1180.56 ms   |
| **gpu-sm (blk:32x32)** | 11.50 ms   | 1138.15 ms   |
| **gpu-gm (blk:16x16)** | 6.11 ms    | 597.62 ms    |
| **gpu-sm (blk:16x16)** | 5.93 ms    | 578.93 ms    |
| **gpu-gm (blk:8x8)**   | 3.67 ms    | 353.96 ms    |
| **gpu-sm (blk:8x8)**   | 3.52 ms    | 342.87 ms    |

**测试方法** 基于上述三种方案，分别在1000 $\times$ 1000和10000 $\times$ 10000的矩阵规模以及K=3的卷积核规模下进行了测试，对于gpu方案，根据共享内存的大小（即，块内的线程数），采用了几种变体。其中blk:32 $\times$ 32表明线程块为线程数为32 $\times$ 32的二维结构。不同的共享内存大小能够容纳不同上限的卷积核。
**测试结果** 测试结果可以看出：
- 当blk为32 $\times$ 32时，全局内存方法gpu-gm在不同的矩阵规模下的性能，分别是cpu的3.74x和3.65x
- 优化效果随着线程块中的线程数的减小而越好
- 共享内存gpu-sm的方法相对于gpu-gm有优化，但不显著，约1.036x。考虑原因和problem1一致，卷积核规模较小，并且没有额外考虑合并访存和bank conflict等问题，优化效果不显著。