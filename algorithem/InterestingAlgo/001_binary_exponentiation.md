# Binary Exponentiation

快速幂算法（也称为二进制 exponentiation、快速 exponentiation 或 exponentiation by squaring）通过将指数分解为二进制形式，它的时间复杂度为 `O(logn)`，而不是朴素方法的 `O(n)`

## 算法描述

计算 \(a^{n}\) 的关键思想是利用指数的二进制表示。例如，假设 \(n = 13\)，它的二进制表示为 \(1101\)，这意味着：

\[13 = 8 + 4 + 1 = 2^3 + 2^2 + 2^0\]

因此，\(a^{13}\) 可以表示为：

\[a^{13} = a^8 \cdot a^4 \cdot a^1\]

所以，计算 \(a^n\) 可以通过计算 \(a\) 的 \(2^k\) 次幂，并将对应二进制位为 1 的那些项相乘来实现。

算法的核心是通过连续平方来计算这些 \(a\) 的 \(2^k\) 次幂。例如：

- \(a^1 = a\)
- \(a^2 = (a^1)^2\)
- \(a^4 = (a^2)^2\)
- \(a^8 = (a^4)^2\)

因此，我们只需要计算 \(\log_2 n\) 个值，而不是 \(n-1\) 个值。

## 实现

### 递归实现

递归方法直接反映了上述思想：

```cpp
long long binpow(long long a, long long b) {
    if (b == 0)
        return 1;
    long long res = binpow(a, b / 2);
    if (b % 2 == 1)
        return res * res * a;
    else
        return res * res;
}
```

### 迭代实现

迭代方法更高效，因为它避免了递归调用的开销：

```cpp
long long binpow(long long a, long long b) {
    long long res = 1;
    while (b > 0) {
        if (b % 2 == 1)
            res = res * a;
        a = a * a;
        b = b / 2;
    }
    return res;
}
```

执行步骤推导，计算 \(3^{13}\)：

- 二进制表示：13 = 8 + 4 + 1 → 1101
- 计算步骤：
  - 初始：res = 1, a = 3, b = 13 (1101)
  - b 是奇数：res = 1 * 3 = 3; a = 9; b = 6 (110)
  - b 是偶数：res = 3; a = 81; b = 3 (11)
  - b 是奇数：res = 3 * 81 = 243; a = 6561; b = 1 (1)
  - b 是奇数：res = 243 * 6561 = 1594323; a = 43046721; b = 0
  - 结果：1594323

godbolt：https://godbolt.org/z/4d6n4PWbr

## 应用场景

### 高效计算大数的模幂运算

问题：计算 \(x^n \bmod m\)。这是一个非常常见的操作，例如在计算模乘法逆元时会用到。

解决方案：由于我们知道模运算不会干扰乘法运算（$a \cdot b \equiv (a \bmod m) \cdot (b \bmod m) \pmod m$），我们可以直接使用相同的代码，只需将每个乘法替换为模乘法：

```cpp
long long binpow(long long a, long long b, long long m) {
    a %= m;
    long long res = 1;
    while (b > 0) {
        if (b & 1)
            res = res * a % m;
        a = a * a % m;
        b >>= 1;
    }
    return res;
}
```

注意：对于 $b \gg m$ 的情况，可以进一步优化该算法。如果 $m$ 是质数，则 $x^n \equiv x^{n \bmod (m-1)} \pmod{m}$；对于合数 $m$，则 $x^n \equiv x^{n \bmod{\phi(m)}} \pmod{m}$。这直接源自费马小定理和欧拉定理，更多细节请参见模逆元相关文章。

### 高效计算斐波那契数列

问题：计算第 $n$ 个斐波那契数 $F_n$。

解决方案：要计算下一个斐波那契数，只需要前两个斐波那契数，因为 $F_n = F_{n-1} + F_{n-2}$。我们可以构建一个 $2 \times 2$ 的矩阵来描述这种转换：从 $F_i$ 和 $F_{i+1}$ 转换到 $F_{i+1}$ 和 $F_{i+2}$。例如，将这个转换应用于 $F_0$ 和 $F_1$ 会得到 $F_1$ 和 $F_2$。因此，我们可以将这个转换矩阵提升到 $n$ 次幂，从而在 $O(\log n)$ 的时间复杂度内找到 $F_n$。

这段代码是使用**矩阵快速幂**计算斐波那契数列第 `n` 项的高效实现，时间复杂度为 \(O(\log n)\)，远快于递归或循环的 \(O(n)\) 复杂度。下面详细解释其原理和实现：

