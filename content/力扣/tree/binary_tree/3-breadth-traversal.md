# 1 层次遍历
```c++
class Solution {
public:
    vector<vector<int>> levelOrder(TreeNode* root) {
        vector<vector<int>> rec;
        vector<int> rec1;
        queue<TreeNode*> q1;
        queue<TreeNode*> q2;
        TreeNode* cur = nullptr;

        if (root == nullptr) {
            return rec;
        }

        q1.push(root);
        while (!q1.empty()) {
            rec1.clear();
            while(!q1.empty()) {  //访问当前层节点，并将该层每个节点的左右节点放入到缓存队列中，逐层访问
                cur = q1.front();
                q1.pop();

                rec1.push_back(cur->val);
                if (cur->left) {
                    q2.push(cur->left);
                }

                if (cur->right) {
                    q2.push(cur->right);
                }   
            }
            rec.push_back(rec1);
            q1.swap(q2);   //准备访问下层节点
        }
        return rec;
    }
};
```