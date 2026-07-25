# 1 路径总和
## 1.1 根节点到叶子节点的路径等于指定值
### 1.1.1 递归
```c++
/*对于给定的targetsum，可以采用遍历过程中节点值累加的方式，也可以采用从targetSum减去节点值的方式*/
bool hasPathSum(TreeNode* root, int targetSum) {
    if (!root) {
        return false;
    }

    if (!root->left && !root->right) {
        return root->val == targetSum;
    }

    return hasPathSum(root->left, targetSum - root->val) || hasPathSum(root->right, targetSum - root->val);
}
```