斐波那契数列的递推公式为：

- \(F(0) = 0\)
- \(F(1) = 1\)
- \(F(n) = F(n-1) + F(n-2)\)（对于 \(n \geq 2\)）

这个递推关系可以用矩阵乘法表示。观察发现：

\[\begin{bmatrix} F(n) \\ F(n-1) \end{bmatrix}
= \begin{bmatrix} 1 & 1 \\ 1 & 0 \end{bmatrix}\times \begin{bmatrix} F(n-1) \\ F(n-2) \end{bmatrix}\]

通过归纳法可推导出：

\[\begin{bmatrix} F(n) \\ F(n-1) \end{bmatrix}= \begin{bmatrix} 1 & 1 \\ 1 & 0 \end{bmatrix}^{n-1}\times \begin{bmatrix} F(1) \\ F(0) \end{bmatrix}\]

由于 \(F(1) = 1\)、\(F(0) = 0\)，最终可简化为：
\[F(n) = \begin{bmatrix} 1 & 1 \\ 1 & 0 \end{bmatrix}^{n-1}\text{ 的左上角元素}\]

因此，计算 \(F(n)\) 等价于求矩阵 \(\begin{bmatrix} 1 & 1 \\ 1 & 0 \end{bmatrix}\) 的 \(n-1\) 次幂，而矩阵幂可以用**快速幂算法**高效计算。

```cpp
using Mat = array<array<long long, 2>, 2>;  // 定义2x2矩阵类型
const long long MOD = 1e9 + 7;              // 取模常数，防止数值溢出

Mat mul(const Mat& a, const Mat& b) {
    Mat c{};
    // 矩阵乘法核心：c[i][j] = sum(a[i][k] * b[k][j]) （k=0,1）
    for (int i = 0; i < 2; ++i)
        for (int k = 0; k < 2; ++k)
            for (int j = 0; j < 2; ++j)
                c[i][j] = (c[i][j] + a[i][k] * b[k][j]) % MOD;
    return c;
}

Mat binpow(Mat a, long long n) {
    Mat res{1, 0, 0, 1};   // 初始化为单位矩阵（矩阵乘法的"1"）
    while (n) {
        if (n & 1)              // 如果n的二进制最后一位是1 
            res = mul(res, a);  // 结果乘以当前矩阵a    
        a = mul(a, a);          // 矩阵a自乘（平方）
        n >>= 1;                // n右移一位（相当于除以2）
    }
    return res;
}

long long fib(long long n) {
    if (n == 0) return 0;
    Mat base{1, 1, 1, 0};
    Mat res = binpow(base, n - 1);
    return res[0][0];
}
```

### 将排列应用 $k$ 次

问题：给定一个长度为 $n$ 的序列。将给定的排列应用于该序列 $k$ 次。

解决方案：只需使用二进制幂将排列提升到 $k$ 次幂，然后将其应用于序列。这将使时间复杂度达到 $O(n \log k)$。

```cpp
vector<int> applyPermutation(vector<int> sequence, vector<int> permutation) {
    vector<int> newSequence(sequence.size());
    for(int i = 0; i < sequence.size(); i++) {
        newSequence[i] = sequence[permutation[i]];
    }
    return newSequence;
}

vector<int> permute(vector<int> sequence, vector<int> permutation, long long k) {
    while (k > 0) {
        if (k & 1) {
            sequence = applyPermutation(sequence, permutation);
        }
        permutation = applyPermutation(permutation, permutation);
        k >>= 1;
    }
    return sequence;
}
```

注意：通过构建排列图并独立考虑每个循环，可以在线性时间内更高效地解决此任务。然后，我们可以计算 $k$ 对循环大小的模，并找到属于该循环的每个数字的最终位置。

### 快速将一组几何变换应用于一组点

问题：给定 $n$ 个点 $p_i$，对每个点应用 $m$ 个变换。每个变换可以是平移、缩放或绕给定轴旋转给定角度。还有一个"循环"操作，它将一组给定的变换应用 $k$ 次（"循环"操作可以嵌套）。要求应用所有变换的速度快于 $O(n \cdot length)$，其中 $length$ 是要应用的变换的总数（在展开"循环"操作之后）。

解决方案：让我们看看不同类型的变换如何改变坐标：

- 平移操作：向每个坐标添加不同的常数。
- 缩放操作：将每个坐标乘以不同的常数。
- 旋转操作：变换更复杂（我们这里不详细讨论），但每个新坐标仍然可以表示为旧坐标的线性组合。

