---
layout: post
title: 最长不重复子串
date: 2017-07-26
tags: 算法
---

### 问题描述：给定一个字符串，找出这个字符串中最长的不重复串。

比如：对于字符串"abcba"，那么返回的结果应该是"abc"或者"cba"（返回一个即可）；对于字符串"acbba"，返回的应是"acb"

思路：利用HaspMap，map的key存储的是字符，value存储的是字符当前的位置，利用containsKey()方法检测是否有重复  
1、如果当前字符出现过并且index大于该字符上一次出现的index，那么将map中该字符对应的value值替换，上一次出现的字符的下一个字符到当前字符变为目前新的子串，（此时子串不一定是最大长度的子串，而是程序运行过程中当前不重复的子串）  
2、如果目前新子串的长度（当前字符的index与startIndex（目前新子串的初始index）的差值 + 1）大于maxLen（最长不重复子串长度），更新maxLen，如果要输出这个不重复子串，需要记录startIndex  
3、记录当前字符的index  
以"abcba"为例：  
1、map.put('a',0)，初始startIndex为0，maxLen为1，map.put('b',1)，startIndex为0，maxLen为2，map.put('c',2)，startIndex为0，maxLen为3，此时子串为"abc"，map.put('b',3)，检测到有重复，则目前新的子串变为‘cb’，将map中字符'b'的index替换为3，maxLen为2，startIndex变为2  
2、目前新的子串为'cb'，index为3，startIndex为2，长度为2，小于最大长度，不更新maxLen  
扫描完以后，根据oriStartIndex（maxLen改变时记录的startIndex）和maxLen来得到最长不重复子串  

具体代码如下：

```
import java.util.HashMap;

public class findLongestSubString {
    public static void main(String[] args) {
        String str = "120135435";
		StringBuilder maxSubString = new StringBuilder("");  
        char[] strCharArr = str.toCharArray();  
        HashMap<Character, Integer> charsIndex = new HashMap<Character, Integer>();  
        int startIndex = 0, oriStartIndex = startIndex, maxLen = 0;  
        for(int index = 0; index < strCharArr.length; index++) {  
            if(charsIndex.containsKey(strCharArr[index])) {  
                int oriIndex = charsIndex.get(strCharArr[index]);  
                if(oriIndex >= startIndex){  
                    startIndex = oriIndex + 1;  
                }  
            }  
            if(index - startIndex + 1 > maxLen) {  
                maxLen = index - startIndex + 1;  
                oriStartIndex = startIndex;  
            }  
            charsIndex.put(strCharArr[index], index);  
        }  
        for(int index =  oriStartIndex; index < oriStartIndex + maxLen; index++) {  
            maxSubString.append(strCharArr[index]);  
        }  
        System.out.println(maxSubString.toString());
}
```