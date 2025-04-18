---
layout: post
title: 经典算法题
date: 2019-06-11
author: Mr Chen
# cover: '/assets/img/shan.jpg'
categories: Notes
tags: 
- 算法
- c++
---

## 二叉树遍历

- 前序遍历
    根结点 ---> 左子树 ---> 右子树

- 中序遍历
    左子树---> 根结点 ---> 右子树

- 后序遍历
    左子树 ---> 右子树 ---> 根结点

- 层次遍历
    只需按层次遍历即可

<!--more-->

二叉树数据结构：

~~~cpp
struct TreeNode{
    int value;
    TreeNode* left;
    TreeNode* right;
    TreeNode(int x) : val(x), left(NULL), right(NULL) {}
}
~~~

### 前序遍历

- 递归遍历

~~~cpp
void PreTraverse(TreeNode* root)
{
    if (!root) {
        return;
    }
    cout << " " << root->value;
    PreOrder(root->left);
    PreOrder(root->right);
}
~~~

- 非递归遍历

~~~cpp
void PreTraverse(TreeNode* root)
{
    if (!root) {
        return;
    }
    stack<TreeNode*> stack_list;
    TreeNode* tmp_root = root;
    while (!tmp_root || !stack_list.empty()) {
        if (!tmp_root) {
            cout << tmp_root->value << " ";
            stack_list.push(tmp_root);
            tmp_root = tmp_root->left;
        } else {
            TreeNode node = stack_list.pop();
            tmp_root = node.right;
        }
    }
}
~~~

### 中序遍历

- 递归遍历

~~~cpp
void MiddleTraverse(TreeNode* root)
{
    if (!root) {
        return;
    }
    MiddleOrder(root->left);
    cout << " " << root->value;
    MiddleOrder(root->right);
}
~~~

- 非递归遍历

~~~cpp
void MiddleTraverse(TreeNode* root) 
{
    stack<TreeNode*> stack_list;
    TreeNode* tmp_root = root;
    while(!tmp_root || !stack_list.empty()) {
        if (!tmp_root) {
            stack.push(tmp_root);
            tmp_root = tmp_root->left;
        } else {
            TreeNode *node = stack_list.pop();
            cout << node->value << " ";
            tmp_root = tmp_root->right;
        }
    }
}
~~~

### 后序遍历

- 递归遍历

~~~cpp
void MiddleTraverse(TreeNode* root)
{
    if (!root) {
        return;
    }
    MiddleOrder(root->left);
    MiddleOrder(root->right);
    cout << " " << root->value;
}
~~~

- 非递归遍历

~~~cpp
void MiddleTraverse(TreeNode* root) 
{
    stack<TreeNode*> stack_list;
    TreeNode* tmp_root = root;
    while(!tmp_root || !stack_list.empty()) {
        if (!tmp_root) {
            stack.push(tmp_root);
            tmp_root = tmp_root->left;
        } else {
            TreeNode *node = stack_list.pop();
            tmp_root = tmp_root->right;
            cout << node->value << " ";
        }
    }
}
~~~

## 广度优先/层次遍历

~~~cpp
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode(int x) : val(x), left(NULL), right(NULL) {}
 * };
 */
class Solution {
public:
    vector<vector<int>> levelOrder(TreeNode* root) {
        vector<vector<int>> ret_vec;
        
        queue<TreeNode*> stack_node;
        stack_node.push(root);
        while(!stack_node.empty()) {
            int count = stack_node.size();
            vector<int> tmp_vec;
            while(count--) {
                TreeNode* node = stack_node.front();
                stack_node.pop();
                tmp_vec.push_back(node->val);
                if(node->left) {
                    stack_node.push(node->left);
                }
                if(node->right) {
                    stack_node.push(node->right);
                }                  
            }   
            ret_vec.push_back(tmp_vec);
        }
        return ret_vec;
    }
};
~~~

## 深度优先遍历

递归遍历


## 求二叉树最大宽度

~~~cpp
class Solution {
public:
    int widthOfBinaryTree(TreeNode* root) {
        if (!root) {
            return 0;
        }
        queue<TreeNode*> stack_node;
        TreeNode* pNode = root;
        int max_width(0);
        stack_node.push(pNode);
        while(pNode) {
            size_t count = stack_node.size();
            if (count > max_width) {
                max_width = count;
            }
            while(count--) {
                TreeNode* node = stack_node.front();
                stack_node.pop();
                if(node->left)
                    stack_node.push(node->left);
                if(node->right)
                    stack_node.push(node->right);
            }
        }
        return max_width;
    }
};
~~~

## 判断二叉树是否对称

- 递归判断

~~~cpp
bool IsMirror(TreeNode* left, TreeNode* right)
{
    if (!left && !right) {
        return true;
    }
    if (!left || !right) {
        return false;
    }
    return (left->val == right->val) 
                && IsMirror(left->left, right->right) 
                && IsMirror(left->right, right->left); 
}

bool IsMirror(TreeNode* root)
{
    return IsMirror(root, root);
}
~~~

- 非递归/迭代法判断

