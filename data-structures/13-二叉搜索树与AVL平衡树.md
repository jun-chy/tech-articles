# 二叉搜索树与平衡策略：从BST到AVL树的工程实现

> 作者：蔡浩宇 | 数据结构与算法笔记

## 引言

在哈希表一文中我们讨论了O(1)查找的极致效率，但哈希表有一个先天短板——它无法支持范围查询。"找出所有分数在80~90之间的学生"、"列出日志时间在最近1小时内的所有事件"，这类需求在数据库索引和文件系统中随处可见，而这正是二叉搜索树（BST）的主场。本文将从BST的基础实现出发，直面其退化问题，最终引入AVL自平衡策略，并用C语言完成可落地的工程代码。

---

## 1. 二叉搜索树：有序数据的天然容器

### 1.1 核心性质

```
二叉搜索树定义：对于任意节点 N——
  • 左子树所有节点的值 < N 的值
  • 右子树所有节点的值 > N 的值
  • 左右子树也都是二叉搜索树

         8          ← 根节点
        / \
       3   10       ← 左子树全部 < 8，右子树全部 > 8
      / \    \
     1   6    14
        /    /
       4    13

中序遍历结果：1, 3, 4, 6, 8, 10, 13, 14  ← 天然有序！
```

**BST与哈希表的核心差异**：

| 维度 | 哈希表 | 二叉搜索树 |
|------|--------|-----------|
| 单点查找 | O(1) 平均 | O(log n) 平衡，O(n) 退化 |
| 范围查询 | 不支持（需全遍历） | O(log n + k)，k为结果数 |
| 有序遍历 | 不支持 | O(n) 天然支持 |
| 前驱/后继 | 不支持 | O(log n) |
| 内存占用 | 较大（需预留槽位） | 紧凑（按需分配节点） |

### 1.2 BST节点结构与基础操作

```c
/* 节点定义 */
typedef struct BSTNode {
    int             key;
    void           *data;         /* 卫星数据 */
    struct BSTNode *left;
    struct BSTNode *right;
} BSTNode;

/* 查找——递归版本（简洁但栈开销大） */
BSTNode* bst_search(BSTNode *root, int key)
{
    if (root == NULL || root->key == key)
        return root;

    return (key < root->key)
        ? bst_search(root->left, key)
        : bst_search(root->right, key);
}

/* 查找——迭代版本（性能更优，嵌入式首选） */
BSTNode* bst_search_iter(BSTNode *root, int key)
{
    BSTNode *cur = root;
    while (cur != NULL && cur->key != key)
    {
        cur = (key < cur->key) ? cur->left : cur->right;
    }
    return cur;
}

/* 插入——返回新的根节点（处理空树情况） */
BSTNode* bst_insert(BSTNode *root, int key, void *data)
{
    if (root == NULL)
    {
        BSTNode *node = (BSTNode *)malloc(sizeof(BSTNode));
        node->key  = key;
        node->data = data;
        node->left = node->right = NULL;
        return node;
    }

    if (key < root->key)
        root->left  = bst_insert(root->left,  key, data);
    else if (key > root->key)
        root->right = bst_insert(root->right, key, data);
    /* key == root->key 时可根据需求更新data或忽略 */

    return root;
}

/* 范围查询——中序遍历收集区间内节点 */
void bst_range_query(BSTNode *root, int low, int high,
                     BSTNode **results, int *count, int max_results)
{
    if (root == NULL || *count >= max_results) return;

    /* 左子树可能包含符合条件的节点 */
    if (root->key > low)
        bst_range_query(root->left, low, high, results, count, max_results);

    /* 当前节点*/
    if (root->key >= low && root->key <= high)
    {
        results[(*count)++] = root;
    }

    /* 右子树可能包含符合条件的节点 */
    if (root->key < high)
        bst_range_query(root->right, low, high, results, count, max_results);
}
```

---

## 2. BST的阿喀琉斯之踵：退化问题

