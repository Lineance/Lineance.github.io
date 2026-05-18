+++
date = '2025-12-17'
draft = false
title = '高性能计算：阶乘'
slug = 'hpc-factorial-optimization'
author = 'lensit'
tags = ['study','hpc', 'cpp', 'cuda', 'algorithm']
showtoc = true
showrightsidebar = true
rightSidebarWidgets = ["profile", "tags", "meta"]
sidebarAuthor = "lineance"
sidebarBio = "Happy Learning"
sidebarAvatar = "/img/avatar_study.webp"
math = true
+++

## 引言

阶乘，可以说是数学计算中最容易实现的一个程序。在学习循环时，我们必定会写过这样的代码：

```cpp
int factorial(int n) {
    int result = 1;
    for (int i = 2; i <= n; i++) {
        result *= i;
    }
    return result;
}
```

然而，这个简单的实现很快就遇到了瓶颈。在C++中，`int`类型只能计算到12!（479,001,600），`long long`也只能计算到20!（2,432,902,008,176,640,000）。当我们需要计算更大的阶乘时，传统数据类型就无能为力了。

这引出了几个有趣的问题：**如何高效地计算大数乘法？** 和 **如何高效地计算大数阶乘？**

## 1. 如何计算大数乘法？

### 1.1 朴素的高精度实现

当需要计算更大的阶乘时，我们首先想到的是使用数组来表示大数。这种思想来源于我们小学学习的竖式计算方法：

```cpp
class HugeInteger {
private:
    vector<int> digits;  // 存储每一位数字
    
public:
    HugeInteger operator*(const HugeInteger& other) const {
        // 实现竖式乘法
        HugeInteger result;
        result.digits.resize(digits.size() + other.digits.size(), 0);
        
        for (size_t i = 0; i < digits.size(); i++) {
            for (size_t j = 0; j < other.digits.size(); j++) {
                result.digits[i + j] += digits[i] * other.digits[j];
                result.digits[i + j + 1] += result.digits[i + j] / 10;
                result.digits[i + j] %= 10;
            }
        }
        
        return result;
    }
};
```

这种方法的时间复杂度是O(n²)，其中n是数字的位数。虽然可以计算任意长度的数字，但效率较低。

### 1.2 压位优化

由于CPU的ALU（算术逻辑单元）在处理32位或64位整数时效率最高，我们可以将多个十进制位压缩到一个数组元素中：

```cpp
HugeInteger operator*(const HugeInteger& other) const {
    HugeInteger result;
    result.digits.resize(digits.size() + other.digits.size(), 0);
    
    for (size_t i = 0; i < digits.size(); i++) {
        long long carry = 0;
        for (size_t j = 0; j < other.digits.size() || carry; j++) {
            long long cur = result.digits[i + j] 
                          + digits[i] * (j < other.digits.size() ? other.digits[j] : 0) 
                          + carry;
            result.digits[i + j] = cur % BASE;
            carry = cur / BASE;
        }
    }
    
    // 去除前导零
    while (result.digits.size() > 1 && result.digits.back() == 0)
        result.digits.pop_back();
    
    return result;
}
```

通过压位优化，我们将乘法次数减少了约9倍（以10^9为基数），显著提高了计算效率。

## 2. 如何高效地计算大数乘法

### 2.1 大数乘法的本质

大数乘法本质上是多项式卷积运算。设有两个大整数：

$$
A = \sum_{i=0}^{n-1} a_i \cdot B^i, \quad B = \sum_{i=0}^{m-1} b_i \cdot B^i
$$

其中B是基数（如10^9）。

乘积C = A × B可表示为：
$$
C_k = \sum_{i+j=k} a_i b_j
$$

### 2.2 FFT的魔法

快速傅里叶变换（FFT）可以将卷积运算的时间复杂度从O(n²)降低到O(n log n)。基本思想是：

1. 将系数表示转换为点值表示（FFT）
2. 在点值表示下进行乘法运算
3. 将结果转换回系数表示（IFFT）