~~~cpp
bool IsMirror(TreeNode* root)
{
    if (!root) {
        return false;
    }
    queue<TreeNode*> queue_node;
    queue_node.push(root);
    queue_node.push(root);
    while(!queue_node.empty()) {
        TreeNode* t1 = queue_node.front();
        queue_node.pop();
        TreeNode* t2 = queue_node.front();
        queue_node.pop();
        if(!t1 && !t2) {
            continue;
        }
        if(!t1 || !t2) {
            return false;
        }
        queue_node.push(t1.left);
        queue_node.push(t2.right);
        queue_node.push(t1.right);
        queue_node.push(t2.left);
    }
    return true;
}
~~~


## 大数相乘

是指那些相乘结果或是乘数本身用long long类型都会溢出的数字，通常这些数字都通过string类型进行表示，借助于可动态调整大小的数据结构（vector,string,deque）模拟实现数字的乘法操作。对于普通的乘法，我们知道m位数和n位数相乘，最后的结果位数在区间内[m+n-1,m+n]。例如34*56，我们通常这么计算： 
将3，4分别于6相乘，记录低位的进位，然后将3，4对5进行相同的操作，知道第二个乘数的最高位乘完，算法结束。 
所以我们可以保存每个位数的相乘结果，最后统一进位转换。

~~~cpp
#include<iostream>
#include<deque>
#include<sstream>

std::string BigNumMultiply(std::string s1, std::string s2)
{
    //记录最终结果
    std::string res = "";
    //使用deque是因为出现进位时可以在队列前插入数据，效率比vector高，大小设为最小
    std::deque<int> vec(s1.size() + s2.size() - 1, 0);
    for (int i = 0; i < s1.size(); ++i) {
        for (int j = 0; j < s2.size(); ++j) {
            vec[i + j] += (s1[i] - '0')*(s2[j] - '0');//记录相乘结果
        }
    }
    // 进位处理
    int addflag = 0;
    // 倒序遍历，是因为最左边的值为最高位，最右边的值在最低位，进位运算要从低位开始
    for (int i = vec.size() - 1; i >= 0; --i) {
        int temp = vec[i] + addflag;//当前值加上进位值
        vec[i] = temp % 10;//当前值
        addflag = temp / 10;//进位值
    }
    //如果有进位，将进位加到队列头部
    while (addflag != 0) {
        int t = addflag % 10;
        vec.push_front(t);
        addflag /= 10;
    }
    // 转换为字符串
    for (auto c : vec) {
        std::ostringstream ss;
        ss << c;
        res = res + ss.str();
    }
    return res;
}
~~~


## 反转链表

反转一个单链表。

解决方案
### 迭代

> 假设存在链表 1 → 2 → 3 → Ø，我们想要把它改成 Ø ← 1 ← 2 ← 3。

在遍历列表时，将当前节点的 next 指针改为指向前一个元素。由于节点没有引用其上一个节点，因此必须事先存储其前一个元素。在更改引用之前，还需要另一个指针来存储下一个节点。不要忘记在最后返回新的头引用！

~~~cpp
ListNode* ReverseList(ListNode* root)
{
    if (!root || !root->next) {
        return root;
    }
    ListNode* prev = NULL;
    ListNode* cur = root;
    while (cur) {
        ListNode* tmp = cur->next;
        cur->next = prev;
        prev = cur;
        cur = tmp;
    }
    return prev;
}
~~~

复杂度分析

- 时间复杂度：O(n)O(n) 。 假设 nn 是列表的长度，时间复杂度是 O(n)O(n)。

- 空间复杂度：O(1)O(1) 。

### 递归

递归版本稍微复杂一些，其关键在于反向工作。假设列表的其余部分已经被反转，现在我该如何反转它前面的部分？假设列表为：n1 → … → nk-1 → nk → nk+1 → … → nm → Ø

若从节点 nk+1 到 nm 已经被反转，而我们正处于 nk。

> n1 → … → nk-1 → nk → nk+1 ← … ← nm

我们希望 nk+1 的下一个节点指向 nk。

所以，

> nk.next.next = nk;

要小心的是 n1 的下一个必须指向 Ø 。如果你忽略了这一点，你的链表中可能会产生循环。如果使用大小为 2 的链表测试代码，则可能会捕获此错误。

~~~cpp
ListNode* ReverseList(ListNode* root)
{
    if (!root || !root->next)
        return root;
    ListNode* tmp_node = reverseList(root->next);
    root->next->next = root;
    root->next = NULL;
    return tmp_node;
}
~~~

复杂度分析

- 时间复杂度：O(n)O(n) 。 假设 nn 是列表的长度，那么时间复杂度为 O(n)O(n)。

- 空间复杂度：O(n)O(n) 。 由于使用递归，将会使用隐式栈空间。递归深度可能会达到 nn 层。


## 二叉树转换为双向链表

题目：输入一棵二叉搜索树（二叉查找树），将该二叉搜索树转换成一个排序的双向链表。要求不能创建任何新的结点，只能调整树中结点指针的指向。

