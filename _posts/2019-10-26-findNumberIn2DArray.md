---
layout: post
title: 二维数组中的查找
date: 2019-10-26
tags: 算法
---

#### [源码链接](https://github.com/zichen-fei/algorithm/blob/master/src/com/feizc/FindNumberIn2DArray.java)

### **题目描述**

在一个 n * m 的二维数组中，每一行都按照从左到右递增的顺序排序，每一列都按照从上到下递增的顺序排序。请完成一个函数，输入这样的一个二维数组和一个整数，判断数组中是否含有该整数。

输入

```
[
  [1,   4,  7, 11, 15],
  [2,   5,  8, 12, 19],
  [3,   6,  9, 16, 22],
  [10, 13, 14, 17, 24],
  [18, 21, 23, 26, 30]
]
```

输出

```
给定 target = 5，返回 true
给定 target = 20，返回 false
```

### **线性查找**

从二维数组的右上角开始查找。如果当前元素等于目标值，则返回 true。如果当前元素大于目标值，则移到左边一列。如果当前元素小于目标值，则移到下边一行。

```
public boolean findNumberIn2DArray(int[][] matrix, int target) {
    if (matrix == null) {
        return false;
    }
    int row = matrix.length;
    if (row == 0) {
        return false;
    }
    int col = matrix[0].length;
    if (col == 0) {
        return false;
    }
    int m = matrix.length;
    int n = matrix[0].length;
    int row = 0;
    int col = n - 1;
    while (row < m && col >= 0) {
        if (matrix[row][col] > target) {
            col--;
        } else if (matrix[row][col] < target) {
            row++;
        } else {
            return true;
        }
    }
    return false;
}
```

### **二叉搜索树**

以右上角为根，左方向为左节点，下方向为右节点，如果找到了则直接返回true，如果要找的值小于根，则递归到"左"孩子处；如果要找的值大于根，则递归到"右"孩子处。

```
public boolean findNumberIn2DArray(int[][] matrix, int target) {
    if (matrix == null) {
        return false;
    }
    int row = matrix.length;
    if (row == 0) {
        return false;
    }
    int col = matrix[0].length;
    if (col == 0) {
        return false;
    }
    return binarySearch(matrix, target, 0, col - 1, row, col);
}

public boolean binarySearch(int[][] matrix, int target, int row_i, int col_j, int row, int col) {
    if (row_i >= row || col_j < 0) {
        return false;
    }
    int root = matrix[row_i][col_j];
    if (root < target) {
        return binarySearch(matrix, target, row_i + 1, col_j, row, col);
    } else if (root > target) {
        return binarySearch(matrix, target, row_i, col_j - 1, row, col);
    } else {
        return true;
    }
}

```

**只能选取左下角或者右上角的数字，但不能选择左上角或者右下角的。以左上角为例，最初数字1位于初始数组的左上角，由于1小于7，
那么7应该位于1的右边或者下边。此时既不能从查找范围内剔除1所在的行，也不能剔除1所在的列，无法缩小查找的范围。**