---
title: 矩形重叠个数
tags: 每日一题 图形图
typora-root-url: ../../dongyifeng.github.io
---

> 平面内有 n 个矩形，第 i 个矩形的左下角坐标为$(x_1[i],y_1[i])$，右上角坐标为：$(x_2[i],y_2[i])$。如果两个或多个矩形有公共区域，则认为他们是相互重叠的（不考虑边界和角落）。请你计算出平面内重叠矩形数量最多的地方，有多少个矩形相互重叠？



分析：

<font color=red>**二维的图像题一般可以转化为一维的图像题**</font>

<font color=green>**任何一个重合区域的底，一定是某一个矩形的底。**</font>

对矩形底边排序，先出处理下方的矩形，依次向上处理。

在处理线段重叠问题时使用了有序表，此时我们假设有一个容器。将的底处在同一条直线的矩阵加入容器，如下图：【A，B，C】，依次向上处理。遇到 D，将 D 也加入容器。遇到 E 时，需要将容器中矩阵的上边小于 E 的底边的矩阵从容器中删除（因为上边都小于 E 的底边，不能与E这一层的矩阵有重叠）。

![](/../typora/images/algorithm/screenshot-20221112-115835.png)

删除容器中上边小于 E 的底边的矩阵后，容器中剩下的矩阵大概如下的样子。矩阵中的矩形，都是在以 E 为辐射范围内可能重叠的矩形。

由于任何一个重合区域的底，一定是某一个矩形的底，因此我们可以将容器中矩形的底边看成在一条直线的线段，只要这些线段重叠，那么对应的矩形也是重叠。至此，我们将矩形重叠问题转化为线段重叠问题。

![](/../typora/images/algorithm/screenshot-20221112-120948.png)

```java
 public static int rectangleCoverMax(int[][] matrix) {
        if (matrix == null || matrix.length == 0) {
            return 0;
        }
        // 按照底边排序
        Arrays.sort(matrix, (e1, e2) -> (e1[1] - e2[1]));

        int res = 0;
        // map 中 value 的和
        int count = 0;

        // key：是 arr 的 end；value：矩阵集合
        TreeMap<Integer, List<int[]>> map = new TreeMap<>();
        for (int i = 0; i < matrix.length; i++) {
            int x1 = matrix[i][0];
            int y1 = matrix[i][1];
            int x2 = matrix[i][2];
            int y2 = matrix[i][3];

            List<int[]> list = map.getOrDefault(y2, new ArrayList<>());
            list.add(new int[]{x1, x2});
            map.put(y2, list);
            count += 1;

            // 将 map 中所有 key( end ) 小于等于 start 的删除掉
            while (true) {
                Map.Entry<Integer, List<int[]>> entry = map.floorEntry(y1);
                if (entry == null) {
                    break;
                }
                count -= entry.getValue().size();
                map.remove(entry.getKey());

                // map 中 value 的和减少 entry.getValue()
                y1 = entry.getKey();
            }

            int[][] segment_matrix = new int[count][2];
            int j = 0;
            for (int key : map.keySet()) {
                for (int[] rec : map.get(key)) {
                    segment_matrix[j][0] = rec[0];
                    segment_matrix[j][1] = rec[1];
                }
                j += 1;
            }
            res = Math.max(res, segmentCoverMax(segment_matrix));
        }
        return res;
    }
```