### 2.1 退化场景

```
有序插入序列 {1, 2, 3, 4, 5, 6, 7} → 退化为链表：

    1
     \
      2
       \
        3
         \
          4
           \
            5
             \
              6
               \
                7

查找时间复杂度从 O(log n) 退化为 O(n)
——此时BST与链表无异，没有任何性能优势
```

### 2.2 退化有多严重？

| 节点数 n | 平衡BST深度 ≈ log₂(n) | 退化链表深度 = n | 查找性能差距 |
|----------|----------------------|-----------------|-------------|
| 100      | 7                    | 100             | 14倍 |
| 10,000   | 14                   | 10,000          | 714倍 |
| 1,000,000| 20                   | 1,000,000       | 50,000倍 |

**结论**：退化是BST的致命缺陷——必须引入自平衡机制。

---

## 3. AVL树：高平衡，低容忍

### 3.1 平衡因子

```
AVL树定义：任意节点的 |左子树高度 - 右子树高度| ≤ 1

平衡因子 (Balance Factor) = H(left) - H(right)

     BF=1 ────►   5
                 / \
      BF=0 ◄──  3   8  ──► BF=-1
                     \
                      10  ◄── BF=0

所有节点 |BF| ≤ 1 → 这是一棵合法的AVL树
```

### 3.2 四种失衡与旋转修复

```
情况1：LL（左-左）——右旋一次

     5 (BF=2)             3
    /                    / \
   3 (BF=1)    右旋→    2   5
  /
 2

操作：将 3 提升为根，5 变为 3 的右子树

──────────────────────────────────────────

情况2：RR（右-右）——左旋一次

   3 (BF=-2)             5
    \                    / \
     5 (BF=-1)   左旋→  3   6
      \
       6

对称操作

──────────────────────────────────────────

情况3：LR（左-右）——先左旋后右旋

     5 (BF=2)          5            4
    /                 /            / \
   3 (BF=-1) 左旋→   4     右旋→  3   5
    \               /
     4             3

先让左子树失衡的那个方向旋转，再整体反向旋转

──────────────────────────────────────────

情况4：RL（右-左）——先右旋后左旋

   3 (BF=-2)        3              4
    \                 \            / \
     5 (BF=1)  右旋→   4    左旋→ 3   5
    /                   \
   4                     5

LR的镜像情况
```

### 3.3 AVL树完整C语言实现

