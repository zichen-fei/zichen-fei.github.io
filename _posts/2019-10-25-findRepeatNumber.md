---
layout: post
title: 数组中重复的数字
date: 2019-10-25
tags: 算法
---

#### [源码链接](https://github.com/zichen-fei/algorithm/blob/master/src/com/feizc/FindRepeatNumber.java)

### **题目描述**
在一个长度为 n 的数组 nums 里的所有数字都在 0～n-1 的范围内。数组中某些数字是重复的，但不知道有几个数字重复了，也不知道每个数字重复了几次。请找出数组中任意一个重复的数字。

输入
```
[2, 3, 1, 0, 2, 5, 3]
```
输出
```
2 或 3
``` 

### **方法一: 先排序再遍历**

先排序，然后看相邻元素是否有相同的，有直接return。 不过很慢，时间O(nlogn)了，空间O(1)，会修改原数组

```
public int findRepeatNumber(int[] nums) {
    quickSort(nums, 0, nums.length-1);
    for(int i = 0;i< nums.length - 1;i++) {
        if(nums[i]== nums[i + 1] ) {
            return nums[i];
        }
    }
    return -1;
}

public void quickSort(int[] a, int left, int right) {
    if (left < right) {
        int pivot = a[left];
        int lo = left;
        int hi = right;
        while (lo < hi) {
            while (lo < hi && a[hi] >= pivot) {
                hi--;
            }
            a[lo] = a[hi];
            while (lo < hi && a[lo] <= pivot) {
                lo++;
            }
            a[hi] = a[lo];
        }
        a[lo] = pivot;
        quickSort(a, left, lo - 1);
        quickSort(a, lo + 1, right);
    }
}
```

### **方法二: 哈希表**

时间O(n)，空间O（n），不改变原数组

```
public int findRepeatNumber(int[] nums) {
    Map<Integer, Integer> map = new HashMap<>();
    for (int i = 0; i < nums.length; i++) {
        if (map.containsKey(nums[i])) {
            return nums[i];
        }
        map.put(nums[i], 1);
    }
    return -1;
}
```

### **方法三: 原地排序**

因为出现的元素值 < nums.size(); 所以我们可以将见到的元素 放到索引的位置，如果交换时，发现索引处已存在该元素，则重复 O(N) 空间O(1)，会修改原数组

```
public int findRepeatNumber(int[] nums) {
    int temp;
    for (int i = 0; i < nums.length; i++) {
        while (nums[i] != i) {
            temp = nums[i];
            if (nums[temp] == temp) {
                return temp;
            }
            nums[i] = nums[temp];
            nums[temp] = temp;
        }
    }
    return -1;
}
```

### **方法四**

另外初始化一个长度一样的数组b，每个元素赋值为-1，然后遍历数组，把数组的元素依照其自身的值，放入数组b中对应编号的位置，如果出现了重复的元素，就返回

```
public int findRepeatNumber(int[] nums) {
    int[] b = new int[nums.length];

    for (int j = 0; j < num.length; j++) {
        b[j] = -1;
    }

    for (int i = 0; i < nums.length; i++) {
        if (nums[i] == num[nums[i]]) {
            return nums[i];
        } else {
            b[nums[i]] = nums[i];
        }
    }
    return -1;
}
```

**题目虽然简单，但是还考察了另一个重点，就是沟通能力，要先询问面试官要求时间或者空间优先**  
+ 时间优先就用哈希表
+ 空间优先就用原地排序
+ 如果要求空间优先并且不能修改原数组就用方法四