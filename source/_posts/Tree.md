---
title: Tree
date: 2018-12-08 23:25:45
tags: [数据结构,大一]
categories: 数据结构
---
二叉树的编程我觉着可以帮助你学习递归 : )
<!-- More -->
```C
#include <stdio.h>
#include <stdlib.h>

typedef int ElemType;
typedef struct node{
	ElemType data;
	struct node* lc;
	struct node* rc;
	int visit;//真的，如果不是这样写最方便，我也不想直接改掉TNode结构体orz 
}TNode;
typedef struct{
	TNode** qu;
	int n;
	int front;
	int rear;
}Tqueue;

TNode* tree_init();//树的初始化
void   preorder(TNode* tree); //递归遍历
void   tree_pre(TNode* tree); //非递归遍历
void   inorder(TNode* tree); 
void   tree_in(TNode* tree); 
void   postorder(TNode* tree); 
void   tree_post(TNode* tree); 
void   level_order(TNode* tree);//层次遍历
int    tree_nodes(TNode* tree);
int    tree_leaves(TNode* tree);
int    tree_depth(TNode* tree);
int    tree_level(TNode* tree,int x,int* j);//查找
int    tree_width(TNode* tree);

Tqueue* queue_init(int n){
	Tqueue* queue = (Tqueue*)malloc(sizeof(Tqueue));
	queue->qu = (TNode**)malloc(sizeof(TNode*)*n);
	queue->front = 0;
	queue->rear = 0;
	queue->n = n;
	return queue;
}

int main(){
	freopen("tree.txt","r",stdin);
	TNode* tree = tree_init();
	preorder(tree);printf("\n");tree_pre(tree);printf("\n");
	inorder(tree);printf("\n");tree_in(tree);printf("\n");
	postorder(tree);printf("\n");tree_post(tree);printf("\n");
	level_order(tree);
	printf("\nTree leaves: %d\tTree depth: %d\tTree nodes: %d",tree_leaves(tree),tree_depth(tree),tree_nodes(tree));
	int num;tree_level(tree,'K',&num);printf("\t'K': %d",num+1);
	printf("\tTree width: %d",tree_width(tree));
} 

TNode* tree_init(){
	TNode* tree = (TNode*)malloc(sizeof(TNode));
	if(tree==NULL){
		printf("tree_init:malloc error.");
		exit(0);
	}
	ElemType a;
	scanf("%d",&a);
	if(a==0) tree=NULL;
	else{
		tree->data = a;
		tree->visit= 0; 
		tree->lc = tree_init();
		tree->rc = tree_init();
	}
	return tree; 
}

void preorder(TNode* tree){
	if(tree==NULL) return;
	else{
		printf("%c  ",tree->data);
		preorder(tree->lc);
		preorder(tree->rc);
	}
}

void tree_pre(TNode* tree){
	Tqueue* stack = queue_init(tree_nodes(tree));//假装这是初始化栈的函数
	while((stack->rear!=0)||(tree!=NULL)){
		if(tree!=NULL){
			stack->qu[stack->rear++] = tree;
			printf("%c  ",tree->data);
			tree = tree->lc;
		}
		else{
			tree = stack->qu[--stack->rear];
			tree = tree->rc;
		}
	}
	free(stack);
} 


void inorder(TNode* tree){
	if(tree==NULL) return;
	else{
		inorder(tree->lc);
		printf("%c  ",tree->data);
		inorder(tree->rc);
	}
}

void tree_in(TNode* tree){
	Tqueue* stack = queue_init(tree_nodes(tree));
	while((stack->rear!=0)||(tree!=NULL)){
		if(tree!=NULL){
			stack->qu[stack->rear++] = tree;
			tree = tree->lc;
		}
		else{
			tree = stack->qu[--stack->rear];
			printf("%c  ",tree->data);
			tree = tree->rc;
		}
	}
	free(stack);
} 

void postorder(TNode* tree){
	if(tree==NULL) return;
	else{
		postorder(tree->lc);
		postorder(tree->rc);
		printf("%c  ",tree->data);
	}
}

void tree_post(TNode* tree){
	Tqueue* stack = queue_init(tree_nodes(tree));
	while((stack->rear!=0)||(tree!=NULL)){
		if(tree!=NULL){
			if(tree->visit==0) stack->qu[stack->rear++] = tree;
			tree = tree->lc;
		}
		else{
			tree = stack->qu[--stack->rear];
			if(tree->visit==1) printf("%c  ",tree->data);
			else{
				tree->visit = 1;
				stack->qu[stack->rear++] = tree;
			}
			tree = tree->rc;
		}
	}
	free(stack);
} 

void level_order(TNode* tree){
	Tqueue* queue = queue_init(tree_nodes(tree));
	queue->qu[queue->rear++%queue->n] = tree;
	while(queue->front%queue->n!=queue->rear%queue->n){
		if(queue->qu[(queue->front)%queue->n]->lc!=NULL) queue->qu[queue->rear++%queue->n] = queue->qu[(queue->front)%queue->n]->lc;
		if(queue->qu[(queue->front)%queue->n]->rc!=NULL) queue->qu[queue->rear++%queue->n] = queue->qu[(queue->front)%queue->n]->rc;
		printf("%c  ",queue->qu[(queue->front++)%queue->n]->data);
	}
	free(queue);
}

int tree_nodes(TNode* tree){
	if(tree==NULL) return 0;
	int a,b;
	a = tree_nodes(tree->lc);
	b = tree_nodes(tree->rc);
	return a+b+1;
}

int tree_leaves(TNode* tree){
	if(tree==NULL) return 0;
	if(tree->lc==NULL&&tree->rc==NULL) return 1;
	return tree_leaves(tree->lc)+tree_leaves(tree->rc);
}

int tree_depth(TNode* tree){
	int dl = 0;
	int dr = 0;
	if(tree==NULL) return 0;
	if(tree->lc==NULL&&tree->rc==NULL) return 1;
	dl = tree_depth(tree->lc);
	dr = tree_depth(tree->rc);
	return 1+((dl>dr)?dl:dr);
}

int tree_level(TNode* tree,int x,int *j){ 
	if(tree==NULL) return -1; // 没找到 返回-1；
	if(tree->data==x) return *j; 
	int t=++*j;  // 层数加1  
	int ret=tree_level(tree->lc,x,j); 
	if(ret>0)return ret;  //在左子树分支找到
	*j=t; //恢复层数
	return tree_level(tree->rc,x,j);
}

int tree_width(TNode* tree){
	if(tree==NULL) return 0;
  	Tqueue* queue = queue_init(tree_nodes(tree));
	queue->qu[queue->rear++%queue->n] = tree;
  	int width=1;
  	while(true){
    	int size=queue->rear-queue->front;//当前层的节点数
    	if(size>width) width = size;
    	if(size==0) break;
   		while(size>0){//如果当前层还有节点就进行下去
      	TNode* node = queue->qu[(queue->front++)%queue->n];size--;
      	if(node->lc) queue->qu[queue->rear++%queue->n] = node->lc;
      	if(node->rc) queue->qu[queue->rear++%queue->n] = node->rc;
      	}
  	}
  	free(queue);
  	return width;
}

​```