```c
/* AVL节点——比BST多了height字段 */
typedef struct AVLNode {
    int             key;
    void           *data;
    struct AVLNode *left;
    struct AVLNode *right;
    int             height;       /* 以本节点为根的子树高度 */
} AVLNode;

/* 工具函数：获取高度（空节点高度为0） */
static inline int avl_height(AVLNode *node)
{
    return (node == NULL) ? 0 : node->height;
}

/* 工具函数：计算平衡因子 */
static inline int avl_balance(AVLNode *node)
{
    return (node == NULL) ? 0 : avl_height(node->left) - avl_height(node->right);
}

/* 工具函数：更新高度 */
static inline void avl_update_height(AVLNode *node)
{
    int hl = avl_height(node->left);
    int hr = avl_height(node->right);
    node->height = (hl > hr ? hl : hr) + 1;
}

/* 右旋 */
static AVLNode* avl_rotate_right(AVLNode *y)
{
    AVLNode *x  = y->left;
    AVLNode *T2 = x->right;

    x->right = y;
    y->left  = T2;

    avl_update_height(y);
    avl_update_height(x);

    return x;  /* 新的子树根 */
}

/* 左旋 */
static AVLNode* avl_rotate_left(AVLNode *x)
{
    AVLNode *y  = x->right;
    AVLNode *T2 = y->left;

    y->left  = x;
    x->right = T2;

    avl_update_height(x);
    avl_update_height(y);

    return y;
}

/* 平衡调整 */
static AVLNode* avl_rebalance(AVLNode *node)
{
    avl_update_height(node);
    int bf = avl_balance(node);

    /* LL：左子树过高，且左子树的左子树更高 */
    if (bf > 1 && avl_balance(node->left) >= 0)
        return avl_rotate_right(node);

    /* LR：左子树过高，但左子树的右子树更高 */
    if (bf > 1 && avl_balance(node->left) < 0)
    {
        node->left = avl_rotate_left(node->left);
        return avl_rotate_right(node);
    }

    /* RR：右子树过高，且右子树的右子树更高 */
    if (bf < -1 && avl_balance(node->right) <= 0)
        return avl_rotate_left(node);

    /* RL：右子树过高，但右子树的左子树更高 */
    if (bf < -1 && avl_balance(node->right) > 0)
    {
        node->right = avl_rotate_right(node->right);
        return avl_rotate_left(node);
    }

    return node;  /* 已平衡 */
}

/* AVL插入——返回新根 */
AVLNode* avl_insert(AVLNode *root, int key, void *data)
{
    /* 1. 标准BST插入 */
    if (root == NULL)
    {
        AVLNode *node = (AVLNode *)malloc(sizeof(AVLNode));
        node->key    = key;
        node->data   = data;
        node->left   = node->right = NULL;
        node->height = 1;
        return node;
    }

    if (key < root->key)
        root->left  = avl_insert(root->left,  key, data);
    else if (key > root->key)
        root->right = avl_insert(root->right, key, data);
    else
        return root;  /* 重复键不插入 */

    /* 2. 回溯时重平衡 */
    return avl_rebalance(root);
}

/* AVL删除——返回新根 */
AVLNode* avl_delete(AVLNode *root, int key)
{
    if (root == NULL) return NULL;

    if (key < root->key)
        root->left = avl_delete(root->left, key);
    else if (key > root->key)
        root->right = avl_delete(root->right, key);
    else
    {
        /* 找到待删节点 */
        if (root->left == NULL || root->right == NULL)
        {
            AVLNode *temp = root->left ? root->left : root->right;
            if (temp == NULL)
            {
                temp = root;
                root = NULL;
            }
            else
            {
                *root = *temp;  /* 把唯一子节点的数据复制过来 */
            }
            free(temp);
        }
        else
        {
            /* 有两个子节点：找中序后继（右子树的最小节点） */
            AVLNode *successor = root->right;
            while (successor->left)
                successor = successor->left;

            /* 用后继的key和data替换 */
            root->key  = successor->key;
            root->data = successor->data;

            /* 删除后继节点 */
            root->right = avl_delete(root->right, successor->key);
        }
    }

    if (root == NULL) return NULL;

    return avl_rebalance(root);
}
```

---

## 4. 工程实践：基于AVL树的嵌入式配置管理

### 4.1 场景描述

嵌入式设备通常需要管理数十到数百个配置参数（如校准值、阈值、通信参数等），要求：按键值快速读写、支持按前缀范围导出（如"全部网络配置"）、内存受限。

### 4.2 设计实现

```c
/* 配置项 */
typedef struct {
    const char *key;        /* "wifi.ssid", "sensor.offset", ... */
    int         value_type; /* INT, FLOAT, STRING 等 */
    union {
        int   int_val;
        float float_val;
        char  str_val[64];
    };
} ConfigItem;

/* 基于AVL树的配置管理器 */
typedef struct {
    AVLNode    *root;
    uint32_t    count;
    /* 读写锁——多任务环境下使用 */
    SemaphoreHandle_t mutex;  /* FreeRTOS */
} ConfigManager;

/* 按前缀范围查询并导出为JSON */
void config_export_by_prefix(ConfigManager *mgr,
                              const char *prefix,
                              char *json_buf, size_t buf_size)
{
    /* 1. 范围查询：找到 [prefix, prefix+最大后缀) 范围内的所有节点 */
    /* 2. 中序遍历输出JSON */
    /* 3. 利用AVL的有序性，遇到超出范围的节点立即停止遍历 */
    // ... 实现略
}
```

