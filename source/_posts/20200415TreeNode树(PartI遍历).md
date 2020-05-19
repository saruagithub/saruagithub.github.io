---
title: TreeNode树(PartI遍历)
date: 2020-04-15 08:50:22
tags:
- TreeNode
- 算法
- 数据结构
categories:
- 算法
---



### 树结构

TreeNode结构：

```c++
struct TreeNode{
    int val;
    TreeNode* left;
    TreeNode* right;
    TreeNode(int x):val(x),left(NULL),right(NULL) {}
};
```

二叉树：由一个根节点及两颗不相交的二叉树组成。

满二叉树：每一个结点或者是一个分支结点，并恰好有两个非空的子节点。

飞空满二叉树的叶结点数等于其分支结点数+1。

完全二叉树：严格的形状要求，从根节点起每一层从左到右填充。一棵高度为d的完全二叉树，除了d-1层以外，每一层都是满的。（完全二叉树不一定是满二叉树）



### 遍历

#### 普通递归遍历

前序遍历，中序遍历，后序遍历

```c++
void travel_preoder0(TreeNode* root){
    if (!root) return;
    cout<<root->val<<endl;
    travel_preoder0(root->left);
    travel_preoder0(root->right);
}
```

如果要返回遍历结果的话，存vector

```c++
vector<int> res;
vector<int> travel_preorder1(TreeNode* root){
    if(root != NULL){
        res.push_back(root->val);
        travel_preorder1(root->left);
        travel_preorder1(root->right);
    }
    return res;
}
```

用stack辅助前序遍历：

```c++
void travel_preoder2(TreeNode* root){
    stack<TreeNode*> Stk;
    if (root) {
        Stk.push(root);
    }
    while (!Stk.empty()) {
        root = Stk.top();
        cout<<root->val<<endl;
        Stk.pop();
        if (root->right) {
            Stk.push(root->right);
        }
        if (root->left) {
            Stk.push(root->left);
        }
    }
}
```

由于stack是先入后出，前序遍历的话，root访问之后，先将right右结点放入stack里，再将left左子节点放入stack。但这样写不好写中序遍历和后序遍历的。



#### 迭代前序遍历算法

注意节点入stack的顺序

```c++
//迭代前序遍历算法，每到一个节点 A，就应该立即访问它。在访问完根节点后，遍历左子树前，要将右子树压入栈。
//时间复杂度为 O(n)
vector<int> preorderTraversal(TreeNode* root){
    vector<int> ans;
    stack<TreeNode*> stk;
    TreeNode* rt = root;
    while (rt || stk.size()) {
        while (rt) {
            //root,left,right
            stk.push(rt->right);
            ans.push_back(rt->val);//first root
            rt = rt->left; // then left
        }
        rt = stk.top();//finally right
        stk.pop();
    }
    return ans;
}

//后序遍历完整棵树后，结果序列逆序即可。
vector<int> postorderTraversal(TreeNode* root) {
    stack<TreeNode*> S;
    vector<int> v;
    TreeNode* rt = root;
    while(rt || S.size()){
        while(rt){
            //root,right,left,then reverse
            S.push(rt->left);
            v.push_back(rt->val);
            rt=rt->right;
        }
        rt=S.top();
        S.pop();
    }
    reverse(v.begin(),v.end());
    return v;
}

//inorder 中序迭代遍历树
//每到一个节点 A，因为根的访问在中间，将 A 入栈。然后遍历左子树，接着访问 A，最后遍历右子树。
vector<int> inorderTraversal(TreeNode* root){
    stack<TreeNode*> S;
    vector<int> v;
    TreeNode* rt = root;
    while (rt || S.size()) {
        while (rt) {
            // all left
            S.push(rt);
            rt = rt->left;
        }
        //mid
        rt = S.top();
        S.pop();
        v.push_back(rt->val);
        //right
        rt = rt->right;
    }
    return v;
}

```



#### 层次遍历

逐层遍历，先根再左边右边逐层访问下来。

```c++
void PrintFromTopToBottom(TreeNode* root){
    if (!root) {
        return;
    }
    queue<TreeNode*> DequeTreeNode;
    DequeTreeNode.push(root);
    while (!DequeTreeNode.empty()) {
        TreeNode* pNode = DequeTreeNode.front();
        DequeTreeNode.pop();
        cout<<pNode->val<<endl;
        if (pNode->left) {
            DequeTreeNode.push(pNode->left);
        }
        if (pNode->right) {
            DequeTreeNode.push(pNode->right);
        }
    }
}
```

