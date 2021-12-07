# 二叉树的常规遍历

> ## 先序遍历     
### 递归实现    
```java
class Solution {
    private List<Integer> ans;
    public List<Integer> preorderTraversal(TreeNode root) {
        ans = new ArrayList<>();
        preOrderRecur(root);
        return ans;
    }
    private void preOrderRecur(TreeNode root){
        if(root == null) return;
        ans.add(root.val);
        preOrderRecur(root.left);
        preOrderRecur(root.right);
    }
}
```    

### 非递归实现（栈）    
&emsp;&emsp;先将root节点压如栈中，之后反复执行：从栈中弹出一个元素，计入遍历结果，将其右节点、左节点分别压入栈中。直到栈为空。    
```java
class Solution {
    public List<Integer> preorderTraversal(TreeNode root) {
        if(root == null) return new ArrayList<>();
        List<Integer> ans = new ArrayList<>();
        Stack<TreeNode> stack = new Stack<>();
        stack.push(root);
        while(!stack.empty()){
            root = stack.pop();
            ans.add(root.val);
            if(root.right != null) stack.push(root.right);
            if(root.left != null) stack.push(root.left);
        }
        return ans;
    }
}
```  

> ## 中序遍历   
### 递归实现   
```java
class Solution {
    private List<Integer> ans;
    public List<Integer> inorderTraversal(TreeNode root) {
        ans = new ArrayList<>();
        inOrderRecur(root);
        return ans;
    }
    private void inOrderRecur(TreeNode root){
        if(root == null) return;
        inOrderRecur(root.left);
        ans.add(root.val);
        inOrderRecur(root.right);
    }
}
```   

### 非递归实现（栈）   
1. 申请一个新的栈，记为stack。初始时，令变量cur=root。
2. 先把cur节点压入栈中，对以cur节点为头的整棵子树来说，依次把左边界压入栈中。即不停地令cur=cur.left，然后重复步骤2。
3. 不断重复步骤2，直到发现cur为空，此时从stack中弹出一个节点，记为node。将node加入遍历结果，并让cur=node.right，然后重复步骤2。
4. 当stack为空且cur为空时，遍历结束。
```java
class Solution {
    public List<Integer> inorderTraversal(TreeNode root) {
        if(root == null) return new ArrayList<>();
        List<Integer> ans = new ArrayList<Integer>();
        Stack<TreeNode> stack = new Stack<>();
        while(!stack.empty() || root != null){
            if(root != null){
                stack.push(root);
                root = root.left;
            }else{
                root = stack.pop();
                ans.add(root.val);
                root = root.right;
            }
        }
        return ans;
    }
}
```    

> ## 后序遍历   
### 递归实现   
```java
class Solution {
    private List<Integer> ans;
    public List<Integer> postorderTraversal(TreeNode root) {
        ans = new ArrayList<>();
        posOrderRecur(root);
        return ans;
    }
    private void posOrderRecur(TreeNode root){
        if(root == null) return;
        posOrderRecur(root.left);
        posOrderRecur(root.right);
        ans.add(root.val);
    }
}
```  

### 非递归实现（栈）   
1. 申请一个栈，记为s1，然后将根节点root压入s1中。
2. 将s1中弹出的节点记为cur，将cur的左右孩子节点依次压入s1中。
3. 在整个过程中，每个从s1弹出的节点，都将其压入s2中。
4. 不断重复步骤2和步骤3，直到s1为空。
5. 将s2中的所有节点弹出，其顺序即为后序遍历。       

&emsp;&emsp;**整体思想其实就是：进入s1中的顺序为“根->左->右”，出s1的顺序（即进入s2的顺序）为“根->右->左”，再用s2将其反转，便成了“左->右->根”。**
```java
class Solution {
    public List<Integer> postorderTraversal(TreeNode root) {
        if(root == null) return new ArrayList<>();
        Stack<TreeNode> s1 = new Stack<>();
        Stack<TreeNode> s2 = new Stack<>();
        s1.push(root);
        while(!s1.empty()){
            root = s1.pop();
            s2.push(root);
            if(root.left != null)
                s1.push(root.left);
            if(root.right != null)
                s1.push(root.right);
        }
        List<Integer> ans = new ArrayList<>();
        while(!s2.empty()){
            ans.add(s2.pop().val);
        }
        return ans;
    }
}
```   

> ## 层序遍历   
1. 申请一个双端队列list，将根节点加入list中。变量last为当前遍历的层的最后一个节点，nLast为下一层的最后一个节点。队列row为当前层中的元素，当前层遍历结束后将其加入遍历结果ans中。      
2. 取list的第一个节点记为cur，将其值加入row中，再将其左右孩子节点依次加入list尾端，并维护nLast为下一层的最右节点。    
3. 当cur等于last时，表明该换行了，令last=nLast，将row加入ans中。    
4. 重复步骤2、3直到list为空。
```java
class Solution {
    public List<List<Integer>> levelOrder(TreeNode root) {
        if(root == null) return new ArrayList<>();
        List<List<Integer>> ans = new ArrayList<>();
        LinkedList<TreeNode> list = new LinkedList<>();
        List<Integer> row = new ArrayList<>();
        list.addLast(root);
        TreeNode cur = null, last = root, nLast = null;
        while(!list.isEmpty()){
            cur = list.pollFirst();
            row.add(cur.val);
            if(cur.left != null){
                list.addLast(cur.left);
                nLast = cur.left;
            }
            if(cur.right != null){
                list.addLast(cur.right);
                nLast = cur.right;
            }
            if(cur == last){
                ans.add(row);
                row = new ArrayList<>();
                last = nLast;
            }
        }
        return ans;
    }
}
```     

> ## ZigZag遍历（锯齿形层序遍历）    
&emsp;&emsp;类似于层序遍历，需添加一个变量记录当前层是从左至右遍历还是从右至左遍历，并且nLast维护方式改变，具体见代码。   
```java
class Solution {
    public List<List<Integer>> zigzagLevelOrder(TreeNode root) {
        if(root == null) return new ArrayList<>();
        List<List<Integer>> ans = new ArrayList<>();
        LinkedList<TreeNode> list = new LinkedList<>();
        List<Integer> row = new ArrayList<>();
        TreeNode last = root, nLast = null, cur = null;
        boolean ltor = true;//第一层为从左至右遍历
        list.addLast(root);
        while(list.size() > 0){
            if(ltor){
                cur = list.pollFirst();
                row.add(cur.val);
                if(cur.left != null){
                    list.addLast(cur.left);
                    if(nLast == null) nLast = cur.left;
                }
                if(cur.right != null){
                    list.addLast(cur.right);
                    if(nLast == null) nLast = cur.right;
                }
            }else{
                cur = list.pollLast();
                row.add(cur.val);
                if(cur.right != null){
                    list.addFirst(cur.right);
                    if(nLast == null) nLast = cur.right;
                }
                if(cur.left != null){
                    list.addFirst(cur.left);
                    if(nLast == null) nLast = cur.left;
                }
            }
            if(cur == last){
                last = nLast;
                nLast = null;
                ltor = !ltor;//下一层与当前层遍历顺序相反
                ans.add(row);
                row = new ArrayList<>();
            }
        }
        return ans;
    }
}
```