**为什么选AVL而不是红黑树？**

| 场景 | AVL | 红黑树 |
|------|-----|--------|
| 查询密集型（写少读多） | **优选**（更严格平衡→更浅深度） | 可用 |
| 写密集型（频繁插入删除） | 可用（旋转次数较多） | **优选**（旋转次数少） |
| 嵌入式配置管理 | **优选**（启动时加载，运行时查询为主） | — |

---

## 5. 性能实测对比

```c
/* 测试代码：对比BST与AVL在有序插入场景下的性能 */
#include <time.h>

void benchmark(int n)
{
    clock_t start, end;
    BSTNode *bst_root = NULL;
    AVLNode *avl_root = NULL;

    /* BST：有序插入 → 退化为链表 */
    start = clock();
    for (int i = 0; i < n; i++)
        bst_root = bst_insert(bst_root, i, NULL);
    end = clock();
    printf("BST ordered insert (%d): %.3f ms\n", n,
           (double)(end - start) * 1000 / CLOCKS_PER_SEC);

    /* AVL：始终保持平衡 */
    start = clock();
    for (int i = 0; i < n; i++)
        avl_root = avl_insert(avl_root, i, NULL);
    end = clock();
    printf("AVL ordered insert (%d): %.3f ms\n", n,
           (double)(end - start) * 1000 / CLOCKS_PER_SEC);

    /* 查找最后一个元素（BST需遍历整个链表） */
    start = clock();
    for (int i = 0; i < 100000; i++)
        bst_search_iter(bst_root, n - 1);
    end = clock();
    printf("BST search last (%d): %.3f ms (100k iterations)\n", n,
           (double)(end - start) * 1000 / CLOCKS_PER_SEC);

    start = clock();
    for (int i = 0; i < 100000; i++)
        bst_search_iter((BSTNode *)avl_root, n - 1);
    end = clock();
    printf("AVL search last (%d): %.3f ms (100k iterations)\n", n,
           (double)(end - start) * 1000 / CLOCKS_PER_SEC);
}

/*
 * 典型输出 (n=10000):
 * BST ordered insert (10000): 1258.4 ms    ← 退化为O(n²)插入
 * AVL ordered insert (10000): 3.2 ms       ← O(n log n)
 * BST search last (10000): 235.8 ms        ← O(n) 查找
 * AVL search last (10000): 0.6 ms          ← O(log n) 查找
 *
 * 差距达 400 倍！
 */
```

---

## 6. 平衡树选择决策树

```
需要范围查询 / 有序遍历 / 前驱后继？
    │
    ├─ NO → 用哈希表（O(1)，更简单）
    │
    └─ YES
        │
        ├─ 读多写少（查询 > 80%）？
        │   └─ YES → AVL树（查询最快，平衡最严格）
        │
        ├─ 写操作频繁？
        │   └─ YES → 红黑树（旋转更少，插入删除更快）
        │
        ├─ 数据量大 + 需要磁盘存储？
        │   └─ YES → B/B+树（减少IO，数据库标配）
        │
        └─ 内存极度受限 + 不要求最坏O(log n)？
            └─ YES → Treap / Splay树（实现简单）
```

---

## 总结

本文从BST的基础操作到AVL自平衡策略，建立了二叉搜索树的完整知识体系：

- **BST的优势**：天然有序、范围查询、前驱后继——这些都是哈希表无法提供的
- **退化陷阱**：有序插入使BST退化为链表，性能从O(log n)骤降至O(n)
- **AVL的解法**：通过平衡因子和四种旋转，保证树高始终为O(log n)
- **工程选型**：读密集场景选AVL，写密集选红黑树，磁盘存储选B+树

数据结构的选择从来不是非黑即白——理解每种结构的trade-off，才是工程师的核心素养。

---

*本文配套代码见 GitHub：[embedded-dslib](https://github.com/jun-chy/embedded-dslib)*
