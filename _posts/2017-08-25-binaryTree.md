---
layout: post
title: 逐层遍历二叉树
date: 2017-08-25
tags: 算法
---

### 题目描述：从上到下按层打印二叉树，同一层结点从左至右输出。每一层输出一行。
### **方法一：**

利用递归方式，搜寻并打印某一层节点，再打印下一层节点

1.要逐层打印二叉树的节点，可以先实现打印该二叉树某一层的算法：

```
public void PrintNodeOfLevel(BinaryTree<int> root, int level){
	if(root == null || level < 0)
		return;
	if(level == 0)
		System.out.println(root.Data + "  ");
	if(root.Left != null)
		PrintNodeOfLevel(root.Left, level - 1);
	if(root.Right != null)
		PrintNodeOfLevel(root.Right, level - 1);
}

```

2.逐层打印需要知道二叉树的深度：

```
public int GetDepth(BinaryTree<int> root){
	int lDepth = 0,rDepth = 0;
	if(root == null)
		return 0;
	if(root.Left != null)
		lDepth = GetDepth(root.Left);
	if(root.Right != null)
		rDepth = GetDepth(root.Right);
	int temp = 0;
	if(lDepth >= rDepth)
		temp = lDepth;
	else
		temp = rDepth;
	return temp + 1;
}
```
3.最后逐层打印二叉树

```
public void PrintBTree(BinaryTree<T> root){
	int depth = GetDepth(root);
	for(int i = 0; i < depth; i++){
		PrintNodeOfLevel(root, i);
		System.out.println("\n");
	}
}
```
这是通过递归，比较耗时的做法，但不需要额外的空间

### **方法二：**
利用队列，先将上层节点入队，节点出队的时候将其孩子节点入队，这样就可以达到按层入队出队的效果。
要打印某一层，可以在出队的时候一层一层的出，同时计算出队的次数，就可以判断出当前是哪一层

代码如下
```
public void printTree(BinaryTree<T> root){
	if(root == null)
		return;
	Queue<BinaryNode<T>> queue = new LinkedList<>();
        
	int current;//当前层 还未打印的结点个数
	int next;//下一层结点个数
        
	queue.offer(root);
	current = 1;
	next = 0;
	while(!queue.isEmpty()){
		Node<T> currentNode = queue.poll();
		System.out.printf("%-4d", currentNode.element);
	    current--;
	            
	    if(currentNode.left != null){
			queue.offer(currentNode.left);
			next++;
		}
		if(currentNode.right != null){
		    queue.offer(currentNode.right);
		    next++;
		}
		if(current ==0){
		    System.out.println();
		    current = next;
		    next = 0;
		}
	}
}
```