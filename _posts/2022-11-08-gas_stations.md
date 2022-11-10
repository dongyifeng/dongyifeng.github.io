---
title: 环形加油站
tags: 每日一题 滑动窗口
typora-root-url: ../../dongyifeng.github.io
---

> N 个加油站组成一个环形，给定两个长度都是 N 的非负数组 oil 和 dis（N > 1）,oil\[i] 表示第 i 个加油站存的油可以跑多少千米，dis\[i] 代表第 i 个加油站到环中下一个加油站相隔多少千米。
>
> 假设你有一辆邮箱足够大的车，初始时车里没有油。如果车从第 i 个加油站出发，最终可以回到这个加油站，那么第 i 个加油站就算良好出发点，否则就不算。请返回长度为 N 的 boolean 数组 res，res\[i] 代表第 i 个加油站是不是良好出发点。



**解法一：暴力算法**

分析：

如果下图绿色节点顺利走完一圈，它就是良好出发点。

![](/images/assets/screenshot-20221108-221900.png)



其实我们可以将 oil - dis 得到纯能数组。那么原问题就转化为：在纯能数组组成的环上，从起始点出发，累加纯能数组上的数值，累加和始终不为负数，那么起始点就是良好起点。



![](/images/assets/screenshot-20221108-222354.png)



时间复杂度：$O(N^2)$

空间复杂度：$O(1)$

```python
def stations(dis, oil):
    if not dis or not oil or len(dis) < 2 or len(dis) != len(oil): return
    res = [False] * len(dis)
    n = len(dis)
    # 起始点（大于 0）
    init = -1
    # 生成纯能数组
    for i in range(n):
        dis[i] = oil[i] - dis[i]
        if dis[i] > 0: init = i

    if init == -1: return res

    # 尝试以每个加油站为出发点
    for i in range(n):
        res[i] = circular(dis, i) >= 0

    return res

# 以 i 为出发点，尝试走完一圈。如果累计和为 0 ，退出
def circular(dis, i):
    if dis[i] < 0: return dis[i]
    n = len(dis)
    sum_value = dis[i]
    j = i + 1
    while j % n != i:
        sum_value += dis[j % n]
        if sum_value < 0: break
        j += 1
    return sum_value
```



**解法二：滑动窗口**



**连通区（窗口）伸缩规则**：使用 [ start, end ) 表示连通区，前闭后开。

start 初始值为 init（初始点：在纯能数组中选择一个值大于 0 的数据作为初始点）

end = next_index( init , n)



如果 end 能扩展连通区：rest >= 0，就逆时针走，一直走到 rest 小于或者走到 start 位置；否则就顺时针移动 start，将所需的油累计在 need 变量上，在 end 扩展时使用 need，使用完毕后将 need 还原成 0。



跑完一圈后会出现两种情况：

- 没找到良好出发点
- 找到了一个良好出发点（start）



**情况一：没找到良好出发点**

![](/images/assets/screenshot-20221109-002800.png)

![](/images/assets/screenshot-20221109-002822.png)

![](/images/assets/screenshot-20221109-002840.png)



<font color=orange>当走完一圈，如果 start 对应加油站不是一个良好出发点，就表明所有加油站不都不是良好出发点。</font>

如下图：连通区是：【G，H，I，A，B】，start 在 G 位置，不是一个良好出发点，那么【H，I，A，B】 也不是良好出发点。因为在连通区内 G 是可以到达【H，I，A，B】任何位置。G 带着剩余油（rest>=0） 到达【H，I，A，B】都没走通，那么以【H，I，A，B】为起始点（rest==0） 更不可能走完一圈。

连通图的 start 没有扩展到【F，E，D，C】说明，【F，E，D，C】无法达到 G 位置，所以【F，E，D，C】也不是良好出发点。

综上所述所此种情况下，所有的加油站都不是良好出发点。



![](/images/assets/screenshot-20221110-172928.png)