```cpp
void fft(vector<Complex>& a, int limit, int type) {
    // 位反转置换
    for (int i = 0; i < limit; i++) {
        if (i < rev[i]) swap(a[i], a[rev[i]]);
    }
    
    // 蝴蝶操作
    for (int mid = 1; mid < limit; mid <<= 1) {
        Complex wn(cos(PI / mid), type * sin(PI / mid));
        for (int left = 0; left < limit; left += (mid << 1)) {
            Complex w(1, 0);
            for (int k = 0; k < mid; k++) {
                Complex x = a[left + k];
                Complex y = w * a[left + k + mid];
                a[left + k] = x + y;
                a[left + k + mid] = x - y;
                w *= wn;
            }
        }
    }
    
    if (type == -1) {
        for (int i = 0; i < limit; i++) {
            a[i] /= limit;
        }
    }
}
```

### 2.3 三次变两次优化

标准FFT乘法需要三次变换（两次FFT，一次IFFT）。通过巧妙的编码，我们可以减少到两次：

1. 将两个实数序列编码到一个复数序列：$P_i = a_i + i \cdot b_i$
2. 计算P的FFT
3. 计算$P(\omega)^2$，其虚部包含卷积结果
4. 对结果进行IFFT并提取虚部的一半

这种优化将复杂度从3n log n降低到2n log n。

## 3. 如何高效地计算大数阶乘

### 引子

聪明的你肯定能想到，为什么我不头尾一乘，将循环次数减半呢？

```cpp
for (int i = 1; i <= n / 2; i++) {
    result *= i * (n - i + 1); 
}
if (n % 2 == 1) result *= (n + 1) / 2;  // 奇数时补中间的项
```

再深入思考一下，既然能把序列从中间配对折叠，那为何不干脆将其一刀两断，让左半段和右半段各自先算出局部乘积，再将两个结果相乘？左半段继续二分，右半段也继续二分……

### 3.1 分治法（二进制分割）

分治法是计算阶乘的经典方法。核心思想是将连续整数序列递归地分成两部分：

$$
n! = \left(\prod_{k=1}^{\lfloor n/2 \rfloor} k\right) \times \left(\prod_{k=\lfloor n/2 \rfloor+1}^{n} k\right)
$$

递归实现如下：

```cpp
void factorial_binary_splitting(mpz_t result, unsigned long n, unsigned long m) {
    if (n - m == 0) {
        mpz_set_ui(result, 1);
        return;
    }
    if (n - m == 1) {
        mpz_set_ui(result, n);
        return;
    }
    
    unsigned long mid = m + (n - m) / 2;
    mpz_t high, low;
    mpz_init(high);
    mpz_init(low);
    
    factorial_binary_splitting(high, n, mid);
    factorial_binary_splitting(low, mid, m);
    
    mpz_mul(result, high, low);
    
    mpz_clear(high);
    mpz_clear(low);
}
```

复杂度分析：如果使用FFT乘法，复杂度为$O(n \log^2 n)$。

### 3.2 PrimeSwing算法

PrimeSwing算法是目前工程中速度最快的阶乘算法之一，由Peter Luschny于2008年提出。核心思想基于以下数学原理：

**Swing数定义**：
$$
S(n) = \prod_{3 \leq p \leq n} p^{e(p,n)}
$$
其中$e(p,n) = v_p(n!) - 2v_p(\lfloor n/2 \rfloor!)$

**主定理**：
$$
\operatorname{odd}(n!) = \operatorname{odd}(\lfloor n/2 \rfloor!)^2 \cdot S(n)
$$

**阶乘公式**：
$$
n! = \operatorname{odd}(\lfloor n/2 \rfloor!)^2 \cdot S(n) \cdot 2^{n - \operatorname{Popcount}(n)}
$$

算法实现：

