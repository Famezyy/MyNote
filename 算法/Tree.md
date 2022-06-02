## 二叉树



### 前序遍历（中左右）

#### 递归

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

#### 非递归

```java
class Solution {
    public List<Integer> preorderTraversal(TreeNode root) {
        Deque<TreeNode> stack = new LinkedList<>();
        List<Integer> result = new ArrayList<>();
        TreeNode cur = root;
        while (cur != null || !stack.isEmpty()) {
            if (cur != null) {
                result.add(cur.val);
                stack.push(cur);
                cur = cur.left;
            } else {
                cur = stack.pop().right;
            }
        }
        return result;
    }
}
```

### 中序遍历（左中右）

#### 递归

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

#### 非递归

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

### 后序遍历（左右中）

#### 递归

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

#### 非递归

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

