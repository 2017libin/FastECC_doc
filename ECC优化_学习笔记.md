ECC优化 学习笔记

# 一. 域运算优化
## 1.1 数据结构

### 1.1.1 256比特的大整数

1. 使用数组表示一个大整数，即256比特的数可以表示为`uint64_t n[4]`，也可以表示为`uint32_t n[8]`

   1. 通常使用小端方式存储：低位放在低地址，高位放在高地址

2. 存储方式

   ```c
   // secp256k1中的表示形式
   typedef struct {
       uint64_t d[4];
   } secp256k1_scalar;
   ```

## 1.2 加/减法

相应分量带进位的加减操作

### 1.2.1 加法

1. 参考：GmSSL-sm2_alg.c

```c
static void sm2_bn288_add(uint64_t r[9], const uint64_t a[9], const uint64_t b[9])
{
	int i;
	r[0] = a[0] + b[0];
	for (i = 1; i < 9; i++) {
		r[i] = a[i] + b[i] + (r[i-1] >> 32);
	}
	for (i = 0; i < 8; i++) {
		r[i] &= 0xffffffff;
	}
}
```

### 1.2.2 减法

1. 参考：GmSSL-sm2_alg.c

```c
static void sm2_bn288_sub(uint64_t ret[9], const uint64_t a[9], const uint64_t b[9])
{
	int i;
	uint64_t r[9];

	// 借1
	r[0] = ((uint64_t)1 << 32) + a[0] - b[0];
	// 借完1后剩下0xffffffff，但是有可能上一轮不需要1，因此加上(r[i - 1] >> 32)
	for (i = 1; i < 8; i++) {
		r[i] = 0xffffffff + a[i] - b[i] + (r[i - 1] >> 32);
		r[i - 1] &= 0xffffffff;
	}
	// 借完1后减去1，但是有可能上一轮不需要1，因此加上(r[i - 1] >> 32)
	r[i] = a[i] - b[i] + (r[i - 1] >> 32) - 1;
	r[i - 1] &= 0xffffffff;

	for (i = 0; i < 9; i++) {
		ret[i] = r[i];
	}
}
```

## 1.3 乘法

### 1.3.1 教科书方法

### 1.3.2 Karatsuba乘法

### 1.3.3 Montgomery乘法