```cpp
class FactorialPrimeSwingOptimized {
private:
    std::vector<unsigned long> primes;
    
    void swing(mpz_t result, unsigned long n) {
        mpz_set_ui(result, 1);
        for (unsigned long p : primes) {
            if (p > n) break;
            
            long exponent = 0;
            unsigned long temp = n;
            while (temp > 0) {
                temp /= p;
                exponent += temp;
            }
            temp = n / 2;
            while (temp > 0) {
                temp /= p;
                exponent -= 2 * temp;
            }
            
            if (exponent > 0) {
                mpz_t pow_val;
                mpz_init(pow_val);
                mpz_ui_pow_ui(pow_val, p, exponent);
                mpz_mul(result, result, pow_val);
                mpz_clear(pow_val);
            }
        }
    }
    
    void rec_factorial(mpz_t result, unsigned long n) {
        if (n < 2) {
            mpz_set_ui(result, 1);
            return;
        }
        
        mpz_t rec_half;
        mpz_init(rec_half);
        rec_factorial(rec_half, n / 2);
        
        mpz_t swing_val;
        mpz_init(swing_val);
        swing(swing_val, n);
        
        mpz_mul(result, rec_half, rec_half);
        mpz_mul(result, result, swing_val);
        
        mpz_clear(rec_half);
        mpz_clear(swing_val);
    }
    
public:
    FactorialPrimeSwingOptimized(unsigned long max_n) {
        primes = sieve_primes(max_n);
    }
    
    void factorial(mpz_t result, unsigned long n) {
        if (n < 20) {
            mpz_set_ui(result, 1);
            for (unsigned long i = 2; i <= n; i++) {
                mpz_mul_ui(result, result, i);
            }
            return;
        }
        
        rec_factorial(result, n);
        unsigned long shift = n - __builtin_popcountll(n);
        mpz_mul_2exp(result, result, shift);
    }
};
```

复杂度分析：$O(n \log^2 n \log \log n)$

## 4. 如何更高效地计算大数阶乘 - CPU

### 4.1 GMP库：CPU上的高性能计算

GNU多精度算术库（GMP）是CPU上最高效的大数运算库。

#### mpz_mul 大数乘法算法栈

GMP的乘法实现是一个自适应算法栈，根据操作数大小自动选择最优算法：

| 区间（limb数） | 算法 | 复杂度 | 备注 |
|------|------|--------|------|
| 1×1 | 单limb汇编 | O(1) | 手写汇编 |
| 2×2 ~ 8×8 | 朴素竖式（Basecase） | O(n²) | 完全展开，无循环分支 |
| 9×9 ~ 30×30 | Toom-2.5 | O(n^2.2) | 分3段，5次乘法 |
| 30 ~ 120 | Toom-3 | O(n^1.46) | 分5段，9次乘法 |
| 120 ~ 600 | Toom-4/6 | O(n^1.40) | 段数继续增加 |
| ≥600且满足阈值 | Schönhage–Strassen FFT | O(n log n log log n) | 模2^ω+1，位打包 |
| ≥超大阈值（万limb级） | Nussbauder分圆卷积 | O(n log n) | 纯整数FFT，避免浮点 |

> 阈值随CPU、缓存、字长变化，运行时可自动调优（tuneup）。

分治调用链示例：

```bash
mpz_mul(n=10000)
└─ mpn_mul(an=10000, bn=10000)
   ├─ 若 an≈bn ≥ FFT_THRESHOLD → mpn_fft_mul
   │   ├─ 位打包
   │   ├─ 三层 FFT（蝴蝶）
   │   └─ 分量乘 → 递归 mpn_mul（小尺寸）
   └─ 否则 → mpn_toom44_mul
       ├─ 5 段插值
       ├─ 9 次点乘 → 递归 mpn_mul（更小）
       └─ 插值逆变换
```

每层"点乘"都可能是更小的Toom、Karatsuba或Basecase，形成自适应树。

#### mpz_fac_ui 阶乘实现分析

GMP的阶乘实现（mpz_fac_ui）采用了精巧的四步流程：

