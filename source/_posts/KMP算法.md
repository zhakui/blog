---
title: KMP算法
date: 2017-01-12 15:25:37
categories:
  算法 
tags:
  - 算法
---
# 什么是KMP算法
  KMP算法是其实就是一种字符串匹配算法（字符串中宣召匹配的子串），由D.E.Knuth、J.H.Morris和V.R.Pratt同时发现的。

# KMP算法的思想
  假设现在有字符串A.那么我们需要在A中找到是否包含字符串B，如果存在就返回起始位置。  
  A:ABCDABCABCDEABCEABDE 
  B:ABCDE
A串中出现了两次B串：  
ABCDABC<font color=#ff0000>ABCDE</font>ABCEABDE  
传统方法是，从左到右一个字符一个字符的比较，当遇到不匹配的字符就回到上次开始匹配的下一位继续匹配。  
下面给出一段暴力搜索的代码：
```java
public class General{

    public static void main(String[] args){
        String txt = "ABCDABCABCDEABCEABDE";
        String pat = "ABCDE";
        int a = search(pat, txt);
        if (a == txt.length()){
            System.out.println("Not Found!");
        }else{
            System.out.println("Found: " + a);
        }
    }

    private static int search(String pat, String txt){
        int M = pat.length();
        int N = txt.length();

        for (int i = 0; i <= N - M; i++){
            int j;
            for (j = 0; j < M; j++){
                if (txt.charAt(i+j) != pat.charAt(j)){
                    break;
                }
            }
            if (j == M) return i;//Found
        }
        return N;//Not Found
    }
}
```
    