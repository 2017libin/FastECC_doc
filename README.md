# FastECC_doc
关于ECC优化技术的学习笔记

## [1. 性能优化](./ECC优化_学习笔记)

### 1.1 大整数运算

| 方案          | 排行 |
| ------------- | ---- |
| 大整数-加法   |      |
| 大整数-乘法   |      |
| 大整数-模运算 |      |
| 大整数-模乘   |      |
| 大整数-模逆   |      |

### 1.2 有限域运算

### 1.3 协议层

## [2.性能测试 ](性能测试-学习笔记)

1. 单线程测试
   1. 运行n次所花的时间
   2. 运行n秒所执行的次数
2. 多线程测试
   1. 基于openmp的多线程测试
3. 多进程测试
   1. fork多进程测试

## 3. 密码库优化技术

### 3.1 大整数运算

| 密码库    | 数据结构 | 大整数-加法 | 大整数-乘法 | 大整数-模运算        | 大整数-模乘 | 大整数-模逆 |      |
| --------- | -------- | ----------- | ----------- | -------------------- | ----------- | ----------- | ---- |
| Secp256k1 |          |             |             | 特定曲线的优化模运算 |             |             |      |
|           |          |             |             |                      |             |             |      |
|           |          |             |             |                      |             |             |      |

### 3.2 域运算

## 4. 调试信息

### 4.1 自定义输出函数

1. debug()可以用于替换原有printf函数，并根据DEBUG来控制是否输出
2. PRINT_FLAG用于调试运行流程，观察是否执行到相应的输出语句

```c
#define DEBUG 0
#if DEBUG
  #define debug(...) printf(__VA_ARGS__)
#else 
  #define debug(...) ;
#endif

#define PRINT_FLAG1(msg) do { \
    fprintf(stdout, "%s:%d: %s\n", __FILE__, __LINE__, msg); \
} while(0)

#define PRINT_FLAG2(msg) do { \
    fprintf(stdout, "[*] PRINT_FLAG: %s\n", msg); \
} while(0)

```