| 步骤 | 描述 |
|------|------|
| ① 奇偶分离 | 只计算**奇部**（odd part），2的幂次单独用移位处理 |
| ② 规模判断 | `n < 阈值` → "小n算法"；否则 → "大n算法" |
| ③ 二进制分割 | 两条路径最终都把"大量小因子"做成二进制分割乘积 |
| ④ 移位补偿 | 根据`popcount(n/2)`算出缺失的2^k，一次性左移补回 |

**① 奇偶分离**：阶乘中包含大量因子2，直接计算会浪费大量乘法。GMP将阶乘拆分为奇部和2的幂次：
$$
n! = \operatorname{odd}(n!) \cdot 2^{v_2(n!)}
$$
其中$v_2(n!) = n - \operatorname{Popcount}(n)$，只计算奇部，最后用移位补回。

**② 规模判断**：

- **小n算法**（fac_small）：阈值约30~128，非递归，用循环把"奇数列表"一次性攒到栈数组，调用`fac_prod()`做二进制分割乘积
- **大n算法**（fac_large）：基于PrimeSwing递归分解，递归深度≈log₂n，叶子落到"小n算法"

**③ 二进制分割**（fac_prod）：这是GMP阶乘的核心，将大量小因子分治相乘：

```c
if (n ≤ MUL_THRESHOLD)          // 很小就直接连乘
    return basecase_mul(vec, n);
mid = n >> 1;
left  = fac_prod(vec, mid);
right = fac_prod(vec + mid, n - mid);
return left * right;            // 大数乘法，自动选 Karatsuba/Toom/FFT
```

乘法次数从O(n)降到O(log n)，顶层一次超大乘法占主要工时。

**④ 移位补偿**：递归计算奇部时，2的幂次被剥离。最终需要补回：

- 2的幂总数：$v_2(n!) = n - \operatorname{Popcount}(n)$
- 递归已含：$v_2(\lfloor n/2 \rfloor!^2) = 2 \cdot (\lfloor n/2 \rfloor - \operatorname{Popcount}(\lfloor n/2 \rfloor))$
- 尚缺：$\Delta = \operatorname{Popcount}(\lfloor n/2 \rfloor)$

因此只需最后把奇部左移`Popcount(n/2)`位即可。


#### 代码示例

```cpp
#include <gmp.h>

int main() {
    mpz_t result;
    mpz_init(result);
    
    // 计算10000!
    mpz_fac_ui(result, 10000);
    
    gmp_printf("10000! = %Zd\n", result);
    
    mpz_clear(result);
    return 0;
}
```

### 4.2 OpenMP多线程并行

OpenMP是CPU上最常用的并行编程模型。在阶乘计算中，我们可以并行化多个独立的计算任务：

```cpp
#include <omp.h>

void factorize_factorial_multithread(unsigned int n,
                                     const std::vector<unsigned int>& primes,
                                     std::vector<unsigned int>& exponents) {
    exponents.resize(primes.size(), 0);
    
    #pragma omp parallel for schedule(dynamic)
    for (size_t i = 0; i < primes.size(); i++) {
        unsigned int exp = 0;
        unsigned long long power = primes[i];
        while (power <= n) {
            exp += n / power;
            if (power > n / primes[i]) break;
            power *= primes[i];
        }
        exponents[i] = exp;
    }
}
```

## 如何更高效地计算大数阶乘 - GPU

### 5.1 CGBN库：CUDA大数运算

CGBN（Cooperative Groups Big Num）是NVIDIA开发的CUDA大数运算库。它利用GPU的并行计算能力，实现了高性能的大数运算。

### 5.2 GPU加速阶乘计算

基于PrimeSwing算法，我们可以设计GPU加速的阶乘计算：

