## 1 用处
1. 检查是否存在环
2. 找到入环点
3. 计算环的长度
## 2 基本思想
1. 定义快慢指针fast和slow，起始位置均位于链表头部，固定fast每次前进2步，slow每次前进一步
2. 若fast遇到null节点，则表示链表无环
3. 若链表有环，fash和slow一定会相遇
4. 当fast和slow相遇时，额外创建指针ptr，并指向链表头部，且每次前进一步，最终slow和ptr会在入环点相遇
## 3 疑问
1. 为什么fast和slow一定会相遇
2. fast和slow相遇时，slow指针是否绕环超过一圈？
3. 为什么ptr和slow相遇的节点一定是入环点？
4. 为什么fast指针每次移动2步，能不能移动3、4、5...步？
## 4 解答
参考：https://zhuanlan.zhihu.com/p/361049436
## 5 代码
```c++

```
bool isHappy(int n) {
    int slow = n;
    int fast = SquareSum(SquareSum(n));

    while (fast != 1 && fast != slow) {  //fast == 1或者形成环退出
        slow = SquareSum(slow); 
        fast = SquareSum(SquareSum(fast)); //注意fast独立前进，不受slow影响
    }
 
    return fast == 1;
        
}
