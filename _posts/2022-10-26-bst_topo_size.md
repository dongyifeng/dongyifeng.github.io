---
title: 搜索二叉树的最大拓扑结构
tags: 每日一题 树型DP
typora-root-url: ../../dongyifeng.github.io
---

> 给定一棵二叉树的头节点 head，已知所有节点的值都不一样，返回其中最大的且符合搜索二叉树的最大拓扑结构的大小。
>
> 拓扑结构：不是子树，只要能连起来的结构都算。



概念：拓扑贡献度

下图蓝色拓扑结构是一棵以节点 “5” 为根节点的搜索二叉树的拓扑贡献度为 5 = node.left.拓扑贡献度 +  node.right.拓扑贡献度 + 1

以节点 “3” 为根节点的搜索二叉树的拓扑贡献度为 2

以节点 “10” 为根节点的搜索二叉树的拓扑贡献度为 2

![](/images/assets/screenshot-20221026-221157.png)

**解法一：暴力算法**

计算以每个节点为根节点拓扑贡献度，取最大值。

```python
class TreeNode:
    def __init__(self, val, left=None, right=None):
        self.val = val
        self.left = left
        self.right = right

def bst_topo_size(head: TreeNode):
    if not head: return 0
	
  	# 计算以 head 为根节点的拓扑贡献度
    res = max_topo(head, head)
    # 计算以 head.left 为根节点的拓扑贡献度
    res = max(res, bst_topo_size(head.left))
    # 计算以 head.right 为根节点的拓扑贡献度
    res = max(res, bst_topo_size(head.right))
    # 最大值
    return res

def max_topo(head: TreeNode, node: TreeNode):
    if head and node and is_bst_node(head, node, node.val):
        return max_topo(head, node.left) + max_topo(head, node.right) + 1
    return 0

def is_bst_node(head: TreeNode, node: TreeNode, value):
    if not head: return False
    if head == node: return True
    return is_bst_node(head.left if head.val > value else head.right, node, value)
```



**解法二：树型 DP**

以节点 “X” 为根节点的搜索二叉树的拓扑贡献度：

- 如果 X.val < X.left.val 并且 X.val > X.right.val （满足搜索二叉树），a + b + 1
- 否则 1
- 最终结果：max( X.拓扑贡献度 , max( X.left.拓扑贡献度, X.right.拓扑贡献度) )



![](/images/assets/screenshot-20221026-222803.png)

如下图：要计算当前节点18 左子树的拓扑贡献度。如果左节点大于 18 ，拓扑贡献度为 0。现在左节点是 10 < 18，左节点有效。左节点的左子树都小于10，所以左节点的左子树拓扑贡献度100，直接收下。左节点的右子树的值都大于10，也有可能大于18，所以需要探查左节点的右边界，一旦遇到大于18的节点，需要删除结点，减去对应的贡献度，并从下向上依次修复对应的拓扑贡献度。

在以往的树型 DP 的问题中，从左子树获取的 left_info 和从右子树获取的 right_info ,用完就丢弃了。本地我们需要使用之前每个节点的 left_info 和 right_info，并且要修改历史的 left_info 和 right_info。因此我们使用 map 存储之前计算过 Info 信息，key ：节点，value ： left_info 和 right-info。

![](/images/assets/screenshot-20221026-224244.png)

以 X 为根节点求拓扑贡献度时，需要处理 X.left 的右边边界和 X.right 的左边界。

![](/images/assets/screenshot-20221026-224657.png)

时间复杂度：O(N)

如下图，看一下每个节点右边界：

- 节点 1  --> 2，5，11
- 节点 2  --> 4，9
- 节点 4  --> 8
- 节点 5 --> 10
- 节点 3 --> 6，13
- 节点 6 --> 12
- 节点 7 --> 14

在处理整棵树每个节点的右边界时，没有重复，是 O(N)

同理在处理整棵树每个节点的左边界时，也是 O(N)，所以整体算法的时间复杂度为：O(N)

![](/images/assets/screenshot-20221026-225348.png)

```python
class Record:
    def __init__(self, left: TreeNode, right: TreeNode):
        self.left = left
        self.right = right


def bst_topo_size2(head: TreeNode):
    map = {}
    return pos_order(head, map)


def pos_order(head: TreeNode, map):
    if not head: return 0

    left_info = pos_order(head.left, map)
    right_info = pos_order(head.right, map)
    modify_map(head.left, head.val, map, True)
    modify_map(head.right, head.val, map, False)
    # 修改后的值
    left_record = map.get(head.left, None)
    right_record = map.get(head.right, None)

    left_bst = 0 if not left_record else left_record.left + left_record.right + 1
    right_bst = 0 if not right_record else right_record.left + right_record.right + 1
    map[head] = Record(left_bst, right_bst)
    return max(left_bst + right_bst + 1, max(left_info, right_info))


# 返回值是要减掉的贡献记录
def modify_map(node: TreeNode, val, map, is_left):
    if not node or node not in map: return 0

    record = map.get(node)
    # 左节点或者右节点不满足搜索二叉树
    if (is_left and node.val > val) or (not is_left and node.val < val):
        map.pop(node)
        # node 要被删除，所以他的贡献记录需要删掉
        return record.left + record.right + 1
    else:
        minus = modify_map(node.right if is_left else node.left, val, map, is_left)
        if is_left:
            record.right -= minus
        else:
            record.left -= minus
        map[node] = record
        return minus
```



**对数器：**

```python
import random

def generator_random_arr(max_size):
    num = range(int(random.random() * max_size) + 1)
    n = len(num)
    return random.sample(num, int(random.random() * n) + 1)

def insert(root, data):
    temp = root
    while temp:
        p = temp
        temp = temp.left if data < temp.val else temp.right

    if data < p.val:
        p.left = TreeNode(data)
    else:
        p.right = TreeNode(data)

def generator_bst(arr):
    root = None
    for item in arr:
        if not root:
            root = TreeNode(item)
            continue

        insert(root, item)
    return root

def check():
    max_size = 10
    for i in range(1000):
        arr = generator_random_arr(max_size)
        root1 = generator_bst(arr)
        root2 = generator_bst(arr)

        res1 = bst_topo_size(root1)
        res2 = bst_topo_size2(root2)

        # print("int", res1, res2)
        if res1 != res2:
            print("ERROR", res1, res2, arr)
    print("OVER")

check()
```