```cpp
// GPU核函数：计算质数的幂
template <class params>
__global__ void kernel_factorial(cgbn_error_report_t *report,
                                 instance_t<params> *instances,
                                 uint32_t count) {
    int32_t instance = (blockIdx.x * blockDim.x + threadIdx.x) / params::TPI;
    if (instance >= count) return;
    
    factorial_gpu_t<params> factorial(cgbn_report_monitor, report, instance);
    factorial(instances);
}

// 主机端：质因数分解和结果合并
void calculate_factorial_gpu(unsigned int n, mpz_t result) {
    // 1. CPU生成质数表
    std::vector<unsigned int> primes = generate_primes(n);
    
    // 2. CPU质因数分解（多线程）
    std::vector<unsigned int> exponents;
    factorize_factorial_multithread(n, primes, exponents);
    
    // 3. GPU并行计算质数幂
    process_batch_gpu(primes, exponents, result);
}
```

### 5.3 CUDA-FFT 大数乘法 (并未实现 学习一下ds老师写的)

在大数阶乘计算中，随着 $n$ 的增长，中间结果的位数急剧膨胀——$10,000,000!$ 的最终结果已超过 $6500$ 万位。此时，大数乘法成为整个计算链路的核心瓶颈。将 FFT 乘法迁移至 GPU，利用数千个 CUDA 核心并行处理傅里叶变换，是突破这一瓶颈的关键路径之一。

#### 5.3.1 数学基础回顾

大数乘法的 FFT 加速基于以下三条等价链：

1. **整数 → 多项式**：将一个大整数 $A = a_{k-1}a_{k-2} \dots a_0$（以 $B$ 为基）表示为多项式 $A(x) = \sum_{i=0}^{k-1} a_i x^i$。此时 $A = A(B)$，两个大整数相乘等价于两个多项式相乘后在 $x = B$ 处求值并执行进位归一化。
2. **多项式乘法 → 线性卷积**：两个多项式 $A(x)$ 与 $B(x)$ 的乘积系数 $c_k = \sum_{i=0}^{k} a_i b_{k-i}$，这正是离散卷积。
3. **卷积 → FFT 频域乘法**：由卷积定理，时域的卷积对应频域的逐点乘积：$\mathcal{F}\{a * b\} = \mathcal{F}\{a\} \cdot \mathcal{F}\{b\}$。因此，计算流程为：FFT 变换两序列 → 逐点相乘 → IFFT 逆变换 → 进位归一化。

这一算法的时间复杂度为 $O(N \log N)$，在处理数万位以上的整数时，相比 $O(N^2)$ 的竖式乘法和 $O(N^{1.585})$ 的 Karatsuba 算法具备显著的渐进优势。

#### 5.3.2 基于 cuFFT 的实现流程

NVIDIA 的 cuFFT 库为 GPU 端 FFT 提供了高度优化的实现，支持批量变换和多 GPU 扩展。以 cuFFT 实现大数乘法的完整流程如下：

```cpp
#include <cufft.h>
#include <cuda_runtime.h>
#include <thrust/device_vector.h>

// 基于 cuFFT 的大数乘法核心流程
void big_integer_multiply_cufft(
    const std::vector<double>& A,   // 被乘数的多项式系数（以 B 为基）
    const std::vector<double>& B,   // 乘数的多项式系数
    std::vector<double>& C)         // 乘积的多项式系数
{
    size_t lenA = A.size();
    size_t lenB = B.size();
    size_t conv_len = lenA + lenB - 1;  // 线性卷积长度

    // 1. 零填充至 2 的幂，避免循环卷积混叠
    size_t fft_len = 1;
    while (fft_len < conv_len) fft_len <<= 1;

    // 2. 将输入数据拷贝至 GPU
    thrust::device_vector<cufftDoubleComplex> d_A(fft_len, make_cuDoubleComplex(0.0, 0.0));
    thrust::device_vector<cufftDoubleComplex> d_B(fft_len, make_cuDoubleComplex(0.0, 0.0));
    for (size_t i = 0; i < lenA; i++) d_A[i] = make_cuDoubleComplex(A[i], 0.0);
    for (size_t i = 0; i < lenB; i++) d_B[i] = make_cuDoubleComplex(B[i], 0.0);

    // 3. 创建 cuFFT plan 并执行正向 FFT
    cufftHandle plan;
    cufftPlan1d(&plan, fft_len, CUFFT_Z2Z, 1);  // Z2Z: 双精度复数到复数
    cufftExecZ2Z(plan, 
        thrust::raw_pointer_cast(d_A.data()),
        thrust::raw_pointer_cast(d_A.data()), 
        CUFFT_FORWARD);
    cufftExecZ2Z(plan,
        thrust::raw_pointer_cast(d_B.data()),
        thrust::raw_pointer_cast(d_B.data()), 
        CUFFT_FORWARD);

    // 4. 频域逐点相乘
    thrust::transform(d_A.begin(), d_A.end(), d_B.begin(), d_A.begin(),
        [] __device__ (cufftDoubleComplex x, cufftDoubleComplex y) {
            return make_cuDoubleComplex(
                x.x * y.x - x.y * y.y,
                x.x * y.y + x.y * y.x);
        });

    // 5. 逆 FFT 并缩放
    cufftExecZ2Z(plan,
        thrust::raw_pointer_cast(d_A.data()),
        thrust::raw_pointer_cast(d_A.data()),
        CUFFT_INVERSE);
    cufftDestroy(plan);

    // 6. 拷贝回主机端并执行进位归一化
    C.resize(conv_len);
    for (size_t i = 0; i < conv_len; i++)
        C[i] = d_A[i].x / fft_len;  // IFFT 的缩放因子
}
```