如您所见，每个变换都可以表示为对坐标的线性操作。因此，一个变换可以写成一个 $4 \times 4$ 的矩阵形式：

$$\begin{pmatrix} a_{11} & a_{12} & a_{13} & a_{14} \\ a_{21} & a_{22} & a_{23} & a_{24} \\ a_{31} & a_{32} & a_{33} & a_{34} \\ a_{41} & a_{42} & a_{43} & a_{44} \end{pmatrix}$$

当它与带有旧坐标和 1 的向量相乘时，会得到一个带有新坐标和 1 的新向量：

$$\begin{pmatrix} x & y & z & 1 \end{pmatrix} \cdot \begin{pmatrix} a_{11} & a_{12} & a_{13} & a_{14} \\ a_{21} & a_{22} & a_{23} & a_{24} \\ a_{31} & a_{32} & a_{33} & a_{34} \\ a_{41} & a_{42} & a_{43} & a_{44} \end{pmatrix} = \begin{pmatrix} x' & y' & z' & 1 \end{pmatrix}$$

（您可能会问，为什么要引入一个虚构的第四个坐标？这就是齐次坐标的妙处，它在计算机图形学中有着广泛的应用。如果没有它，就不可能将平移等仿射操作实现为单个矩阵乘法，因为它需要我们向坐标添加常数。仿射变换在更高维度中变成了线性变换！）

以下是变换如何以矩阵形式表示的一些示例：

平移操作：将 $x$ 坐标平移 5，$y$ 坐标平移 7，$z$ 坐标平移 9。

$$\begin{pmatrix} 1 & 0 & 0 & 0 \\ 0 & 1 & 0 & 0 \\ 0 & 0 & 1 & 0 \\ 5 & 7 & 9 & 1 \end{pmatrix}$$

缩放操作：将 $x$ 坐标缩放 10 倍，其他两个坐标缩放 5 倍。

$$\begin{pmatrix} 10 & 0 & 0 & 0 \\ 0 & 5 & 0 & 0 \\ 0 & 0 & 5 & 0 \\ 0 & 0 & 0 & 1 \end{pmatrix}$$

旋转操作：按照右手规则（逆时针方向）绕 $x$ 轴旋转 $\theta$ 度。

$$\begin{pmatrix} 1 & 0 & 0 & 0 \\ 0 & \cos \theta & -\sin \theta & 0 \\ 0 & \sin \theta & \cos \theta & 0 \\ 0 & 0 & 0 & 1 \end{pmatrix}$$

现在，一旦每个变换都被描述为矩阵，变换序列就可以被描述为这些矩阵的乘积，而重复 $k$ 次的"循环"可以被描述为矩阵的 $k$ 次幂（可以使用二进制幂在 $O(\log{k})$ 时间内计算）。这样，表示所有变换的矩阵可以首先在 $O(m \log{k})$ 时间内计算，然后可以在 $O(n)$ 时间内应用于 $n$ 个点中的每一个，总复杂度为 $O(n + m \log{k})$。

### 二进制幂的变体：两个数的模 $m$ 乘法

问题：计算两个数 $a$ 和 $b$ 的模 $m$ 乘积。$a$ 和 $b$ 可以放入内置数据类型中，但它们的乘积太大而无法放入 64 位整数中。我们的想法是在不使用大数算术的情况下计算 $a \cdot b \pmod m$。

解决方案：我们只需应用上面描述的二进制构造算法，只是执行加法而不是乘法。换句话说，我们已经将两个数的乘法"扩展"为 $O (\log m)$ 次加法和乘以二的操作（本质上是加法）。

$$a \cdot b = \begin{cases} 0 &\text{如果 }a = 0 \\ 2 \cdot \frac{a}{2} \cdot b &\text{如果 }a > 0 \text{ 且 }a \text{ 为偶数} \\ 2 \cdot \frac{a-1}{2} \cdot b + b &\text{如果 }a > 0 \text{ 且 }a \text{ 为奇数} \end{cases}$$

注意：您可以通过使用浮点运算以不同的方式解决此任务。首先使用浮点数计算表达式 $\frac{a \cdot b}{m}$ 并将其转换为无符号整数 $q$。使用无符号整数算术从 $a \cdot b$ 中减去 $q \cdot m$，然后对 $m$ 取模以找到答案。这个解决方案看起来相当不可靠，但它非常快，并且易于实现。有关更多信息，请参见此处。
