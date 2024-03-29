# Tree

## 1.二叉树

### 1.1 遍历

#### 1.前序遍历（中左右）

##### 1.1 递归

```java
class Solution {

    List<Integer> result = new ArrayList<>();

    public List<Integer> preorderTraversal(TreeNode root) {
        preorder(root);
        return result;
    }

    public void preorder(TreeNode cur) {
        if (cur == null) return;
        result.add(cur.val);
        preorder(cur.left);
        preorder(cur.right);
    }
}
```

##### 1.2 非递归

```java
class Solution {
    public List<Integer> preorderTraversal(TreeNode root) {
        // 用于存放父节点
        Deque<TreeNode> stack = new LinkedList<>();
        List<Integer> result = new ArrayList<>();
        TreeNode cur = root;
        while (cur != null || !stack.isEmpty()) {
            if (cur != null) {
                result.add(cur.val);
                stack.push(cur);
                // 先遍历左节点
                cur = cur.left;
            } else {
                // 左节点为空则遍历右节点
                cur = stack.pop().right;
            }
        }
        return result;
    }
}
```

#### 2.中序遍历（左中右）

##### 2.1 递归

```java
class Solution {

    List<Integer> result = new ArrayList<>();

    public List<Integer> inorderTraversal(TreeNode root) {
        inorder(root);
        return result;
    }

    public void inorder(TreeNode cur) {
        if (cur == null) return;
        inorder(cur.left);
        result.add(cur.val);
        inorder(cur.right);
    }
}
```

##### 2.2 非递归

```java
class Solution {
    public List<Integer> inorderTraversal(TreeNode root) {
        Deque<TreeNode> stack = new LinkedList<>();
        List<Integer> result = new ArrayList<>();
        TreeNode cur = root;
        while (cur != null || !stack.isEmpty()) {
            if (cur != null) {
                stack.push(cur);
                cur = cur.left;
            } else {
                cur = stack.pop();
                result.add(cur.val);
                cur = cur.right;
            }
        }
        return result;
    }
}
```

#### 3.后序遍历（左右中）

##### 3.1 递归

```java
class Solution {

    List<Integer> result = new ArrayList<>();

    public List<Integer> postorderTraversal(TreeNode root) {
        postorder(root);
        return result;
    }

    public void postorder(TreeNode cur) {
        if (cur == null) return;
        postorder(cur.left);
        postorder(cur.right);
        result.add(cur.val);
    }
}
```

##### 3.2 非递归

```java
class Solution {

    public List<Integer> postorderTraversal(TreeNode root) {
        Deque<TreeNode> stack = new LinkedList<>();
        List<Integer> result = new ArrayList<>();
        if (root == null) return result;
        stack.push(root);
        TreeNode cur, pre = null;

        while (!stack.isEmpty()) {
            cur = stack.peek();
            if ((cur.left == null && cur.right == null) 
            || pre != null && (pre == cur.left || pre == cur.right)) {
                result.add(cur.val);
                stack.pop();
                pre = cur;
            } else {
                if (cur.right != null) {
                    stack.push(cur.right);
                }
                if (cur.left != null) {
                    stack.push(cur.left);
                }
            }
        }
        return result;
    }
}
```

### 1.2 大顶堆小顶堆

