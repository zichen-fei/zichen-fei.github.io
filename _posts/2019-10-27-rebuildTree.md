---
layout: post
title: 重建二叉树
date: 2019-10-27
tags: 算法
---

#### [源码链接](https://github.com/zichen-fei/algorithm/blob/master/src/com/feizc/BuildTree.java)

### **题目描述**

输入某二叉树的前序遍历和中序遍历的结果，请重建该二叉树。假设输入的前序遍历和中序遍历的结果中都不含重复的数字。

输入
```
前序遍历 preorder = [3,9,20,15,7]
中序遍历 inorder = [9,3,15,20,7]
```
输出
```
    3
   / \
  9  20
    /  \
   15   7
```

### **分治, 递归**

二叉树的问题一般都是分治思想，递归去做。因为二叉树本身就是递归定义的。

+ 前序遍历的第 1 个结点一定是二叉树的根结点  
+ 在中序遍历中，根结点把中序遍历序列分成了两个部分，左边部分构成了二叉树的根结点的左子树，右边部分构成了二叉树的根结点的右子树
+ 查找根结点在中序遍历序列中的位置，可以遍历，也可以在一开始就记录下来。

先找到preorder中的起始元素作为根节点，在inorder中找到根节点的索引mid，那preorder[1:mid + 1]为左子树，preorder[mid + 1:]为右子树，
inorder[0:mid]为左子树，inorder[mid + 1:]为右子树。递归建立二叉树。

```
public TreeNode buildTree(int[] preorder, int[] inorder) {
    if (preorder == null || preorder.length == 0) {
        return null;
    }
    int length = preorder.length;
    Map<Integer, Integer> map = new HashMap<>();
    for (int i = 0; i < length; i++) {
        map.put(inorder[i], i);
    }
    return buildTree(preorder, 0, length - 1, inorder, 0, length - 1, map);
}

public TreeNode buildTree(int[] preorder, int preStart, int preEnd, int[] inorder, int inStart, int inEnd, Map<Integer, Integer> map) {
    if (preStart > preEnd) {
        return null;
    }
    int rootVal = preorder[preStart];
    TreeNode root = new TreeNode(rootVal);
    if (preStart == preEnd) {
        return root;
    } else {
        int rootIndex = map.get(rootVal);
        int leftNodes = rootIndex - inStart;
        int rightNodes = inEnd - rootIndex;
        TreeNode leftSubTree = buildTree(preorder, preStart + 1, preStart + leftNodes, inorder, inStart, rootIndex - 1, map);
        TreeNode rightSubTree = buildTree(preorder, preEnd - rightNodes + 1, preEnd, inorder, rootIndex + 1, inEnd, map);
        root.left = leftSubTree;
        root.right = rightSubTree;
        return root;
    }
}
```
