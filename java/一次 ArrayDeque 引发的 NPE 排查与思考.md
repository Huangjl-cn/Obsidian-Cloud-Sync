
## 问题引入

在 LeetCode 上写判断二叉树是否对称的迭代解法时，我采用了经典的“成对比较”思路：左子树按“根-左-右”入队，右子树按“根-右-左”入队，然后每次从队列中弹出两个节点进行比较。为了让空子节点也能作为位置占位符参与比较，我需要将 `null` 也放入队列。初始实现用了 `ArrayDeque`：

```java
/**  
 * 同时遍历根节点的左右子树，左子树按“根-左-右”顺序，右子树按“根-右-左”顺序
 * 用队列迭代地去比较  
 */  
public boolean isSymmetricIterative(TreeNode root) {  
    // 需存储 null 占位，故用 LinkedList（ArrayDeque 不支持存储 null）。  
    Deque<TreeNode> queue = new LinkedList<>();  
    queue.offer(root.left);  
    queue.offer(root.right);  
    while (!queue.isEmpty()) {  
        TreeNode leftNode = queue.poll();  
        TreeNode rightNode = queue.poll();  
        if (leftNode == null && rightNode == null) continue;  
        if ((leftNode == null || rightNode == null) || (leftNode.val != rightNode.val)) return false;  
        queue.offer(leftNode.left);  
        queue.offer(rightNode.right);  
        queue.offer(leftNode.right);  
        queue.offer(rightNode.left);  
    }  
    return true;  
}
```

结果运行时直接抛出了 `NullPointerException`，但这段代码的逻辑看起来并没有问题——我既没有用 `null` 去调用方法，也没有访问 `null` 的属性。问题出在哪儿？

## 根因：ArrayDeque 的设计禁区

`ArrayDeque` 底层虽然是数组，但判断队列空满是通过头尾指针（head 与 tail）的数值比较实现的 `（head == tail）`，判断扩容是 `(tail = (tail + 1) & (elements.length - 1)) == head`，与数组元素是否为 `null` 毫无关系。真正的原因是：**`ArrayDeque` 在设计上就禁止存储 `null`**。

为什么？因为 `Deque` 接口中有两个关键方法 —— `poll()`（弹出并移除）和 `peek()`（只查看不移除），它们的规范约定是：**当队列为空时，返回 `null`**。如果 `ArrayDeque` 允许用户存入 `null` 作为业务元素，那么调用者面对 `poll()` 返回的 `null` 时将陷入两难：这个 `null` 是表示“队列空了”，还是表示“我刚好取出了一个存进去的 `null`？这是一个无法通过 API 返回值本身来消除的语义歧义。为了彻底杜绝这种模糊性，设计者直接在源头做了物理拦截 —— `ArrayDeque` 拒绝任何 `null` 元素的插入。

## 扩展讨论：关于语义歧义与 API 设计的关联

这个问题之所以值得深究，是因为它涉及 Java 集合框架中一个容易被忽略的设计差异。`ArrayDeque` 是纯粹的 `Deque` 实现，它的职责就是做一个无歧义的队列/栈工具，因此它选择了**禁止 null**来保证 `poll()` 和 `peek()` 返回的 null 只传达一种信息：“队列空了”。

而 `LinkedList` 之所以“可以”存 null，并非因为它比 `ArrayDeque` 更聪明地解决了歧义，而是因为它同时实现了 `List` 接口 —— List 规范明确规定允许存储 null 元素，因为 List 通过 `size()` 判空、通过 `get(index)` 取值，不存在“返回 null 表示不存在”这种场景。所以 `LinkedList` 是被 List 的身份“绑架”着去支持 null 的，它把区分歧义的责任完全交给了调用者 —— 官方建议是配合 `isEmpty()` 前置判断来使用。

## 为什么我的代码用 LinkedList 就安全了

回看我的算法实现，循环条件是 `while(!queue.isEmpty())`，这意味着每次执行 `poll()` 之前，队列都保证非空。因此，`poll()` 返回的 `null` 绝不可能是“队列为空”的信号，只能是实实在在存入的 `null` 占位符。这种用法正好符合 `LinkedList` 的安全使用范式，因此切换后程序正常运行。

## 延伸：ArrayDeque 与 LinkedList 的定位差异

| 维度        | ArrayDeque      | LinkedList        |
| --------- | --------------- | ----------------- |
| null 支持   | 不支持             | 支持（因 List 接口约束）   |
| 按索引插入 API | 无               | 有，但需 O(n) 遍历查找    |
| 两端操作性能    | 极高（连续内存 + 指针移动） | 一般（节点分散，缓存不友好）    |
| 核心定位      | 高性能纯栈/纯队列       | 同时作为 List 和 Deque |

补充一点：`ArrayDeque` 虽然没有公开的按索引插入方法，但在 OpenJDK 的讨论中，未来可能会为其增加 `get(index)` 和 `set(index)` 这样基于索引的随机访问方法，不过仍然不会支持在中间位置插入 —— 因为中间插入需要移动大量元素，时间复杂度为 O(n)，且这种操作违背了 `ArrayDeque` 作为“两端操作专属工具”的设计初衷。而 `LinkedList` 虽然提供了按索引插入的 API，但其“先遍历查找再指针调整”的实现，在实际运行中往往比 `ArrayDeque` 的内存复制慢得多。

## 总结

这次踩坑的本质是容器特性与业务场景之间的匹配问题：算法需要把 `null` 当作占位符来用，而 `ArrayDeque` 恰好不支持 `null`。这并非性能优劣之争，而是“能不能用”的问题。在做技术选型时，不能只看“大家都说 ArrayDeque 性能更好”，还要看清自己的业务逻辑是否依赖了该容器不支持的特性。选对了容器，代码才能跑起来；理解了背后的设计哲学，下次遇到类似问题才能快速定位。