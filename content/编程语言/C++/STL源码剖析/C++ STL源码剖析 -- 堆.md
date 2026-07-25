# 1 二叉堆

二叉堆，就是一种完全二叉树，即整个二叉树除了最底层的叶子节点之外，是填满的，而最底层的叶子节点中间也没有空隙,如下图所示：
![二叉树][二叉树实例]

使用数组层级存储    
| * | A | B | C | D | E | F | G | H | I | J |   

如果0位置保留，从1位置开始保存根节点，那么i位置的某个节点，其左孩子节点必定在**2i**位置，其右孩子节点则在**2i+1**位置，其父节点则位于**i/2**。

如果0位置使用，从0位置开始保存根节点，那么i位置的某个节点，其左孩子节点必定在**2i+1**位置，其右孩子节点则在**2i+2**位置，其父节点在位于**(i-1)/2**位置

由于Array无法动态改变大小，使用vector可以更好的实现堆算法。堆分为max_heap和min_heap，前者根节点大于等于所有子节点，后者根节点小于等于所有子节点，本文将以max_heap为例讲解堆。

# 2 堆算法

## 2.1 push_heap
新添加的元素要放在二叉树的最下一层作为叶子节点，填充在vector的尾部，然后再对齐位置进行调整。

为满足max_heap的特性，即max_heap对应的二叉树的根节点一定要是最大的节点，将新插入的节点和其父节点的值进行比较，直到其父节点大于等于新插入节点的值，或者新插入节点变为根节点。

![push_up][push_up]

*stl_heap.h*
```c++
//使用push_heap之前，需要将元素已经插入到vector尾部
//_RandomAccessIterator表明是随机访问迭代器，起始位置为0
//_Compare是数值比较函数
template <class _RandomAccessIterator, class _Compare>
inline void 
push_heap(_RandomAccessIterator __first, _RandomAccessIterator __last,
          _Compare __comp)
{
  __push_heap_aux(__first, __last, __comp,
                  __DISTANCE_TYPE(__first), __VALUE_TYPE(__first));
}


template <class _RandomAccessIterator, class _Compare,
          class _Distance, class _Tp>
inline void 
__push_heap_aux(_RandomAccessIterator __first,
                _RandomAccessIterator __last, _Compare __comp,
                _Distance*, _Tp*) 
{
  __push_heap(__first, _Distance((__last - __first) - 1), _Distance(0),  
              _Tp(*(__last - 1)), __comp);
}

template <class _RandomAccessIterator, class _Distance, class _Tp, 
          class _Compare>
void
__push_heap(_RandomAccessIterator __first, _Distance __holeIndex,
            _Distance __topIndex, _Tp __value, _Compare __comp)
{
  _Distance __parent = (__holeIndex - 1) / 2; //计算父节点相对位置
  //__comps比较*(__first + __parent)与__value的关系，对于max_heap而言，__comp为判断*(__first + __parent)是否小于__value
  //__first + __parent表示parent的绝对位置, value是新插入的元素值
  //__holeIndex初始为末尾位置，何为holeIndex? 其实就是从当前节点向父节点方向向上依次比较时的当前值，将当前位置称为holeindex
  while (__holeIndex > __topIndex && __comp(*(__first + __parent), __value)) {
    *(__first + __holeIndex) = *(__first + __parent); //交换parent与holeindex的值
    __holeIndex = __parent; 
    __parent = (__holeIndex - 1) / 2;
  }
  *(__first + __holeIndex) = __value; //插入元素的最终位置
}
```
以上是C++标准库的抽象实现，如果使用具象的vector实现，代码如下：
```c++
std::vector<int> heap;
void push_heap(int value) {
  heap.push_back(value); //插入末尾
  int parent = (heap.size() - 1) / 2;
  int hole_index = heap.size() - 1;

  while (hole_index > 0 && heap[parent] < value) {
    heap[hole_index] = heap[parent];
    //向上继续比较
    hole_index = parent;
    parent = (hole_index - 1) / 2;
  }

  //循环结束，更新value的最终位置
  heap[hole_index] = value;
}
```

## 2.2 pop_heap
max_heap的顶点是heap的最大值，pop heap将顶点弹出之后，在顶点形成一个“空洞”，为满足max_heap的特征，可以将max_heap最后一层最右边的叶子节点填充到顶点位置，然后将该新顶点元素与其左右孩子节点进行比较，将其与较大的孩子节点位置进行交换，直到该元素大于其所有孩子节点或者到达叶子位置。

