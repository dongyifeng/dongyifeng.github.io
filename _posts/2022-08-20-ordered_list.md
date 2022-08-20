

---
layout: post
title: 有序表
description: 有序表的概述
tag: 算法,数据结构,有序表,AVL树,红黑树,Size Balanced Tree,跳表
---



- AVL 树
- 红黑树
- SB（Size Blanced Tree）
- 跳表



## 有序表的简单介绍

1. 有序表在使用层面上可以理解为一种集合结构
2. 如果只有 key，没有伴随数据 value，可以使用 TreeSet 结构
3. 如果既有 key，又有伴随数据 value，可以使用 TreeMap 结构
4. 有无伴随数据，是 TreeSet 和 TreeMap 唯一的区别，底层的实际结构是一回事
5. 有序表和哈希表的区别是，有序表把 key 按照顺序组织起来，而哈希表完全不组织
6. 红黑树、AVL 树、size-balance-tree 和跳表等都属于有序表，只是底层具体实现不同
7. 放入有序表中的数据，如果是基础类型，内部按值传递，内存占用就是数据的大小
8. 放入有序表中的数据，如果不是基础类型，必须提供比较器，内部按引用传递，内存占用是数据内存地址的大小
9. 不管是什么底层具体实现，只要是有序表，都有固定的基本功能和固定的时间复杂度。



## 有序表的固定操作

1.  put(key , value) ：将一个（key，value）记录加入到表中，或者将 key 的记录更新成 value
2.  V get( key )：根据给定的 key，查询 value 并返回。
3.  remove( key )：移除 key 的记录
4.  containsKey( key )：询问是否有关于 key 的记录 
5.  K firstKey()：返回所有键值的排序结果中，最左（最小）的那个
6.  K lastKey()：返回所有键值的排序结果中，最右（最大）的那个
7.  K floorKey( key )：如果表中存入过 key，返回 key，否则返回所有键值的排序结果中，key 的前一个
8.  K ceilingKey( key )：如果表中存入过 key，返回 key，否则返回所有键值的排序结果中，key 的后一个

以上所有操作的时间复杂度都是：O( logN)，N 为有序表含有的记录数



```java
    public static void main(String[] args) {
        TreeMap<Integer, String> treeMap = new TreeMap<Integer, String>();

        treeMap.put(7, "我是7");
        treeMap.put(5, "我是5");
        treeMap.put(4, "我是4");
        treeMap.put(3, "我是3");
        treeMap.put(9, "我是9");
        treeMap.put(2, "我是2");

        System.out.println(treeMap.containsKey(5));
        System.out.println(treeMap.get(5));
        System.out.println(treeMap.firstKey() + ",我最小");
        System.out.println(treeMap.lastKey() + ",我最大");
        System.out.println(treeMap.floorKey(8) + ",在表中所有 <= 8 的数中，我离 8 最近");
        System.out.println(treeMap.ceilingKey(8) + ",在表中所有 >= 8 的数中，我离 8 最近");
        System.out.println(treeMap.floorKey(7) + ",在表中所有 <= 7 的数中，我离 7 最近");
        System.out.println(treeMap.ceilingKey(7) + ",在表中所有 >= 7 的数中，我离 7 最近");
      
        treeMap.remove(5);
      
      System.out.println(treeMap.get(5) + ",删除就没有了哦");
    }
```



红黑树

为了保持树的平衡性。添加的硬性规则

1. 节点有红黑两种元素
1. 根节点为黑色
1. 叶子节点为黑色
1. 从每个叶子节点到根节点的路径上，不能出现两个连续的红色节点
1. 从任一节点到其每个叶子节点的所有路径上很色节点数目相同。