[源码解析](https://zhuanlan.zhihu.com/p/630992857)

大顶堆（Max Heap）和小顶堆（Min Heap）是两种特殊的二叉堆，它们都是一种完全二叉树（或近似完全二叉树），并且满足堆的性质。

1. **大顶堆（Max Heap）：**

   ```bash
          9
         / \
        7   5
       / \ / \
      6  2 4  3
   ```

   - 在大顶堆中，每个节点的值都大于或等于其子节点的值
   - 对于任意节点 `i`，其值大于等于节点 `2i` 和 `2i+1` 的值
   - 大顶堆的根节点是整个堆中的最大值

2. **小顶堆（Min Heap）：**

   ```bash
          2
         / \
        4   3
       / \ / \
      6  7 5  9
   ```

   - 在小顶堆中，每个节点的值都小于或等于其子节点的值
   - 对于任意节点 `i`，其值小于等于节点 `2i` 和 `2i+1` 的值
   - 小顶堆的根节点是整个堆中的最小值

在Java中，`PriorityQueue` 类是一个实现了小顶堆（Min Heap）的优先队列。默认情况下，`PriorityQueue` 使用自然顺序（元素的自然顺序，或者通过实现 `Comparable` 接口）来维护小顶堆的性质，即队头元素是最小的。

以下是一个简单的示例，演示如何使用 `PriorityQueue` 来创建一个小顶堆：

```java
// 创建一个小顶堆
PriorityQueue<Integer> minHeap = new PriorityQueue<>();
// 输出小顶堆的元素（按照升序）
while (!minHeap.isEmpty()) {
    System.out.print(minHeap.poll() + " ");
}

// 创建一个大顶堆，需要通过自定义比较器（Comparator）来实现
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Collections.reverseOrder());
// 输出大顶堆的元素（按照降序）
while (!maxHeap.isEmpty()) {
    System.out.print(maxHeap.poll() + " ");
}
```

## 2.线段树

多用于以线段为单位的统计计算，例如给定一个数组，每次更新区间时判断是否有区间重合。

[LeetCode 731](https://leetcode.cn/problems/my-calendar-ii/submissions/)—— 检查是否有重复插入 3 次的区间

```java
class MyCalendarTwo {

    Sector root;

    public MyCalendarTwo() {
        root = new Sector(0, 1000000000);
    }

    public boolean book(int start, int end) {
        // 检查区间
        if (query(root, start, end - 1) >= 2) {
            return false;
        }
        // 更新区间
        updateTree(root, start, end - 1);
        return true;
    }

    public int query(Sector sector, int start, int end) {

        // 如果 sector 不在 [start,end] 范围内直接返回
        if (sector.start > end || sector.end < start) return 0;

        // 如果 sector 完全被 [start,end] 覆盖则返回结果
        if (sector.start >= start && sector.end <= end) {
            return sector.val;
        }

        // 创建左右节点
        createTree(sector);
        // 查询左右子树中的最大值
        return Math.max(query(sector.left, start, end), query(sector.right, start, end));
    }

    public void updateTree(Sector sector, int start, int end) {
        // 如果 sector 不在 [start,end] 范围内直接返回
        if (sector.start > end || sector.end < start) return;

        // 如果 sector 完全被 [start,end] 覆盖则 lazy++, val++
        if (sector.start >= start && sector.end <= end) {
            sector.lazy++;
            sector.val++;
            return;
        }
        // 创建左右节点
        createTree(sector);
        
        // 递归更新左右子树
        updateTree(sector.left, start, end);
        updateTree(sector.right, start, end);
        // 更新当前值
        sector.val = Math.max(sector.left.val, sector.right.val);
    }

    public void pushDown(Sector sector) {
        sector.left.lazy += sector.lazy;
        sector.right.lazy += sector.lazy;
        sector.left.val += sector.lazy;
        sector.right.val += sector.lazy;
        sector.lazy = 0;
    }

    public void createTree(Sector sector) {
        int mid = sector.start + ((sector.end - sector.start) >> 1);
        if (sector.left == null) {
            sector.left = new Sector(sector.start, mid);
        }
        if (sector.right == null) {
            sector.right = new Sector(mid + 1, sector.end);
        }
        // 下传 lazy
        pushDown(sector);
    }
}

class Sector {
    Sector left, right;
    int start, end;
    int val;
    int lazy;
    public Sector(int start, int end) {
        this.start = start;
        this.end = end;
    }
}
```

<img src="img/Tree/image-20220720004831315-aebc269660d82d44eaa6d5adf5354be8-01e0bb.png" alt="image-20220720004831315" style="zoom:80%;" />

### 2.1 查询树

<img src="img/Tree/image-20220720004903615-140d97bd941125d627ded6723880ff26-46b5f0.png" alt="image-20220720004903615" style="zoom:80%;" />

### 2.2 更新树

<img src="img/Tree/image-20220720004604270-7b8c002159ee19c9a85f3ffd8948a25e-f45289.png" alt="image-20220720004604270" style="zoom:80%;" />

> 创建左右节点时将懒更新的值下推到子节点