![pop_down][pop_down]

*stl_heap.h*
```c++
template <class _RandomAccessIterator>
inline void pop_heap(_RandomAccessIterator __first, 
                     _RandomAccessIterator __last)
{
  __STL_REQUIRES(_RandomAccessIterator, _Mutable_RandomAccessIterator);
  __STL_REQUIRES(typename iterator_traits<_RandomAccessIterator>::value_type,
                 _LessThanComparable);
  __pop_heap_aux(__first, __last, __VALUE_TYPE(__first));
}

template <class _RandomAccessIterator, class _Tp>
inline void 
__pop_heap_aux(_RandomAccessIterator __first, _RandomAccessIterator __last,
               _Tp*)
{
  __pop_heap(__first, __last - 1, __last - 1, 
             _Tp(*(__last - 1)), __DISTANCE_TYPE(__first));
}

template <class _RandomAccessIterator, class _Tp, class _Compare, 
          class _Distance>
inline void 
__pop_heap(_RandomAccessIterator __first, _RandomAccessIterator __last,
           _RandomAccessIterator __result, _Tp __value, _Distance*)
{
  *__result = *__first; //最后一个位置用于存储顶部值，即执行完pop_heap之后，最大值暂存在尾部，如果底层数据结构使用vector，那么back可以访问该元素，pop_back移除该元素，需要对[first, last-1)进行adjust，原last-1位置存储result，即存储最大值
  __adjust_heap(__first, _Distance(0), _Distance(__last - __first), __value);
}

//核心函数
template <class _RandomAccessIterator, class _Distance, class _Tp>
void 
__adjust_heap(_RandomAccessIterator __first, _Distance __holeIndex,
              _Distance __len, _Tp __value)
{
  _Distance __topIndex = __holeIndex;  //初始值设置为顶点
  _Distance __secondChild = 2 * __holeIndex + 2;  //即右子节点
  while (__secondChild < __len) {  //存在右子节点
    if (*(__first + __secondChild) < *(__first + (__secondChild - 1))) //比较左右子节点， 找到较大者
      __secondChild--;
    *(__first + __holeIndex) = *(__first + __secondChild); //交换hole位置节点与较大的子节点
    __holeIndex = __secondChild;  //hole index移到较大子节点位置
    __secondChild = 2 * (__secondChild + 1);  //再次找到新的右子节点，即__secondChild = 2 * __holeIndex + 2
  }
  if (__secondChild == __len) {
    *(__first + __holeIndex) = *(__first + (__secondChild - 1));
    __holeIndex = __secondChild - 1;
  }
  __push_heap(__first, __holeIndex, __topIndex, __value);
}

```

## 2.3 sort_heap

每次pop_heap会将max_heap中最大的元素移动到底层数据结构的尾部（比如vector，经过一次pop_heap，会将原vector[0]位置元素放到当前有效heap的尾部），如果持续调用pop_heap,那么最终max_heap会变为一个单调递增的有序序列。

![sort_heap][sort_heap]

*stl_heap.h*
```c++
template <class _RandomAccessIterator>
void sort_heap(_RandomAccessIterator __first, _RandomAccessIterator __last)
{
  __STL_REQUIRES(_RandomAccessIterator, _Mutable_RandomAccessIterator);
  __STL_REQUIRES(typename iterator_traits<_RandomAccessIterator>::value_type,
                 _LessThanComparable);
  while (__last - __first > 1)
    pop_heap(__first, __last--); //__last--逐渐缩小范围
}
```

## 2.4 make_heap

*stl_heap.h*
```c++
//将[first, last)的区间调整为max_heap
template <class _RandomAccessIterator, class _Tp, class _Distance>
void __make_heap(_RandomAccessIterator __first,
            _RandomAccessIterator __last, _Tp*, _Distance*)
{
  if (__last - __first < 2) return;
  _Distance __len = __last - __first;
  _Distance __parent = (__len - 2)/2;
    
  while (true) {
    __adjust_heap(__first, __parent, __len, _Tp(*(__first + __parent))); //__adjust_heap为调整堆的过程
    if (__parent == 0) return;
    __parent--;
  }
}
```
