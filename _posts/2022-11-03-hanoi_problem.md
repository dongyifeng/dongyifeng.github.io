---
title: 汉诺塔
tags: 每日一题 递归
typora-root-url: ../../dongyifeng.github.io
---

> 汉诺塔：汉诺塔（又称河内塔）问题是源于印度一个古老传说的益智玩具。大梵天创造世界的时候做了三根金刚石柱子，在一根柱子上从下往上按照大小顺序摞着64片黄金圆盘。大梵天命令婆罗门把圆盘从下面开始按大小顺序重新摆放在另一根柱子上。并且规定，在小圆盘上不能放大圆盘，在三根柱子之间一次只能移动一个圆盘。



汉诺塔问题分三步：

1. 将 0 ~ n -1 号圆盘从 from 柱子移动到 other 柱子。
2. 将 n 号圆盘从 from 柱子移动到 to 柱子。
3. 将 0 ~ n - 1 号圆盘从 other 柱子移动到 to 柱子。

![](/images/assets/screenshot-20221104-124442.png)

![](/images/assets/screenshot-20221104-124514.png)

```python
def hanoi(n):
    if n <= 0: return
    func(n, "左", "右", "中")

def func(i, start, end, other):
    if i == 1:
        print("Move 1 from " + start + " to " + end)
        return
    func(i - 1, start, other, end)
    print("Move " + str(i) + " from" + start + " to " + end)
    func(i - 1, other, end, start)

hanoi(3)
```



> 汉诺塔游戏的要求把所有的圆盘从左边都移到右边的柱子上，给定一个整型数组arr，其中只含有1、2、3，代表所有圆盘目前的状态，1 代表左柱，2代表中柱，3 代表右柱，arr[i] 的值代表第 i + 1 个圆盘的位置。
>
> 比如：arr = [3,3,2,1] ，代表第 1 个圆盘在右柱上，第 2 个圆盘在右柱上，第 3 个圆盘在中柱上，第 4 个圆盘在左柱上。
>
> 如果 arr 代表的状态是最优移动轨迹过程中出现的状态，返回 arr 这种状态是最优移动轨迹中第几状态；如果 arr 代表的状态不是最优移动轨迹过程中出现的状态，则返回 -1。



**解法一：暴力递归**

分析：

结论：n 层汉诺塔一共需要 $2^n-1$ 次移动圆盘。

汉诺塔问题分三步：

1. 将 0 ~ n -1 号圆盘从 from 柱子移动到 other 柱子，需要 $2^{n-1}-1$ 次移动圆盘。
2. 将 n 号圆盘从 from 柱子移动到 to 柱子，需要 1 次移动圆盘。
3. 将 0 ~ n - 1 号圆盘从 other 柱子移动到 to 柱子，需要 $2^{n-1}-1$ 次移动圆盘。



可能性分析：

- 假设 n 号圆盘在 from 柱子上，说明第一大步还没走完，此时最优移动轨迹中的第几状态，取决于第一大步移动了多少次圆盘。
- 假设 n 号圆盘在 other 柱子上，从下图可知 n 号圆盘不可能出现在 other 柱子上，说明次走法的不是最优移动轨迹，返回 -1。
- 假设 n 号圆盘在 to 柱子上，说明第一大步和第二大步已经走完，此时最优移动轨迹中的第几状态 = 第一大步所有的移动次数 + 第二大步所有的移动次数 + 第三大步真实移动次数 = $(2^{n-1}-1) + 1 + rest  = 2{n-1}+rest$。



时间复杂度：O(N)

```python
def step(arr):
    if not arr: return 0
    return f(arr, len(arr) - 1, 1, 2, 3)

# 目标：把 arr[0~i] 的圆盘，从 from 全部挪到 to 上
# 返回：根据 arr 中状态 arr[0~i] ,它是最优解的第几步？
# 时间复杂度：O(N)
def f(arr, i, fro, other, to):
    if i == -1: return 0
    if arr[i] != fro and arr[i] != to:
        return -1
    if arr[i] == fro:
        # 第一大步没走完，arr[0~i-1] from --> other
        return f(arr, i - 1, fro, to, other)
    # 第三步完成的程度
    rest = f(arr, i - 1, other, fro, to)
    if rest == -1:
        return rest
    return (1 << i) + rest
```



**解法二：非递归**

```python
def step1(arr):
    if not arr: return 0
    fro = 1
    other = 2
    to = 3
    i = len(arr) - 1
    res = 0
    while i >= 0:
        if arr[i] != fro and arr[i] != to: return -1
        if arr[i] == fro:
            res += 1 << i
            tmp = fro
            fro = other
        else:
            tmp = to
            to = other
        other = tmp
        i -= 1
    return res
```



**对数器**

```python
import random

def check():
    for _ in range(1000):
        arr = [int(random.random() * 3) + 1 for _ in range(int(random.random() * 100))]
        res = step(arr)
        res1 = step(arr)
        if res != res1:
            print("ERROR","res=",res,"res1=",res1,arr)
    print("Nice")

check()
```

