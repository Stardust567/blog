---
title: Huffman
date: 2018-12-08 23:22:03
tags: [数据结构,大一]
categories: 数据结构
---
### 概念：
哈夫曼树也称为最优二叉树，是一种带权路径长度最短的二叉树。（树的带权路径长度：树中所有的叶结点的权值乘上其到根结点的路径长度，然后进行加和的结果）哈夫曼树中根结点为第0层，N个权值构成一棵有N个叶结点的二叉树，相应的叶结点的路径长度为其层数。树的路径长度是从树根到每一结点的路径长度之和，记为WPL 易证哈夫曼树的WPL是最小的。

<!-- More -->

### 原理：
编码表是通过对源符号出现的概率进行评估的方式得到，出现概率高的符号使用较短的编码，反之则使用较长的编码，由此实现编码后字符串的平均长度的期望值降低，从而达到无损压缩数据的目的。



### 举例：
根据文本文件得出45个不同字符，通过所给的函数初始化哈夫曼树，根据45个初始节点完善填充整个（2*n-1 = 89）哈夫曼数组，再根据哈夫曼数组制作密码本，进行文件的压缩（加密）与解压（解密）的功能实现。

```C​
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define LEN 100

typedef struct{
	char ch;
	int weight;int parent;
	int lchild;int rchild;
}TNode;

typedef struct{
	char ch;
	char code[LEN];
}TCode;

typedef struct{
	char ch;
	int weight;
}TW;


void select_subtree(TNode* pht,int n,int* pa,int* pb){
	for(int i=0;i<n;i++){
		if(pht[i].parent==-1){
			*pa = i;
			break;
		}
	}
	for(int j=*pa+1;j<n;j++){
		if(pht[j].parent==-1){
			*pb = j;
			break;
		}
	}
	//printf("\n%d\t%d\t%d\t%d\t",*pa,*pb,pht[*pa].parent,pht[*pb].parent); 
	int temp = *pa;
	*pa = (pht[*pa].weight>pht[*pb].weight)?*pb:*pa;
	*pb = (pht[*pb].weight<pht[temp].weight)?temp:*pb;
	//printf("\n%d\t%d",*pa,*pb); 
	for(int i=0;i<n;i++){
		if(pht[i].parent==-1){
			if(pht[i].weight < pht[*pa].weight&&i!=*pa&&i!=*pb) {*pb = *pa;*pa = i;}
			else if(pht[i].weight < pht[*pb].weight&&i!=*pa&&i!=*pb) {*pb = i;}
		}
	}
	//printf("\n%d\t%d",*pa,*pb); 
}

TNode* create_htree(TW weights[],int n){
	TNode* pht = (TNode*)malloc(sizeof(TNode)*(2*n-1));
	for(int i=0;i<n*2-1;i++){
		pht[i].ch = (i<n)?weights[i].ch:' ';
		pht[i].weight = (i<n)?weights[i].weight:0;
		pht[i].parent = -1;
		pht[i].lchild = -1;pht[i].rchild = -1;
	}
	int pa,pb;
	for(int i=n;i<n*2-1;i++){
		select_subtree(pht,i,&pa,&pb);
		pht[pa].parent=i;pht[pb].parent=i;
		pht[i].lchild = pa;pht[i].rchild = pb;
		pht[i].weight = pht[pa].weight+pht[pb].weight;
		//printf("\n%d\t%02d\t%02d\t%02d\t%02d\n",i,pht[i].weight,pht[i].parent,pht[i].lchild,pht[i].rchild);
		//printf("%d\t%d\n",pht[pht[i].lchild].parent,pht[pht[i].rchild].parent);
	}
	return pht;
}

void encoding(TNode* pht,TCode book[],int n){
	char* str = (char*)malloc(n+1);
	str[n] = '\0';
	for(int i=0;i<n;i++){
		int idx = i;int j = n;
		while(pht[idx].parent!=-1){
			if(pht[pht[idx].parent].lchild==idx){
				j--;str[j]='0';
			}
			if(pht[pht[idx].parent].rchild==idx){
				j--;str[j]='1';
			}	
			idx = pht[idx].parent;
		}
		book[i].ch = pht[i].ch;
		strcpy(book[i].code,&str[j]);
		printf("%c  :  ",book[i].ch);
		puts(book[i].code);
	}
}

void decoding(TNode* pht,char codes[],int n){
	freopen("your_love.txt","w",stdout);
	int i=0,p = 2*n-2; 
	
	while(codes[i]!='\0'){
		while(pht[p].lchild!=-1&&pht[p].rchild!=-1){
			if(codes[i]=='0') p = pht[p].lchild;
			else p = pht[p].rchild;
			i++;
		}
		printf("%c",pht[p].ch);	
		p = 2*n-2;
		
	} 
	printf("\n");
	fclose(stdout);
	
}

// 统计字符串text中字符出现的频率，参数n为字符串长度
// 返回值为：text中出现的不同种类的字符个数
// 副作用：通过指针参数间接返回两个数组，其中：
//		dict：字符数组，存放 text中出现的不同种类的字符
//		freq：整型数组，存放 text中出现的不同种类的字符的出现频率 
int calc_freq(char text[], int **freq, char **dict, int n)
{
	int i, k, nchar = 0;
	int * pwght;
	char * pch;
	int tokens[256] = {0};

	// 根据输入的文本字符串逐一统计字符出现的频率 
	for(i = 0; i < n; ++i){
		tokens[text[i]]++;
	}
	
	// 统计共有多少个相异的字符出现在文本串中 
	for(i = 0; i < 256; i++){
		if( tokens[i] > 0 ){
			nchar++;
		}
	}

	// 为权重数组分配空间 
	pwght = (int*)malloc(sizeof(int)*nchar);
	if( !pwght ){
		printf("为权重数组分配空间失败！\n");
		exit(0);
	}
	
	// 为字符数组（字典）分配空间 
	pch = (char *)malloc(sizeof(char)*nchar);
	if( !pch ){
		printf("为字符数组（字典）分配空间失败！\n");
		exit(0);
	}
	
	k = 0;
	for(i = 0; i < 256; ++i){
		if( tokens[i] > 0 ){
			pwght[k] = tokens[i];
			pch[k] = (char)i;  //强制类型转换 
			k++;
		}
	}
	
	*freq = pwght;
	*dict = pch;
	
	return nchar;
} 
 
int main(){
	freopen("love_letter.txt","r",stdin);
	char* str = (char*)malloc(2000);	
	gets(str);
	fclose(stdin);
	
	int** ch_f = (int**)malloc(4);
	char**ch_c = (char**)malloc(4);	
	int n = calc_freq(str,ch_f,ch_c,strlen(str));
	free(str);	
	TW weights[n];
	for(int i=0;i<n;i++){
		weights[i].weight = (*ch_f)[i];
		weights[i].ch = (*ch_c)[i];
	}
	free(ch_f);free(ch_c);
	
	TNode* pht = create_htree(weights,n);
	TCode codebook[n];
	encoding(pht,codebook,n);
	freopen("love_letter.txt","r",stdin);
	char* word = (char*)malloc(2000);
	gets(word);
	fclose(stdin);
	freopen("codebook.txt","w",stdout);
	while(*word!='\0'){
		for(int i=0;i<n;i++){
			if(*word==codebook[i].ch){
				for(int j=0;j<strlen(codebook[i].code);j++){
					printf("%c",codebook[i].code[j]);
				}
			}
		}  
		word++;
	}
	fclose(stdout);
	//free(word);
	freopen("codebook.txt","r",stdin);
	char* codes = (char*)malloc(6000);
	gets(codes);
	fclose(stdin);
	decoding(pht,codes,n);
	return 0;
}
​```