>需要将结果从蒙哥马利域转到实数域，计算效率较低
>
>[蒙哥马利乘法原理的介绍](https://blog.csdn.net/BjarneCpp/article/details/77644958)
>
>针对Montgomery方法进行优化
>
>1. The SOS method
>2. The CIOS method

- 蒙哥马利模乘通过转换到另外一个域（蒙哥马利域）上进行计算，来避免模乘时需要进行模逆运算

  - 计算$x*y=z$的步骤，蒙哥马利算法通过以下步骤进行计算

    - 将标准域转换为蒙哥马利域，x转换为$x'=xR$，y转换为$y'=yR$，计算$x'=REDC((x\ mod\ N)(R^2\ mod\ N))$，$y'$的计算方法也一样
    - 使用REDC完成在蒙哥马利域上的乘法$z'=REDC(x'*y')=(x*y)R$
    - 将蒙哥马利域上的结果转换为标准域结果，使用的是REDC算法，即计算$z=REDC(z')=(x*y)$

  - 另外一种计算x*y的方式如下：

    ![image-20231005154521389](img/ECC优化_学习笔记/image-20231005154521389.png)

  - 加法和乘法不同，加法直接使用**普通的加法**进行运算即可。具体的加法运算规则如下：

    - 将标准域转换为蒙哥马利域，x转换为$x'=xR$，y转换为$y'=yR$，计算$x'=REDC((x\ mod\ N)(R^2\ mod\ N))$，$y'$的计算方法也一样
    - 使用普通加法完成在蒙哥马利域上的加法$z'=xR+yR=(x+y)R$
    - 将蒙哥马利域上的结果转换为标准域结果，使用的是REDC算法，即计算$z=REDC(z')=(x+y)$

![image-20231004213226897](img/ECC优化_学习笔记/image-20231004213226897.png)

### 1.3 二进制的蒙哥马利乘法

> 对硬件友好的计算方法

1. 计算$x*y*2^{-k}$通过变换，每次移动一位来完成计算
2. 对于每次移动一位，可以通过算法二完成计算

![image-20231005153942626](img/ECC优化_学习笔记/image-20231005153942626.png)

1. 除二的计算方法

![image-20231005154021542](img/ECC优化_学习笔记/image-20231005154021542.png)

1. 二进制蒙哥马利乘积的伪代码如下

![image-20231005153915008](img/ECC优化_学习笔记/image-20231005153915008.png)

### 1.3.4 RNS表示下的并行乘法

### 1.3.5 KOA算法

> KOA算法使用的是分治的方法
>
> KOA就是K

![image-20230914163505010](img/ECC优化_学习笔记/image-20230914163505010.png)

![image-20230921133459164](img/ECC优化_学习笔记/image-20230921133459164.png)



## 4. 模约减算法（规约 reduction）

> 其实就是取模运算，但是使用减法而不是除法来取模，所以也叫做约减

### 1.4.1 普通规约（classical）

### 1.4.2 Barrett规约

1. 原理：计算一个近似的除数$e'$，并计算$c'=d-e'\times N$，最后通过$c=c'-kN$求出$c$
2. 分析：
   1. 输入$d=a\times b$和$N$，计算 $c = a\times b\ mod\ N$
   2. 存在e满足：$d=a\times b=eN+c$
   3. 目标：求出e，并通过$d=a\times b-eN$计算$c$
   4. 问题是，e不好求，但是e的近似值$e'=\lfloor d/N \rfloor$容易求
      1. $e'=\lfloor d/N \rfloor \approx \lfloor(d\times2^{2n}/N)/2^{2n}\rfloor\approx \lfloor(\lfloor d/2^{n}\rfloor \times2^{2n}/N)/2^{n}\rfloor$，其中$n$表示N的比特长度，$\lfloor d/2^{n}\rfloor$等价于$d>>n$
      2. 预计算 $u=2^{2n}/N$
      3. 计算$e'$的过程
         1. $c=d >> n$
         2. $c=c\times u$
         3. $c=c>>n$
3. barrett算法的过程
   1. 计算e'
   2. 计算d'=e'*N
   3. 计算c=d - e'*N
   4. 如果c>N，那么c-=N
   5. 返回c
4. 对于sm2_N来说，u刚好为257比特，因此拆成了u=u[255:0]+2^256
   1. 下图的算法3，模数就是为sm2_N的情况

![image-20230920214340808](img/ECC优化_学习笔记/image-20230920214340808.png)

```python
MAX = 0
def barrett(a, b):
    global MAX
    n = 256
    N = 0x8542D69E4C044F18E8B92435BF6FF7DD297720630485628D5AE74EE7C32E79B7
    u1 = 0xebc9563c60576bb99de7a14155fb561c0c5ddb2eabdad96ba06cd2ff7f30f1bb
    d = a * b
    e1 = u1 * (d >> n)
    e2 = (d >> n) << 256
    e = (e1 + e2) >> n
    c = d - e*N
    subcount = 0
    while c > N:
        subcount += 1
        # print(subcount)
        c = c - N
    MAX = max(MAX, subcount)
    assert c < N
    assert c == d%N

def test_barrett(count):
    N = 0x8542D69E4C044F18E8B92435BF6FF7DD297720630485628D5AE74EE7C32E79B7
    for _ in range(count):
        x = random.randint(1, N)
        y = random.randint(1, N)
        barrett(x, y)
    print(MAX)

if __name__ == "__main__":
    test_barrett(1000000)

```

### 1.4.3 快速模约减算法

> 如果模数为梅森素数，可以快速对512比特的数进行求模
>
> 可以参考：基于软硬件协同的SM2椭圆曲线公钥密码算法加速_邓尧慷

在SM2中，点运算涉及的取模运算可以通过快速模约减算法实现

### 1.4.4 基于查表的模规约运算

A Fast Modular Reduction Method

## 5. 模乘运算

> 模乘结果可以直接由模乘算法得到，也可以**先进行乘法运算，再进行约减运算得到**

1. 模乘运算
   1. 蒙哥马利模乘
2. 先乘法，再求模
   1. **乘法：**
      1. KOA算法
   2. **求模：**
      1. 快速模约减算法
      2. Barrett算法

## 6. 求逆运算

### 6.1 扩展欧几里得

### 6.2 Montgomery

## 7. 平方根

### 7.1 shanks算法

### 7.2 Tonelli-shanks算法

# 二. 曲线运算优化

# 三. 协议层优化

## SM2

## SM9

# 四. 配对运算优化

# 五. 抗侧信道手段

# 六. 常见密码库的优化