**情况二：找到了一个良好出发点（start）**



![](/images/assets/screenshot-20221109-095613.png)

![](/images/assets/screenshot-20221109-095631.png)



当走完一圈，如果 start 对应加油站是一个良好出发点（rest >= 0），我们需要寻找其他良好出发点。

如下图：连通区是：【G，H，I，A，B，C，D，E，F】，start 所在的 G 位置是一个良好出发点，那么从 start 上一个加油站 start1 出发到一路追溯到 init ，任何一个能到 G 的加油站都是良好出发点。



![](/images/assets/screenshot-20221110-201745.png)





时间复杂度：$O(N)$

空间复杂度：$O(1)$



```python
def stations2(dis, oil):
    if not dis or not oil or len(dis) < 2 or len(dis) != len(oil): return
    n = len(dis)
    # 起始点（大于 0）
    init = -1
    # 生成纯能数组
    for i in range(n):
        dis[i] = oil[i] - dis[i]
        if dis[i] > 0: init = i

    return [False] * len(dis) if init < 0 else enlarge_area(dis, init)

def enlarge_area(dis, init):
    n = len(dis)
    res = [False] * n
    # 连通区起始点
    start = init
    # 连通区终点
    end = next_index(init, n)
    # 突破 start 需要油量
    need = 0
    # 剩余油量
    rest = 0

    # 以 init 为起始点，跑一圈
    while True:
        # 连通区 start 扩展（如果 end 无法突破，就扩展 start，所需的油都累计在 need 中）
        if dis[start] < need:
            # 如果 dis[start] 为负数，need 值增加
            # 如果 dis[start] 为正数，need 值减少
            need -= dis[start]
        else:
            # 将 need 的累积的油计算到 rest
            rest += dis[start] - need
            # 重置 need
            need = 0
            # end 连续突破
            while rest >= 0 and end != start:
                rest += dis[end]
                end = next_index(end, n)

            # 如果 end 连续突破后，rest 还有剩余，说明是 end == start 的条件跳出循环的，已经跑了一圈了。
            # 跑过一圈后，rest >= 0 油有剩余，说明以 start 是良好起始点
            # 跑过一圈后，rest < 0 油没有剩余，说明没有一个良好起始点，直接返回
            if rest >= 0:
                res[start] = True
                # 寻找其他的良好起始点
                # 所有能正常能达到 start 的加油站都是良好起始点
                # 所有从 start 上一个节点开始一路向上寻找能正常穿过 start 的加油站，并将对应 res 设置为 True
                connect_good(dis, last_index(start, n), init, res)
                # 已经跑了一圈了，其他良好起始点也寻找完毕，任务完成，跳出。
                break
        start = last_index(start, n)
        if start == init or start == last_index(end, n):
            break
    return res

def connect_good(dis, start, init, res):
    need = 0
    n = len(dis)
    while start != init:
        # 如果当前节点 start 无法穿越，用 need 记录所需要油，继续向上寻找
        if dis[start] < need:
            need -= dis[start]
        else:
            # 成功穿越
            res[start] = True
            need = 0
        start = last_index(start, n)

      
# 数组需要循环访问，需要在两个端点做特殊处理
# 获取 index 前一个索引
def last_index(index, size):
    return size - 1 if index == 0 else index - 1

# 获取 index 后一个索引
def next_index(index, size):
    return 0 if index == size - 1 else index + 1
```



**对数器**

```python
import random

def check():
    for _ in range(100):
        n = int(random.random() * 5) + 1
        oil = [int(random.random() * 5) + 1 for _ in range(n)]
        dis = [int(random.random() * 5) + 1 for _ in range(n)]

        res = stations(dis[:], oil[:])
        res2 = stations2(dis[:], oil[:])

        if res != res2:
            print("ERROR", "res=", res, "res2=", res2, "oil=", oil, "dis=", dis)
    print("Nice")

check()
```

