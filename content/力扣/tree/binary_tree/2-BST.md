# 1 BST概述
二叉搜索树有很多操作值得留意，比如验证二叉搜索树、二叉搜索树迭代器、二叉搜索树的插入/删除等等
# 2 验证二叉搜索树
假设一个二叉搜索树具有如下特征：
- 节点的左子树只包含小于当前节点的数。
- 节点的右子树只包含大于当前节点的数。
- 所有左子树和右子树自身必须也是二叉搜索树
  
以上的定义细节处可能有所差异，比如
- 节点的左子树只包含小于当前节点的数。
- 节点的右子树只包含大于等于当前节点的数。
- 所有左子树和右子树自身必须也是二叉搜索树

因此需要如何判断需要根据具体的定义进行
## 2.1 解题思路

下列的解题不正确，因为局部最优解并非全局最优解，单棵子树满足二叉搜索树并不能保证全局满足二叉搜索树，就比如: [5,4,6,null,null,3,7]
```c++
class Solution {
public:
    bool isValidBST(TreeNode* root) {
        if (!root) {
            return true;
        }

        if (root->left && (root->left->val >= root->val)) {
            return false;
        }

        if (root->right && (root->right->val <= root->val)) {
            return false;
        }

        if (!isValidBST(root->left)) {
            return false;
        }

        if (!isValidBST(root->right)) {
            return false;
        } 

        return true;
    }
};
```

正确的解题思路需要考虑全局情况，即：
- 当前节点的值是其左子树的值的上界（最大值）
- 当前节点的值是其右子树的值的下界（最小值）
  
动态调整左右子树的界限值，上面的解法之所以有误，是只考虑局部子树的上下值，而没有考虑整体。
### 2.1.1 递归
```c++
class Solution {
public:
    //注意lef和right的类型，不能用int，防止无法判断INT_MAX
    bool isValid(TreeNode* root, long left, long right) {
        if (!root) {
            return true;
        }
        if (root->val <= left) {
            return false;
        }
        if (root->val >= right) {
            return false;
        }

        if (!isValid(root->left, left, root->val)) {
            return false;
        }

        if (!isValid(root->right, root->val, right)) {
            return false;
        }

        return true;
    }
    bool isValidBST(TreeNode* root) {
        return isValid(root, LONG_MIN, LONG_MAX);
    }
};
```
### 2.1.2 迭代


# 3 二叉搜索树迭代器

# 4 二叉搜索树操作
# 4.1 插入

# 4.2 删除