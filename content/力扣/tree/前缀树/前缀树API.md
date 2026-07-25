# 前缀树基本node

```c++
class Node {
  public:
  Array<Node *, 26> children;	//表示当前节点的子节点
  bool is_word;						//当前节点是否是某个word的末尾字符

  Node () {
    for (int i = 0; i < 26; i++) {
        children[i] = nullptr;
    }
  }
}

class Tree {
  public:
  Node root;
  void insert(string word);
  bool search(string word);
  bool start_with(string word);
}

Tree trie_tree;
```

# 插入API

```c++
void Tree::insert(string word) {
  Tree *node = &trie_tree;
  
  for (char cn : word) {
    if (node->children[cn - 'a'] == null) {
      node->children[cn - 'a'] = new Node();
    }
    
    node = node->children[cn - 'a'];
  }
  
  node.is_word = tree;		//最后一个字符，需要标记一下
}
```

# search API

```c++
void Tree::search(string word) {
  Tree *node = &trie_tree;
  
  for (char cn : word) {
    if (node->children[cn - 'a'] == null) {
      return false;
    }
    
    node = node->children[cn - 'a'];
  }
  
  return node->is_word;
}
```

# start_with API

```c++
void Tree::start_with(string word) {
  Tree *node == &trie_tree;
  
  for (char cn : word) {
    if (node->children[cn - 'a'] == null) {
      return false;
    }
    
    node = node->children[cn - 'a'];
  }
  
  return true;		//只要word中每个字符在tree中均按顺序存在，即返回tree，与is_word无关
}
```