**关键步骤说明**：

1. **零填充（Zero-Padding）**：线性卷积的结果长度为 `lenA + lenB - 1`，而 FFT 默认执行循环卷积。必须将序列零填充至不小于卷积长度的最小 2 的幂，以避免时域混叠误差。

2. **Plan 复用**：cuFFT plan 的创建开销较大，对于相同尺寸的变换应只创建一次并反复使用，以显著降低总延迟。

3. **双精度选择**：大数乘法对精度敏感。单精度浮点（`float`）的 24 位有效尾数在处理大基数时容易产生舍入误差，导致进位计算错误。双精度（`double`）可提供 53 位有效尾数，显著提高数值稳定性。

4. **批量处理**：cuFFT 支持 `cufftPlanMany`，可一次对多个等长序列并行执行 FFT，最大化 GPU 利用率，尤其适合分块乘法或同时处理多个中间结果。

#### 5.3.3 浮点精度与纠错策略

FFT 大数乘法的核心挑战在于：**浮点运算的舍入误差可能破坏整数卷积的精确结果**。当一个整数采用基数 $B = 10^4$ 存储时，卷积结果的每个元素 $C_k = \sum_{i+j=k} a_i b_j$ 的最大值可达 $N \cdot (B-1)^2$（$N$ 为序列长度）。对于大数乘法，这个值很容易超过 $2^{53}$（双精度的精确整数表示上限），导致部分结果的低位数被舍入。

常用对策包括：

- **降低基数 / 分段存储**：使用更小的基数（如 $10^3$ 而非 $10^9$），使每个系数和卷积中间值的幅度受限。代价是序列长度增加，FFT 规模变大。
- **模数归约 + 中国剩余定理（CRT）**：将问题分解到多个小模数域上分别计算，再通过 CRT 重构最终结果。这是 GMP 等工业级库中 FFT 乘法的常用方案，但实现复杂度较高。
- **Number Theoretic Transform（NTT）**：在有限整数域（而非复数域）上执行类似 FFT 的变换，完全规避浮点误差，代价是模数选择受限且操作复杂度更高。
- **两阶段验证**：先用 FFT 快速计算近似结果，再用精确但较慢的竖式乘法验证低位的关键进位，或用误差边界公式提前判断当前基数与 FFT 长度下是否安全。

实际工程中，"3 次变 2 次"的实数编码技巧（参见 §2.3）在 GPU 上同样适用——通过将两个实序列编码到一个复数序列的实部和虚部，可将 FFT/IFFT 调用次数减半，进一步缩小与 CPU FFT 的性能差距。

#### 5.3.5 在阶乘计算中的应用