思路： 这道题目关键在于不能创建新的节点，如不然，我们可以直接将二叉排序树中序遍历保存到一个数组中，而后再建立一个双性链表，将数据保存到双向链表里。
    这里不能创建新节点，我们只能改变节点的指向左右子树的节点，让其变为指向二叉链表中的前后节点，很明显这里同样用的是中序遍历，因此这道题目依然是中序遍历的变种，中序递归构造实现即可，每次递归都保存一个指向已构造好的双向链表的尾节点的指针，将其与下一个节点连接起来。

代码实现（递归）：

~~~cpp
#include<stdio.h>
#include<stdlib.h>
 
typedef struct BSTNode
{
	int data;
	struct BSTNode *left;
	struct BSTNode *right;
}BSTNode,*BSTree;
 
/*
根据题目要求的格式创建二叉排序树
*/
void CreateBST(BSTree *pRoot)
{
	int data;
	scanf("%d",&data);
	if(data == 0)
		pRoot = NULL;
	else
	{
		*pRoot = (BSTree)malloc(sizeof(BSTNode));
		if(*pRoot == NULL)
			exit(EXIT_FAILURE);
		(*pRoot)->data = data;
		(*pRoot)->left = NULL;
		(*pRoot)->right = NULL;
		CreateBST(&((*pRoot)->left));
		CreateBST(&((*pRoot)->right));
	}
}
 
/*
采用中序遍历的方式将二叉树转化为双向链表，
*pLas指向双向链表的最后一个节点
*/
void ConvertNode(BSTree pRoot,BSTree *pLast)
{
	if(pRoot == NULL)
		return;
	
	//先转化左子树
	if(pRoot->left != NULL)
		ConvertNode(pRoot->left,pLast);
 
	//将双向链表的最后一个节点与根节点连接在一起
	pRoot->left = *pLast;
	if(*pLast != NULL)
		(*pLast)->right = pRoot;
	*pLast = pRoot;
 
	//转换右子树
	if(pRoot->right != NULL)
		ConvertNode(pRoot->right,pLast);
}
 
/*
返回双向链表的头结点
*/
BSTree Convert(BSTree pRoot)
{
	if(pRoot == NULL)
		return NULL;
	if(pRoot->left==NULL && pRoot->right==NULL)
		return pRoot;
 
	BSTree pLast = NULL;
	ConvertNode(pRoot,&pLast);
	
	//依次从 last 返回 first 头结点
	BSTree pHead = pLast;
	while(pHead->left != NULL)
		pHead = pHead->left;
 
	return pHead;
}
 
int main()
{
	int n;
	while(scanf("%d",&n) != EOF)
	{
		int i;
		for(i=0;i<n;i++)
		{
			BSTree pRoot = NULL;
			CreateBST(&pRoot);
			BSTree pHead = Convert(pRoot);
			while(pHead != NULL)
			{
				printf("%d ",pHead->data);
				pHead = pHead->right;
			}
 
			printf("\n");
			free(pRoot);
			pRoot = NULL;
		}
	}
	return 0;
}
~~~

## 反序链表求和

Input: (2 -> 4 -> 3) + (5 -> 6 -> 4)
Output: 7 -> 0 -> 8

思路：
首先要理解题目，输入的两个链表都是反序存储的。也就是链表首位代表数字的个位，第二位代表数字百位。接下来就和一般求和的题目一样了，维护当前求和结果和进位即可。注意最后要判断，是否还有进位，如果有，还需要再生成一位。

代码实现：

~~~cpp
struct ListNode {
    int value;
    ListNode* next;
    ListNode(int vul) {
        value = vul;
    }
}
// 注意返回值需要手动释放
ListNode* AddTwoListNumbers(ListNode* list1, ListNode* list2)
{
    ListNode* l1 = list1;
    ListNode* l2 = list2;
    ListNode* dest = new ListNode(0);
    ListNode* tmp_dest = dest;
    int carry = 0; // 记录进位
    while(l1 || l2) {
        int sum = 0;
        if (l1) {
            sum += l1->value;
            l1 = l1->next;
        }
        if (l2) {
            sum += l2->value;
            l2 = l2->next;
        }
        tmp_dest->value = carry + sum % 10;
        tmp_dest->next = new ListNode(0);
        tmp_dest = tmp_dest->next;
        carry = sum / 10;
    }
    if (carry > 0) {
        tmp_dest->next->value = carry;
    }
    return dest;
}
~~~

时间复杂度：O(n)


## 查找字符串中字典序最大的序列

题目描述：给一个字符串，得到它字典序最大的子序列，即：删除一些字符，使得剩下的字符构成的字符串字典序是最大的

代码实现：

~~~cpp
string GetMaxString(const string& src_str)
{
    if (src_str.empty()) {
        return "";
    }
    stack<char> dest;
    string dest_str;
    for (size_t i = 0; i < src_str.size(); i++) {
        if (dest.empty()) {
            dest.push(src_str.at(i));
            continue;
        }
        while (!dest.empty() && src_str[i] > dest.top()) {
            dest.pop();
        }
        dest.push(src_str.at(i));
    }
    while (!dest.empty()) {
        dest_str.insert(dest_str.begin(), dest.top());
        dest.pop();
    }
    return dest_str;
}
~~~

