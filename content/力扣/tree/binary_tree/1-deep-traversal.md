# 1 前序遍历
题目：144
## 1.1 递归
```c++
class Solution {
public:
    void preorder(TreeNode *root, vector<int> &rec) {
        if (!root) {
            return;
        }

        rec.push_back(root->val);
        preorder(root->left, rec);
        preorder(root->right, rec);
    }

    vector<int> preorderTraversal(TreeNode* root) {
        vector<int> result;
        preorder(root, result);
        return result;
    }
};
```
## 1.2 迭代
```c++
class Solution {
public:
    vector<int> preorderTraversal(TreeNode* root) {
        vector<int> result;
        stack<TreeNode*> s;
        TreeNode* next = root;

        while(next || !s.empty() ) { //处理栈
            while(next) { //左向贪婪到底
                result.push_back(next->val);
                s.push(next);
                next = next->left;
            }

            TreeNode* cur = s.top();
            next = cur->right;
            s.pop();
        }

        return result;
    }
};
```
# 2 中序遍历
题目：94
## 2.1 递归
```c++
class Solution {
public:
    void midorder(TreeNode *root, vector<int> &rec) {
        if (!root) {
            return;
        }
        midorder(root->left, rec);
        rec.push_back(root->val);
        midorder(root->right, rec);
    }

    vector<int> midorderTraversal(TreeNode* root) {
        vector<int> result;
        midorder(root, result);
        return result;
    }
};
```
## 2.2 迭代
```c++
class Solution {
public:
    vector<int> inorderTraversal(TreeNode* root) {
        TreeNode* next = root;
        stack<TreeNode*> s;
        vector<int> res;

        while(next || !s.empty()) {
            while (next) {
                s.push(next);
                next = next->left;  //左向贪婪
            }

            TreeNode* cur = s.top(); //贪婪到头则访问val，然后访问右侧节点
            res.push_back(cur->val);
            s.pop();
            next = cur->right;
        }

        return res;
    }
};
```
# 3 后序遍历
题目：145
## 3.1 递归
```c++
class Solution {
public:
    void postTraversal(TreeNode *root, vector<int>& vec) {
        if (root == nullptr) {
            return;
        }

        postTraversal(root->left, vec);
        postTraversal(root->right, vec);
        vec.push_back(root->val);
    }

    vector<int> postorderTraversal(TreeNode* root) {
        vector<int> result;
        postTraversal(root, result);
        return result;
    }
};
```
## 3.2 迭代
```c++
/*后续遍历的顺序是 左右中，即中右左的反序，因此只需要按照中右左的顺序遍历，然后在将结果reverse即可*/
class Solution {
public:
    vector<int> postorderTraversal(TreeNode* root) {
        vector<int> res;
        vector<int> res_reverse;
        stack<TreeNode *> s;
        TreeNode *next = root;

        while (next || !s.empty()) { //中右左的顺序遍历
            while(next) {
                res.push_back(next->val);
                s.push(next);
                next = next->right;
            }

            TreeNode* cur = s.top();
            next = cur->left;
            s.pop();
        }

        vector<int>::reverse_iterator it = res.rbegin();
        for (; it != res.rend(); it++) { //将结果reverse
            res_reverse.push_back(*it);
        }

        return res_reverse;
    }
};
```