在 PrimeSwing 或二进制分割阶乘算法的顶层，**大数乘法**是最耗时的步骤。将这一步卸载到 GPU 上，理论上可以通过 cuFFT 加速中间乘积的计算。以下是结合 PrimeSwing 的 GPU 加速阶乘的简化伪代码：

```cpp
void factorial_gpu_fft(mpz_t result, unsigned long n) {
    // 1. CPU 端完成 PrimeSwing 分解，生成 Swing 数序列
    // 2. 将大数乘法密集的子任务打包发送到 GPU
    // 3. GPU 端调用 cuFFT 完成频域卷积
    // 4. 主机端回收结果并执行进位归一化
    // 5. 按 PrimeSwing 公式组装最终阶乘
}
```

但在工程实践中，阶乘计算的瓶颈往往不止乘法本身——质因数分解、小规模连乘、结果组装等步骤大量依赖 CPU 端的整数操作与内存管理。因此，一个高效的 GPU 加速阶乘需要精细的**CPU-GPU 协同设计**：在 CPU 上完成质数筛选和 Swing 分解，将计算密集、数据量足够的乘法批量卸载到 GPU，同时尽可能减少中间结果的来回传输次数。这种异构计算的工程复杂度较高，通常只在百万位以上规模时才能体现出超过纯 CPU GMP 实现的性能优势。

## 6. 性能对比与总结

### 6.1 算法对比总结

| 算法 | 核心思想 | 优化策略 | 适用场景 | 复杂度 |
| ------ | --------- | --------- | --------- | -------- |
| **GMP原生** | PrimeSwing递归分解 | 自适应乘法、移位补偿 | 通用场景，基准验证 | O(n log² n) |
| **分治算法** | 二进制分割乘积 | 多线程并行、动态阈值 | 中等规模阶乘 | O(n log² n) |
| **PrimeSwing** | 数学恒等式分解 | 并行质数筛、递归优化 | 大规模阶乘计算 | O(n log² n log log n) |
| **FFT加速** | 分治+FFT乘法 | 3变2优化、压位存储、分块 | 超大数乘法优化 | O(n log² n) |
| **CUDA GPU** | 质因数分解并行 | GPU并行计算、内存优化 | 显存充足的GPU环境 | O(π(n) log n) |

### 6.2 完整性能测试数据

#### 不同规模的性能对比

| 阶乘规模 | 算法/实现 | 总耗时(秒) | 结果位数 | 峰值内存(KB) |
| :--- | :--- | :--- | :--- | :--- |
| **14,000!** | GMP内置(`mpz_fac_ui`) | 0.02 | 51,969位 | 4,640 |
| | 二进制分割(单线程) | 0.03 | 51,969位 | 4,640 |
| | 二进制分割(多线程20) | 5.04 | 51,969位 | 5,440 |
| | FFT基础版本 | 72.90 | 51,969位 | 6,224 |
| | FFT分块优化(v1) | 0.15 | 51,969位 | 5,500 |
| | CGBN GPU融合分级 | 0.69 | 51,969位 | 114,184 |
| **104,000!** | GMP内置(`mpz_fac_ui`) | 0.13 | 476,608位 | 6,344 |
| | 二进制分割(单线程) | 0.17 | 476,608位 | 6,320 |
| | FFT分块优化(v1) | 1.48 | 476,608位 | 19,668 |
| **1,004,000!** | GMP内置(`mpz_fac_ui`) | 2.37 | 5,589,713位 | 28,568 |
| | 二进制分割(单线程) | 2.78 | 5,589,713位 | 27,652 |
| | FFT分块优化(v1) | 33.96 | 5,589,713位 | 264,868 |
| **10,004,000!** | GMP内置(`mpz_fac_ui`) | 46.46 | 65,685,060位 | 273,336 |
| | 二进制分割(单线程) | 53.07 | 65,685,060位 | 229,484 |
| **100,004,000!** | GMP内置(`mpz_fac_ui`) | 806.06 | 756,602,557位 | 2,936,468 |
| | 二进制分割(多线程20) | 786.27 | 756,602,557位 | 2,743,428 |

