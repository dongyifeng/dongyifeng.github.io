---
title: 最大宽度坡
tags: 每日一题 单调栈
typora-root-url: ../../dongyifeng.github.io
---

> 给定一个整数数组 A，坡是元组 (i, j)，其中  i < j 且 A[i] <= A[j]。这样的坡的宽度为 j - i。
>
> 找出 A 中的坡的最大宽度，如果不存在，返回 0 。

**示例 1：**

```python
输入：[6,0,8,2,1,5]
输出：4
解释：
最大宽度的坡为 (i, j) = (1, 5): A[1] = 0 且 A[5] = 5.
```

**示例 2：**

```
输入：[9,8,1,0,1,9,4,0,4,1]
输出：7
解释：
最大宽度的坡为 (i, j) = (2, 9): A[2] = 1 且 A[9] = 1.
```



分析：

坡的元组( i , j ) 分为左数据节点和右数据节点，使用单调栈存放左数据节点（从栈顶到栈底是从小到大）。然后从后向前遍历（j 位置大）作为右数据节点，去匹配，如果匹配成功，作为一个坡，单调栈弹出元素。如果匹配不成成功，j -=1 换一个右数据节点。

单调栈中是左数据节点，由于单调性，一旦使用后续就不需要再考虑的。

从右向左遍历，保证了右数据节点的单调性。

![](/images/assets/screenshot-20221029-155313.png)

时间复杂度：O(N)

空间复杂度：O(N)

```python
def max_width_ramp(arr):
    stack = [0]
    for i in range(1, len(arr)):
        if arr[i] < arr[stack[-1]]:
            stack.append(i)

    i = len(arr) - 1
    res = 0
    while stack:
        if arr[stack[-1]] <= arr[i]:
            res = max(res, i - stack[-1])
            stack.pop()
        else:
            i -= 1
```

