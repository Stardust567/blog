---
title: KMP
date: 2018-12-08 23:09:54
tags: [数据结构,大一]
categories: 数据结构
---
普通的查找匹配需要在失配时回溯到失配串第二位继续开始查找是否匹配，复杂度过高。

于是我们想着能不能一次不回头的走到底，针对模式串创建了一个辅助数组（next数组）
<!-- More -->
next数组各个元的值：固定字符串的最长真前缀（第一个字符伊始，但不含最后一个）和最长真后缀相同的长度，以下举例。


n e x t 数 组 |	各元固定字符串 | 相同最长前后缀 | 前后缀相同长度
:----:|:----:|:----:|:----: 
n e x t [0] | a	| “ ” |0
n e x t [1]	| ab | “ ” | 0
n e x t [2]	| aba | “a” | 1
n e x t [3]	| abab | “ab” | 2
n e x t [4]	| ababa	| “aba”	| 3
n e x t [5]	| ababab | “abab” | 4
n e x t [6]	| abababc | “ ”	| 0
n e x t [7]	| abababca | “a” | 1
  a b a b a b c a   的 n e x t 数 组 为 ：

-1 0 0 1 2 3 4 0 1

因为你的目标串没有必要完全回溯，可以把前面相同的位置掠过，因此有了基于next数组的KMP算法，以下举例：

目标串T：ababaacababbabababca

模式串P：abababca



指针i指向目标串，指针j指向模式串，if（T[i]==P[j]）//匹配成功    i++ , j++//往后继续尝试是否匹配

如果失配，i指针不回溯，j指针回溯（j = next[j]），当回溯到首元时，无法再回溯，继续往后试探 i++ , j++

```C
​
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef struct{
	char *pch;
	int len;
}Tstr;

int* KMP(Tstr* T,Tstr* P,int *next){
	int i=0,j=0,k=0,m = T->len,n = P->len;
	int num[m];
	for(int i=0;i<m;i++) num[i]=0;
	while(i<=m-n){ //不超过目标串的长度，且i>m-n时不可能存在匹配
		while(j==-1||(j<n&&T->pch[i]==P->pch[j])){ //只要相同且不超过模式串长度就继续往前走
			i++;
			j++;
		}
		if(j==n) num[k++] = (i-n+1);//成功找到一个匹配
		j = next[j];//通过next数组回溯模式串，继续寻找匹配
	}
	return num;
}

int* KMP_next(Tstr* P){
	int *next = (int*)malloc(sizeof(int)*(P->len));
	int j=0,k=-1,n = P->len;
	next[0] = -1;
	while(j<n){
		if(k==-1||P->pch[j]==P->pch[k]){
			next[j+1] = k+1;
			j++;
			k++;
		}
		else k = next[k];
	}
	return next;
	
}

int* KMP_nextval(Tstr* P){
	int* nextval = KMP_next(P);
	for(int j=1;j<P->len;j++){
		if(P->pch[j]==P->pch[nextval[j]]) nextval[j]=nextval[nextval[j]];
	}
	return nextval;
}
```