#### 结果验证数据

| n 值 | 十进制总位数 | 末尾0个数 | 数位和 | 计算用时（最快） |
|------|-------------|-----------|--------|----------|
| 14,000! | 51,969 | 3,498 | 217,242 | 0.02秒 |
| 104,000! | 476,608 | 25,998 | 2,028,321 | 0.13秒 |
| 1,004,000! | 5,589,713 | 250,997 | 24,026,301 | 2.37秒 |
| 10,004,000! | 65,685,060 | 2,500,998 | 284,299,920 | 46.46秒 |
| 100,004,000! | 756,602,557 | 25,000,998 | 3,292,149,348 | 786.27秒 |

**项目代码**：[cpp-factorial-hpc-lab](https://github.com/lensit/cpp-factorial-hpc-lab)

**参考资料**：

1. Henrik Vestermark. *Fast Factorial & Binomial Computation for Arbitrary Precision*. [www.hvks.com/Numerical/papers.html]
2. Torbjörn Granlund and the GMP development team. *GMP - The GNU Multiple Precision Arithmetic Library (Version 6.3.0)*. [https://gmplib.org/manual/]
3. Y. Hao et al., "Cambricon-P: A Biflow Architecture for Arbitrary Precision Computing," 2021.
4. Peter Luschny, "Conclusions from the Factorial Function Contest". [http://www.luschny.de/math/factorial/conclusions.html]
5. Peter Luschny, "Fast Factorial Functions". [http://www.luschny.de/math/factorial/FastFactorialFunctions.htm]
6. "HVE Fast Factorial in arbitrary precision". [https://www.hvks.com/Numerical/Downloads/HVE%20Fast%20Factorial%20in%20arbitrary%20precision.pdf]

---

## 附录A：项目评价 from huan-yp

### 总体评价

- 技术栈全面、算法优化深入，文档专业清晰，代码管理合理。
- 整体工程复杂度过高，效率提升有限，部分设计缺乏必要的权衡。
- 算法效率甚至不如一个方法经过2h简要深度优化的50%
- 缺少最后的验证环节，如何确保计算出来的结果一定是正确的
- 缺少单元测试，无法确保各个模块的正确性，调试难度大
- 建议使用GitHub或GitLab等更主流的平台，方便代码审查和协作
- 比较正确的方式是调研出一种最高效的策略，然后围绕这一种策略进行优化
- 当然，这次项目还是以学习为主，多尝试也是有价值的

## 附录B: 写在最后

1. CGBN缺乏持续维护，更新停滞; 并且包含32K位固有约束，限制了可计算的阶乘规模（约n≤30000），感觉大部分hpc还是需要手写cuda核。

2. 现在看当时的项目完全没有硬件层面的理解，所以大部分工作的时间在于漫无目的的搜索和调库，而并非深入了解具体的硬件原理和算法实现，所以浪费了非常多的时间。现在看来当时应该不急于计算出结果或者尝试更多的方法，而是应该借此机会了解CPU、GPU、并发线程、cuda等相关知识。

3. 本次是一个团队任务：FFT的部分由 xcx-14514(见那天光) 完成，GMP naive 由 pieceofcorn完成，GMP PrimeSwing 由 LengA。但是在合作部分并没有达到预期的效果，分析在于并没有一个人完全理解全部项目，而理想状态应该是每一个人都理解全部项目。关于这方面会持续思考，当真实感受到成功执行可能会专门写一篇blog，感谢huan-yp的指点和解惑

4. 第一次熟悉了一下团队代码协作的流程和经验、makefile、单元测试等一系列工程工具。但是实现过程还是非常的naive，古法vibe-coding、不足之处数不胜数。现在回顾看来属于是来时路的一次总结，有太多当时没有做好的地方，也反思现在其实也没有能力做好的方向。前路茫茫，道阻且长，还需努力。
