# 面向大厂编程总结——基础版
## **章节一：必会基础习题**
### **一、链表、栈、队列**
#### **1、反转链表（ LeetCode 206）**

**题目：** 给你单链表的头节点 `head` ，请你反转链表，并返回反转后的链表。

**示例：** 

```
输入：head = [1,2,3,4,5]
输出：[5,4,3,2,1]
```

**题解：** 

java版本
```java
/**
 * 反转链表
 * 
 * 此题经常考，而且要求递归和迭代都需要会
 * 
 * 递归只考虑两两结点的反转，反转操作，主要是两步：先反转，然后断链
 * 
 * 迭代要考虑前驱、当前、后继结点，反转操作
 * 
 */
public class LeetCode206 {

    /**
     * 使用递归方法
     * @param head
     * @return
     */
    public ListNode reverseList1(ListNode head) {
        /**
         * 递归出口，即什么时候不满足条件了，就跳出当前递归栈
         * 
         * 因为是反转，因此肯定是要前后两个反转，也就是说，至少要有两个结点才能反转，如果不满足，只需要返回当前结点就可以
         */ 
        if (head == null || head.next == null) {
            return head;
        }
        /**
         * 当满足条件时，需要不停的将下一个结点放入到递归函数中，并创建一个新的指针指向最后一个元素
         * 为什么仅仅使用ListNode newHead = reverseList1(head.next)这句话，就能够将指针指向最后一个结点呢？
         * 因为程序运行实际上使用栈，那么递归实际上最先返回的就是最后一个入栈的，栈是先进后出型。
         */
        ListNode newHead = reverseList1(head.next);
        /**
         * 递归方法的指针反转操作主要是两步骤：
         * 1. 反转
         * 2. 断链
         */
        head.next.next = head;
        head.next = null;
        return newHead;
    }

    /**
     * 使用迭代方法
     * @param head
     * @return
     */
    public ListNode reverseList2(ListNode head) {
        /**
         * 使用迭代方法，首先直接创建一个前驱，和一个游标指针(也就是指向当前结点的指针)
         * 前驱指针指向null，因为反转链表，原来是最后一个结点指向null，反转后，需要第一个结点指向null
         * 游标指针指向当前的head，因为要从链表第一个开始
         */
        ListNode pre = null;
        ListNode curr = head;

        /**
         * 然后，让游标指针开始遍历链表，也就是用while让它循环遍历，因为我们不知道要遍历多少次，因此使用while
         */
        while(curr != null) {
            /**
             * 我们刚开始说过，迭代方法要考虑，前驱、当前、后继结点，前两个我们已经定义了，现在要定义后继了
             * 为什么要在这里才定义后继结点，因为当前指针(也就是游标指针不停的变化，而后继结点也需要跟着游标指针变化)
             */
            ListNode next = curr.next;
            /**
             * 迭代方法的反转步骤：
             * 1.游标指针的next域指向前驱
             * 2.前驱指针移动到游标指针的位置
             * 3.游标指针移动到后继指针的位置
             * 从这个方法可以看出，迭代法的本质就是游标和前驱结点的链反转
             */
            curr.next = pre;
            pre = curr;
            curr = next;
        }
        // 最后，别忘了返回头节点
        return pre;
    }
}
```
go版本
```go
// 方法一：迭代法
func reverseList(head *utils.ListNode) *utils.ListNode {
	var pre *utils.ListNode
	cur := head
	for cur != nil {
		next := cur.Next
		// 反转
		cur.Next = pre
		pre = cur
		cur = next
	}
	return pre
}

// 方法二：递归法
func reverseList2(head *utils.ListNode) *utils.ListNode {
	if head == nil || head.Next == nil {
		return head
	}

	newHead := reverseList2(head.Next)
	// 反转
	head.Next.Next = head
	head.Next = nil
	return newHead
}
```

#### **2、相交链表（ LeetCode 160）**

**题目：** 给你两个单链表的头节点 `headA` 和 `headB` ，请你找出并返回两个单链表相交的起始节点。如果两个链表不存在相交节点，返回 `null` 。

**示例：** 

```
输入：intersectVal = 8, listA = [4,1,8,4,5], listB = [5,6,1,8,4,5], skipA = 2, skipB = 3  
输出：Intersected at '8'
解释：相交节点的值为 8 （注意，如果两个链表相交则不能为 0）。
从各自的表头开始算起，链表 A 为 [4,1,8,4,5]，链表 B 为 [5,6,1,8,4,5]。在 A 中，相交节点前有 2 个节点；在 B 中，相交节点前有 3 个节点。
```

**题解：** 

java版本

```java
/**
 * 其实这就是判断链表有环的变种题
 * 判断两个链表是否相交，只需要定义两个指针A、B分别指向两个链表
 * 然后，同时遍历自己的链表，当遍历完自己的链表时，就去遍历别人的链表
 * 如果，两个链表能够相交，那么A、B两个指针会重合，那么重合的结点就是两个链表相交的起始点
 */
public class LeetCode160 {

    public ListNode getIntersectionNode(ListNode headA, ListNode headB) {
        /**
         * 先判断两个链表是否有空的
         */
        if (headA == null || headB == null) {
            return null;
        }

        ListNode pa = headA;
        ListNode pb = headB;

        /**
         * 循环遍历的条件，因为两个结点重合，表示相交，因此我们的循环条件，就是两个指针不相等的时候，一直循环
         */
        while (pa != pb) {

            // 循环遍历，如果当前结点不为空，就遍历下一个，为空了，就说明该链表已经遍历完，那就应该让它指向另一个链表

            pa = pa != null ? pa.next : headB;
            pb = pb != null ? pb.next : headA;
        }

        return pa;
    }

}
```
go版本

```go
func getIntersectionNode(headA, headB *ListNode) *ListNode {
    if headA == nil || headB == nil {
		return nil
	}

	curA, curB := headA, headB

	for curA != curB {
        // 注意 go语法没有三目运算
		if curA == nil {
			curA = headB
		} else {
			curA = curA.Next
		}

		if curB == nil {
			curB = headA
		} else {
			curB = curB.Next
		}
	}
	return curA
}
```

#### **3、合并两个有序链表（ LeetCode 21）**

**题目：** 将两个升序链表合并为一个新的升序链表并返回。新链表是通过拼接给定的两个链表的所有节点组成的。 

**示例：** 

```
输入：l1 = [1,2,4], l2 = [1,3,4]    
输出：[1,1,2,3,4,4]
```

**题解：** 

java版本
```java
/**
 * 合并两个有序链表：最需要注意的是两个地方
 * 1.一个已经结束了，要把另一个直接添加到最后
 * 2.遇到相同的，排序问题，也就是你自己定义谁在前，谁在后
 * 3.穿针引线方法（只要涉及到穿针引线法，都要加一个虚拟节点）
 */
public class LeetCode21 {

    public ListNode mergeTwoLists(ListNode list1, ListNode list2) {
        // 构造一个虚拟结点，方便我们后续操作，取值随便写一个就行
        ListNode preHead = new ListNode(-1);
        // 定义一个穿线的针
        ListNode curr = preHead;

        /**
         * 循环遍历两个链表，只有当两个都不为空的时候，才遍历。
         * 因为如果有一个为空，那么就直接返回另一个就行，
         * 如果一个遍历完了，那么就直接退出，将另一个剩下的元素，直接挂在最后面
         */
        while(list1 != null && list2 !=null) {
            // 定义谁在前，谁在后
            if(list1.val < list2.val) {
                // 用针将其穿起来
                curr.next = list1;
                // 传完后，让遍历指针走向下一个
                list1 = list1.next;
            }else {
                curr.next = list2;
                list2 = list2.next;
            }
            // 能穿的都传完了之后，让curr指向最后一个结点
            curr = curr.next;
        }

        // 如果有一个没完，我们还要将其连接上，其实这里为了方便记忆，可以记成，一个为空的话，就返回另一个
        curr.next = list1 == null ? list2 : list1;

        return preHead.next;
    }

}
```
go版本
```go
func mergeTwoLists(list1 *utils.ListNode, list2 *utils.ListNode) *utils.ListNode {
	// 定义一个虚拟结点
	preHead := &utils.ListNode{Val: -1}
	// 定义一个针
	cur := preHead

	// 穿针引线
	for list1 != nil && list2 != nil {
		if list1.Val < list2.Val {
			cur.Next = list1
			list1 = list1.Next
		} else {
			cur.Next = list2
			list2 = list2.Next
		}
		cur = cur.Next
	}

	if list1 == nil {
		cur.Next = list2
	} else {
		cur.Next = list1
	}
	return preHead.Next
}
```

#### **4、环形链表Ⅰ（ LeetCode 141）**

**题目：** 给你一个链表的头节点 head ，判断链表中是否有环。如果链表中有某个节点，可以通过连续跟踪 next 指针再次到达，则链表中存在环。 为了表示给定链表中的环，评测系统内部使用整数 pos 来表示链表尾连接到链表中的位置（索引从 0 开始）。注意：pos 不作为参数进行传递 。仅仅是为了标识链表的实际情况。

**示例：** 

```
输入：head = [3,2,0,-4], pos = 1
输出：true
解释：链表中有一个环，其尾部连接到第二个节点。
```

**题解：** 

java版本
```java
public class Solution {
    // 快慢指针法 或者 称为 弗洛伊德法
    public boolean hasCycle(ListNode head) {
        if(null == head){
            return false;
        } 
        // 定义快慢指针，初始时，两个都指向同一个头节点
        ListNode fastPrt = head, slowPrt = head;

        while(null != fastPrt && null != fastPrt.next) {
            // 慢指针一次只走一步
            slowPrt = slowPrt.next;
            // 快指针一次走两步
            fastPrt = fastPrt.next.next;
            // 走完一次，就判断两个指针是否相等，相等则有环
            if(slowPrt == fastPrt) {
                return true;
            }
        }
        // 循环结束后，表示无环
        return false;
    }
}
```

go版本
```go
func hasCycle(head *utils.ListNode) bool {
	if head == nil {
		return false
	}

	fast, slow := head, head

	for fast != nil && fast.Next != nil {
		fast = fast.Next.Next
		slow = slow.Next
		if fast == slow {
			return true
		}
	}
	return false
}
```

#### **5、环形链表Ⅱ（ LeetCode 142）**

**题目：** 给定一个链表的头节点  head ，返回链表开始入环的第一个节点。 如果链表无环，则返回 null。如果链表中有某个节点，可以通过连续跟踪 next 指针再次到达，则链表中存在环。 为了表示给定链表中的环，评测系统内部使用整数 pos 来表示链表尾连接到链表中的位置（索引从 0 开始）。如果 pos 是 -1，则在该链表中没有环。注意：pos 不作为参数进行传递，仅仅是为了标识链表的实际情况。不允许修改链表。

**示例：** 

```
输入：head = [3,2,0,-4], pos = 1  输出：返回索引为 1 的链表节点
解释：链表中有一个环，其尾部连接到第二个节点。
```

**题解：** 

java版本
```java
/**
 * 判断链表有环的问题，但是这个题让输出的是下标，也就是交点的位置，且从0开始计算
 * 这种题，我们一般设置快慢指针，慢指针一次只走一步，快指针一次走两步，如果有环，就能够相交
 * 但是，本题给出的环形链表不是从头到尾的环形，而是从链表中某一个节点开始到结尾
 * 因此，此题还需要注意一点，两个指针相遇了，但是不一定是交点，有可能在环里的某个结点
 * 所以，我们在快慢指针相遇的时候，引入第三个指针，指向链表的起始，然后让其与slow指针一起向后每次移动一个
 * 两指针相遇的时候，就是交点。
 * 为什么呢？
 * 任意时刻，fast指针走过的距离都为slow指针的2倍。因此，我们有a+(n+1)b+nc=2(a+b)  ⟹  a=c+(n−1)(b+c)
 * 有了 a=c+(n−1)(b+c)的等量关系，因此，从相遇点到入环点的距离加上 n−1 圈的环长，恰好等于从链表头部到入环点的距离。
 */
public class LeetCode142 {

    public ListNode detectCycle(ListNode head) {
        // 定义快慢指针
        ListNode fast = head, slow = head;

        while(fast != null && fast.next != null) {
            //先让两个指针相交
            slow = slow.next;
            fast = fast.next.next;
            // 有相交，则必有焦点
            if(slow == fast) {
                // 定义游标，让其指向头节点
                ListNode cur = head;
                // 当游标不等于慢指针得时候，让两个都走一步，重合结点就是焦点
                while(cur != slow) {
                    cur = cur.next;
                    slow = slow.next;
                }
                return cur;
            }

        }
        return null;
    }
}
```
go版本
```go
func detectCycle(head *utils.ListNode) *utils.ListNode {
	if head == nil {
		return head
	}

	fast, slow := head, head

	for fast != nil && fast.Next != nil {
		fast = fast.Next.Next
		slow = slow.Next
		if fast == slow {
			cur := head
			for cur != slow {
				cur = cur.Next
				slow = slow.Next
			}
			return cur
		}
	}
	return nil
}
```
#### **6、反转链表Ⅱ（ LeetCode 92）**

**题目：** 给你单链表的头指针 head 和两个整数 left 和 right ，其中 left <= right 。请你反转从位置 left 到位置 right 的链表节点，返回 反转后的链表 。

**示例：**
```
输入：head = [1,2,3,4,5], left = 2, right = 4  
输出：[1,4,3,2,5]
```
**题解：** 
```java
/**
 * 反转链表2
 * 其实就是，反转给定区间的链表。
 * 本题需要注意的是：
 * 1.凡是需要考虑左右边界的问题, 加个虚拟头节点
 * 2.使用三个指针，前驱、当前、后继，然后进行反转
 * 3.反转策略有所变化
 */
public class LeetCode92 {
    public ListNode reverseBetween(ListNode head, int left, int right) {

        // 首先定义虚拟结点，让虚拟节点指向头节点，虚拟头节点得创建是为了防止从第一个结点开始反转
        ListNode dummyHead = new ListNode(0);
        dummyHead.next = head;

        // 先定义两个指针，分别是前驱和当前
        ListNode pre = dummyHead;
        ListNode curr = dummyHead.next;

        // 移位，当前指针移到左边界上，前驱结点在当前结点前一个
        for (int i = 0; i < left - 1; i++) {
            pre = pre.next;
            curr = curr.next;
        }

        // 当前都在左边界上，因此对区间中元素进行反转
        for (int i = 0; i < right - left; i++) {
            // 定义后继结点
            ListNode next = curr.next;
            // 反转策略
            // 当前结点next域指向它后面的后面
            curr.next = curr.next.next;
            // 后驱结点next域指向前驱结点的后一个
            next.next = pre.next;
            // 前驱结点的next域指向后驱结点
            pre.next = next; 
        }
        return dummyHead.next;
    }
}
```

#### **7、链表的中间结点（ LeetCode 876）**
**思路：** 快慢指针，设定的快指针比慢指针大n步，则当快指针走到最后一个，慢指针就会走到1/(n + 1)处。

**题目：** 给定一个头结点为 head 的非空单链表，返回链表的中间结点。如果有两个中间结点，则返回第二个中间结点。

**示例：** 
```
输入：[1,2,3,4,5]
输出：此列表中的结点 3 (序列化形式：[3,4,5])
返回的结点值为 3 。 (测评系统对该结点序列化表述是 [3,4,5])。
注意，我们返回了一个 ListNode 类型的对象 ans，这样：
ans.val = 3, ans.next.val = 4, ans.next.next.val = 5, 以及 ans.next.next.next = NULL.

输入：[1,2,3,4,5,6]
输出：此列表中的结点 4 (序列化形式：[4,5,6])
由于该列表有两个中间结点，值分别为 3 和 4，我们返回第二个结点。
```
**题解：** 
```java
// 本地思路：快慢指针，快指针比慢指针快n步，则当快指针走到最后一个，慢指针就会走到1/(n + 1)处
class Solution {
    public ListNode middleNode(ListNode head) {
        if(head == null) {
            return null;
        }
        ListNode slow = head , fast = head;
        // 循环结束条件是快指针结点为空和快指针的下一个结点为空
        while(fast !=null && fast.next != null){
            fast = fast.next.next;
            slow = slow.next;
        }
        return slow;
    }
}
```

如果本地要求返回中间两个数字的第一个，则循环条件应该更改为下面的写法

```java
while(fast.next != null && fast.next.next != null){
    fast = fast.next.next;
    slow = slow.next;
}
```

#### **8、删除链表的倒数第 N 个结点（ LeetCode 19）**
**思路：** 快慢指针，设定的快指针比慢指针大n步，则当快指针走到最后一个，慢指针就会走到1/(n + 1)处。

**题目：** 给你一个链表，删除链表的倒数第 n 个结点，并且返回链表的头结点。

**示例：** 
```
输入：head = [1,2,3,4,5], n = 2
输出：[1,2,3,5]

输入：head = [1], n = 1
输出：[]
```
**题解：** 
```java
// 本题也是快慢指针的思想，要删除链表中的某个节点，需要知道其前一个节点。对于头节点来说，其没有前一个节点，如果要删除头节点，则没有办法了。因此，需定义虚拟头节点。
// 思路，删除倒数第n个结点，那就要找到倒数第n+1个结点。如何找到第n+1个结点呢？让快指针先走n+1步，然后快慢指针同时向后移动一步，当快指针走到null时，慢指针刚好在倒数第n+1个位置，然后再执行删除操作。

class Solution {
    public ListNode removeNthFromEnd(ListNode head, int n) {
        // 创建虚拟结点
        ListNode dumyNode = new ListNode(0);
        // 指向头节点
        dumyNode.next = head;
        // 让快慢指针指向虚拟结点，这个必须要指向虚拟结点，这样才能保证指向null
        ListNode fast = dumyNode, slow = dumyNode;

        // 让快指针先走 n + 1步
        for(int i = 0; i <= n; i++) {
            fast = fast.next;
        }

        // 快指针指向null时，停止循环。快慢指针都走一步。
        while(fast != null) {
            slow = slow.next;
            fast = fast.next;
        }

        ListNode removeNode = slow.next;
        slow.next = slow.next.next;
        removeNode.next = null;
        // 一定要返回虚拟结点的下一个
        return dumyNode.next;
    }
}
```
#### **9、复制带随机指针的链表（ LeetCode 138）**

**题目：** 给你一个长度为 n 的链表，每个节点包含一个额外增加的随机指针 random ，该指针可以指向链表中的任何节点或空节点。

构造这个链表的 深拷贝。 深拷贝应该正好由 n 个 全新 节点组成，其中每个新节点的值都设为其对应的原节点的值。新节点的 next 指针和 random 指针也都应指向复制链表中的新节点，并使原链表和复制链表中的这些指针能够表示相同的链表状态。复制链表中的指针都不应指向原链表中的节点 。

例如，如果原链表中有 X 和 Y 两个节点，其中 X.random --> Y 。那么在复制链表中对应的两个节点 x 和 y ，同样有 x.random --> y 。

返回复制链表的头节点。

用一个由 n 个节点组成的链表来表示输入/输出中的链表。每个节点用一个 [val, random_index] 表示：

    val：一个表示 Node.val 的整数。
    random_index：随机指针指向的节点索引（范围从 0 到 n-1）；如果不指向任何节点，则为  null 。

你的代码 只 接受原链表的头节点 head 作为传入参数。

**示例：** 
```
输入：head = [[7,null],[13,0],[11,4],[10,2],[1,0]]
输出：[[7,null],[13,0],[11,4],[10,2],[1,0]]
```
**题解：** 

```java
class Node {
    int val;
    Node next;
    Node random;

    public Node(int val) {
        this.val = val;
        this.next = null;
        this.random = null;
    }
}

/**
 * 深拷贝的问题：其实际上就是按照给定链表，再复制一份链表
 */

class Solution {
    public Node copyRandomList(Node head) {
        // 第一步：复制结点，并将他们链接到一个链表中
        for(Node p = head; p != null; p = p.next.next){
            Node q = new Node(p.val);
            q.next = p.next;
            p.next = q;
        }

        // 第二步: 复制random
        for(Node p = head; p != null; p = p.next.next) {
            if(p.random != null) {
                p.next.random = p.random.next;
            }
        }


        // 第三步: 穿针引线
        Node dummyNode = new Node(-1);
        Node cur = dummyNode;

        for(Node p = head; p != null; p = p.next) {
            Node q = p.next;
            cur.next = q;
            cur = cur.next;
            // 断链、恢复原来的链表
            p.next = q.next;
        }
        return dummyNode.next;
    }
}
```
#### **10、LRU（ LeetCode 146）**

**题目：** 请你设计并实现一个满足  LRU (最近最少使用) 缓存 约束的数据结构。实现 LRUCache 类：
```
LRUCache(int capacity) 以 正整数 作为容量 capacity 初始化 LRU 缓存
int get(int key) 如果关键字 key 存在于缓存中，则返回关键字的值，否则返回 -1 。
void put(int key, int value) 如果关键字 key 已经存在，则变更其数据值 value ；如果不存在，则向缓存中插入该组 key-value 。如果插入操作导致关键字数量超过 capacity ，则应该 逐出 最久未使用的关键字。函数 get 和 put 必须以 O(1) 的平均时间复杂度运行。
```
**示例：** 

```
输入
["LRUCache", "put", "put", "get", "put", "get", "put", "get", "get", "get"]
[[2], [1, 1], [2, 2], [1], [3, 3], [2], [4, 4], [1], [3], [4]]
输出
[null, null, null, 1, null, -1, null, -1, 3, 4]

解释
LRUCache lRUCache = new LRUCache(2);
lRUCache.put(1, 1); // 缓存是 {1=1}
lRUCache.put(2, 2); // 缓存是 {1=1, 2=2}
lRUCache.get(1);    // 返回 1
lRUCache.put(3, 3); // 该操作会使得关键字 2 作废，缓存是 {1=1, 3=3}
lRUCache.get(2);    // 返回 -1 (未找到)
lRUCache.put(4, 4); // 该操作会使得关键字 1 作废，缓存是 {4=4, 3=3}
lRUCache.get(1);    // 返回 -1 (未找到)
lRUCache.get(3);    // 返回 3
lRUCache.get(4);    // 返回 4
```

**题解：** 
```java
// 整体思路，基本是：哈希表 + 双向链表
// 双向链表为缓存的主体，哈希表是单另维护的一个表，用来存放每个结点和其对应的key
import java.util.HashMap;
import java.util.Map;

class DLinkedNode {
    int key;
    int value;
    DLinkedNode prev;
    DLinkedNode next;
    public DLinkedNode() {}
    public DLinkedNode(int _key, int _value) {key = _key; value = _value;}
}

public class LeetCode146 {
    // 定义一个映射缓存
    private Map<Integer, DLinkedNode> cache = new HashMap<Integer, DLinkedNode>();
    // 表长大小
    private int size;
    // 容量
    private int capacity;
    // 虚拟头节点，方便头插和尾删除
    private DLinkedNode head, tail;

    public LeetCode146(int capacity) {
        this.size = 0;
        this.capacity = capacity;
        // 使用伪头部和伪尾部节点
        head = new DLinkedNode();
        tail = new DLinkedNode();
        head.next = tail;
        tail.prev = head;
    }

    public int get(int key) {
        // 先查缓存表
        DLinkedNode node = cache.get(key);
        if (node == null) {
            return -1;
        }
        // 如果 key 存在，先通过哈希表定位，再移到头部
        moveToHead(node);
        return node.value;
    }

    public void put(int key, int value) {
        DLinkedNode node = cache.get(key);
        if (node == null) {
            // 如果 key 不存在，创建一个新的节点
            DLinkedNode newNode = new DLinkedNode(key, value);
            // 添加进哈希表
            cache.put(key, newNode);
            // 添加至双向链表的头部
            addToHead(newNode);
            ++size;
            if (size > capacity) {
                // 如果超出容量，删除双向链表的尾部节点
                DLinkedNode tail = removeTail();
                // 删除哈希表中对应的项
                cache.remove(tail.key);
                --size;
            }
        }
        else {
            // 如果 key 存在，先通过哈希表定位，再修改 value，并移到头部，注意put已有的值时，也会将其移动到头部
            node.value = value;
            moveToHead(node);
        }
    }

    private void addToHead(DLinkedNode node) {
        node.prev = head;
        node.next = head.next;
        head.next.prev = node;
        head.next = node;
    }

    private void removeNode(DLinkedNode node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }

    private void moveToHead(DLinkedNode node) {
        removeNode(node);
        addToHead(node);
    }

    private DLinkedNode removeTail() {
        DLinkedNode res = tail.prev;
        removeNode(res);
        return res;
    }

}
```
#### **11、比较含退格的字符串（ LeetCode 844）**

**题目：** 给定 s 和 t 两个字符串，当它们分别被输入到空白的文本编辑器后，如果两者相等，返回 true 。# 代表退格字符。注意：如果对空文本输入退格字符，文本继续为空。

**示例：**
```
输入：s = "ab#c", t = "ad#c"
输出：true
解释：s 和 t 都会变成 "ac"。
```

**题解：** 
```java
public class LeetCode844 {

    public boolean backspaceCompare(String S, String T) {

        // 首先定义两个指针，指向字符串的最后一位
        int i = S.length() - 1;
        int j = T.length() - 1;
        // 其次定义两个skip，表示记录每个字符串中#号的个数
        int skipS = 0, skipT = 0;

        // 进入循环
        while(i >= 0 || j >= 0) {
            // 分别进行循环,先找到 S 中第一个需要比较的字符（即去除 # 影响后的第一个待比较字符）
            while(i >= 0) {
                if(S.charAt(i) == '#'){
                    skipS++;
                    i--;
                }else if(skipS > 0) {
                    skipS --;
                    i--;
                }else{
                    break;
                }
            }

            // 再找到 T 中第一个需要比较的字符（即去除 # 影响后的第一个待比较字符）
            while(j >= 0) {
                if(T.charAt(i) == '#'){
                    skipT++;
                    j--;
                }else if(skipT > 0) {
                    skipT --;
                    j--;
                }else{
                    break;
                }
            }
            // (i >= 0 && j >= 0) 为 false 情况为
            // 1. i < 0 && j >= 0
            // 2. i >= 0 && j < 0
            // 3. i < 0 && j < 0
            // 其中，第 3 种情况为符合题意情况，因为这种情况下 s 和 t 都是 index = 0 的位置为 '#' 而这种情况下
            // 退格空字符即为空字符，也符合题意，应当返回 True。
            // 但是，情况 1 和 2 不符合题意，因为 s 和 t 其中一个是在 index >= 0 处找到了待比较字符，另一个没有找到
            // 这种情况显然不符合题意，应当返回 False，下式便处理这种情况。
            if(i>=0 && j >=0){
                if(S.charAt(i) == T.charAt(j)){
                    return false;
                }
            }else if(i>=0 || j >=0){
                return false;
            }

            i--;
            j--;

        }
        return true;
    }
}
```
#### **12、棒球比赛（ LeetCode 682）**

**题目：** 你现在是一场采用特殊赛制棒球比赛的记录员。这场比赛由若干回合组成，过去几回合的得分可能会影响以后几回合的得分。
```
比赛开始时，记录是空白的。你会得到一个记录操作的字符串列表 ops，其中 ops[i] 是你需要记录的第 i 项操作，ops 遵循下述规则：

    整数 x - 表示本回合新获得分数 x
    "+" - 表示本回合新获得的得分是前两次得分的总和。题目数据保证记录此操作时前面总是存在两个有效的分数。
    "D" - 表示本回合新获得的得分是前一次得分的两倍。题目数据保证记录此操作时前面总是存在一个有效的分数。
    "C" - 表示前一次得分无效，将其从记录中移除。题目数据保证记录此操作时前面总是存在一个有效的分数。

请你返回记录中所有得分的总和。
```
**示例：**
```
输入：ops = ["5","2","C","D","+"]
输出：30
解释：
"5" - 记录加 5 ，记录现在是 [5]
"2" - 记录加 2 ，记录现在是 [5, 2]
"C" - 使前一次得分的记录无效并将其移除，记录现在是 [5].
"D" - 记录加 2 * 5 = 10 ，记录现在是 [5, 10].
"+" - 记录加 5 + 10 = 15 ，记录现在是 [5, 10, 15].
所有得分的总和 5 + 10 + 15 = 30
```

**题解：** 
```java

```
#### **13、下一个更大元素 I（ LeetCode 496）**
**题目：** nums1 中数字 x 的 下一个更大元素 是指 x 在 nums2 中对应位置 右侧 的 第一个 比 x 大的元素。

给你两个 没有重复元素 的数组 nums1 和 nums2 ，下标从 0 开始计数，其中nums1 是 nums2 的子集。

对于每个 0 <= i < nums1.length ，找出满足 nums1[i] == nums2[j] 的下标 j ，并且在 nums2 确定 nums2[j] 的 下一个更大元素 。如果不存在下一个更大元素，那么本次查询的答案是 -1 。

返回一个长度为 nums1.length 的数组 ans 作为答案，满足 ans[i] 是如上所述的 下一个更大元素 。

**示例：**
```
输入：nums1 = [4,1,2], nums2 = [1,3,4,2].
输出：[-1,3,-1]
解释：nums1 中每个值的下一个更大元素如下所述：
- 4 ，用加粗斜体标识，nums2 = [1,3,4,2]。不存在下一个更大元素，所以答案是 -1 。
- 1 ，用加粗斜体标识，nums2 = [1,3,4,2]。下一个更大元素是 3 。
- 2 ，用加粗斜体标识，nums2 = [1,3,4,2]。不存在下一个更大元素，所以答案是 -1 。
```

**题解：** 
```java

```
#### **14、有效的括号（ LeetCode 20）**

**题目：** 给定一个只包括 '('，')'，'{'，'}'，'['，']' 的字符串 s ，判断字符串是否有效。

**示例：** 输入：s = "()"    输出：true

**题解：** 

```java
/**
 * 括号匹配规则，本题比较简单
 */
public class LeetCode20 {

    public boolean isValid(String s) {

        // 第一步：首先判断字符串长度是否能够被2整除，不能整除，必定不是有效括号
        if (s.length() % 2 != 0) {
            return false;
        }

        // 第二步，定义一个栈
        Stack<Character> stack = new Stack<>();

        // 第三步，遍历字符串
        for (char c : s.toCharArray()) {
            if(c=='('){
                stack.push(')');
            }else if(c == '['){
                stack.push(']');
            }else if(c == '{') {
                stack.push('}');

            /**
             *  前三个判断语句，用来将括号入栈，
             *  第四个判断语句，实际作用就是括号匹配的关键算法
             */    

            } else if(stack.isEmpty() || c != stack.pop()){
                return false;
            }
        }

        return stack.isEmpty();
    }
}
```

#### **15、基本计算器（ LeetCode 224）**

**题目：** 给你一个字符串表达式 `s` ，请你实现一个基本计算器来计算并返回它的值。

注意:不允许使用任何将字符串作为数学表达式计算的内置函数，比如 `eval()` 

**示例：** 输入：s = "1 + 1"    输出：2

**题解：** 

```java
public class LeetCode224 {
    public int calculate(String s) {
        int[] res = dfs(s.toCharArray(), 0, s.length());
        return res[0];
    }
    // 计算s[index....]的值遇到右括号或者字符串结尾停止
    // 返回当前计算结果和计算到了哪个位置
    private int[] dfs(char[] s, int index, int N) {
        int res = 0;
        int sign = 1;
        int i = index;
        while (i < N) {
            char c = s[i];
            if (c != '(' && c != ')') {
                if (c >= '0' && c <= '9') {
                    // 遇到数字，将连续的数字值计算出来，累加到结果中
                    int num = c - '0';
                    while (i+1 < N && s[i+1] >= '0' && s[i+1] <= '9') {
                        num = num * 10 + s[++i] - '0';
                    }
                    res += sign * num;
                }
                else if (c == '+') sign = 1;
                else if (c == '-') sign = -1;
                i++;
            } else { // 遇到左括号或右括号
                if (c == '(') {
                    // 遇到左括号，递归调用s[i+1...]的计算结果
                    if (i + 1 < N) {
                        int[] nextRes = dfs(s, i + 1, N);
                        res = res + sign * nextRes[0];
                        i = nextRes[1] + 1;
                    } else i++;
                } else if (c == ')')  {
                    // 遇到右括号，返回当前计算结果和计算到了哪个位置
                    return new int[]{res, i++};
                }
            }
        }
        // 遇到字符串结尾，返回当前计算结果和计算到了哪个位置
        return new int[]{res, i};
    }
}
```

#### **16、逆波兰表达式求值（ LeetCode 150 ）**

**题目：** 根据 逆波兰表示法，求表达式的值。有效的算符包括 +、-、*、/ 。每个运算对象可以是整数，也可以是另一个逆波兰表达式。

注意 两个整数之间的除法只保留整数部分。

可以保证给定的逆波兰表达式总是有效的。换句话说，表达式总会得出有效数值且不存在除数为 0 的情况。

**示例：** 输入：tokens = ["2","1","+","3","*"]  输出：9

解释：该算式转化为常见的中缀算术表达式为：((2 + 1) * 3) = 9

**题解：** 

```java
public class LeetCode150 {

    /**
     * 数组模拟栈写法
     * @param tokens
     * @return
     */
    public int evalRPN(String[] tokens) {
        int n = tokens.length;
        int[] stack = new int[(n + 1) / 2];
        int index = -1;
        for (int i = 0; i < n; i++) {
            String token = tokens[i];
            switch (token) {
                case "+":
                    index--;
                    stack[index] += stack[index + 1];
                    break;
                case "-":
                    index--;
                    stack[index] -= stack[index + 1];
                    break;
                case "*":
                    index--;
                    stack[index] *= stack[index + 1];
                    break;
                case "/":
                    index--;
                    stack[index] /= stack[index + 1];
                    break;
                default:
                    index++;
                    stack[index] = Integer.parseInt(token);
            }
        }
        return stack[index];
    }


    /**
     * 递归写法
     * @param tokens
     * @param index
     * @return
     */
    private int evo(String[] tokens, int[] index){
        switch(tokens[--index[0]]){
            case "+":
                return evo(tokens, index) + evo(tokens, index);
            case "-":
                return -1*evo(tokens, index) + evo(tokens, index);
            case "*":
                return evo(tokens, index) * evo(tokens, index);
            case "/":
                int divisor = evo(tokens, index);
                return evo(tokens, index)/divisor;
            default:
                return Integer.parseInt(tokens[index[0]]);
        }
    }

}
```

#### **17、最小栈（ LeetCode 155）**

**题目：** 设计一个支持 push ，pop ，top 操作，并能在常数时间内检索到最小元素的栈。

实现 MinStack 类:

    MinStack() 初始化堆栈对象。
    void push(int val) 将元素val推入堆栈。
    void pop() 删除堆栈顶部的元素。
    int top() 获取堆栈顶部的元素。
    int getMin() 获取堆栈中的最小元素。

**示例：** 

```
["MinStack","push","push","push","getMin","pop","top","getMin"]
[[],[-2],[0],[-3],[],[],[],[]]

输出：
[null,null,null,null,-3,null,0,-2]

解释：
MinStack minStack = new MinStack();
minStack.push(-2);
minStack.push(0);
minStack.push(-3);
minStack.getMin();   --> 返回 -3.
minStack.pop();
minStack.top();      --> 返回 0.
minStack.getMin();   --> 返回 -2.
```

输入：

**题解：** 面试要求，不适用辅助栈来实现

```java
public class LeetCode155 {
    private Node head;

    public LeetCode155() {
    }

    public void push(int val) {
        if (head == null) {
            head = new Node(val, val);
        } else {
            head = new Node(val, Math.min(val, head.min), head);
        }
    }

    public void pop() {
        head = head.next;
    }

    public int top() {
        return head.val;
    }

    public int getMin() {
        return head.min;
    }

    private class Node {
        int val;
        int min;
        Node next;

        public Node(int val, int min) {
            this(val, min, null);
        }

        public Node(int val, int min, Node next) {
            this.val = val;
            this.min = min;
            this.next = next;
        }
    }

}
```

#### **18、验证栈序列（ LeetCode 946）**

**题目：** 给定 pushed 和 popped 两个序列，每个序列中的 值都不重复，只有当它们可能是在最初空栈上进行的推入 push 和弹出 pop 操作序列的结果时，返回 true；否则，返回 false 。

**示例：** 

```
输入：pushed = [1,2,3,4,5], popped = [4,5,3,2,1]
输出：true
解释：我们可以按以下顺序执行：
push(1), push(2), push(3), push(4), pop() -> 4,
push(5), pop() -> 5, pop() -> 3, pop() -> 2, pop() -> 1
```

**题解：** 

```java
class Solution {
    public boolean validateStackSequences(int[] pushed, int[] popped) {
        int N = pushed.length;
        Stack<Integer> stack = new Stack<>();
        // 用来指向出栈队列
        int j = 0;
        for(int x : pushed) {
            stack.push(x);
            while(!stack.isEmpty() && j < N && stack.peek() == popped[j]) {
                stack.pop();
                j++;
            }
        }
        return j == N;
    }
}
```

#### **19、每日温度（ LeetCode 739）**

**题目：** 给定一个整数数组 temperatures ，表示每天的温度，返回一个数组 answer ，其中 answer[i] 是指对于第 i 天，下一个更高温度出现在几天后。如果气温在这之后都不会升高，请在该位置用 0 来代替。

**示例：** 输入: temperatures = [73,74,75,71,69,72,76,73]  输出: [1,1,4,2,1,1,0,0]

**题解：** 

```java
class Solution {
    public int[] dailyTemperatures(int[] temperatures) {
        Deque<Integer> stack = new ArrayDeque<>();
        int[] result = new int[temperatures.length];

        for(int i =0; i < temperatures.length; i++) {
            while(!stack.isEmpty() && temperatures[i] > temperatures[stack.peek()]){
                int pre = stack.pop();
                result[pre] = i - pre;
            }
            stack.push(i);
        }
        return result;
    }
}
```

#### **20、接雨水（ LeetCode 42）**

**题目：** 给定 n 个非负整数表示每个宽度为 1 的柱子的高度图，计算按此排列的柱子，下雨之后能接多少雨水。

**示例：** 输入：height = [0,1,0,2,1,0,1,3,2,1,2,1]  输出：6

解释：上面是由数组 [0,1,0,2,1,0,1,3,2,1,2,1] 表示的高度图，在这种情况下，可以接 6 个单位的雨水（蓝色部分表示雨水）。 

**题解：** 

```java
class Solution {
    public int trap(int[] walls) {
        if (walls == null || walls.length <= 2) {
            return 0;
        }

        // 思路：
        // 单调不增栈，walls元素作为右墙依次入栈
        // 出现入栈元素（右墙）比栈顶大时，说明在右墙左侧形成了低洼处，低洼处出栈并结算该低洼处能接的雨水

        int water = 0;
        Deque<Integer> stack = new ArrayDeque();
        for (int right=0; right<walls.length; right++) {
            // 栈不为空，且当前元素（右墙）比栈顶（右墙的左侧）大：说明形成低洼处了
            while (!stack.isEmpty() && walls[right]>walls[stack.peek()]) {
                // 低洼处弹出，尝试结算此低洼处能积攒的雨水
                int bottom = stack.pop();
                // 看看栈里还有没有东西（左墙是否存在）
                // 有右墙+有低洼+没有左墙=白搭
                if (stack.isEmpty()) {
                    break;
                }

                // 左墙位置以及左墙、右墙、低洼处的高度
                int left = stack.peek();
                int leftHeight = walls[left];
                int rightHeight = walls[right];
                int bottomHeight = walls[bottom];

                // 能积攒的水=(右墙位置-左墙位置-1) * (min(右墙高度, 左墙高度)-低洼处高度)
                water += (right-left-1) * (Math.min(leftHeight, rightHeight)-bottomHeight);
            }

            // 上面的pop循环结束后再push，保证stack是单调不增
            stack.push(right);
        }

        return water;
    }
}
```

#### **21、用栈实现队列（ LeetCode 232）**

**题目：** 请你仅使用两个栈实现先入先出队列。队列应当支持一般队列支持的所有操作（push、pop、peek、empty）：

实现 MyQueue 类：

    void push(int x) 将元素 x 推到队列的末尾
    int pop() 从队列的开头移除并返回元素
    int peek() 返回队列开头的元素
    boolean empty() 如果队列为空，返回 true ；否则，返回 false

说明：你 只能 使用标准的栈操作 —— 也就是只有 push to top, peek/pop from top, size, 和 is empty 操作是合法的。你所使用的语言也许不支持栈。你可以使用 list 或者 deque（双端队列）来模拟一个栈，只要是标准的栈操作即可。

**示例：** 

```
输入：
["MyQueue", "push", "push", "peek", "pop", "empty"]
[[], [1], [2], [], [], []]
输出：
[null, null, null, 1, 1, false]

解释：
MyQueue myQueue = new MyQueue();
myQueue.push(1); // queue is: [1]
myQueue.push(2); // queue is: [1, 2] (leftmost is front of the queue)
myQueue.peek(); // return 1
myQueue.pop(); // return 1, queue is [2]
myQueue.empty(); // return false
```

**题解：** 

```java
class MyQueue {

    Stack<Integer> s = new Stack<Integer>();
    Stack<Integer> sQueue = new Stack<Integer>();

    /** Initialize your data structure here. */
    public MyQueue() {

    }

    /** Push element x to the back of queue. */
    public void push(int x) {
        while(!sQueue.isEmpty())  s.push(sQueue.pop());
        sQueue.push(x);
        while(!s.isEmpty())  sQueue.push(s.pop());
    }

    /** Removes the element from in front of queue and returns that element. */
    public int pop() {
        if(sQueue.isEmpty()) return -1;
        return sQueue.pop();
    }

    /** Get the front element. */
    public int peek() {
        if(sQueue.isEmpty()) return -1;
        return sQueue.peek();
    }

    /** Returns whether the queue is empty. */
    public boolean empty() {
        return sQueue.isEmpty();
    }
}

/**
 * Your MyQueue object will be instantiated and called as such:
 * MyQueue obj = new MyQueue();
 * obj.push(x);
 * int param_2 = obj.pop();
 * int param_3 = obj.peek();
 * boolean param_4 = obj.empty();
 */
```

#### **22、滑动窗口最大值（ LeetCode 239）**

**题目：** 给你一个整数数组 nums，有一个大小为 k 的滑动窗口从数组的最左侧移动到数组的最右侧。你只可以看到在滑动窗口内的 k 个数字。滑动窗口每次只向右移动一位。返回 滑动窗口中的最大值 。

**示例：** 

```
输入：nums = [1,3,-1,-3,5,3,6,7], k = 3
输出：[3,3,5,5,6,7]
解释：
滑动窗口的位置                最大值
---------------               -----
[1  3  -1] -3  5  3  6  7       3
 1 [3  -1  -3] 5  3  6  7       3
 1  3 [-1  -3  5] 3  6  7       5
 1  3  -1 [-3  5  3] 6  7       5
 1  3  -1  -3 [5  3  6] 7       6
 1  3  -1  -3  5 [3  6  7]      7
```

**题解：** 

```java
class Solution {
    public int[] maxSlidingWindow(int[] nums, int k) {
        if(nums == null || nums.length < 2){
            return nums;
        } 
        // 双向队列 保存当前窗口最大值的数组位置 保证队列中数组位置的数值按从大到小排序
        LinkedList<Integer> queue = new LinkedList();
        // 结果数组
        int[] result = new int[nums.length-k+1];
        // 遍历nums数组
        for(int i = 0;i < nums.length;i++){
            // 保证从大到小 如果前面数小则需要依次弹出，直至满足要求
            while(!queue.isEmpty() && nums[queue.peekLast()] <= nums[i]){
                queue.pollLast();
            }
            // 添加当前值对应的数组下标
            queue.addLast(i);
            // 判断当前队列中队首的值是否有效
            if(queue.peek() <= i-k){
                queue.poll();   
            } 
            // 当窗口长度为k时 保存当前窗口中最大值
            if(i+1 >= k){
                result[i+1-k] = nums[queue.peek()];
            }
        }
        return result;
    }
}
```

#### **23、设计循环双端队列（ LeetCode 641）**

**题目：** 设计实现双端队列。

实现 MyCircularDeque 类:

    MyCircularDeque(int k) ：构造函数,双端队列最大为 k 。
    boolean insertFront()：将一个元素添加到双端队列头部。 如果操作成功返回 true ，否则返回 false 。
    boolean insertLast() ：将一个元素添加到双端队列尾部。如果操作成功返回 true ，否则返回 false 。
    boolean deleteFront() ：从双端队列头部删除一个元素。 如果操作成功返回 true ，否则返回 false 。
    boolean deleteLast() ：从双端队列尾部删除一个元素。如果操作成功返回 true ，否则返回 false 。
    int getFront() )：从双端队列头部获得一个元素。如果双端队列为空，返回 -1 。
    int getRear() ：获得双端队列的最后一个元素。 如果双端队列为空，返回 -1 。
    boolean isEmpty() ：若双端队列为空，则返回 true ，否则返回 false  。
    boolean isFull() ：若双端队列满了，则返回 true ，否则返回 false 。

**示例：** 

```
输入
["MyCircularDeque", "insertLast", "insertLast", "insertFront", "insertFront", "getRear", "isFull", "deleteLast", "insertFront", "getFront"]
[[3], [1], [2], [3], [4], [], [], [], [4], []]
输出
[null, true, true, true, false, 2, true, true, true, 4]

解释
MyCircularDeque circularDeque = new MycircularDeque(3); // 设置容量大小为3
circularDeque.insertLast(1);                    // 返回 true
circularDeque.insertLast(2);                    // 返回 true
circularDeque.insertFront(3);                    // 返回 true
circularDeque.insertFront(4);                    // 已经满了，返回 false
circularDeque.getRear();                  // 返回 2
circularDeque.isFull();                        // 返回 true
circularDeque.deleteLast();                    // 返回 true
circularDeque.insertFront(4);                    // 返回 true
circularDeque.getFront();                // 返回 4
```

**题解：** 需要注意的点：1.数组大小是给定大小+1，因为rear所指向的位置始终是空的 2.末尾插入要先赋值再移动rear指针 3.队满和队空的条件

```java
public class MyCircularDeque {

    // 1、不用设计成动态数组，使用静态数组即可
    // 2、设计 head 和 tail 指针变量
    // 3、head == tail 成立的时候表示队列为空
    // 4、tail + 1 == head

    private int capacity;
    private int[] arr;
    private int front;
    private int rear;

    /**
     * Initialize your data structure here. Set the size of the deque to be k.
     */
    public MyCircularDeque(int k) {
        capacity = k + 1;
        arr = new int[capacity];

        // 头部指向第 1 个存放元素的位置
        // 插入时，先减，再赋值
        // 删除时，索引 +1（注意取模）
        front = 0;
        // 尾部指向下一个插入元素的位置
        // 插入时，先赋值，再加
        // 删除时，索引 -1（注意取模）
        rear = 0;
    }

    /**
     * Adds an item at the front of Deque. Return true if the operation is successful.
     */
    public boolean insertFront(int value) {
        if (isFull()) {
            return false;
        }
        front = (front - 1 + capacity) % capacity;
        arr[front] = value;
        return true;
    }

    /**
     * Adds an item at the rear of Deque. Return true if the operation is successful.
     */
    public boolean insertLast(int value) {
        if (isFull()) {
            return false;
        }
        arr[rear] = value;
        rear = (rear + 1) % capacity;
        return true;
    }

    /**
     * Deletes an item from the front of Deque. Return true if the operation is successful.
     */
    public boolean deleteFront() {
        if (isEmpty()) {
            return false;
        }
        // front 被设计在数组的开头，所以是 +1
        front = (front + 1) % capacity;
        return true;
    }

    /**
     * Deletes an item from the rear of Deque. Return true if the operation is successful.
     */
    public boolean deleteLast() {
        if (isEmpty()) {
            return false;
        }
        // rear 被设计在数组的末尾，所以是 -1
        rear = (rear - 1 + capacity) % capacity;
        return true;
    }

    /**
     * Get the front item from the deque.
     */
    public int getFront() {
        if (isEmpty()) {
            return -1;
        }
        return arr[front];
    }

    /**
     * Get the last item from the deque.
     */
    public int getRear() {
        if (isEmpty()) {
            return -1;
        }
        // 当 rear 为 0 时防止数组越界
        return arr[(rear - 1 + capacity) % capacity];
    }

    /**
     * Checks whether the circular deque is empty or not.
     */
    public boolean isEmpty() {
        return front == rear;
    }

    /**
     * Checks whether the circular deque is full or not.
     */
    public boolean isFull() {
        // 注意：这个设计是非常经典的做法
        return (rear + 1) % capacity == front;
    }
}
```

#### **24、移除链表元素（ LeetCode 203）**

**题目：** 给你一个链表的头节点 head 和一个整数 val ，请你删除链表中所有满足 Node.val == val 的节点，并返回 新的头节点。

**示例：** 输入：head = [1,2,6,3,4,5,6], val = 6   输出：[1,2,3,4,5]

**题解：** 

```java
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode(int x) { val = x; }
 * }
 */
class Solution {
 public ListNode removeElements(ListNode head, int val) {
        while(head != null && head.val == val){
            ListNode delNode = head;
            head = head.next;
            delNode.next = null;
        }

        if(head == null){
            return null;
        }

        ListNode prev = head;
        while(prev.next != null){
            if(prev.next.val == val){
                ListNode delNode = prev.next;
                prev.next = delNode.next;
                delNode.next = null;
            }else{
                prev = prev.next;
            }
        }
        return head;
    }
}
```

#### **25、k个一组反转链表（ LeetCode 25）**

**题目：** 给你链表的头节点 head ，每 k 个节点一组进行翻转，请你返回修改后的链表。k 是一个正整数，它的值小于或等于链表的长度。如果节点总数不是 k 的整数倍，那么请将最后剩余的节点保持原有顺序。你不能只是单纯的改变节点内部的值，而是需要实际进行节点交换。

**示例：** 输入：head = [1,2,3,4,5], k = 2   输出：[2,1,4,3,5]

**题解：** 

```java
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode() {}
 *     ListNode(int val) { this.val = val; }
 *     ListNode(int val, ListNode next) { this.val = val; this.next = next; }
 * }
 */
class Solution {

    public ListNode reverseKGroup(ListNode head, int k) {
        ListNode hair = new ListNode(0);
        hair.next = head;
        ListNode pre = hair;

        while (head != null) {
            ListNode tail = pre;
            // 查看剩余部分长度是否大于等于 k
            for (int i = 0; i < k; ++i) {
                tail = tail.next;
                if (tail == null) {
                    return hair.next;
                }
            }
            ListNode nex = tail.next;
            ListNode[] reverse = myReverse(head, tail);
            head = reverse[0];
            tail = reverse[1];
            // 把子链表重新接回原链表
            pre.next = head;
            tail.next = nex;
            pre = tail;
            head = tail.next;
        }

        return hair.next;
    }

    public ListNode[] myReverse(ListNode head, ListNode tail) {
        ListNode prev = tail.next;
        ListNode p = head;
        while (prev != tail) {
            ListNode nex = p.next;
            p.next = prev;
            prev = p;
            p = nex;
        }
        return new ListNode[]{tail, head};
    }

}
```

#### **26、回文链表（ LeetCode 234）**

**题目：** 给你一个单链表的头节点 head ，请你判断该链表是否为回文链表。如果是，返回 true ；否则，返回 false 。

**示例：** 输入：head = [1,2,2,1]   输出：true

**题解：** 

```java
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode(int x) { val = x; }
 * }
 */
class Solution {
    private ListNode frontPointer;
    public boolean isPalindrome(ListNode head) {
        frontPointer = head;
        return recursivelyCheck(head);
    }

    private boolean recursivelyCheck(ListNode head) {
        if(head != null){
            if (!recursivelyCheck(head.next)) return false;
            if(head.val != frontPointer.val) return false;
            frontPointer = frontPointer.next;
        }
        return true;
    }
}
```

#### **27、奇偶链表（ LeetCode 328）**

**题目：** 给定单链表的头节点 head ，将所有索引为奇数的节点和索引为偶数的节点分别组合在一起，然后返回重新排序的列表。第一个节点的索引被认为是 奇数 ， 第二个节点的索引为 偶数 ，以此类推。请注意，偶数组和奇数组内部的相对顺序应该与输入时保持一致。你必须在 O(1) 的额外空间复杂度和 O(n) 的时间复杂度下解决这个问题。

**示例：** 输入: head = [1,2,3,4,5]   输出: [1,3,5,2,4]

**题解：** 

```java
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode() {}
 *     ListNode(int val) { this.val = val; }
 *     ListNode(int val, ListNode next) { this.val = val; this.next = next; }
 * }
 */
class Solution {
    public ListNode oddEvenList(ListNode head) {
        if (head == null) {
            return head;
        }
        ListNode evenHead = head.next;
        ListNode odd = head, even = evenHead;
        while (even != null && even.next != null) {
            odd.next = even.next;
            odd = odd.next;
            even.next = odd.next;
            even = even.next;
        }
        odd.next = evenHead;
        return head;
    }
}
```

#### **30、数组中第K大的元素（ LeetCode 215）**

**题目：** 

**示例：** 

**题解：** 

```java

```




### **二、递归、排序**
#### **1、合并两个有序数组（ LeetCode 88）**

**题目：** 给你两个按 非递减顺序 排列的整数数组 nums1 和 nums2，另有两个整数 m 和 n ，分别表示 nums1 和 nums2 中的元素数目。请你 合并 nums2 到 nums1 中，使合并后的数组同样按 非递减顺序 排列。注意：最终，合并后数组不应由函数返回，而是存储在数组 nums1 中。为了应对这种情况，nums1 的初始长度为 m + n，其中前 m 个元素表示应合并的元素，后 n 个元素为 0 ，应忽略。nums2 的长度为 n 。

**示例：** 输入：nums1 = [1,2,3,0,0,0], m = 3, nums2 = [2,5,6], n = 3  输出：[1,2,2,3,5,6]

解释：需要合并 [1,2,3] 和 [2,5,6] 。合并结果是 [1,2,2,3,5,6] ，其中斜体加粗标注的为 nums1 中的元素。

**题解：** 

一、双指针法

```java
class Solution {
    public void merge(int[] nums1, int m, int[] nums2, int n) {
        int p1 = 0, p2 = 0;
        int[] sorted = new int[m + n];
        int cur;
        while (p1 < m || p2 < n) {
            if (p1 == m) {
                cur = nums2[p2++];
            } else if (p2 == n) {
                cur = nums1[p1++];
            } else if (nums1[p1] < nums2[p2]) {
                cur = nums1[p1++];
            } else {
                cur = nums2[p2++];
            }
            sorted[p1 + p2 - 1] = cur;
        }
        for (int i = 0; i != m + n; ++i) {
            nums1[i] = sorted[i];
        }
    }
}
```

二、逆向双指针法

```java
class Solution {
    public void merge(int[] nums1, int m, int[] nums2, int n) {
        int p1 = m - 1, p2 = n - 1;
        int tail = m + n - 1;
        int cur;
        while (p1 >= 0 || p2 >= 0) {
            if (p1 == -1) {
                cur = nums2[p2--];
            } else if (p2 == -1) {
                cur = nums1[p1--];
            } else if (nums1[p1] > nums2[p2]) {
                cur = nums1[p1--];
            } else {
                cur = nums2[p2--];
            }
            nums1[tail--] = cur;
        }
    }
}
```

#### **2、颜色分类（ LeetCode 75）**

**题目：** 给定一个包含红色、白色和蓝色、共 n 个元素的数组 nums ，原地对它们进行排序，使得相同颜色的元素相邻，并按照红色、白色、蓝色顺序排列。我们使用整数 0、 1 和 2 分别表示红色、白色和蓝色。必须在不使用库的sort函数的情况下解决这个问题。

**示例：** 输入：nums = [2,0,2,1,1,0]    输出：[0,0,1,1,2,2]

**题解：** 

```java
class Solution {
    public void sortColors(int[] nums) {
        int n = nums.length;
        int p0 = 0, p2 = n - 1;
        for (int i = 0; i <= p2; ++i) {
            while (i <= p2 && nums[i] == 2) {
                int temp = nums[i];
                nums[i] = nums[p2];
                nums[p2] = temp;
                --p2;
            }
            if (nums[i] == 0) {
                int temp = nums[i];
                nums[i] = nums[p0];
                nums[p0] = temp;
                ++p0;
            }
        }
    }
}
```

#### **3、部分排序（面试 16）**

**题目：** 给定一个整数数组，编写一个函数，找出索引m和n，只要将索引区间[m,n]的元素排好序，整个数组就是有序的。注意：n-m尽量最小，也就是说，找出符合条件的最短序列。函数返回值为[m,n]，若不存在这样的m和n（例如整个数组是有序的），请返回[-1,-1]。

**示例：** 输入： [1,2,4,7,10,11,7,12,6,7,16,18,19]    输出： [3,9]

**题解：** 

```java
class Solution {
    public int[] subSort(int[] nums) {
        if (nums == null || nums.length <= 1) {
            return new int[]{-1, -1};
        }

        int n = nums.length;
        int min = Integer.MAX_VALUE;
        int max = Integer.MIN_VALUE;
        int minChange = -1;
        int maxChange = -1;
        for (int i = 0; i < n; i++) {
            if (nums[i] < max) {
                maxChange = i;
            } else {
                max = nums[i];
            }
        }

        for (int i = n - 1; i >= 0; i--) {
            if (nums[i] > min) {
                minChange = i;
            } else {
                min = nums[i];
            }
        }
        return new int[]{minChange, maxChange};
    }
}
```

#### **4、计算右侧小于当前元素的个数（ LeetCode 315）**

**题目：** 给你一个整数数组 nums ，按要求返回一个新数组 counts 。数组 counts 有该性质： counts[i] 的值是  nums[i] 右侧小于 nums[i] 的元素的数量。

**示例：** 

    输入：nums = [5,2,6,1]
    输出：[2,1,1,0] 
    解释：
    5 的右侧有 2 个更小的元素 (2 和 1)
    2 的右侧仅有 1 个更小的元素 (1)
    6 的右侧有 1 个更小的元素 (1)
    1 的右侧有 0 个更小的元素

**题解：** 

```java
class Solution {
    private int[] index;
    private int[] temp;
    private int[] tempIndex;
    private int[] ans;

    public List<Integer> countSmaller(int[] nums) {
        this.index = new int[nums.length];
        this.temp = new int[nums.length];
        this.tempIndex = new int[nums.length];
        this.ans = new int[nums.length];
        for (int i = 0; i < nums.length; ++i) {
            index[i] = i;
        }
        int l = 0, r = nums.length - 1;
        mergeSort(nums, l, r);
        List<Integer> list = new ArrayList<Integer>();
        for (int num : ans) {
            list.add(num);
        }
        return list;
    }

    public void mergeSort(int[] a, int l, int r) {
        if (l >= r) {
            return;
        }
        int mid = (l + r) >> 1;
        mergeSort(a, l, mid);
        mergeSort(a, mid + 1, r);
        merge(a, l, mid, r);
    }

    public void merge(int[] a, int l, int mid, int r) {
        int i = l, j = mid + 1, p = l;
        while (i <= mid && j <= r) {
            if (a[i] <= a[j]) {
                temp[p] = a[i];
                tempIndex[p] = index[i];
                ans[index[i]] += (j - mid - 1);
                ++i;
                ++p;
            } else {
                temp[p] = a[j];
                tempIndex[p] = index[j];
                ++j;
                ++p;
            }
        }
        while (i <= mid)  {
            temp[p] = a[i];
            tempIndex[p] = index[i];
            ans[index[i]] += (j - mid - 1);
            ++i;
            ++p;
        }
        while (j <= r) {
            temp[p] = a[j];
            tempIndex[p] = index[j];
            ++j;
            ++p;
        }
        for (int k = l; k <= r; ++k) {
            index[k] = tempIndex[k];
            a[k] = temp[k];
        }
    }
}
```

#### **5、合并k个升序链表（ LeetCode 23）**

**题目：** 给你一个链表数组，每个链表都已经按升序排列。请你将所有链表合并到一个升序链表中，返回合并后的链表。

**示例：** 

    输入：lists = [[1,4,5],[1,3,4],[2,6]]
    输出：[1,1,2,3,4,4,5,6]
    解释：链表数组如下：
    [
    1->4->5,
    1->3->4,
    2->6
    ]
    将它们合并到一个有序链表中得到。
    1->1->2->3->4->4->5->6

**题解：** 

```java
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode() {}
 *     ListNode(int val) { this.val = val; }
 *     ListNode(int val, ListNode next) { this.val = val; this.next = next; }
 * }
 */
class Solution {
    public ListNode mergeKLists(ListNode[] lists) {
        return merge(lists, 0, lists.length - 1);
    }

    public ListNode merge(ListNode[] lists, int l, int r) {
        if (l == r) {
            return lists[l];
        }
        if (l > r) {
            return null;
        }
        int mid = (l + r) >> 1;
        return mergeTwoLists(merge(lists, l, mid), merge(lists, mid + 1, r));
    }

    public ListNode mergeTwoLists(ListNode a, ListNode b) {
        if (a == null || b == null) {
            return a != null ? a : b;
        }
        ListNode head = new ListNode(0);
        ListNode tail = head, aPtr = a, bPtr = b;
        while (aPtr != null && bPtr != null) {
            if (aPtr.val < bPtr.val) {
                tail.next = aPtr;
                aPtr = aPtr.next;
            } else {
                tail.next = bPtr;
                bPtr = bPtr.next;
            }
            tail = tail.next;
        }
        tail.next = (aPtr != null ? aPtr : bPtr);
        return head.next;
    }
}
```

#### **6、盛最多水的容器（ LeetCode 11）**

**题目：** 给定一个长度为 n 的整数数组 height 。有 n 条垂线，第 i 条线的两个端点是 (i, 0) 和 (i, height[i]) 。找出其中的两条线，使得它们与 x 轴共同构成的容器可以容纳最多的水。返回容器可以储存的最大水量。说明：你不能倾斜容器。

**示例：** 输入：[1,8,6,2,5,4,8,3,7]   输出：49 

解释：图中垂直线代表输入数组 [1,8,6,2,5,4,8,3,7]。在此情况下，容器能够容纳水（表示为蓝色部分）的最大值为 49。

**题解：** 

```java
class Solution {
    public int maxArea(int[] height) {
        int left = 0, right = height.length - 1, max = 0;
        while (left < right) {
            int minHeight = Math.min(height[left], height[right]);
            max = Math.max(max, (right - left) * minHeight);
            if (height[left] == minHeight) {
                while ((height[left] <= minHeight) && (left < right)) {
                    left++;
                }
            } else {
                while ((height[right] <= minHeight) && (left < right)) {
                    right--;
                }
            }
        }
        return max;
    }
}
```

#### **7、两数之和（ LeetCode 1）**

**题目：** 给定一个整数数组 nums 和一个整数目标值 target，请你在该数组中找出 和为目标值 target  的那 两个 整数，并返回它们的数组下标。你可以假设每种输入只会对应一个答案。但是，数组中同一个元素在答案里不能重复出现。

**示例：** 输入：nums = [2,7,11,15], target = 9  输出：[0,1]

解释：因为 nums[0] + nums[1] == 9 ，返回 [0, 1] 。

**题解：** 

```java
class Solution {
    public int[] twoSum(int[] nums, int target) {

        Map<Integer, Integer> hMap = new HashMap<>();
        for (int i = 0; i < nums.length; i++) {
            int temp = target - nums[i];
            if(hMap.containsKey(temp)){
                return new int[]{hMap.get(temp), i};
            }
            hMap.put(nums[i], i);
        }
        return new int[0];
    }
}
```

#### **8、分发饼干（ LeetCode 455）**

**题目：** 假设你是一位很棒的家长，想要给你的孩子们一些小饼干。但是，每个孩子最多只能给一块饼干。对每个孩子 i，都有一个胃口值 g[i]，这是能让孩子们满足胃口的饼干的最小尺寸；并且每块饼干 j，都有一个尺寸 s[j] 。如果 s[j] >= g[i]，我们可以将这个饼干 j 分配给孩子 i ，这个孩子会得到满足。你的目标是尽可能满足越多数量的孩子，并输出这个最大数值。

**示例：** 

    输入: g = [1,2,3], s = [1,1]
    输出: 1
    解释: 
    你有三个孩子和两块小饼干，3个孩子的胃口值分别是：1,2,3。
    虽然你有两块小饼干，由于他们的尺寸都是1，你只能让胃口值是1的孩子满足。
    所以你应该输出1。

**题解：** 

```java
class Solution {
    public int findContentChildren(int[] g, int[] s) {
        int ret = 0;
        Arrays.sort(g);
        Arrays.sort(s);
        int i = 0, j = 0;
        while (i < g.length && j < s.length) {
            if (g[i] <= s[j]) {
                ret++;
                i++;
                j++;
            }else if (g[i] > s[j]) {
                j++;
            }
        }
        return ret;
    }
}
```

#### **9、跳跃游戏 Ⅱ（ LeetCode 45 ）**

**题目：**

**示例：** 

**题解：** 

```java

```

#### **10、K次取反后最⼤化的数组和（ LeetCode 1005 ）**

**题目：**

**示例：** 

**题解：** 

```java

```

#### **11、根据身⾼重建队列 （ LeetCode 406 ）**

**题目：**

**示例：** 

**题解：** 

```java

```

#### **12、柠檬水找零（ LeetCode 860 ）**

**题目：** 在柠檬水摊上，每一杯柠檬水的售价为 5 美元。顾客排队购买你的产品，（按账单 bills 支付的顺序）一次购买一杯。每位顾客只买一杯柠檬水，然后向你付 5 美元、10 美元或 20 美元。你必须给每个顾客正确找零，也就是说净交易是每位顾客向你支付 5 美元。注意，一开始你手头没有任何零钱。给你一个整数数组 bills ，其中 bills[i] 是第 i 位顾客付的账。如果你能给每位顾客正确找零，返回 true ，否则返回 false 。

**示例：** 
    输入：bills = [5,5,5,10,20]
    输出：true
    解释：
    前 3 位顾客那里，我们按顺序收取 3 张 5 美元的钞票。
    第 4 位顾客那里，我们收取一张 10 美元的钞票，并返还 5 美元。
    第 5 位顾客那里，我们找还一张 10 美元的钞票和一张 5 美元的钞票。
    由于所有客户都得到了正确的找零，所以我们输出 true。

**题解：** 

```java
class Solution {
    public boolean lemonadeChange(int[] bills) {
        int five = 0, ten = 0;
        for (int bill : bills) {
            if (bill == 5) {
                five++;
            } else if (bill == 10) {
                if (five == 0) {
                    return false;
                }
                five--;
                ten++;
            } else {
                if (five > 0 && ten > 0) {
                    five--;
                    ten--;
                } else if (five >= 3) {
                    five -= 3;
                } else {
                    return false;
                }
            }
        }
        return true;
    }
}
```

#### **13、用最少数量的箭引爆气球（ LeetCode 452 ）**

**题目：** 有一些球形气球贴在一堵用 XY 平面表示的墙面上。墙面上的气球记录在整数数组 points ，其中points[i] = [xstart, xend] 表示水平直径在 xstart 和 xend之间的气球。你不知道气球的确切 y 坐标。一支弓箭可以沿着 x 轴从不同点 完全垂直 地射出。在坐标 x 处射出一支箭，若有一个气球的直径的开始和结束坐标为 xstart，xend， 且满足  xstart ≤ x ≤ xend，则该气球会被 引爆 。可以射出的弓箭的数量 没有限制 。 弓箭一旦被射出之后，可以无限地前进。给你一个数组 points ，返回引爆所有气球所必须射出的 最小 弓箭数 。

**示例：** 

    输入：points = [[10,16],[2,8],[1,6],[7,12]]
    输出：2
    解释：气球可以用2支箭来爆破:
    -在x = 6处射出箭，击破气球[2,8]和[1,6]。
    -在x = 11处发射箭，击破气球[10,16]和[7,12]。

**题解：** 

```java
class Solution {
    public int findMinArrowShots(int[][] points) {
        //按照右边界排序
        Arrays.sort(points, new Comparator<int[]>() {
            public int compare(int[] point1, int[] point2) {
                if (point1[1] > point2[1]) {
                    return 1;
                } else if (point1[1] < point2[1]) {
                    return -1;
                } else {
                    return 0;
                }
            }
        });
        int count = 1;
        int pre = points[0][1];
        for (int[] point: points) {
            if (point[0] > pre) {
                count++;
                pre = point[1];
            }
        }
        return count;
    }
}
```

#### **14、移掉 K 位数字（ LeetCode 402 ）**

**题目：** 给你一个以字符串表示的非负整数 num 和一个整数 k ，移除这个数中的 k 位数字，使得剩下的数字最小。请你以字符串形式返回这个最小的数字。

**示例：** 输入：num = "1432219", k = 3  输出："1219"

解释：移除掉三个数字 4, 3, 和 2 形成一个新的最小的数字 1219 。

**题解：** 

```java
class Solution {
    public String removeKdigits(String num, int k) {
        if (num.length() == k) return "0";

        StringBuilder sb = new StringBuilder();
        for (char c : num.toCharArray()) {
            while (k > 0 && sb.length() != 0 && sb.charAt(sb.length() - 1) > c) {
                sb.deleteCharAt(sb.length() - 1);
                k--;
            }
            if (c == '0' && sb.length() == 0) continue;
            sb.append(c);
        }

        String res = sb.substring(0, Math.max(sb.length() - k, 0));
        return res.length() == 0 ? "0" : res;
    }
}
```

#### **15、跳跃游戏（ LeetCode 55 ）**

**题目：** 给定一个非负整数数组 nums ，你最初位于数组的第一个下标 。数组中的每个元素代表你在该位置可以跳跃的最大长度。判断你是否能够到达最后一个下标。

**示例：** 
输入：nums = [2,3,1,1,4]  输出：true

解释：可以先跳 1 步，从下标 0 到达下标 1, 然后再从下标 1 跳 3 步到达最后一个下标。

**题解：** 

```java
public class Solution {
    public boolean canJump(int[] nums) {
        int n = nums.length;
        int rightmost = 0;
        for (int i = 0; i < n; ++i) {
            if (i <= rightmost) {
                rightmost = Math.max(rightmost, i + nums[i]);
                if (rightmost >= n - 1) {
                    return true;
                }
            }
        }
        return false;
    }
}
```

#### **16、摆动序列（ LeetCode 376 ）**

**题目：** 如果连续数字之间的差严格地在正数和负数之间交替，则数字序列称为 摆动序列 。第一个差（如果存在的话）可能是正数或负数。仅有一个元素或者含两个不等元素的序列也视作摆动序列。

    例如， [1, 7, 4, 9, 2, 5] 是一个 摆动序列 ，因为差值 (6, -3, 5, -7, 3) 是正负交替出现的。
    相反，[1, 4, 7, 2, 5] 和 [1, 7, 4, 5, 5] 不是摆动序列，第一个序列是因为它的前两个差值都是正数，第二个序列是因为它的最后一个差值为零。

子序列 可以通过从原始序列中删除一些（也可以不删除）元素来获得，剩下的元素保持其原始顺序。给你一个整数数组 nums ，返回 nums 中作为 摆动序列 的 最长子序列的长度 。

**示例：** 

    输入：nums = [1,7,4,9,2,5]
    输出：6
    解释：整个序列均为摆动序列，各元素之间的差值为 (6, -3, 5, -7, 3) 。

**题解：** 

```java
class Solution {
    public int wiggleMaxLength(int[] nums) {
        int n = nums.length;
        if (n < 2) {
            return n;
        }
        int prevdiff = nums[1] - nums[0];
        int ret = prevdiff != 0 ? 2 : 1;
        for (int i = 2; i < n; i++) {
            int diff = nums[i] - nums[i - 1];
            if ((diff > 0 && prevdiff <= 0) || (diff < 0 && prevdiff >= 0)) {
                ret++;
                prevdiff = diff;
            }
        }
        return ret;
    }
}
```

#### **17、买卖股票的最佳时机 II（ LeetCode 122 ）**

**题目：** 给你一个整数数组 prices ，其中 prices[i] 表示某支股票第 i 天的价格。在每一天，你可以决定是否购买和/或出售股票。你在任何时候 最多 只能持有 一股 股票。你也可以先购买，然后在 同一天 出售。返回 你能获得的 最大 利润 。

**示例：** 

    输入：prices = [7,1,5,3,6,4]
    输出：7
    解释：在第 2 天（股票价格 = 1）的时候买入，在第 3 天（股票价格 = 5）的时候卖出, 这笔交易所能获得利润 = 5 - 1 = 4 。
        随后，在第 4 天（股票价格 = 3）的时候买入，在第 5 天（股票价格 = 6）的时候卖出, 这笔交易所能获得利润 = 6 - 3 = 3 。
        总利润为 4 + 3 = 7 。

**题解：** 

```java
class Solution {
    public int maxProfit(int[] prices) {
        int ans = 0;
        int n = prices.length;
        for (int i = 1; i < n; ++i) {
            ans += Math.max(0, prices[i] - prices[i - 1]);
        }
        return ans;
    }
}
```

#### **18、三数之和（LeetCode 15）**

**题目：** 给你一个包含 n 个整数的数组 nums，判断 nums 中是否存在三个元素 a，b，c ，使得 a + b + c = 0 ？请你找出所有和为 0 且不重复的三元组。注意：答案中不可以包含重复的三元组。

**示例：** 

    输入：nums = [-1,0,1,2,-1,-4]
    输出：[[-1,-1,2],[-1,0,1]]

**题解：** 

```java
class Solution {

/**
 * 1. 排序
 * 2. 三指针
 * 3. 固定一个，遍历其他两个，将其变成两数之和，后两个数的和为前一个数的负数
 */
    public List<List<Integer>> threeSum(int[] nums) {
        List<List<Integer>> ans = new ArrayList<List<Integer>>();
        Arrays.sort(nums);
        int n = nums.length;
        // 遍历第一个
        for(int first = 0; first < n; first++){
            // 去重
            if(first > 0 && nums[first] == nums[first - 1]){
                continue;
            }
            // 第三个指针
            int third = n - 1;
            // 后两个数的和
            int target = -nums[first];
            // 遍历第二个
            for(int second = first + 1; second < n; second++){
                // 去重
                if(second > first + 1 && nums[second] == nums[second - 1]){
                    continue;
                }
                // 保证第三个指针在第二个右侧
                while(second < third && nums[second] + nums[third] > target) {
                    --third;
                }
                // 当二三相等时，就退出循环
                if(second == third){
                    break;
                }
                // 找到满足条件的数
                if((nums[second] + nums[third]) == target){
                    List<Integer> list = new ArrayList<>();
                    list.add(nums[first]);
                    list.add(nums[second]);
                    list.add(nums[third]);
                    ans.add(list);
                }
            }
        }
        return ans;
    }
}
```

#### **19、最接近三数之和（LeetCode 16）**

**题目：** 给你一个长度为 n 的整数数组 nums 和 一个目标值 target。请你从 nums 中选出三个整数，使它们的和与 target 最接近。返回这三个数的和。假定每组输入只存在恰好一个解。

**示例：** 输入：nums = [-1,2,1,-4], target = 1   输出：2

解释：与 target 最接近的和是 2 (-1 + 2 + 1 = 2) 。

**题解：** 

```java
public class Solution 
{
    public int threeSumClosest(int[] nums, int target) 
    {
        // sort the nums array
        Arrays.sort(nums);
        int closestSum = 0;
        int diff = Integer.MAX_VALUE;
        // iterate nums array, no need for the last two numbers because we need at least three numbers
        for(int i=0; i<nums.length-2; i++)
        {
            int left = i + 1;
            int right = nums.length - 1;
            // use two pointers to iterate rest array
            while(left < right)
            {
                int temp_sum = nums[i] + nums[left] + nums[right];
                int temp_diff = Math.abs(temp_sum - target);
                // if find a new closer sum, then update sum and diff
                if(temp_diff < diff)
                {
                    closestSum = temp_sum;
                    diff = temp_diff;
                }

                if(temp_sum < target) // meaning need larger sum
                    left++;
                else if(temp_sum > target) // meaning need smaller sum
                    right--;
                else // meaning temp_sum == target, this is the closestSum 
                    return temp_sum;
            }
        }

        return closestSum;
    }
}
```

#### **20、加油站（ LeetCode 134 ）**

**题目：** 在一条环路上有 n 个加油站，其中第 i 个加油站有汽油 gas[i] 升。你有一辆油箱容量无限的的汽车，从第 i 个加油站开往第 i+1 个加油站需要消耗汽油 cost[i] 升。你从其中的一个加油站出发，开始时油箱为空。给定两个整数数组 gas 和 cost ，如果你可以绕环路行驶一周，则返回出发时加油站的编号，否则返回 -1 。如果存在解，则 保证 它是 唯一 的。

**示例：** 

    输入: gas = [1,2,3,4,5], cost = [3,4,5,1,2]
    输出: 3
    解释:
    从 3 号加油站(索引为 3 处)出发，可获得 4 升汽油。此时油箱有 = 0 + 4 = 4 升汽油
    开往 4 号加油站，此时油箱有 4 - 1 + 5 = 8 升汽油
    开往 0 号加油站，此时油箱有 8 - 2 + 1 = 7 升汽油
    开往 1 号加油站，此时油箱有 7 - 3 + 2 = 6 升汽油
    开往 2 号加油站，此时油箱有 6 - 4 + 3 = 5 升汽油
    开往 3 号加油站，你需要消耗 5 升汽油，正好足够你返回到 3 号加油站。
    因此，3 可为起始索引。

**题解：** 

```java
class Solution {
    public int canCompleteCircuit(int[] gas, int[] cost) {
        int n = gas.length;
        int i = 0;
        while (i < n) {
            int sumOfGas = 0, sumOfCost = 0;
            int cnt = 0;
            while (cnt < n) {
                int j = (i + cnt) % n;
                sumOfGas += gas[j];
                sumOfCost += cost[j];
                if (sumOfCost > sumOfGas) {
                    break;
                }
                cnt++;
            }
            if (cnt == n) {
                return i;
            } else {
                i = i + cnt + 1;
            }
        }
        return -1;
    }
}
```

#### **21、分发糖果（ LeetCode 135 ）**

**题目：**

**示例：** 

**题解：** 

```java

```

#### **22、无重叠区间（ LeetCode 435 ）**

**题目：**

**示例：** 

**题解：** 

```java

```

#### **23、划分字母区间（ LeetCode 763 ）**

**题目：**

**示例：** 

**题解：** 

```java

```

#### **24、合并区间（ LeetCode 56 ）**

**题目：**

**示例：** 

**题解：** 

```java

```

#### **25、单调递增的数字（ LeetCode 738 ）**

**题目：**

**示例：** 

**题解：** 

```java

```

#### **26、监控二叉树（ LeetCode 968 ）**

**题目：**

**示例：** 

**题解：** 

```java

```

### **三、二叉树**

**注：**本章节比较重要，因此先总结一些重要的概念知识，再梳理算法题解。

#### **二叉树相关知识点**

1. 由n个节点构成的二叉树共有
   
   $$
   \frac {C_{2n}^n}{(n + 1)}种
   $$

2. 度为 0 的节点数 = 度为 2 的节点数 + 1、度的和 = 度为 1 的节点数 + 2倍度为 2 的节点数 或 = 总结点数 - 1、

3. 非空二叉树第  i  层上 最多有
   
   $$
   2^{i-1} 个节点
   $$

4. 高度为 h 的二叉树最多有
   
   $$
   2^{h} -1 个节点
   $$

5. 具有 n 个（n > 0）个节点的完全二叉树的高度为 
   
   $$
   \lceil log_2n \rceil + 1 或 \lfloor log_2(n + 1) \rfloor
   $$

6. 完全二叉树中度为 1 的数 = 0 或 = 1

7. 哈夫曼树中只有0 或 2 的节点

8. 对于有n个叶子节点的哈夫曼树，共有2 n - 1 个节点

9. 

#### **1、二叉树的前序遍历（ LeetCode 144 ）**

```java
/**
 * Definition for a binary tree node.
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode(int x) { val = x; }
 * }
 */
class Solution {
    public List<Integer> preorderTraversal(TreeNode root) {
        List<Integer> ans = new ArrayList<>();
        traversal(root,ans);
        return ans;
    }

    private void traversal(TreeNode root, List<Integer> ans) {
        if(root==null) return;
        ans.add(root.val);
        if(root.left!=null) traversal(root.left,ans);
        if(root.right!=null) traversal(root.right,ans); 
    }
}
```

面试必备：Morris遍历

```java
class Solution {
    public List<Integer> preorderTraversal(TreeNode root) {
        List<Integer> res = new ArrayList<Integer>();
        if (root == null) {
            return res;
        }

        TreeNode p1 = root, p2 = null;

        while (p1 != null) {
            p2 = p1.left;
            if (p2 != null) {
                while (p2.right != null && p2.right != p1) {
                    p2 = p2.right;
                }
                if (p2.right == null) {
                    res.add(p1.val);
                    p2.right = p1;
                    p1 = p1.left;
                    continue;
                } else {
                    p2.right = null;
                }
            } else {
                res.add(p1.val);
            }
            p1 = p1.right;
        }
        return res;
    }
}
```

#### **2、二叉树的中序遍历（ LeetCode 94 ）**

```java
/**
 * Definition for a binary tree node.
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode() {}
 *     TreeNode(int val) { this.val = val; }
 *     TreeNode(int val, TreeNode left, TreeNode right) {
 *         this.val = val;
 *         this.left = left;
 *         this.right = right;
 *     }
 * }
 */
class Solution {
    public List<Integer> inorderTraversal(TreeNode root) {
        List<Integer> ans = new ArrayList<>();
        traversal(root,ans);
        return ans;
    }
     private void traversal(TreeNode root, List<Integer> ans) {
        if(root==null) return;
        if(root.left!=null) traversal(root.left,ans);
        ans.add(root.val);
        if(root.right!=null) traversal(root.right,ans); 
    }
}
```

#### **3、二叉树的后序遍历（ LeetCode 145 ）**

```java
/**
 * Definition for a binary tree node.
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode() {}
 *     TreeNode(int val) { this.val = val; }
 *     TreeNode(int val, TreeNode left, TreeNode right) {
 *         this.val = val;
 *         this.left = left;
 *         this.right = right;
 *     }
 * }
 */
class Solution {
    public List<Integer> inorderTraversal(TreeNode root) {
        List<Integer> ans = new ArrayList<>();
        traversal(root,ans);
        return ans;
    }
     private void traversal(TreeNode root, List<Integer> ans) {
        if(root==null) return;
        if(root.left!=null) traversal(root.left,ans);
        if(root.right!=null) traversal(root.right,ans); 
        ans.add(root.val);
    }
}
```

#### **4、二叉树的层序遍历（ LeetCode 102 ）**

**题目：** 给你二叉树的根节点 root ，返回其节点值的 层序遍历 。 （即逐层地，从左到右访问所有节点）。

**示例：** 

    输入：root = [3,9,20,null,null,15,7]
    输出：[[3],[9,20],[15,7]]

**题解：** 

```java
// 广度优先
class Solution {
    public List<List<Integer>> levelOrder(TreeNode root) {
        List<List<Integer>> ret = new ArrayList<List<Integer>>();
        if (root == null) {
            return ret;
        }

        Queue<TreeNode> queue = new LinkedList<TreeNode>();
        queue.offer(root);
        while (!queue.isEmpty()) {
            List<Integer> level = new ArrayList<Integer>();
            int currentLevelSize = queue.size();
            for (int i = 1; i <= currentLevelSize; ++i) {
                TreeNode node = queue.poll();
                level.add(node.val);
                if (node.left != null) {
                    queue.offer(node.left);
                }
                if (node.right != null) {
                    queue.offer(node.right);
                }
            }
            ret.add(level);
        }

        return ret;
    }
}
```

#### **5、二叉树的锯齿形层序遍历（ LeetCode 103 ）**

**题目：** 给你二叉树的根节点 root ，返回其节点值的 锯齿形层序遍历 。（即先从左往右，再从右往左进行下一层遍历，以此类推，层与层之间交替进行）。

**示例：** 

    输入：root = [3,9,20,null,null,15,7]
    输出：[[3],[20,9],[15,7]]

**题解：** 

```java
// 广度优算法
class Solution {
    public List<List<Integer>> zigzagLevelOrder(TreeNode root) {
        List<List<Integer>> ans = new LinkedList<List<Integer>>();
        if (root == null) {
            return ans;
        }

        Queue<TreeNode> nodeQueue = new ArrayDeque<TreeNode>();
        nodeQueue.offer(root);
        boolean isOrderLeft = true;

        while (!nodeQueue.isEmpty()) {
            Deque<Integer> levelList = new LinkedList<Integer>();
            int size = nodeQueue.size();
            for (int i = 0; i < size; ++i) {
                TreeNode curNode = nodeQueue.poll();
                if (isOrderLeft) {
                    levelList.offerLast(curNode.val);
                } else {
                    levelList.offerFirst(curNode.val);
                }
                if (curNode.left != null) {
                    nodeQueue.offer(curNode.left);
                }
                if (curNode.right != null) {
                    nodeQueue.offer(curNode.right);
                }
            }
            ans.add(new LinkedList<Integer>(levelList));
            isOrderLeft = !isOrderLeft;
        }

        return ans;
    }
}
```

#### **6、从前序与中序遍历序列构造二叉树（ LeetCode 105 ）**

**题目：** 给定两个整数数组 preorder 和 inorder ，其中 preorder 是二叉树的先序遍历， inorder 是同一棵树的中序遍历，请构造二叉树并返回其根节点。

**示例：** 

    输入: preorder = [3,9,20,15,7], inorder = [9,3,15,20,7]
    输出: [3,9,20,null,null,15,7]

**题解：** 

```java
//方法一：递归
class Solution {
    private Map<Integer, Integer> indexMap;

    public TreeNode myBuildTree(int[] preorder, int[] inorder, int preorder_left, int preorder_right, int inorder_left, int inorder_right) {
        if (preorder_left > preorder_right) {
            return null;
        }

        // 前序遍历中的第一个节点就是根节点
        int preorder_root = preorder_left;
        // 在中序遍历中定位根节点
        int inorder_root = indexMap.get(preorder[preorder_root]);

        // 先把根节点建立出来
        TreeNode root = new TreeNode(preorder[preorder_root]);
        // 得到左子树中的节点数目
        int size_left_subtree = inorder_root - inorder_left;
        // 递归地构造左子树，并连接到根节点
        // 先序遍历中「从 左边界+1 开始的 size_left_subtree」个元素就对应了中序遍历中「从 左边界 开始到 根节点定位-1」的元素
        root.left = myBuildTree(preorder, inorder, preorder_left + 1, preorder_left + size_left_subtree, inorder_left, inorder_root - 1);
        // 递归地构造右子树，并连接到根节点
        // 先序遍历中「从 左边界+1+左子树节点数目 开始到 右边界」的元素就对应了中序遍历中「从 根节点定位+1 到 右边界」的元素
        root.right = myBuildTree(preorder, inorder, preorder_left + size_left_subtree + 1, preorder_right, inorder_root + 1, inorder_right);
        return root;
    }

    public TreeNode buildTree(int[] preorder, int[] inorder) {
        int n = preorder.length;
        // 构造哈希映射，帮助我们快速定位根节点
        indexMap = new HashMap<Integer, Integer>();
        for (int i = 0; i < n; i++) {
            indexMap.put(inorder[i], i);
        }
        return myBuildTree(preorder, inorder, 0, n - 1, 0, n - 1);
    }
}
// 方法二：迭代
class Solution {
    public TreeNode buildTree(int[] preorder, int[] inorder) {
        if (preorder == null || preorder.length == 0) {
            return null;
        }
        TreeNode root = new TreeNode(preorder[0]);
        Deque<TreeNode> stack = new LinkedList<TreeNode>();
        stack.push(root);
        int inorderIndex = 0;
        for (int i = 1; i < preorder.length; i++) {
            int preorderVal = preorder[i];
            TreeNode node = stack.peek();
            if (node.val != inorder[inorderIndex]) {
                node.left = new TreeNode(preorderVal);
                stack.push(node.left);
            } else {
                while (!stack.isEmpty() && stack.peek().val == inorder[inorderIndex]) {
                    node = stack.pop();
                    inorderIndex++;
                }
                node.right = new TreeNode(preorderVal);
                stack.push(node.right);
            }
        }
        return root;
    }
}
```

#### **7、路径总和 II（ LeetCode 113 ）**

**题目：** 给你二叉树的根节点 root 和一个整数目标和 targetSum ，找出所有 从根节点到叶子节点 路径总和等于给定目标和的路径。叶子节点 是指没有子节点的节点。

**示例：** 

    输入：root = [5,4,8,11,null,13,4,7,2,null,null,5,1], targetSum = 22
    输出：[[5,4,11,2],[5,8,4,5]]

**题解：** 

```java
// 深度优先
class Solution {
    List<List<Integer>> ret = new LinkedList<List<Integer>>();
    Deque<Integer> path = new LinkedList<Integer>();

    public List<List<Integer>> pathSum(TreeNode root, int targetSum) {
        dfs(root, targetSum);
        return ret;
    }

    public void dfs(TreeNode root, int targetSum) {
        if (root == null) {
            return;
        }
        path.offerLast(root.val);
        targetSum -= root.val;
        if (root.left == null && root.right == null && targetSum == 0) {
            ret.add(new LinkedList<Integer>(path));
        }
        dfs(root.left, targetSum);
        dfs(root.right, targetSum);
        path.pollLast();
    }
}

// 广度优先
class Solution {
    List<List<Integer>> ret = new LinkedList<List<Integer>>();
    Map<TreeNode, TreeNode> map = new HashMap<TreeNode, TreeNode>();

    public List<List<Integer>> pathSum(TreeNode root, int targetSum) {
        if (root == null) {
            return ret;
        }

        Queue<TreeNode> queueNode = new LinkedList<TreeNode>();
        Queue<Integer> queueSum = new LinkedList<Integer>();
        queueNode.offer(root);
        queueSum.offer(0);

        while (!queueNode.isEmpty()) {
            TreeNode node = queueNode.poll();
            int rec = queueSum.poll() + node.val;

            if (node.left == null && node.right == null) {
                if (rec == targetSum) {
                    getPath(node);
                }
            } else {
                if (node.left != null) {
                    map.put(node.left, node);
                    queueNode.offer(node.left);
                    queueSum.offer(rec);
                }
                if (node.right != null) {
                    map.put(node.right, node);
                    queueNode.offer(node.right);
                    queueSum.offer(rec);
                }
            }
        }

        return ret;
    }

    public void getPath(TreeNode node) {
        List<Integer> temp = new LinkedList<Integer>();
        while (node != null) {
            temp.add(node.val);
            node = map.get(node);
        }
        Collections.reverse(temp);
        ret.add(new LinkedList<Integer>(temp));
    }
}
```

#### **8、二叉树的最近公共祖先（ LeetCode 236 ）**

**题目：** 给定一个二叉树, 找到该树中两个指定节点的最近公共祖先。百度百科中最近公共祖先的定义为：“对于有根树 T 的两个节点 p、q，最近公共祖先表示为一个节点 x，满足 x 是 p、q 的祖先且 x 的深度尽可能大（一个节点也可以是它自己的祖先）。

**示例：** 

    输入：root = [3,5,1,6,2,0,8,null,null,7,4], p = 5, q = 1
    输出：3
    解释：节点 5 和节点 1 的最近公共祖先是节点 3 。

**题解：** 

```java
// 方法一：递归方法
class Solution {

    private TreeNode ans;

    public Solution() {
        this.ans = null;
    }

    private boolean dfs(TreeNode root, TreeNode p, TreeNode q) {
        if (root == null) return false;
        boolean lson = dfs(root.left, p, q);
        boolean rson = dfs(root.right, p, q);
        if ((lson && rson) || ((root.val == p.val || root.val == q.val) && (lson || rson))) {
            ans = root;
        } 
        return lson || rson || (root.val == p.val || root.val == q.val);
    }

    public TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
        this.dfs(root, p, q);
        return this.ans;
    }
}

// 方法二：存储父节点
class Solution {
    Map<Integer, TreeNode> parent = new HashMap<Integer, TreeNode>();
    Set<Integer> visited = new HashSet<Integer>();

    public void dfs(TreeNode root) {
        if (root.left != null) {
            parent.put(root.left.val, root);
            dfs(root.left);
        }
        if (root.right != null) {
            parent.put(root.right.val, root);
            dfs(root.right);
        }
    }

    public TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
        dfs(root);
        while (p != null) {
            visited.add(p.val);
            p = parent.get(p.val);
        }
        while (q != null) {
            if (visited.contains(q.val)) {
                return q;
            }
            q = parent.get(q.val);
        }
        return null;
    }
}
```

#### **9、二叉树的右视图（ LeetCode 199 ）**

**题目：** 给定一个二叉树的 根节点 root，想象自己站在它的右侧，按照从顶部到底部的顺序，返回从右侧所能看到的节点值。

**示例：** 

    输入: [1,2,3,null,5,null,4]
    输出: [1,3,4]

**题解：** 

```java
// 深度优先搜索
class Solution {
    public List<Integer> rightSideView(TreeNode root) {
        Map<Integer, Integer> rightmostValueAtDepth = new HashMap<Integer, Integer>();
        int max_depth = -1;

        Deque<TreeNode> nodeStack = new ArrayDeque<TreeNode>();
        Deque<Integer> depthStack = new ArrayDeque<Integer>();
        nodeStack.push(root);
        depthStack.push(0);

        while (!nodeStack.isEmpty()) {
            TreeNode node = nodeStack.pop();
            int depth = depthStack.pop();

            if (node != null) {
                // 维护二叉树的最大深度
                max_depth = Math.max(max_depth, depth);

                // 如果不存在对应深度的节点我们才插入
                if (!rightmostValueAtDepth.containsKey(depth)) {
                    rightmostValueAtDepth.put(depth, node.val);
                }

                nodeStack.push(node.left);
                nodeStack.push(node.right);
                depthStack.push(depth + 1);
                depthStack.push(depth + 1);
            }
        }

        List<Integer> rightView = new ArrayList<Integer>();
        for (int depth = 0; depth <= max_depth; depth++) {
            rightView.add(rightmostValueAtDepth.get(depth));
        }

        return rightView;
    }
}

// 广度优先搜索
class Solution {
    public List<Integer> rightSideView(TreeNode root) {
        Map<Integer, Integer> rightmostValueAtDepth = new HashMap<Integer, Integer>();
        int max_depth = -1;

        Queue<TreeNode> nodeQueue = new ArrayDeque<TreeNode>();
        Queue<Integer> depthQueue = new ArrayDeque<Integer>();
        nodeQueue.add(root);
        depthQueue.add(0);

        while (!nodeQueue.isEmpty()) {
            TreeNode node = nodeQueue.remove();
            int depth = depthQueue.remove();

            if (node != null) {
                // 维护二叉树的最大深度
                max_depth = Math.max(max_depth, depth);

                // 由于每一层最后一个访问到的节点才是我们要的答案，因此不断更新对应深度的信息即可
                rightmostValueAtDepth.put(depth, node.val);

                nodeQueue.add(node.left);
                nodeQueue.add(node.right);
                depthQueue.add(depth + 1);
                depthQueue.add(depth + 1);
            }
        }

        List<Integer> rightView = new ArrayList<Integer>();
        for (int depth = 0; depth <= max_depth; depth++) {
            rightView.add(rightmostValueAtDepth.get(depth));
        }

        return rightView;
    }
}
```

#### **10、二叉树展开为链表（ LeetCode 114 ）**

**题目：** 给你二叉树的根结点 root ，请你将它展开为一个单链表：展开后的单链表应该同样使用 TreeNode ，其中 right 子指针指向链表中下一个结点，而左子指针始终为 null 。展开后的单链表应该与二叉树 先序遍历 顺序相同。

**示例：** 

    输入：root = [1,2,5,3,4,null,6]
    输出：[1,null,2,null,3,null,4,null,5,null,6]

**题解：** 

```java
// 递归
class Solution {
    public void flatten(TreeNode root) {
        if (root == null) return;
        flatten(root.left);
        TreeNode right = root.right;
        root.right = root.left;
        root.left = null;
        while (root.right != null) {
            root = root.right;
        }
        flatten(right);
        root.right = right;
    }
}

// 迭代
class Solution {
    public void flatten(TreeNode root) {
        List<TreeNode> list = new ArrayList<TreeNode>();
        Deque<TreeNode> stack = new LinkedList<TreeNode>();
        TreeNode node = root;
        while (node != null || !stack.isEmpty()) {
            while (node != null) {
                list.add(node);
                stack.push(node);
                node = node.left;
            }
            node = stack.pop();
            node = node.right;
        }
        int size = list.size();
        for (int i = 1; i < size; i++) {
            TreeNode prev = list.get(i - 1), curr = list.get(i);
            prev.left = null;
            prev.right = curr;
        }
    }
}

// 寻找前驱结点
class Solution {
    public void flatten(TreeNode root) {
        TreeNode curr = root;
        while (curr != null) {
            if (curr.left != null) {
                TreeNode next = curr.left;
                TreeNode predecessor = next;
                while (predecessor.right != null) {
                    predecessor = predecessor.right;
                }
                predecessor.right = curr.right;
                curr.left = null;
                curr.right = next;
            }
            curr = curr.right;
        }
    }
}
```

#### **11、将有序数组转换为二叉搜索树（ LeetCode 108 ）**

**题目：** 给你一个整数数组 nums ，其中元素已经按 升序 排列，请你将其转换为一棵 高度平衡 二叉搜索树。高度平衡 二叉树是一棵满足「每个节点的左右两个子树的高度差的绝对值不超过 1 」的二叉树。

**示例：** 

    输入：nums = [-10,-3,0,5,9]
    输出：[0,-3,9,-10,null,5]
    解释：[0,-10,5,null,-3,null,9] 也将被视为正确答案

**题解：** 

```java
// 方法一：中序遍历，总是选择中间位置左边的数字作为根节点
class Solution {
    public TreeNode sortedArrayToBST(int[] nums) {
        return helper(nums, 0, nums.length - 1);
    }

    public TreeNode helper(int[] nums, int left, int right) {
        if (left > right) {
            return null;
        }

        // 总是选择中间位置左边的数字作为根节点
        int mid = (left + right) / 2;

        TreeNode root = new TreeNode(nums[mid]);
        root.left = helper(nums, left, mid - 1);
        root.right = helper(nums, mid + 1, right);
        return root;
    }
}

// 方法二：中序遍历，总是选择中间位置右边的数字作为根节点
class Solution {
    public TreeNode sortedArrayToBST(int[] nums) {
        return helper(nums, 0, nums.length - 1);
    }

    public TreeNode helper(int[] nums, int left, int right) {
        if (left > right) {
            return null;
        }

        // 总是选择中间位置右边的数字作为根节点
        int mid = (left + right + 1) / 2;

        TreeNode root = new TreeNode(nums[mid]);
        root.left = helper(nums, left, mid - 1);
        root.right = helper(nums, mid + 1, right);
        return root;
    }
}

// 方法三：中序遍历，选择任意一个中间位置数字作为根节点
class Solution {
    Random rand = new Random();

    public TreeNode sortedArrayToBST(int[] nums) {
        return helper(nums, 0, nums.length - 1);
    }

    public TreeNode helper(int[] nums, int left, int right) {
        if (left > right) {
            return null;
        }

        // 选择任意一个中间位置数字作为根节点
        int mid = (left + right + rand.nextInt(2)) / 2;

        TreeNode root = new TreeNode(nums[mid]);
        root.left = helper(nums, left, mid - 1);
        root.right = helper(nums, mid + 1, right);
        return root;
    }
}
```

#### **12、把二叉搜索树转换为累加树（ LeetCode 538 ）**

**题目：** 给出二叉 搜索 树的根节点，该树的节点值各不相同，请你将其转换为累加树（Greater Sum Tree），使每个节点 node 的新值等于原树中大于或等于 node.val 的值之和。提醒一下，二叉搜索树满足下列约束条件：

    节点的左子树仅包含键 小于 节点键的节点。
    节点的右子树仅包含键 大于 节点键的节点。
    左右子树也必须是二叉搜索树。

**示例：** 

    输入：[4,1,6,0,2,5,7,null,null,null,3,null,null,null,8]
    输出：[30,36,21,36,35,26,15,null,null,null,33,null,null,null,8]

**题解：** 

```java
// 方法一：反序中序遍历
class Solution {
    int sum = 0;

    public TreeNode convertBST(TreeNode root) {
        if (root != null) {
            convertBST(root.right);
            sum += root.val;
            root.val = sum;
            convertBST(root.left);
        }
        return root;
    }
}

// Morris 遍历
class Solution {
    public TreeNode convertBST(TreeNode root) {
        int sum = 0;
        TreeNode node = root;

        while (node != null) {
            if (node.right == null) {
                sum += node.val;
                node.val = sum;
                node = node.left;
            } else {
                TreeNode succ = getSuccessor(node);
                if (succ.left == null) {
                    succ.left = node;
                    node = node.right;
                } else {
                    succ.left = null;
                    sum += node.val;
                    node.val = sum;
                    node = node.left;
                }
            }
        }

        return root;
    }

    public TreeNode getSuccessor(TreeNode node) {
        TreeNode succ = node.right;
        while (succ.left != null && succ.left != node) {
            succ = succ.left;
        }
        return succ;
    }
}
```

#### **13、删除二叉搜索树中的节点（ LeetCode 450 ）**

**题目：** 给定一个二叉搜索树的根节点 root 和一个值 key，删除二叉搜索树中的 key 对应的节点，并保证二叉搜索树的性质不变。返回二叉搜索树（有可能被更新）的根节点的引用。一般来说，删除节点可分为两个步骤：

    首先找到需要删除的节点；
    如果找到了，删除它。

**示例：** 

    输入：root = [5,3,6,2,4,null,7], key = 3
    输出：[5,4,6,2,null,null,7]
    解释：给定需要删除的节点值是 3，所以我们首先找到 3 这个节点，然后删除它。
    一个正确的答案是 [5,4,6,2,null,null,7]。
    另一个正确答案是 [5,2,6,null,4,null,7]。

**题解：** 

```java
// 递归
class Solution {
    public TreeNode deleteNode(TreeNode root, int key) {
        if (root == null) {
            return null;
        }
        if (root.val > key) {
            root.left = deleteNode(root.left, key);
            return root;
        }
        if (root.val < key) {
            root.right = deleteNode(root.right, key);
            return root;
        }
        if (root.val == key) {
            if (root.left == null && root.right == null) {
                return null;
            }
            if (root.right == null) {
                return root.left;
            }
            if (root.left == null) {
                return root.right;
            }
            TreeNode successor = root.right;
            while (successor.left != null) {
                successor = successor.left;
            }
            root.right = deleteNode(root.right, successor.val);
            successor.right = root.right;
            successor.left = root.left;
            return successor;
        }
        return root;
    }
}

// 迭代
class Solution {
    public TreeNode deleteNode(TreeNode root, int key) {
        TreeNode cur = root, curParent = null;
        while (cur != null && cur.val != key) {
            curParent = cur;
            if (cur.val > key) {
                cur = cur.left;
            } else {
                cur = cur.right;
            }
        }
        if (cur == null) {
            return root;
        }
        if (cur.left == null && cur.right == null) {
            cur = null;
        } else if (cur.right == null) {
            cur = cur.left;
        } else if (cur.left == null) {
            cur = cur.right;
        } else {
            TreeNode successor = cur.right, successorParent = cur;
            while (successor.left != null) {
                successorParent = successor;
                successor = successor.left;
            }
            if (successorParent.val == cur.val) {
                successorParent.right = successor.right;
            } else {
                successorParent.left = successor.right;
            }
            successor.right = cur.right;
            successor.left = cur.left;
            cur = successor;
        }
        if (curParent == null) {
            return cur;
        } else {
            if (curParent.left != null && curParent.left.val == key) {
                curParent.left = cur;
            } else {
                curParent.right = cur;
            }
            return root;
        }
    }
}
```

#### **14、二叉树的序列化与反序列化（ LeetCode 297 ）**

**题目：** 序列化是将一个数据结构或者对象转换为连续的比特位的操作，进而可以将转换后的数据存储在一个文件或者内存中，同时也可以通过网络传输到另一个计算机环境，采取相反方式重构得到原数据。请设计一个算法来实现二叉树的序列化与反序列化。这里不限定你的序列 / 反序列化算法执行逻辑，你只需要保证一个二叉树可以被序列化为一个字符串并且将这个字符串反序列化为原始的树结构。提示: 输入输出格式与 LeetCode 目前使用的方式一致，详情请参阅 LeetCode 序列化二叉树的格式。你并非必须采取这种方式，你也可以采用其他的方法解决这个问题。

**示例：** 

    输入：root = [1,2,3,null,null,4,5]
    输出：[1,2,3,null,null,4,5]

**题解：** 

```java
// 方法一：深度优先搜索
public class Codec {
    public String serialize(TreeNode root) {
        return rserialize(root, "");
    }

    public TreeNode deserialize(String data) {
        String[] dataArray = data.split(",");
        List<String> dataList = new LinkedList<String>(Arrays.asList(dataArray));
        return rdeserialize(dataList);
    }

    public String rserialize(TreeNode root, String str) {
        if (root == null) {
            str += "None,";
        } else {
            str += str.valueOf(root.val) + ",";
            str = rserialize(root.left, str);
            str = rserialize(root.right, str);
        }
        return str;
    }

    public TreeNode rdeserialize(List<String> dataList) {
        if (dataList.get(0).equals("None")) {
            dataList.remove(0);
            return null;
        }

        TreeNode root = new TreeNode(Integer.valueOf(dataList.get(0)));
        dataList.remove(0);
        root.left = rdeserialize(dataList);
        root.right = rdeserialize(dataList);

        return root;
    }
}
// 方法二：括号表示编码 + 递归下降解码
public class Codec {
    public String serialize(TreeNode root) {
        if (root == null) {
            return "X";
        }
        String left = "(" + serialize(root.left) + ")";
        String right = "(" + serialize(root.right) + ")";
        return left + root.val + right;
    }

    public TreeNode deserialize(String data) {
        int[] ptr = {0};
        return parse(data, ptr);
    }

    public TreeNode parse(String data, int[] ptr) {
        if (data.charAt(ptr[0]) == 'X') {
            ++ptr[0];
            return null;
        }
        TreeNode cur = new TreeNode(0);
        cur.left = parseSubtree(data, ptr);
        cur.val = parseInt(data, ptr);
        cur.right = parseSubtree(data, ptr);
        return cur;
    }

    public TreeNode parseSubtree(String data, int[] ptr) {
        ++ptr[0]; // 跳过左括号
        TreeNode subtree = parse(data, ptr);
        ++ptr[0]; // 跳过右括号
        return subtree;
    }

    public int parseInt(String data, int[] ptr) {
        int x = 0, sgn = 1;
        if (!Character.isDigit(data.charAt(ptr[0]))) {
            sgn = -1;
            ++ptr[0];
        }
        while (Character.isDigit(data.charAt(ptr[0]))) {
            x = x * 10 + data.charAt(ptr[0]++) - '0';
        }
        return x * sgn;
    }
}
```

#### **15、完全二叉树的节点个数（ LeetCode 222 ）**

**题目：** 给你一棵 完全二叉树 的根节点 root ，求出该树的节点个数。

完全二叉树 的定义如下：在完全二叉树中，除了最底层节点可能没填满外，其余每层节点数都达到最大值，并且最下面一层的节点都集中在该层最左边的若干位置。若最底层为第 h 层，则该层包含 1~ 2h 个节点。

**示例：** 

    输入：root = [1,2,3,4,5,6]
    输出：6

**题解：** 

```java
// 二分查找 + 位运算
class Solution {
    public int countNodes(TreeNode root) {
        if (root == null) {
            return 0;
        }
        int level = 0;
        TreeNode node = root;
        while (node.left != null) {
            level++;
            node = node.left;
        }
        int low = 1 << level, high = (1 << (level + 1)) - 1;
        while (low < high) {
            int mid = (high - low + 1) / 2 + low;
            if (exists(root, level, mid)) {
                low = mid;
            } else {
                high = mid - 1;
            }
        }
        return low;
    }

    public boolean exists(TreeNode root, int level, int k) {
        int bits = 1 << (level - 1);
        TreeNode node = root;
        while (node != null && bits > 0) {
            if ((bits & k) == 0) {
                node = node.left;
            } else {
                node = node.right;
            }
            bits >>= 1;
        }
        return node != null;
    }
}
```

#### **16、二叉树的最大深度（ LeetCode 104 ）**

**题目：** 给定一个二叉树，找出其最大深度。二叉树的深度为根节点到最远叶子节点的最长路径上的节点数。说明: 叶子节点是指没有子节点的节点。

**题解：** 

```java
// 方法一：深度优先搜索
class Solution {
    public int maxDepth(TreeNode root) {
        if (root == null) {
            return 0;
        } else {
            int leftHeight = maxDepth(root.left);
            int rightHeight = maxDepth(root.right);
            return Math.max(leftHeight, rightHeight) + 1;
        }
    }
}

// 广度优先搜索
class Solution {
    public int maxDepth(TreeNode root) {
        if (root == null) {
            return 0;
        }
        Queue<TreeNode> queue = new LinkedList<TreeNode>();
        queue.offer(root);
        int ans = 0;
        while (!queue.isEmpty()) {
            int size = queue.size();
            while (size > 0) {
                TreeNode node = queue.poll();
                if (node.left != null) {
                    queue.offer(node.left);
                }
                if (node.right != null) {
                    queue.offer(node.right);
                }
                size--;
            }
            ans++;
        }
        return ans;
    }
}
```

#### **17、二叉树的最小深度（ LeetCode 111 ）**

**题目：** 给定一个二叉树，找出其最小深度。最小深度是从根节点到最近叶子节点的最短路径上的节点数量。说明：叶子节点是指没有子节点的节点。

**示例：** 

    输入：root = [3,9,20,null,null,15,7]
    输出：2

**题解：** 

```java
// 方法一：深度优先搜索
class Solution {
    public int minDepth(TreeNode root) {
        if (root == null) {
            return 0;
        }

        if (root.left == null && root.right == null) {
            return 1;
        }

        int min_depth = Integer.MAX_VALUE;
        if (root.left != null) {
            min_depth = Math.min(minDepth(root.left), min_depth);
        }
        if (root.right != null) {
            min_depth = Math.min(minDepth(root.right), min_depth);
        }

        return min_depth + 1;
    }
}

// 方法二：广度优先搜索
class Solution {
    class QueueNode {
        TreeNode node;
        int depth;

        public QueueNode(TreeNode node, int depth) {
            this.node = node;
            this.depth = depth;
        }
    }

    public int minDepth(TreeNode root) {
        if (root == null) {
            return 0;
        }

        Queue<QueueNode> queue = new LinkedList<QueueNode>();
        queue.offer(new QueueNode(root, 1));
        while (!queue.isEmpty()) {
            QueueNode nodeDepth = queue.poll();
            TreeNode node = nodeDepth.node;
            int depth = nodeDepth.depth;
            if (node.left == null && node.right == null) {
                return depth;
            }
            if (node.left != null) {
                queue.offer(new QueueNode(node.left, depth + 1));
            }
            if (node.right != null) {
                queue.offer(new QueueNode(node.right, depth + 1));
            }
        }

        return 0;
    }
}
```

#### **18、二叉树的所有路径（ LeetCode 257 ）**

**题目：** 给你一个二叉树的根节点 root ，按 任意顺序 ，返回所有从根节点到叶子节点的路径。叶子节点 是指没有子节点的节点。

**示例：** 

    输入：root = [1,2,3,null,5]
    输出：["1->2->5","1->3"]

**题解：** 

```java
// 方法一：深度优先搜索
class Solution {
    public List<String> binaryTreePaths(TreeNode root) {
        List<String> paths = new ArrayList<String>();
        constructPaths(root, "", paths);
        return paths;
    }

    public void constructPaths(TreeNode root, String path, List<String> paths) {
        if (root != null) {
            StringBuffer pathSB = new StringBuffer(path);
            pathSB.append(Integer.toString(root.val));
            if (root.left == null && root.right == null) {  // 当前节点是叶子节点
                paths.add(pathSB.toString());  // 把路径加入到答案中
            } else {
                pathSB.append("->");  // 当前节点不是叶子节点，继续递归遍历
                constructPaths(root.left, pathSB.toString(), paths);
                constructPaths(root.right, pathSB.toString(), paths);
            }
        }
    }
}

// 广度优先算法
class Solution {
    public List<String> binaryTreePaths(TreeNode root) {
        List<String> paths = new ArrayList<String>();
        if (root == null) {
            return paths;
        }
        Queue<TreeNode> nodeQueue = new LinkedList<TreeNode>();
        Queue<String> pathQueue = new LinkedList<String>();

        nodeQueue.offer(root);
        pathQueue.offer(Integer.toString(root.val));

        while (!nodeQueue.isEmpty()) {
            TreeNode node = nodeQueue.poll(); 
            String path = pathQueue.poll();

            if (node.left == null && node.right == null) {
                paths.add(path);
            } else {
                if (node.left != null) {
                    nodeQueue.offer(node.left);
                    pathQueue.offer(new StringBuffer(path).append("->").append(node.left.val).toString());
                }

                if (node.right != null) {
                    nodeQueue.offer(node.right);
                    pathQueue.offer(new StringBuffer(path).append("->").append(node.right.val).toString());
                }
            }
        }
        return paths;
    }
}
```

#### **19、平衡二叉树（ LeetCode 110 ）**

**题目：** 给定一个二叉树，判断它是否是高度平衡的二叉树。本题中，一棵高度平衡二叉树定义为：一个二叉树每个节点 的左右两个子树的高度差的绝对值不超过 1 。

**示例：** 

    输入：root = [3,9,20,null,null,15,7]
    输出：true

**题解：** 

```java
// 方法一：自顶向下的递归
class Solution {
    public boolean isBalanced(TreeNode root) {
        if (root == null) {
            return true;
        } else {
            return Math.abs(height(root.left) - height(root.right)) <= 1 && isBalanced(root.left) && isBalanced(root.right);
        }
    }

    public int height(TreeNode root) {
        if (root == null) {
            return 0;
        } else {
            return Math.max(height(root.left), height(root.right)) + 1;
        }
    }
}
// 方法二：自底向上的递归
class Solution {
    public boolean isBalanced(TreeNode root) {
        return height(root) >= 0;
    }

    public int height(TreeNode root) {
        if (root == null) {
            return 0;
        }
        int leftHeight = height(root.left);
        int rightHeight = height(root.right);
        if (leftHeight == -1 || rightHeight == -1 || Math.abs(leftHeight - rightHeight) > 1) {
            return -1;
        } else {
            return Math.max(leftHeight, rightHeight) + 1;
        }
    }
}
```

#### **20、左叶子之和（ LeetCode 404 ）**

**题目：** 给定二叉树的根节点 root ，返回所有左叶子之和。

**示例：** 

    输入: root = [3,9,20,null,null,15,7] 
    输出: 24 
    解释: 在这个二叉树中，有两个左叶子，分别是 9 和 15，所以返回 24

**题解：** 

```java
// 方法一：深度优先搜索
class Solution {
    public int sumOfLeftLeaves(TreeNode root) {
        return root != null ? dfs(root) : 0;
    }

    public int dfs(TreeNode node) {
        int ans = 0;
        if (node.left != null) {
            ans += isLeafNode(node.left) ? node.left.val : dfs(node.left);
        }
        if (node.right != null && !isLeafNode(node.right)) {
            ans += dfs(node.right);
        }
        return ans;
    }

    public boolean isLeafNode(TreeNode node) {
        return node.left == null && node.right == null;
    }
}

// 方法二：广度优先搜索
class Solution {
    public int sumOfLeftLeaves(TreeNode root) {
        if (root == null) {
            return 0;
        }

        Queue<TreeNode> queue = new LinkedList<TreeNode>();
        queue.offer(root);
        int ans = 0;
        while (!queue.isEmpty()) {
            TreeNode node = queue.poll();
            if (node.left != null) {
                if (isLeafNode(node.left)) {
                    ans += node.left.val;
                } else {
                    queue.offer(node.left);
                }
            }
            if (node.right != null) {
                if (!isLeafNode(node.right)) {
                    queue.offer(node.right);
                }
            }
        }
        return ans;
    }

    public boolean isLeafNode(TreeNode node) {
        return node.left == null && node.right == null;
    }
}
```

#### **21、找树左下角的值（ LeetCode 513 ）**

**题目：** 给定一个二叉树的 根节点 root，请找出该二叉树的 最底层 最左边 节点的值。假设二叉树中至少有一个节点。

**示例：** 

    输入: root = [2,1,3]
    输出: 1

**题解：** 

```java
// 方法一：深度优先搜索
class Solution {
    int curVal = 0;
    int curHeight = 0;

    public int findBottomLeftValue(TreeNode root) {
        int curHeight = 0;
        dfs(root, 0);
        return curVal;
    }

    public void dfs(TreeNode root, int height) {
        if (root == null) {
            return;
        }
        height++;
        dfs(root.left, height);
        dfs(root.right, height);
        if (height > curHeight) {
            curHeight = height;
            curVal = root.val;
        }
    }
}

// 方法二：广度优先搜索
class Solution {
    public int findBottomLeftValue(TreeNode root) {
        int ret = 0;
        Queue<TreeNode> queue = new ArrayDeque<TreeNode>();
        queue.offer(root);
        while (!queue.isEmpty()) {
            TreeNode p = queue.poll();
            if (p.right != null) {
                queue.offer(p.right);
            }
            if (p.left != null) {
                queue.offer(p.left);
            }
            ret = p.val;
        }
        return ret;
    }
}
```

#### **22、修剪二叉搜索树（ LeetCode 669 ）**

**题目：** 给你二叉搜索树的根节点 root ，同时给定最小边界low 和最大边界 high。通过修剪二叉搜索树，使得所有节点的值在[low, high]中。修剪树 不应该 改变保留在树中的元素的相对结构 (即，如果没有被移除，原有的父代子代关系都应当保留)。 可以证明，存在 唯一的答案 。所以结果应当返回修剪好的二叉搜索树的新的根节点。注意，根节点可能会根据给定的边界发生改变。

**示例：** 

    输入：root = [1,0,2], low = 1, high = 2
    输出：[1,null,2]

**题解：** 

```java
// 递归
class Solution {
    public TreeNode trimBST(TreeNode root, int L, int R) {
        if (root == null) return root;
        if (root.val > R) return trimBST(root.left, L, R);
        if (root.val < L) return trimBST(root.right, L, R);

        root.left = trimBST(root.left, L, R);
        root.right = trimBST(root.right, L, R);
        return root;
    }
}
```

#### **23、二叉搜索树的最近公共祖先（ LeetCode 235 ）**

**题目：** 给定一个二叉搜索树, 找到该树中两个指定节点的最近公共祖先。百度百科中最近公共祖先的定义为：“对于有根树 T 的两个结点 p、q，最近公共祖先表示为一个结点 x，满足 x 是 p、q 的祖先且 x 的深度尽可能大（一个节点也可以是它自己的祖先）。”例如，给定如下二叉搜索树:  root = [6,2,8,0,4,7,9,null,null,3,5]

**示例：** 

    输入: root = [6,2,8,0,4,7,9,null,null,3,5], p = 2, q = 8
    输出: 6 
    解释: 节点 2 和节点 8 的最近公共祖先是 6。

**题解：** 

```java
// 方法一：两次遍历
class Solution {
    public TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
        List<TreeNode> path_p = getPath(root, p);
        List<TreeNode> path_q = getPath(root, q);
        TreeNode ancestor = null;
        for (int i = 0; i < path_p.size() && i < path_q.size(); ++i) {
            if (path_p.get(i) == path_q.get(i)) {
                ancestor = path_p.get(i);
            } else {
                break;
            }
        }
        return ancestor;
    }

    public List<TreeNode> getPath(TreeNode root, TreeNode target) {
        List<TreeNode> path = new ArrayList<TreeNode>();
        TreeNode node = root;
        while (node != target) {
            path.add(node);
            if (target.val < node.val) {
                node = node.left;
            } else {
                node = node.right;
            }
        }
        path.add(node);
        return path;
    }
}
// 方法二：一次遍历
class Solution {
    public TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
        TreeNode ancestor = root;
        while (true) {
            if (p.val < ancestor.val && q.val < ancestor.val) {
                ancestor = ancestor.left;
            } else if (p.val > ancestor.val && q.val > ancestor.val) {
                ancestor = ancestor.right;
            } else {
                break;
            }
        }
        return ancestor;
    }
}
```

#### **24、二叉搜索树的最小绝对差（ LeetCode 530 ）**

**题目：** 给你一个二叉搜索树的根节点 root ，返回 树中任意两不同节点值之间的最小差值 。差值是一个正数，其数值等于两值之差的绝对值。

**示例：** 

    输入：root = [4,2,6,1,3]
    输出：1

**题解：** 

```java
// 方法一：中序遍历
class Solution {
    int pre;
    int ans;

    public int getMinimumDifference(TreeNode root) {
        ans = Integer.MAX_VALUE;
        pre = -1;
        dfs(root);
        return ans;
    }

    public void dfs(TreeNode root) {
        if (root == null) {
            return;
        }
        dfs(root.left);
        if (pre == -1) {
            pre = root.val;
        } else {
            ans = Math.min(ans, root.val - pre);
            pre = root.val;
        }
        dfs(root.right);
    }
}
```

#### **25、最大二叉树（ LeetCode 654 ）**

**题目：** 
给定一个不重复的整数数组 nums 。 最大二叉树 可以用下面的算法从 nums 递归地构建:

    创建一个根节点，其值为 nums 中的最大值。
    递归地在最大值 左边 的 子数组前缀上 构建左子树。
    递归地在最大值 右边 的 子数组后缀上 构建右子树。

返回 nums 构建的 最大二叉树 。

**题解：** 

```java
// 方法一：递归
public class Solution {
    public TreeNode constructMaximumBinaryTree(int[] nums) {
        return construct(nums, 0, nums.length);
    }
    public TreeNode construct(int[] nums, int l, int r) {
        if (l == r)
            return null;
        int max_i = max(nums, l, r);
        TreeNode root = new TreeNode(nums[max_i]);
        root.left = construct(nums, l, max_i);
        root.right = construct(nums, max_i + 1, r);
        return root;
    }
    public int max(int[] nums, int l, int r) {
        int max_i = l;
        for (int i = l; i < r; i++) {
            if (nums[max_i] < nums[i])
                max_i = i;
        }
        return max_i;
    }
}
```
#### **26、求根节点到叶节点数字之和（ LeetCode 129 ）**

**题目：** 

**题解：**

```java

```
#### **27、 二叉树的最大深度（ LeetCode 104 ）**

**题目：** 

**题解：**

```java

```



#### **28、验证二叉搜索树（ LeetCode 98）**

**题目：** 

**示例：** 

**题解：** 

```java

```


#### **29、对称二叉树（ LeetCode 101）**

**题目：** 

**示例：** 

**题解：** 

```java

```


#### **30、二叉树中的最大路径和（ LeetCode 124）**

**题目：** 

**示例：** 

**题解：** 

```java

```

#### **31、二叉树的最大宽度（ LeetCode 662）**

**题目：** 

**示例：** 

**题解：** 

```java

```

#### **32、二叉树的序列化与反序列化（ LeetCode 297）**

**题目：** 

**示例：** 

**题解：** 

```java

```

#### **33、二叉树的完全性检验（ LeetCode 958）**

**题目：** 

**示例：** 

**题解：** 

```java

```

#### **34、二叉搜索树中第K小的元素（ LeetCode 230）**

**题目：** 

**示例：** 

**题解：** 

```java

```

#### **35、另一个树的子树（ LeetCode 572）**

**题目：** 

**示例：** 

**题解：** 

```java

```

#### **38、相同的树（ LeetCode 100）**

**题目：** 

**示例：** 

**题解：** 

```java

```

### **四、滑动窗口（字串问题都可以用它）**

#### **1、无重复字符的最长串（ LeetCode 3 ）**

**题目：** 给定一个字符串 s ，请你找出其中不含有重复字符的 最长子串 的长度。

**示例：** 
```
输入: s = "abcabcbb"
输出: 3 
解释: 因为无重复字符的最长子串是 "abc"，所以其长度为 3。
```

**题解：** 

```java
public int lengthOfLongestSubstring(String s) {
    HashMap<Character, Integer> map = new HashMap<>();
    //用于记录最大不重复子串的长度
    int maxLen = 0;
    //滑动窗口左指针，也就是左边界
    int left = 0;
    for (int i = 0; i < s.length() ; i++)
    {
        /**
        1、首先，判断当前字符是否包含在map中，如果不包含，将该字符添加到map（字符，字符在数组下标）,
            此时没有出现重复的字符，左指针不需要变化。此时不重复子串的长度为：i-left+1，与原来的maxLen比较，取最大值；

        2、如果当前字符 ch 包含在 map中，此时有2类情况：
            1）当前字符包含在当前有效的子段中，如：abca，当我们遍历到第二个a，当前有效最长子段是 abc，我们又遍历到a，
            那么此时更新 left 为 map.get(a)+1=1，当前有效子段更新为 bca；
            2）当前字符不包含在当前最长有效子段中，如：abba，我们先添加a,b进map，此时left=0，我们再添加b，发现map中包含b，
            而且b包含在最长有效子段中，就是1）的情况，我们更新 left=map.get(b)+1=2，此时子段更新为 b，而且map中仍然包含a，map.get(a)=0；
            随后，我们遍历到a，发现a包含在map中，且map.get(a)=0，如果我们像1）一样处理，就会发现 left=map.get(a)+1=1，实际上，left此时
            应该不变，left始终为2，子段变成 ba才对。

            为了处理以上2类情况，我们每次更新left，left=Math.max(left , map.get(ch)+1).
            另外，更新left后，不管原来的 s.charAt(i) 是否在最长子段中，我们都要将 s.charAt(i) 的位置更新为当前的i，
            因此此时新的 s.charAt(i) 已经进入到 当前最长的子段中！
            */
        if(map.containsKey(s.charAt(i)))
        {
            left = Math.max(left , map.get(s.charAt(i))+1);
        }
        // 不存在，就把这个字符加入
        map.put(s.charAt(i) , i);
        // i-left + 1就是当前滑动窗口的长度
        maxLen = Math.max(maxLen , i-left+1);
    }
    
    return maxLen;
}
```

#### **2、串联所有单词的子串（ LeetCode 30 ）**

**题目：**

**示例：** 
```
```

**题解：** 

```java

```


#### **3、最小覆盖子串（ LeetCode 76 ）**

**题目：**

**题解：** 

```java

```

#### **4、至多包含两个不同字符的最长子串（ LeetCode 159 ）**

**题目：**

**示例：** 
```
```

**题解：** 

```java

```

#### **5、长度最小的子数组（ LeetCode 209 ）**

**题目：**

**示例：** 
```
```

**题解：** 

```java

```

#### **6、滑动窗口最大值（ LeetCode 239）**

**题目：**

**示例：** 
```
```

**题解：** 

```java

```

#### **7、字符串的排列（ LeetCode 567）**

**题目：**

**示例：** 
```
```

**题解：** 

```java

```

#### **8、最小区间（ LeetCode 632）**

**题目：**

**示例：** 
```
```

**题解：** 

```java

```

#### **9、最小窗口子序列（ LeetCode 727）**

**题目：**

**示例：** 
```
```

**题解：** 

```java

```
#### **10、找到字符串中所有字母异位词（ LeetCode 438）**

**题目：**

**示例：** 
```
```

**题解：** 

```java

```

#### **11、字符串的排列（ LeetCode 567）**

**题目：**

**示例：** 
```
```

**题解：** 

```java

```

#### **12、最小覆盖字串（ LeetCode 76）**

**题目：**

**示例：** 
```
```

**题解：** 

```java

```

### **五、图论与搜索算法**

#### **1、所有可能的路径（ LeetCode 797）**

**题目：**

**示例：** 

```

```

**题解：** 

```java

```

#### **3、课程表（ LeetCode 207）**

**题目：**

**示例：** 

```

```

**题解：** 

```java

```

#### **4、课程表Ⅱ（ LeetCode 210）**

**题目：**

**示例：** 

```

```

**题解：** 

```java

```

#### **6、判断二分图（ LeetCode 785）**

**题目：**

**示例：** 

```

```

**题解：** 

```java

```

#### **7、可能的二分法（ LeetCode 886）**

**题目：**

**示例：** 

```

```

**题解：** 

```java

```

#### **9、最低成本联通所有城市（ LeetCode 1135）**

**题目：**

**示例：** 

```

```

**题解：** 

```java

```

#### **10、连接所有点的最小费用（ LeetCode 1584）**

**题目：**

**示例：** 

```

```

**题解：** 

```java

```

#### **11、以图判树（ LeetCode 261）**

**题目：**

**示例：** 

```

```

**题解：** 

```java

```

#### **12、概率最大的路径（ LeetCode 1514）**

**题目：**

**示例：** 

```

```

**题解：** 

```java

```

#### **13、最小体力消耗路径（ LeetCode 1631）**

**题目：**

**示例：** 

```

```

**题解：** 

```java

```

#### **14、网络延迟时间（ LeetCode 743）**

**题目：**

**示例：** 

```

```

**题解：** 

```java

```

#### **15、飞地的数量（ LeetCode 1020）**

**题目：**

**示例：** 

```

```

**题解：** 

```java

```

#### **16、统计封闭岛屿的数目（ LeetCode 1254）**

**题目：**

**示例：** 

```

```

**题解：** 

```java

```

#### **17、统计子岛屿（ LeetCode 1905）**

**题目：**

**示例：** 

```

```

**题解：** 

```java

```

#### **18、岛屿数量（ LeetCode 200）**

**题目：**

**示例：** 

```

```

**题解：** 

```java

```

#### **19、不同岛屿的数量（ LeetCode 694）**

**题目：**

**示例：** 

```

```

**题解：** 

```java

```

#### **20、岛屿的最大面积（ LeetCode 695）**

**题目：**

**示例：** 

```

```

**题解：** 

```java

```

#### **21、二叉树的最小深度（ LeetCode 111）**

**题目：**

**示例：** 

```

```

**题解：** 

```java

```

#### **22、打开转盘锁（ LeetCode 752）**

**题目：**

**示例：** 

```

```

**题解：** 

```java

```

#### **24、滑动谜题（ LeetCode 773）**

**题目：**

**示例：** 

```

```

**题解：** 

```java

```

### **六、回溯算法**

#### **1、全排列（ LeetCode 46）**

**题目：**

**示例：** 

```
```

**题解：** 

```java

```

#### **2、N皇后（ LeetCode 51）**

**题目：**

**示例：** 
```
```

**题解：** 

```java

```

#### **4、划分为k个相等的子集（ LeetCode 698）**

**题目：**

**示例：** 
```
```

**题解：** 

```java

```

#### **5、全排列（ LeetCode 46）**

**题目：**

**示例：** 
```
```

**题解：** 

```java

```

#### **6、数组总和Ⅲ（ LeetCode 216）**

**题目：**

**示例：** 
```
```

**题解：** 

```java

```

#### **7、数组总和（ LeetCode 39）**

**题目：**

**示例：** 
```
```

**题解：** 

```java

```

#### **8、数组总和Ⅱ（ LeetCode 40）**

**题目：**

**示例：** 
```
```

**题解：** 

```java

```

#### **9、全排列Ⅱ（ LeetCode 47）**

**题目：**

**示例：** 
```
```

**题解：** 

```java

```

#### **10、组合（ LeetCode 77）**

**题目：**

**示例：** 
```
```

**题解：** 

```java

```

#### **11、子集（ LeetCode 78）**

**题目：**

**示例：** 
```
```

**题解：** 

```java

```

#### **12、子集Ⅱ（ LeetCode 90）**

**题目：**

**示例：** 
```
```

**题解：** 

```java

```

#### **19、解数独（ LeetCode 37）**

**题目：**

**示例：** 
```
```

**题解：** 

```java

```
#### **20、括号生成（ LeetCode 22）**

**题目：**

**示例：** 
```
```

**题解：** 

```java

```
### **七、贪心**

#### **1、Yogurt factory 机器工厂 （洛谷P1376）**
**题目：** 小 T 开办了一家机器工厂，在 $N$个星期内，原材料成本和劳动力价格不断起伏，第 $i$ 周生产一台机器需要花费 $C_i$ 元。若没把机器卖出去，每保养一台机器，每周需要花费 $S$ 元，这个费用不会发生变化。机器工厂接到订单，在第 $i$ 周需要交付 $Y_i$ 台机器给委托人，第 $i$ 周刚生产的机器，或者之前的存货，都可以进行交付。请你计算出这 $n$ 周时间内完成订单的最小代价。

输入格式：第一行输入两个整数 $N$ 和 $S$，接下来 $N$ 行每行两个数 $C_i$ 和 $Y_i$。
输出格式：输出一个整数，表示最少的代价。

**示例：** 
```
样例输入：
4 5
88 200
89 400
97 300
91 500

样例输出：
126900
```

**题解：** 

```java
import java.util.Scanner;

public class P1376 {

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        // n 是周
        int n = sc.nextInt();
        // s 是维护费
        int s = sc.nextInt();
        // p是一台花费，y是交付台数，min_p是花费最低
        int p, y, min_p = Integer.MAX_VALUE - s;
        long total = 0;
        for (int i = 1; i <= n; i++) {
            // 读入第i周的生产一台的花费p，以及要交付的机器台数y
            p = sc.nextInt();
            y = sc.nextInt();
            // 上一周的最第低价格加上储藏一周的费用，得到利用之前库存的最低价格
            min_p += s;
            // 如果这周生产一台的价格比利用之前库存的价格低，就全部由这周生产
            if (min_p > p){
                min_p = p;
            }
            // 计算交付本周所有机器的最低价格，并且累加
            total += min_p * y;
        }
        System.out.println(total);
        sc.close();
    }

}
```
#### **2、数字拆分**

**题目：** 给定一个数字N，将其拆分为若干个不相同的正整数的和，要求这些数乘积最大。

**示例：** 
```
第一个行输入一个N，第二行输出拆分的整数，用空格隔开，第三行输出最大乘积
10
2 3 5
30 
解释：10 = 2+3+5
12
3 4 5
60
解释：12 = 3+4+5
```

**题解：** 

```java

public class Code2 {
    public static void main(String args[]) {
        Scanner sc = new Scanner(System.in);
        int n = sc.nextInt();
        // 特殊情况1：n = 3，分解方案1 2
        if (n == 3) {
            System.out.println("1 2");
            System.out.println(2);
            return;
        }
        // 特殊情况2：n = 4，分解方案1 3
        if (n == 4) {
            System.out.println("1 3");
            System.out.println(3);
            return;
        }

        int[] res = new int[n];
        int k = 0;
        // 从2开始从小到大每个数都试一次
        for (int i = 2; i < 10000; i++){
            // 如果剩余的数字大于i，那么i就可以作为一个分解出的数字，然后给n减去i
            if (n >= i) {
                res[k ++] = i;
                n -= i;
            } else {
                // 否则，按从大到小的顺序给每个数字都+1，同时每次给当前数字+1
                // 都给n减去1，直到n减为0
                while (n > 0) {
                    for (int j = k - 1; j >= Math.max(0, k - n); j--){
                        res[j] ++;
                    }
                    n -= k;
                }
                break;
            }
        }
        // 高精度乘法，b是乘积，初始值为1
        // 然后循环将划分得到的k个数字乘起来
        int[] b = new int[10000];
        int digit = 0;
        b[0] = 1;
        for (int i = 0; i < k; i++) {
            for (int t = 0; t <= digit; t++)
                b[t] *= res[i];
            for (int t = 0; t <= digit; t++) {
                b[t + 1] += b[t] / 10;
                b[t] %= 10;
            }
            while (b[digit + 1] != 0) {
                digit += 1;
                b[digit + 1] = b[digit] / 10;
                b[digit] %= 10;
            }
        }

        // 输出划分方案和乘积b
        System.out.print(res[0]);
        for (int i = 1; i < k; i++){
            System.out.print(" " + res[i]);
        }
        System.out.println();
        for (int i = digit; i >= 0; i--){
            System.out.print(b[i]);
        }
        System.out.println();

        sc.close();
    }
}
```
#### **3、守望者的逃离（洛谷P1095）**

**题目：**
守望者在与尤迪安的交锋中遭遇了围杀，被困在一个荒芜的大岛上。

为了杀死守望者，尤迪安开始对这个荒岛施咒，这座岛很快就会沉下去。到那时，岛上的所有人都会遇难。

守望者的跑步速度为 $17m/s$，以这样的速度是无法逃离荒岛的。庆幸的是守望者拥有闪烁法术，可在 $1s$ 内移动 $60m$，不过每次使用闪烁法术都会消耗魔法值 $10$ 点。守望者的魔法值恢复的速度为 $4$ 点每秒，只有处在原地休息状态时才能恢复。

现在已知守望者的魔法初值 $M$，他所在的初始位置与岛的出口之间的距离 $S$，岛沉没的时间 $T$。你的任务是写一个程序帮助守望者计算如何在最短的时间内逃离荒岛，若不能逃出，则输出守望者在剩下的时间内能走的最远距离。

注意：守望者跑步、闪烁或休息活动均以秒为单位，且每次活动的持续时间为整数秒。距离的单位为米。

输入数据共一行三个非负整数，分别表示 $M$，$S$，$T$。

输出数据共两行。

第一行一个字符串 $\texttt{Yes}$ 或 $\texttt{No}$，即守望者是否能逃离荒岛。

第二行包含一个整数。第一行为 $\texttt{Yes}$ 时表示守望者逃离荒岛的最短时间；第一行为 $\texttt{No}$ 时表示守望者能走的最远距离。

**示例：** 
```
39 200 4
No
197
```

**题解：** 

```java

```

#### **4、无重叠区间（ LeetCode435）**

**题目：**

**示例：** 
```
```

**题解：** 

```java

```

#### **5、用最少数量的箭引爆气球（ LeetCode 452）**

**题目：**

**示例：** 
```
```

**题解：** 

```java

```

#### **6、会议室Ⅱ（ LeetCode 253）**

**题目：**

**示例：** 
```
```

**题解：** 

```java

```

#### **7、视频拼接（ LeetCode 1024）**

**题目：**

**示例：** 
```
```

**题解：** 

```java

```

#### **8、跳跃游戏Ⅱ（ LeetCode 45）**

**题目：**

**示例：** 
```
```

**题解：** 

```java

```

#### **9、跳跃游戏（ LeetCode 55）**

**题目：**

**示例：** 
```
```

**题解：** 

```java

```

#### **10、加油站（ LeetCode 134）**

**题目：**

**示例：** 
```
```

**题解：** 

```java

```

#### **11、跳石头（ 洛谷P2678）**

**题目：**

**示例：** 

```
```

**题解：** 

```java

```

#### **10、旅行家的预算（ 洛谷P1016）**
**题目：**

一个旅行家想驾驶汽车以最少的费用从一个城市到另一个城市（假设出发时油箱是空的）。给定两个城市之间的距离 $D_1$、汽车油箱的容量 $C$（以升为单位）、每升汽油能行驶的距离 $D_2$、出发点每升汽油价格$P$和沿途油站数 $N$（$N$ 可以为零），油站 $i$ 离出发点的距离 $D_i$、每升汽油价格 $P_i$（$i=1,2,…,N$）。计算结果四舍五入至小数点后两位。如果无法到达目的地，则输出 `No Solution`。

输入格式

第一行，$D_1$，$C$，$D_2$，$P$，$N$。

接下来有 $N$ 行。

第 $i+1$ 行，两个数字，油站 $i$ 离出发点的距离 $D_i$ 和每升汽油价格 $P_i$。

输出格式

所需最小费用，计算结果四舍五入至小数点后两位。如果无法到达目的地，则输出 `No Solution`。

**示例：** 

```
输入：
275.6 11.9 27.4 2.8 2
102.0 2.9
220.0 2.2
输出：
26.95
```

**题解：** 

```java

```
### **八、动态规划**

动态规划问题目标其实就是求最值，常见的题型就是最长序列、最小距离等等。动态规划的三要素：重叠子问题、最优子结构、状态转移方程。其中状态转移方程最难，也是动态规划最重要的步骤。状态转移的套路：明确基本问题——》明确状态——》明确选择——》定义DP数组/函数的含义。

重叠子问题的解决方案：定义备忘录，每次算出某个子问题的答案后，先记到备忘录中再返回，每次遇到一个子问题，先去备忘录里查一下，看看是否已经解决这个问题，有的话直接拿来用，一般使用一个数组充当备忘录。

#### **1、斐波那契数（ LeetCode 509 ）**

**题目：**

**示例：** 

**题解：** 

```java

```

#### **2、使用最小花费爬楼梯（ LeetCode 746 ）**

**题目：**

**示例：** 

**题解：** 

```java

```

#### **3、组合总和 Ⅳ（ LeetCode 377 ）**

**题目：**

**示例：** 

**题解：** 

```java

```

#### **4、单词拆分（ LeetCode 139 ）**

**题目：**

**示例：** 

**题解：** 

```java

```

#### **5、一和零（ LeetCode 474 ）**

**题目：**

**示例：** 

**题解：** 

```java

```


#### **6、判断子序列（ LeetCode 392 ）**

**题目：**

**示例：** 

**题解：** 

```java

```

#### **7、爬楼梯（ LeetCode 70 ）**

**题目：** 假设你正在爬楼梯。需要 n 阶你才能到达楼顶。每次你可以爬 1 或 2 个台阶。你有多少种不同的方法可以爬到楼顶呢？

**示例：** 

    输入：n = 2
    输出：2
    解释：有两种方法可以爬到楼顶。
    1. 1 阶 + 1 阶
    2. 2 阶

**题解：** 

```java
// 方法一
class Solution {
    public int climbStairs(int n) {
        double sqrt_5 = Math.sqrt(5);
        double fib_n = Math.pow((1 + sqrt_5) / 2, n + 1) - Math.pow((1 - sqrt_5) / 2,n + 1);
        return (int)(fib_n / sqrt_5);
    }
}
// 方法二
class Solution {
    public int climbStairs(int n) {
        int[] dp = new int[n + 1];
        dp[0] = 1;
        dp[1] = 1;
        for(int i = 2; i <= n; i++) {
            dp[i] = dp[i - 1] + dp[i - 2];
        }
        return dp[n];
    }
}
```

#### **8、最大子序和/最大子数组和（ LeetCode 53 ）**

**题目：** 给你一个整数数组 nums ，请你找出一个具有最大和的连续子数组（子数组最少包含一个元素），返回其最大和。子数组 是数组中的一个连续部分。

**示例：** 

    输入：nums = [-2,1,-3,4,-1,2,1,-5,4]
    输出：6
    解释：连续子数组 [4,-1,2,1] 的和最大，为 6 。

**题解：** 

```java
// 方法一：动态规划
class Solution {
    public int maxSubArray(int[] nums) {
        int pre = 0, maxAns = nums[0];
        for (int x : nums) {
            pre = Math.max(pre + x, x);
            maxAns = Math.max(maxAns, pre);
        }
        return maxAns;
    }
}

// 方法二：分治
class Solution {
    public class Status {
        public int lSum, rSum, mSum, iSum;

        public Status(int lSum, int rSum, int mSum, int iSum) {
            this.lSum = lSum;
            this.rSum = rSum;
            this.mSum = mSum;
            this.iSum = iSum;
        }
    }

    public int maxSubArray(int[] nums) {
        return getInfo(nums, 0, nums.length - 1).mSum;
    }

    public Status getInfo(int[] a, int l, int r) {
        if (l == r) {
            return new Status(a[l], a[l], a[l], a[l]);
        }
        int m = (l + r) >> 1;
        Status lSub = getInfo(a, l, m);
        Status rSub = getInfo(a, m + 1, r);
        return pushUp(lSub, rSub);
    }

    public Status pushUp(Status l, Status r) {
        int iSum = l.iSum + r.iSum;
        int lSum = Math.max(l.lSum, l.iSum + r.lSum);
        int rSum = Math.max(r.rSum, r.iSum + l.rSum);
        int mSum = Math.max(Math.max(l.mSum, r.mSum), l.rSum + r.lSum);
        return new Status(lSum, rSum, mSum, iSum);
    }
}
```

#### **9、零钱兑换（ LeetCode 322 ）**

**题目：** 给你一个整数数组 coins ，表示不同面额的硬币；以及一个整数 amount ，表示总金额。计算并返回可以凑成总金额所需的 最少的硬币个数 。如果没有任何一种硬币组合能组成总金额，返回 -1 。你可以认为每种硬币的数量是无限的。

**示例：** 

    输入：coins = [1, 2, 5], amount = 11
    输出：3 
    解释：11 = 5 + 5 + 1

**题解：** 

```java
// 方法一：记忆化搜索
public class Solution {
    public int coinChange(int[] coins, int amount) {
        if (amount < 1) {
            return 0;
        }
        return coinChange(coins, amount, new int[amount]);
    }

    private int coinChange(int[] coins, int rem, int[] count) {
        if (rem < 0) {
            return -1;
        }
        if (rem == 0) {
            return 0;
        }
        if (count[rem - 1] != 0) {
            return count[rem - 1];
        }
        int min = Integer.MAX_VALUE;
        for (int coin : coins) {
            int res = coinChange(coins, rem - coin, count);
            if (res >= 0 && res < min) {
                min = 1 + res;
            }
        }
        count[rem - 1] = (min == Integer.MAX_VALUE) ? -1 : min;
        return count[rem - 1];
    }
}
// 方法二：动态规划
public class Solution {
    public int coinChange(int[] coins, int amount) {
        int max = amount + 1;
        int[] dp = new int[amount + 1];
        Arrays.fill(dp, max);
        dp[0] = 0;
        for (int i = 1; i <= amount; i++) {
            for (int j = 0; j < coins.length; j++) {
                if (coins[j] <= i) {
                    dp[i] = Math.min(dp[i], dp[i - coins[j]] + 1);
                }
            }
        }
        return dp[amount] > amount ? -1 : dp[amount];
    }
}
```

#### **10、零钱兑换 II（ LeetCode 518 ）**

**题目：** 给你一个整数数组 coins 表示不同面额的硬币，另给一个整数 amount 表示总金额。请你计算并返回可以凑成总金额的硬币组合数。如果任何硬币组合都无法凑出总金额，返回 0 。假设每一种面额的硬币有无限个。 题目数据保证结果符合 32 位带符号整数。

**示例：** 

    输入：amount = 5, coins = [1, 2, 5]
    输出：4
    解释：有四种方式可以凑成总金额：
    5=5
    5=2+2+1
    5=2+1+1+1
    5=1+1+1+1+1

**题解：** 

```java
// 动态规划
class Solution {
    public int change(int amount, int[] coins) {
        int[] dp = new int[amount + 1];
        dp[0] = 1;
        for (int coin : coins) {
            for (int i = coin; i <= amount; i++) {
                dp[i] += dp[i - coin];
            }
        }
        return dp[amount];
    }
}
```

#### **11、最小路径和（ LeetCode 64 ）**

**题目：** 给定一个包含非负整数的 m x n 网格 grid ，请找出一条从左上角到右下角的路径，使得路径上的数字总和为最小。说明：每次只能向下或者向右移动一步。

**示例：** 

    输入：grid = [[1,3,1],[1,5,1],[4,2,1]]
    输出：7
    解释：因为路径 1→3→1→1→1 的总和最小。

**题解：** 

```java
// 动态规划
class Solution {
    public int minPathSum(int[][] grid) {
        if (grid == null || grid.length == 0 || grid[0].length == 0) {
            return 0;
        }
        int rows = grid.length, columns = grid[0].length;
        int[][] dp = new int[rows][columns];
        dp[0][0] = grid[0][0];
        for (int i = 1; i < rows; i++) {
            dp[i][0] = dp[i - 1][0] + grid[i][0];
        }
        for (int j = 1; j < columns; j++) {
            dp[0][j] = dp[0][j - 1] + grid[0][j];
        }
        for (int i = 1; i < rows; i++) {
            for (int j = 1; j < columns; j++) {
                dp[i][j] = Math.min(dp[i - 1][j], dp[i][j - 1]) + grid[i][j];
            }
        }
        return dp[rows - 1][columns - 1];
    }
}
```

#### **12、编辑距离（ LeetCode 72 ）**

**题目：** 给你两个单词 word1 和 word2， 请返回将 word1 转换成 word2 所使用的最少操作数  。你可以对一个单词进行如下三种操作：

    插入一个字符
    删除一个字符
    替换一个字符

**示例：** 

    输入：word1 = "horse", word2 = "ros"
    输出：3
    解释：
    horse -> rorse (将 'h' 替换为 'r')
    rorse -> rose (删除 'r')
    rose -> ros (删除 'e')

**题解：** 

```java
// 动态规划
class Solution {
    public int minDistance(String word1, String word2) {
        int n = word1.length();
        int m = word2.length();

        // 有一个字符串为空串
        if (n * m == 0) {
            return n + m;
        }

        // DP 数组
        int[][] D = new int[n + 1][m + 1];

        // 边界状态初始化
        for (int i = 0; i < n + 1; i++) {
            D[i][0] = i;
        }
        for (int j = 0; j < m + 1; j++) {
            D[0][j] = j;
        }

        // 计算所有 DP 值
        for (int i = 1; i < n + 1; i++) {
            for (int j = 1; j < m + 1; j++) {
                int left = D[i - 1][j] + 1;
                int down = D[i][j - 1] + 1;
                int left_down = D[i - 1][j - 1];
                if (word1.charAt(i - 1) != word2.charAt(j - 1)) {
                    left_down += 1;
                }
                D[i][j] = Math.min(left, Math.min(down, left_down));
            }
        }
        return D[n][m];
    }
}
```

#### **13、买卖股票的最佳时机（ LeetCode 121 ）**

**题目：** 给定一个数组 prices ，它的第 i 个元素 prices[i] 表示一支给定股票第 i 天的价格。你只能选择 某一天 买入这只股票，并选择在 未来的某一个不同的日子 卖出该股票。设计一个算法来计算你所能获取的最大利润。返回你可以从这笔交易中获取的最大利润。如果你不能获取任何利润，返回 0 

**示例：** 

    输入：[7,1,5,3,6,4]
    输出：5
    解释：在第 2 天（股票价格 = 1）的时候买入，在第 5 天（股票价格 = 6）的时候卖出，最大利润 = 6-1 = 5 。
        注意利润不能是 7-1 = 6, 因为卖出价格需要大于买入价格；同时，你不能在买入前卖出股票。

**题解：** 

```java
public class Solution {
    public int maxProfit(int prices[]) {
        int minprice = Integer.MAX_VALUE;
        int maxprofit = 0;
        for (int i = 0; i < prices.length; i++) {
            if (prices[i] < minprice) {
                minprice = prices[i];
            } else if (prices[i] - minprice > maxprofit) {
                maxprofit = prices[i] - minprice;
            }
        }
        return maxprofit;
    }
}
```

#### **14、买卖股票的最佳时机II（ LeetCode 122 ）**

**题目：** 给你一个整数数组 prices ，其中 prices[i] 表示某支股票第 i 天的价格。在每一天，你可以决定是否购买和/或出售股票。你在任何时候 最多 只能持有 一股 股票。你也可以先购买，然后在 同一天 出售。返回 你能获得的 最大 利润 。

**示例：**

    输入：prices = [7,1,5,3,6,4]
    输出：7
    解释：在第 2 天（股票价格 = 1）的时候买入，在第 3 天（股票价格 = 5）的时候卖出, 这笔交易所能获得利润 = 5 - 1 = 4 。
        随后，在第 4 天（股票价格 = 3）的时候买入，在第 5 天（股票价格 = 6）的时候卖出, 这笔交易所能获得利润 = 6 - 3 = 3 。
        总利润为 4 + 3 = 7 。

**题解：** 

```java
// 方法一：动态规划
class Solution {
    public int maxProfit(int[] prices) {
        int n = prices.length;
        int[][] dp = new int[n][2];
        dp[0][0] = 0;
        dp[0][1] = -prices[0];
        for (int i = 1; i < n; ++i) {
            dp[i][0] = Math.max(dp[i - 1][0], dp[i - 1][1] + prices[i]);
            dp[i][1] = Math.max(dp[i - 1][1], dp[i - 1][0] - prices[i]);
        }
        return dp[n - 1][0];
    }
}
// 方法二：贪心
class Solution {
    public int maxProfit(int[] prices) {
        int ans = 0;
        int n = prices.length;
        for (int i = 1; i < n; ++i) {
            ans += Math.max(0, prices[i] - prices[i - 1]);
        }
        return ans;
    }
}
```

#### **15、买卖股票的最佳时机III（ LeetCode 123 ）**

**题目：** 给定一个数组，它的第 i 个元素是一支给定的股票在第 i 天的价格。设计一个算法来计算你所能获取的最大利润。你最多可以完成 两笔 交易。注意：你不能同时参与多笔交易（你必须在再次购买前出售掉之前的股票）。

**示例：** 

    输入：prices = [3,3,5,0,0,3,1,4]
    输出：6
    解释：在第 4 天（股票价格 = 0）的时候买入，在第 6 天（股票价格 = 3）的时候卖出，这笔交易所能获得利润 = 3-0 = 3 。
        随后，在第 7 天（股票价格 = 1）的时候买入，在第 8 天 （股票价格 = 4）的时候卖出，这笔交易所能获得利润 = 4-1 = 3 。

**题解：** 

```java
// 动态规划
class Solution {
    public int maxProfit(int[] prices) {
        int n = prices.length;
        int buy1 = -prices[0], sell1 = 0;
        int buy2 = -prices[0], sell2 = 0;
        for (int i = 1; i < n; ++i) {
            buy1 = Math.max(buy1, -prices[i]);
            sell1 = Math.max(sell1, buy1 + prices[i]);
            buy2 = Math.max(buy2, sell1 - prices[i]);
            sell2 = Math.max(sell2, buy2 + prices[i]);
        }
        return sell2;
    }
}
```

#### **16、买卖股票的最佳时机IV（ LeetCode 188 ）**

**题目：** 给定一个整数数组 prices ，它的第 i 个元素 prices[i] 是一支给定的股票在第 i 天的价格。设计一个算法来计算你所能获取的最大利润。你最多可以完成 k 笔交易。注意：你不能同时参与多笔交易（你必须在再次购买前出售掉之前的股票）。

**示例：** 

    输入：k = 2, prices = [2,4,1]
    输出：2
    解释：在第 1 天 (股票价格 = 2) 的时候买入，在第 2 天 (股票价格 = 4) 的时候卖出，这笔交易所能获得利润 = 4-2 = 2 。

**题解：** 

```java
// 动态规划
class Solution {
    public int maxProfit(int k, int[] prices) {
        if (prices.length == 0) {
            return 0;
        }

        int n = prices.length;
        k = Math.min(k, n / 2);
        int[][] buy = new int[n][k + 1];
        int[][] sell = new int[n][k + 1];

        buy[0][0] = -prices[0];
        sell[0][0] = 0;
        for (int i = 1; i <= k; ++i) {
            buy[0][i] = sell[0][i] = Integer.MIN_VALUE / 2;
        }

        for (int i = 1; i < n; ++i) {
            buy[i][0] = Math.max(buy[i - 1][0], sell[i - 1][0] - prices[i]);
            for (int j = 1; j <= k; ++j) {
                buy[i][j] = Math.max(buy[i - 1][j], sell[i - 1][j] - prices[i]);
                sell[i][j] = Math.max(sell[i - 1][j], buy[i - 1][j - 1] + prices[i]);   
            }
        }

        return Arrays.stream(sell[n - 1]).max().getAsInt();
    }
}
// 优化
class Solution {
    public int maxProfit(int k, int[] prices) {
        if (prices.length == 0) {
            return 0;
        }

        int n = prices.length;
        k = Math.min(k, n / 2);
        int[] buy = new int[k + 1];
        int[] sell = new int[k + 1];

        buy[0] = -prices[0];
        sell[0] = 0;
        for (int i = 1; i <= k; ++i) {
            buy[i] = sell[i] = Integer.MIN_VALUE / 2;
        }

        for (int i = 1; i < n; ++i) {
            buy[0] = Math.max(buy[0], sell[0] - prices[i]);
            for (int j = 1; j <= k; ++j) {
                buy[j] = Math.max(buy[j], sell[j] - prices[i]);
                sell[j] = Math.max(sell[j], buy[j - 1] + prices[i]);   
            }
        }

        return Arrays.stream(sell).max().getAsInt();
    }
}
```

#### **17、最佳买卖股票时机含冷冻期( LeetCode 309 )**

**题目：** 给定一个整数数组prices，其中第  prices[i] 表示第 i 天的股票价格 。​设计一个算法计算出最大利润。在满足以下约束条件下，你可以尽可能地完成更多的交易（多次买卖一支股票）:卖出股票后，你无法在第二天买入股票 (即冷冻期为 1 天)。注意：你不能同时参与多笔交易（你必须在再次购买前出售掉之前的股票）。

**示例：** 

    输入: prices = [1,2,3,0,2]
    输出: 3 
    解释: 对应的交易状态为: [买入, 卖出, 冷冻期, 买入, 卖出]

**题解：** 

```java
class Solution {
    public int maxProfit(int[] prices) {
        if (prices.length == 0) {
            return 0;
        }

        int n = prices.length;
        // f[i][0]: 手上持有股票的最大收益
        // f[i][1]: 手上不持有股票，并且处于冷冻期中的累计最大收益
        // f[i][2]: 手上不持有股票，并且不在冷冻期中的累计最大收益
        int[][] f = new int[n][3];
        f[0][0] = -prices[0];
        for (int i = 1; i < n; ++i) {
            f[i][0] = Math.max(f[i - 1][0], f[i - 1][2] - prices[i]);
            f[i][1] = f[i - 1][0] + prices[i];
            f[i][2] = Math.max(f[i - 1][1], f[i - 1][2]);
        }
        return Math.max(f[n - 1][1], f[n - 1][2]);
    }
}
// 优化
class Solution {
    public int maxProfit(int[] prices) {
        if (prices.length == 0) {
            return 0;
        }

        int n = prices.length;
        int f0 = -prices[0];
        int f1 = 0;
        int f2 = 0;
        for (int i = 1; i < n; ++i) {
            int newf0 = Math.max(f0, f2 - prices[i]);
            int newf1 = f0 + prices[i];
            int newf2 = Math.max(f1, f2);
            f0 = newf0;
            f1 = newf1;
            f2 = newf2;
        }

        return Math.max(f1, f2);
    }
}
```

#### **18、买卖股票的最佳时机含手续费( LeetCode 714 )**

**题目：** 给定一个整数数组 prices，其中 prices[i]表示第 i 天的股票价格 ；整数 fee 代表了交易股票的手续费用。你可以无限次地完成交易，但是你每笔交易都需要付手续费。如果你已经购买了一个股票，在卖出它之前你就不能再继续购买股票了。返回获得利润的最大值。注意：这里的一笔交易指买入持有并卖出股票的整个过程，每笔交易你只需要为支付一次手续费。

**示例：** 

    输入：prices = [1, 3, 2, 8, 4, 9], fee = 2
    输出：8
    解释：能够达到的最大利润:  
    在此处买入 prices[0] = 1
    在此处卖出 prices[3] = 8
    在此处买入 prices[4] = 4
    在此处卖出 prices[5] = 9
    总利润: ((8 - 1) - 2) + ((9 - 4) - 2) = 8

**题解：** 

```java
// 动态规划
class Solution {
    public int maxProfit(int[] prices, int fee) {
        int n = prices.length;
        int[][] dp = new int[n][2];
        dp[0][0] = 0;
        dp[0][1] = -prices[0];
        for (int i = 1; i < n; ++i) {
            dp[i][0] = Math.max(dp[i - 1][0], dp[i - 1][1] + prices[i] - fee);
            dp[i][1] = Math.max(dp[i - 1][1], dp[i - 1][0] - prices[i]);
        }
        return dp[n - 1][0];
    }
}

// 优化
class Solution {
    public int maxProfit(int[] prices, int fee) {
        int n = prices.length;
        int sell = 0, buy = -prices[0];
        for (int i = 1; i < n; ++i) {
            sell = Math.max(sell, buy + prices[i] - fee);
            buy = Math.max(buy, sell - prices[i]);
        }
        return sell;
    }
}
```

#### **19、完全平方数（ LeetCode 279 ）**

**题目：** 给你一个整数 n ，返回 和为 n 的完全平方数的最少数量 。完全平方数 是一个整数，其值等于另一个整数的平方；换句话说，其值等于一个整数自乘的积。例如，1、4、9 和 16 都是完全平方数，而 3 和 11 不是。

**示例：** 

    输入：n = 12
    输出：3 
    解释：12 = 4 + 4 + 4

**题解：** 

```java
class Solution {
    public int numSquares(int n) {
        int[] f = new int[n + 1];
        for (int i = 1; i <= n; i++) {
            int minn = Integer.MAX_VALUE;
            for (int j = 1; j * j <= i; j++) {
                minn = Math.min(minn, f[i - j * j]);
            }
            f[i] = minn + 1;
        }
        return f[n];
    }
}
```

#### **20、三角形最小路径和（ LeetCode 120 ）**

**题目：** 给定一个三角形 triangle ，找出自顶向下的最小路径和。每一步只能移动到下一行中相邻的结点上。相邻的结点 在这里指的是 下标 与 上一层结点下标 相同或者等于 上一层结点下标 + 1 的两个结点。也就是说，如果正位于当前行的下标 i ，那么下一步可以移动到下一行的下标 i 或 i + 1 。

**示例：** 

    输入：triangle = [[2],[3,4],[6,5,7],[4,1,8,3]]
    输出：11
    解释：如下面简图所示：
         2
        3 4
       6 5 7
      4 1 8 3
    自顶向下的最小路径和为 11（即，2 + 3 + 5 + 1 = 11）。

**题解：** 

```java
class Solution {
    public int minimumTotal(List<List<Integer>> triangle) {
        int n = triangle.size();
        int[][] f = new int[n][n];
        f[0][0] = triangle.get(0).get(0);
        for (int i = 1; i < n; ++i) {
            f[i][0] = f[i - 1][0] + triangle.get(i).get(0);
            for (int j = 1; j < i; ++j) {
                f[i][j] = Math.min(f[i - 1][j - 1], f[i - 1][j]) + triangle.get(i).get(j);
            }
            f[i][i] = f[i - 1][i - 1] + triangle.get(i).get(i);
        }
        int minTotal = f[n - 1][0];
        for (int i = 1; i < n; ++i) {
            minTotal = Math.min(minTotal, f[n - 1][i]);
        }
        return minTotal;
    }
}
```

#### **21、不同路径（ LeetCode 62 ）**

**题目：** 一个机器人位于一个 m x n 网格的左上角 （起始点在下图中标记为 “Start” ）。机器人每次只能向下或者向右移动一步。机器人试图达到网格的右下角（在下图中标记为 “Finish” ）。问总共有多少条不同的路径？

**示例：** 

    输入：m = 3, n = 7
    输出：28

**题解：** 

```java
class Solution {
    public int uniquePaths(int m, int n) {
        int[][] f = new int[m][n];
        for (int i = 0; i < m; ++i) {
            f[i][0] = 1;
        }
        for (int j = 0; j < n; ++j) {
            f[0][j] = 1;
        }
        for (int i = 1; i < m; ++i) {
            for (int j = 1; j < n; ++j) {
                f[i][j] = f[i - 1][j] + f[i][j - 1];
            }
        }
        return f[m - 1][n - 1];
    }
}
```

#### **22、不同路径II（ LeetCode 63 ）**

**题目：** 一个机器人位于一个 m x n 网格的左上角 （起始点在下图中标记为 “Start” ）。机器人每次只能向下或者向右移动一步。机器人试图达到网格的右下角（在下图中标记为 “Finish”）。现在考虑网格中有障碍物。那么从左上角到右下角将会有多少条不同的路径？网格中的障碍物和空位置分别用 1 和 0 来表示。

**示例：** 

    输入：obstacleGrid = [[0,0,0],[0,1,0],[0,0,0]]
    输出：2
    解释：3x3 网格的正中间有一个障碍物。
    从左上角到右下角一共有 2 条不同的路径：
    1. 向右 -> 向右 -> 向下 -> 向下
    2. 向下 -> 向下 -> 向右 -> 向右

**题解：** 

```java
class Solution {
    public int uniquePathsWithObstacles(int[][] obstacleGrid) {
        int n = obstacleGrid.length, m = obstacleGrid[0].length;
        int[] f = new int[m];

        f[0] = obstacleGrid[0][0] == 0 ? 1 : 0;
        for (int i = 0; i < n; ++i) {
            for (int j = 0; j < m; ++j) {
                if (obstacleGrid[i][j] == 1) {
                    f[j] = 0;
                    continue;
                }
                if (j - 1 >= 0 && obstacleGrid[i][j - 1] == 0) {
                    f[j] += f[j - 1];
                }
            }
        }

        return f[m - 1];
    }
}
```

#### **23、整数拆分（ LeetCode 343 ）**

**题目：** 给定一个正整数 n ，将其拆分为 k 个 正整数 的和（ k >= 2 ），并使这些整数的乘积最大化。返回 你可以获得的最大乘积 。

**示例：** 

    输入: n = 2
    输出: 1
    解释: 2 = 1 + 1, 1 × 1 = 1。

**题解：** 

```java
class Solution {
    public int integerBreak(int n) {
        int[] dp = new int[n + 1];
        for (int i = 2; i <= n; i++) {
            int curMax = 0;
            for (int j = 1; j < i; j++) {
                curMax = Math.max(curMax, Math.max(j * (i - j), j * dp[i - j]));
            }
            dp[i] = curMax;
        }
        return dp[n];
    }
}
// 优化
class Solution {
    public int integerBreak(int n) {
        if (n < 4) {
            return n - 1;
        }
        int[] dp = new int[n + 1];
        dp[2] = 1;
        for (int i = 3; i <= n; i++) {
            dp[i] = Math.max(Math.max(2 * (i - 2), 2 * dp[i - 2]), Math.max(3 * (i - 3), 3 * dp[i - 3]));
        }
        return dp[n];
    }
}
```

#### **24、不同的二叉搜索树（ LeetCode 96 ）**

**题目：** 给你一个整数 n ，求恰由 n 个节点组成且节点值从 1 到 n 互不相同的 二叉搜索树 有多少种？返回满足题意的二叉搜索树的种数。

**示例：** 

    输入：n = 3
    输出：5

**题解：** 

```java
class Solution {
    public int numTrees(int n) {
        int[] G = new int[n + 1];
        G[0] = 1;
        G[1] = 1;

        for (int i = 2; i <= n; ++i) {
            for (int j = 1; j <= i; ++j) {
                G[i] += G[j - 1] * G[i - j];
            }
        }
        return G[n];
    }
}
```

#### **25、地下城游戏（ LeetCode 174 ）**

**题目：** 一些恶魔抓住了公主（P）并将她关在了地下城的右下角。地下城是由 M x N 个房间组成的二维网格。我们英勇的骑士（K）最初被安置在左上角的房间里，他必须穿过地下城并通过对抗恶魔来拯救公主。骑士的初始健康点数为一个正整数。如果他的健康点数在某一时刻降至 0 或以下，他会立即死亡。有些房间由恶魔守卫，因此骑士在进入这些房间时会失去健康点数（若房间里的值为负整数，则表示骑士将损失健康点数）；其他房间要么是空的（房间里的值为 0），要么包含增加骑士健康点数的魔法球（若房间里的值为正整数，则表示骑士将增加健康点数）。为了尽快到达公主，骑士决定每次只向右或向下移动一步。编写一个函数来计算确保骑士能够拯救到公主所需的最低初始健康点数。

例如，考虑到如下布局的地下城，如果骑士遵循最佳路径 右 -> 右 -> 下 -> 下，则骑士的初始健康点数至少为 7。
|1|2|3|
|--|--|--|
|-2(K)|-3|3|
|-5|-10|1|
|10|30|-5(P)|

**题解：** 

```java
class Solution {
    public int calculateMinimumHP(int[][] dungeon) {
        int n = dungeon.length, m = dungeon[0].length;
        int[][] dp = new int[n + 1][m + 1];
        for (int i = 0; i <= n; ++i) {
            Arrays.fill(dp[i], Integer.MAX_VALUE);
        }
        dp[n][m - 1] = dp[n - 1][m] = 1;
        for (int i = n - 1; i >= 0; --i) {
            for (int j = m - 1; j >= 0; --j) {
                int minn = Math.min(dp[i + 1][j], dp[i][j + 1]);
                dp[i][j] = Math.max(minn - dungeon[i][j], 1);
            }
        }
        return dp[0][0];
    }
}
```

#### **26、打家劫舍（ LeetCode 198 ）**

**题目：** 你是一个专业的小偷，计划偷窃沿街的房屋。每间房内都藏有一定的现金，影响你偷窃的唯一制约因素就是相邻的房屋装有相互连通的防盗系统，如果两间相邻的房屋在同一晚上被小偷闯入，系统会自动报警。给定一个代表每个房屋存放金额的非负整数数组，计算你 不触动警报装置的情况下 ，一夜之内能够偷窃到的最高金额。

**示例：** 

    输入：[1,2,3,1]
    输出：4
    解释：偷窃 1 号房屋 (金额 = 1) ，然后偷窃 3 号房屋 (金额 = 3)。偷窃到的最高金额 = 1 + 3 = 4 。

**题解：** 

```java
class Solution {
    public int rob(int[] nums) {
        if (nums == null || nums.length == 0) {
            return 0;
        }
        int length = nums.length;
        if (length == 1) {
            return nums[0];
        }
        int[] dp = new int[length];
        dp[0] = nums[0];
        dp[1] = Math.max(nums[0], nums[1]);
        for (int i = 2; i < length; i++) {
            dp[i] = Math.max(dp[i - 2] + nums[i], dp[i - 1]);
        }
        return dp[length - 1];
    }
}
// 优化
class Solution {
    public int rob(int[] nums) {
        if (nums == null || nums.length == 0) {
            return 0;
        }
        int length = nums.length;
        if (length == 1) {
            return nums[0];
        }
        int first = nums[0], second = Math.max(nums[0], nums[1]);
        for (int i = 2; i < length; i++) {
            int temp = second;
            second = Math.max(first + nums[i], second);
            first = temp;
        }
        return second;
    }
}
```

#### **27、打家劫舍II（ LeetCode 213 ）**

**题目：** 你是一个专业的小偷，计划偷窃沿街的房屋，每间房内都藏有一定的现金。这个地方所有的房屋都 围成一圈 ，这意味着第一个房屋和最后一个房屋是紧挨着的。同时，相邻的房屋装有相互连通的防盗系统，如果两间相邻的房屋在同一晚上被小偷闯入，系统会自动报警 。给定一个代表每个房屋存放金额的非负整数数组，计算你 在不触动警报装置的情况下 ，今晚能够偷窃到的最高金额。

**示例：** 

    输入：nums = [2,3,2]
    输出：3
    解释：你不能先偷窃 1 号房屋（金额 = 2），然后偷窃 3 号房屋（金额 = 2）, 因为他们是相邻的。

**题解：** 

```java
class Solution {
    public int rob(int[] nums) {
        int length = nums.length;
        if (length == 1) {
            return nums[0];
        } else if (length == 2) {
            return Math.max(nums[0], nums[1]);
        }
        return Math.max(robRange(nums, 0, length - 2), robRange(nums, 1, length - 1));
    }

    public int robRange(int[] nums, int start, int end) {
        int first = nums[start], second = Math.max(nums[start], nums[start + 1]);
        for (int i = start + 2; i <= end; i++) {
            int temp = second;
            second = Math.max(first + nums[i], second);
            first = temp;
        }
        return second;
    }
}
```

#### **28、打家劫舍III（ LeetCode 337 ）**

**题目：** 小偷又发现了一个新的可行窃的地区。这个地区只有一个入口，我们称之为 root 。除了 root 之外，每栋房子有且只有一个“父“房子与之相连。一番侦察之后，聪明的小偷意识到“这个地方的所有房屋的排列类似于一棵二叉树”。 如果 两个直接相连的房子在同一天晚上被打劫 ，房屋将自动报警。给定二叉树的 root 。返回 在不触动警报的情况下 ，小偷能够盗取的最高金额 。

**示例：** 

    输入: root = [3,2,3,null,3,null,1]
    输出: 7 
    解释: 小偷一晚能够盗取的最高金额 3 + 3 + 1 = 7

**题解：** 

```java
// 动态规划
class Solution {
    Map<TreeNode, Integer> f = new HashMap<TreeNode, Integer>();
    Map<TreeNode, Integer> g = new HashMap<TreeNode, Integer>();

    public int rob(TreeNode root) {
        dfs(root);
        return Math.max(f.getOrDefault(root, 0), g.getOrDefault(root, 0));
    }

    public void dfs(TreeNode node) {
        if (node == null) {
            return;
        }
        dfs(node.left);
        dfs(node.right);
        f.put(node, node.val + g.getOrDefault(node.left, 0) + g.getOrDefault(node.right, 0));
        g.put(node, Math.max(f.getOrDefault(node.left, 0), g.getOrDefault(node.left, 0)) + Math.max(f.getOrDefault(node.right, 0), g.getOrDefault(node.right, 0)));
    }
}

// 优化
class Solution {
    public int rob(TreeNode root) {
        int[] rootStatus = dfs(root);
        return Math.max(rootStatus[0], rootStatus[1]);
    }

    public int[] dfs(TreeNode node) {
        if (node == null) {
            return new int[]{0, 0};
        }
        int[] l = dfs(node.left);
        int[] r = dfs(node.right);
        int selected = node.val + l[1] + r[1];
        int notSelected = Math.max(l[0], l[1]) + Math.max(r[0], r[1]);
        return new int[]{selected, notSelected};
    }
}
```

#### **29、最长递增子序列（ LeetCode 300 ）**

**题目：** 给你一个整数数组 nums ，找到其中最长严格递增子序列的长度。子序列 是由数组派生而来的序列，删除（或不删除）数组中的元素而不改变其余元素的顺序。例如，[3,6,2,7] 是数组 [0,3,1,6,2,2,7] 的子序列。

**示例：** 

    输入：nums = [10,9,2,5,3,7,101,18]
    输出：4
    解释：最长递增子序列是 [2,3,7,101]，因此长度为 4 。

**题解：** 

```java
// 动态规划
class Solution {
    public int lengthOfLIS(int[] nums) {
        if (nums.length == 0) {
            return 0;
        }
        int[] dp = new int[nums.length];
        dp[0] = 1;
        int maxans = 1;
        for (int i = 1; i < nums.length; i++) {
            dp[i] = 1;
            for (int j = 0; j < i; j++) {
                if (nums[i] > nums[j]) {
                    dp[i] = Math.max(dp[i], dp[j] + 1);
                }
            }
            maxans = Math.max(maxans, dp[i]);
        }
        return maxans;
    }
}
// 贪心 + 二分查找
class Solution {
    public int lengthOfLIS(int[] nums) {
        int len = 1, n = nums.length;
        if (n == 0) {
            return 0;
        }
        int[] d = new int[n + 1];
        d[len] = nums[0];
        for (int i = 1; i < n; ++i) {
            if (nums[i] > d[len]) {
                d[++len] = nums[i];
            } else {
                int l = 1, r = len, pos = 0; // 如果找不到说明所有的数都比 nums[i] 大，此时要更新 d[1]，所以这里将 pos 设为 0
                while (l <= r) {
                    int mid = (l + r) >> 1;
                    if (d[mid] < nums[i]) {
                        pos = mid;
                        l = mid + 1;
                    } else {
                        r = mid - 1;
                    }
                }
                d[pos + 1] = nums[i];
            }
        }
        return len;
    }
}
```

#### **30、最长连续递增序列（ LeetCode 674 ）**

**题目：** 给定一个未经排序的整数数组，找到最长且 连续递增的子序列，并返回该序列的长度。连续递增的子序列 可以由两个下标 l 和 r（l < r）确定，如果对于每个 l <= i < r，都有 nums[i] < nums[i + 1] ，那么子序列 [nums[l], nums[l + 1], ..., nums[r - 1], nums[r]] 就是连续递增子序列。

**示例：** 

    输入：nums = [1,3,5,4,7]
    输出：3
    解释：最长连续递增序列是 [1,3,5], 长度为3。
    尽管 [1,3,5,7] 也是升序的子序列, 但它不是连续的，因为 5 和 7 在原数组里被 4 隔开。 

**题解：** 

```java
class Solution {
    public int findLengthOfLCIS(int[] nums) {
        int ans = 0;
        int n = nums.length;
        int start = 0;
        for (int i = 0; i < n; i++) {
            if (i > 0 && nums[i] <= nums[i - 1]) {
                start = i;
            }
            ans = Math.max(ans, i - start + 1);
        }
        return ans;
    }
}
```

#### **31、分割等和子集（ LeetCode 416 ）**

**题目：** 给你一个 只包含正整数 的 非空 数组 nums 。请你判断是否可以将这个数组分割成两个子集，使得两个子集的元素和相等

**示例：** 

    输入：nums = [1,5,11,5]
    输出：true
    解释：数组可以分割成 [1, 5, 5] 和 [11] 。

**题解：** 

```java
//动态规划
class Solution {
    public boolean canPartition(int[] nums) {
        int n = nums.length;
        if (n < 2) {
            return false;
        }
        int sum = 0, maxNum = 0;
        for (int num : nums) {
            sum += num;
            maxNum = Math.max(maxNum, num);
        }
        if (sum % 2 != 0) {
            return false;
        }
        int target = sum / 2;
        if (maxNum > target) {
            return false;
        }
        boolean[][] dp = new boolean[n][target + 1];
        for (int i = 0; i < n; i++) {
            dp[i][0] = true;
        }
        dp[0][nums[0]] = true;
        for (int i = 1; i < n; i++) {
            int num = nums[i];
            for (int j = 1; j <= target; j++) {
                if (j >= num) {
                    dp[i][j] = dp[i - 1][j] | dp[i - 1][j - num];
                } else {
                    dp[i][j] = dp[i - 1][j];
                }
            }
        }
        return dp[n - 1][target];
    }
}
// 优化
class Solution {
    public boolean canPartition(int[] nums) {
        int n = nums.length;
        if (n < 2) {
            return false;
        }
        int sum = 0, maxNum = 0;
        for (int num : nums) {
            sum += num;
            maxNum = Math.max(maxNum, num);
        }
        if (sum % 2 != 0) {
            return false;
        }
        int target = sum / 2;
        if (maxNum > target) {
            return false;
        }
        boolean[] dp = new boolean[target + 1];
        dp[0] = true;
        for (int i = 0; i < n; i++) {
            int num = nums[i];
            for (int j = target; j >= num; --j) {
                dp[j] |= dp[j - num];
            }
        }
        return dp[target];
    }
}
```

#### **32、最长重复子数组（ LeetCode 718 ）**

**题目：** 给两个整数数组 nums1 和 nums2 ，返回 两个数组中 公共的 、长度最长的子数组的长度 。

**示例：** 

    输入：nums1 = [1,2,3,2,1], nums2 = [3,2,1,4,7]
    输出：3
    解释：长度最长的公共子数组是 [3,2,1] 。

**题解：** 

```java
// 方法一：动态规划
class Solution {
    public int findLength(int[] A, int[] B) {
        int n = A.length, m = B.length;
        int[][] dp = new int[n + 1][m + 1];
        int ans = 0;
        for (int i = n - 1; i >= 0; i--) {
            for (int j = m - 1; j >= 0; j--) {
                dp[i][j] = A[i] == B[j] ? dp[i + 1][j + 1] + 1 : 0;
                ans = Math.max(ans, dp[i][j]);
            }
        }
        return ans;
    }
}
// 方法二：滑动窗口
class Solution {
    public int findLength(int[] A, int[] B) {
        int n = A.length, m = B.length;
        int ret = 0;
        for (int i = 0; i < n; i++) {
            int len = Math.min(m, n - i);
            int maxlen = maxLength(A, B, i, 0, len);
            ret = Math.max(ret, maxlen);
        }
        for (int i = 0; i < m; i++) {
            int len = Math.min(n, m - i);
            int maxlen = maxLength(A, B, 0, i, len);
            ret = Math.max(ret, maxlen);
        }
        return ret;
    }

    public int maxLength(int[] A, int[] B, int addA, int addB, int len) {
        int ret = 0, k = 0;
        for (int i = 0; i < len; i++) {
            if (A[addA + i] == B[addB + i]) {
                k++;
            } else {
                k = 0;
            }
            ret = Math.max(ret, k);
        }
        return ret;
    }
}
```

#### **33、最长公共子序列（ LeetCode 1143 ）**

**题目：** 给定两个字符串 text1 和 text2，返回这两个字符串的最长 公共子序列 的长度。如果不存在 公共子序列 ，返回 0 。一个字符串的 子序列 是指这样一个新的字符串：它是由原字符串在不改变字符的相对顺序的情况下删除某些字符（也可以不删除任何字符）后组成的新字符串。例如，"ace" 是 "abcde" 的子序列，但 "aec" 不是 "abcde" 的子序列。两个字符串的 公共子序列 是这两个字符串所共同拥有的子序列。

**示例：** 

    输入：text1 = "abcde", text2 = "ace" 
    输出：3  
    解释：最长公共子序列是 "ace" ，它的长度为 3 。

**题解：** 

```java
// 动态规划
class Solution {
    public int longestCommonSubsequence(String text1, String text2) {
        int m = text1.length(), n = text2.length();
        int[][] dp = new int[m + 1][n + 1];
        for (int i = 1; i <= m; i++) {
            char c1 = text1.charAt(i - 1);
            for (int j = 1; j <= n; j++) {
                char c2 = text2.charAt(j - 1);
                if (c1 == c2) {
                    dp[i][j] = dp[i - 1][j - 1] + 1;
                } else {
                    dp[i][j] = Math.max(dp[i - 1][j], dp[i][j - 1]);
                }
            }
        }
        return dp[m][n];
    }
}
```

#### **34、最长回文子序列（ LeetCode 516 ）**

**题目：** 给你一个字符串 s ，找出其中最长的回文子序列，并返回该序列的长度。子序列定义为：不改变剩余字符顺序的情况下，删除某些字符或者不删除任何字符形成的一个序列。

**示例：** 

    输入：s = "bbbab"
    输出：4
    解释：一个可能的最长回文子序列为 "bbbb" 。

**题解：** 

```java
// 动态规划
class Solution {
    public int longestPalindromeSubseq(String s) {
        int n = s.length();
        int[][] dp = new int[n][n];
        for (int i = n - 1; i >= 0; i--) {
            dp[i][i] = 1;
            char c1 = s.charAt(i);
            for (int j = i + 1; j < n; j++) {
                char c2 = s.charAt(j);
                if (c1 == c2) {
                    dp[i][j] = dp[i + 1][j - 1] + 2;
                } else {
                    dp[i][j] = Math.max(dp[i + 1][j], dp[i][j - 1]);
                }
            }
        }
        return dp[0][n - 1];
    }
}
```

#### **35、最长回文子串（ LeetCode 5 ）**

**题目：** 给你一个字符串 s，找到 s 中最长的回文子串。

**示例：** 

    输入：s = "babad"
    输出："bab"
    解释："aba" 同样是符合题意的答案。

**题解：** 

```java
public class Solution {

    public String longestPalindrome(String s) {
        int len = s.length();
        if (len < 2) {
            return s;
        }

        int maxLen = 1;
        int begin = 0;
        // dp[i][j] 表示 s[i..j] 是否是回文串
        boolean[][] dp = new boolean[len][len];
        // 初始化：所有长度为 1 的子串都是回文串
        for (int i = 0; i < len; i++) {
            dp[i][i] = true;
        }

        char[] charArray = s.toCharArray();
        // 递推开始
        // 先枚举子串长度
        for (int L = 2; L <= len; L++) {
            // 枚举左边界，左边界的上限设置可以宽松一些
            for (int i = 0; i < len; i++) {
                // 由 L 和 i 可以确定右边界，即 j - i + 1 = L 得
                int j = L + i - 1;
                // 如果右边界越界，就可以退出当前循环
                if (j >= len) {
                    break;
                }

                if (charArray[i] != charArray[j]) {
                    dp[i][j] = false;
                } else {
                    if (j - i < 3) {
                        dp[i][j] = true;
                    } else {
                        dp[i][j] = dp[i + 1][j - 1];
                    }
                }

                // 只要 dp[i][L] == true 成立，就表示子串 s[i..L] 是回文，此时记录回文长度和起始位置
                if (dp[i][j] && j - i + 1 > maxLen) {
                    maxLen = j - i + 1;
                    begin = i;
                }
            }
        }
        return s.substring(begin, begin + maxLen);
    }
}
```

#### **36、目标和（ LeetCode 494 ）**

**题目：** 给你一个整数数组 nums 和一个整数 target 。向数组中的每个整数前添加 '+' 或 '-' ，然后串联起所有整数，可以构造一个 表达式 ： 例如，nums = [2, 1] ，可以在 2 之前添加 '+' ，在 1 之前添加 '-' ，然后串联起来得到表达式 "+2-1" 。返回可以通过上述方法构造的、运算结果等于 target 的不同 表达式 的数目。

**示例：** 

    输入：nums = [1,1,1,1,1], target = 3
    输出：5
    解释：一共有 5 种方法让最终目标和为 3 。
    -1 + 1 + 1 + 1 + 1 = 3
    +1 - 1 + 1 + 1 + 1 = 3
    +1 + 1 - 1 + 1 + 1 = 3
    +1 + 1 + 1 - 1 + 1 = 3
    +1 + 1 + 1 + 1 - 1 = 3

**题解：** 

```java
// 回溯
class Solution {
    int count = 0;

    public int findTargetSumWays(int[] nums, int target) {
        backtrack(nums, target, 0, 0);
        return count;
    }

    public void backtrack(int[] nums, int target, int index, int sum) {
        if (index == nums.length) {
            if (sum == target) {
                count++;
            }
        } else {
            backtrack(nums, target, index + 1, sum + nums[index]);
            backtrack(nums, target, index + 1, sum - nums[index]);
        }
    }
}
// 方法二：动态规划
class Solution {
    public int findTargetSumWays(int[] nums, int target) {
        int sum = 0;
        for (int num : nums) {
            sum += num;
        }
        int diff = sum - target;
        if (diff < 0 || diff % 2 != 0) {
            return 0;
        }
        int n = nums.length, neg = diff / 2;
        int[][] dp = new int[n + 1][neg + 1];
        dp[0][0] = 1;
        for (int i = 1; i <= n; i++) {
            int num = nums[i - 1];
            for (int j = 0; j <= neg; j++) {
                dp[i][j] = dp[i - 1][j];
                if (j >= num) {
                    dp[i][j] += dp[i - 1][j - num];
                }
            }
        }
        return dp[n][neg];
    }
}
// 优化
class Solution {
    public int findTargetSumWays(int[] nums, int target) {
        int sum = 0;
        for (int num : nums) {
            sum += num;
        }
        int diff = sum - target;
        if (diff < 0 || diff % 2 != 0) {
            return 0;
        }
        int neg = diff / 2;
        int[] dp = new int[neg + 1];
        dp[0] = 1;
        for (int num : nums) {
            for (int j = neg; j >= num; j--) {
                dp[j] += dp[j - num];
            }
        }
        return dp[neg];
    }
}
```

#### **37、最后一块石头的重量 II（ LeetCode 1049 ）**

**题目：** 有一堆石头，用整数数组 stones 表示。其中 stones[i] 表示第 i 块石头的重量。每一回合，从中选出任意两块石头，然后将它们一起粉碎。假设石头的重量分别为 x 和 y，且 x <= y。那么粉碎的可能结果如下：

    如果 x == y，那么两块石头都会被完全粉碎；
    如果 x != y，那么重量为 x 的石头将会完全粉碎，而重量为 y 的石头新重量为 y-x。

最后，最多只会剩下一块 石头。返回此石头 最小的可能重量 。如果没有石头剩下，就返回 0。

**示例：** 

    输入：stones = [2,7,4,1,8,1]
    输出：1
    解释：
    组合 2 和 4，得到 2，所以数组转化为 [2,7,1,8,1]，
    组合 7 和 8，得到 1，所以数组转化为 [2,1,1,1]，
    组合 2 和 1，得到 1，所以数组转化为 [1,1,1]，
    组合 1 和 1，得到 0，所以数组转化为 [1]，这就是最优值。

**题解：** 

```java
//动态规划
class Solution {
    public int lastStoneWeightII(int[] stones) {
        int sum = 0;
        for (int weight : stones) {
            sum += weight;
        }
        int n = stones.length, m = sum / 2;
        boolean[][] dp = new boolean[n + 1][m + 1];
        dp[0][0] = true;
        for (int i = 0; i < n; ++i) {
            for (int j = 0; j <= m; ++j) {
                if (j < stones[i]) {
                    dp[i + 1][j] = dp[i][j];
                } else {
                    dp[i + 1][j] = dp[i][j] || dp[i][j - stones[i]];
                }
            }
        }
        for (int j = m;; --j) {
            if (dp[n][j]) {
                return sum - 2 * j;
            }
        }
    }
}

// 优化
class Solution {
    public int lastStoneWeightII(int[] stones) {
        int sum = 0;
        for (int weight : stones) {
            sum += weight;
        }
        int m = sum / 2;
        boolean[] dp = new boolean[m + 1];
        dp[0] = true;
        for (int weight : stones) {
            for (int j = m; j >= weight; --j) {
                dp[j] = dp[j] || dp[j - weight];
            }
        }
        for (int j = m;; --j) {
            if (dp[j]) {
                return sum - 2 * j;
            }
        }
    }
}
```

### **八、位运算、二分查找**
#### **1、二分查找（ LeetCode 704 ）**

**题目：** 给定一个 n 个元素有序的（升序）整型数组 nums 和一个目标值 target  ，写一个函数搜索 nums 中的 target，如果目标值存在返回下标，否则返回 -1。

**示例：** 输入: nums = [-1,0,3,5,9,12], target = 9 输出: 4

解释: 9 出现在 nums 中并且下标为 4

**题解：** 

```java
class Solution {
    public int search(int[] nums, int target) {
        int left = 0, right = nums.length - 1;
        while (left <= right) {
            int mid = (right - left) / 2 + left;
            int num = nums[mid];
            if (num == target) {
                return mid;
            } else if (num > target) {
                right = mid - 1;
            } else {
                left = mid + 1;
            }
        }
        return -1;
    }
}
```

#### **2、搜索插入位置（ LeetCode 35 ）**

**题目：** 给定一个排序数组和一个目标值，在数组中找到目标值，并返回其索引。如果目标值不存在于数组中，返回它将会被按顺序插入的位置。请必须使用时间复杂度为 O(log n) 的算法。

**示例：** 输入: nums = [1,3,5,6], target = 5   输出: 2

**题解：** 

```java
class Solution {
    public int searchInsert(int[] nums, int target) {
        int n = nums.length;
        int left = 0, right = n - 1, ans = n;
        while (left <= right) {
            int mid = ((right - left) >> 1) + left;
            if (target <= nums[mid]) {
                ans = mid;
                right = mid - 1;
            } else {
                left = mid + 1;
            }
        }
        return ans;
    }
}
```

#### **3、在排序数组中查找元素的第一个和最后一个位置（ LeetCode 34 ）**

**题目：**  给你一个按照非递减顺序排列的整数数组 nums，和一个目标值 target。请你找出给定目标值在数组中的开始位置和结束位置。如果数组中不存在目标值 target，返回 [-1, -1]。你必须设计并实现时间复杂度为 O(log n) 的算法解决此问题。

**示例：** 输入：nums = [5,7,7,8,8,10], target = 8  输出：[3,4]

**题解：** 

```java
class Solution {
    public int[] searchRange(int[] nums, int target) {
        int leftIdx = binarySearch(nums, target, true);
        int rightIdx = binarySearch(nums, target, false) - 1;
        if (leftIdx <= rightIdx && rightIdx < nums.length && nums[leftIdx] == target && nums[rightIdx] == target) {
            return new int[]{leftIdx, rightIdx};
        } 
        return new int[]{-1, -1};
    }

    public int binarySearch(int[] nums, int target, boolean lower) {
        int left = 0, right = nums.length - 1, ans = nums.length;
        while (left <= right) {
            int mid = (left + right) / 2;
            if (nums[mid] > target || (lower && nums[mid] >= target)) {
                right = mid - 1;
                ans = mid;
            } else {
                left = mid + 1;
            }
        }
        return ans;
    }
}
```

#### **4、搜索旋转排序数组（ LeetCode 33 ）**

**题目：** 整数数组 nums 按升序排列，数组中的值 互不相同 。在传递给函数之前，nums 在预先未知的某个下标 k（0 <= k < nums.length）上进行了 旋转，使数组变为 [nums[k], nums[k+1], ..., nums[n-1], nums[0], nums[1], ..., nums[k-1]]（下标 从 0 开始 计数）。例如， [0,1,2,4,5,6,7] 在下标 3 处经旋转后可能变为 [4,5,6,7,0,1,2] 。给你 旋转后 的数组 nums 和一个整数 target ，如果 nums 中存在这个目标值 target ，则返回它的下标，否则返回 -1 。你必须设计一个时间复杂度为 O(log n) 的算法解决此问题。

**示例：** 输入：nums = [4,5,6,7,0,1,2], target = 0   输出：4

**题解：** 

```java
class Solution {
    public int search(int[] nums, int target) {
        int n = nums.length;
        if (n == 0) {
            return -1;
        }
        if (n == 1) {
            return nums[0] == target ? 0 : -1;
        }
        int l = 0, r = n - 1;
        while (l <= r) {
            int mid = (l + r) / 2;
            if (nums[mid] == target) {
                return mid;
            }
            if (nums[0] <= nums[mid]) {
                if (nums[0] <= target && target < nums[mid]) {
                    r = mid - 1;
                } else {
                    l = mid + 1;
                }
            } else {
                if (nums[mid] < target && target <= nums[n - 1]) {
                    l = mid + 1;
                } else {
                    r = mid - 1;
                }
            }
        }
        return -1;
    }
}
```

#### **5、搜索二维矩阵（ LeetCode 74 ）**

**题目：** 编写一个高效的算法来判断 m x n 矩阵中，是否存在一个目标值。该矩阵具有如下特性：每行中的整数从左到右按升序排列。每行的第一个整数大于前一行的最后一个整数。

**示例：** 输入：matrix = [[1,3,5,7],[10,11,16,20],[23,30,34,60]], target = 3  输出：true

**思路：** 若将矩阵每一行拼接在上一行的末尾，则会得到一个升序数组，我们可以在该数组上二分找到目标元素。代码实现时，可以二分升序数组的下标，将其映射到原矩阵的行和列上。

**题解：** 

```java
class Solution {
    public boolean searchMatrix(int[][] matrix, int target) {
        int m = matrix.length, n = matrix[0].length;
        int low = 0, high = m * n - 1;
        while (low <= high) {
            int mid = (high - low) / 2 + low;
            int x = matrix[mid / n][mid % n];
            if (x < target) {
                low = mid + 1;
            } else if (x > target) {
                high = mid - 1;
            } else {
                return true;
            }
        }
        return false;
    }
}
```

#### **6、寻找两个正序数组的中位数（ LeetCode 4 ）**

**题目：** 给定两个大小分别为 m 和 n 的正序（从小到大）数组 nums1 和 nums2。请你找出并返回这两个正序数组的 中位数 。算法的时间复杂度应该为 O(log (m+n)) 。

**示例：** 输入：nums1 = [1,3], nums2 = [2]  输出：2.00000

解释：合并数组 = [1,2,3] ，中位数 2

**题解：** 

```java
class Solution {
    public double findMedianSortedArrays(int[] nums1, int[] nums2) {
        int length1 = nums1.length, length2 = nums2.length;
        int totalLength = length1 + length2;
        if (totalLength % 2 == 1) {
            int midIndex = totalLength / 2;
            double median = getKthElement(nums1, nums2, midIndex + 1);
            return median;
        } else {
            int midIndex1 = totalLength / 2 - 1, midIndex2 = totalLength / 2;
            double median = (getKthElement(nums1, nums2, midIndex1 + 1) + getKthElement(nums1, nums2, midIndex2 + 1)) / 2.0;
            return median;
        }
    }

    public int getKthElement(int[] nums1, int[] nums2, int k) {
        /* 主要思路：要找到第 k (k>1) 小的元素，那么就取 pivot1 = nums1[k/2-1] 和 pivot2 = nums2[k/2-1] 进行比较
         * 这里的 "/" 表示整除
         * nums1 中小于等于 pivot1 的元素有 nums1[0 .. k/2-2] 共计 k/2-1 个
         * nums2 中小于等于 pivot2 的元素有 nums2[0 .. k/2-2] 共计 k/2-1 个
         * 取 pivot = min(pivot1, pivot2)，两个数组中小于等于 pivot 的元素共计不会超过 (k/2-1) + (k/2-1) <= k-2 个
         * 这样 pivot 本身最大也只能是第 k-1 小的元素
         * 如果 pivot = pivot1，那么 nums1[0 .. k/2-1] 都不可能是第 k 小的元素。把这些元素全部 "删除"，剩下的作为新的 nums1 数组
         * 如果 pivot = pivot2，那么 nums2[0 .. k/2-1] 都不可能是第 k 小的元素。把这些元素全部 "删除"，剩下的作为新的 nums2 数组
         * 由于我们 "删除" 了一些元素（这些元素都比第 k 小的元素要小），因此需要修改 k 的值，减去删除的数的个数
         */

        int length1 = nums1.length, length2 = nums2.length;
        int index1 = 0, index2 = 0;
        int kthElement = 0;

        while (true) {
            // 边界情况
            if (index1 == length1) {
                return nums2[index2 + k - 1];
            }
            if (index2 == length2) {
                return nums1[index1 + k - 1];
            }
            if (k == 1) {
                return Math.min(nums1[index1], nums2[index2]);
            }

            // 正常情况
            int half = k / 2;
            int newIndex1 = Math.min(index1 + half, length1) - 1;
            int newIndex2 = Math.min(index2 + half, length2) - 1;
            int pivot1 = nums1[newIndex1], pivot2 = nums2[newIndex2];
            if (pivot1 <= pivot2) {
                k -= (newIndex1 - index1 + 1);
                index1 = newIndex1 + 1;
            } else {
                k -= (newIndex2 - index2 + 1);
                index2 = newIndex2 + 1;
            }
        }
    }
}
```

#### **7、有效三角形的个数（ LeetCode 611 ）**

**题目：** 给定一个包含非负整数的数组 nums ，返回其中可以组成三角形三条边的三元组个数。

**示例：** 

    输入: nums = [2,2,3,4]
    输出: 3
    解释:有效的组合是: 
    2,3,4 (使用第一个 2)
    2,3,4 (使用第二个 2)
    2,2,3

**题解：** 

```java
class Solution {
    public int triangleNumber(int[] nums) {

        int res = 0;
        Arrays.sort(nums);
        for (int i = 0; i < nums.length; i++) {
            for (int j = i - 1, k = 0; k < j; j--) {
                while (k < j && nums[k] + nums[j] <= nums[i]) {
                    k++;
                }
                res+=j-k;
            }
        }
        return res;

    }
}
```

#### **8、寻找峰值（ LeetCode 162 ）**

**题目：** 峰值元素是指其值严格大于左右相邻值的元素。给你一个整数数组 nums，找到峰值元素并返回其索引。数组可能包含多个峰值，在这种情况下，返回 任何一个峰值 所在位置即可。你可以假设 nums[-1] = nums[n] = -∞ 。你必须实现时间复杂度为 O(log n) 的算法来解决此问题。

**示例：** 输入：nums = [1,2,3,1]  输出：2

解释：3 是峰值元素，你的函数应该返回其索引 2。

**题解：** 

```java
// 方法一：
class Solution {
    public int findPeakElement(int[] nums) {
        int idx = 0;
        for (int i = 1; i < nums.length; ++i) {
            if (nums[i] > nums[idx]) {
                idx = i;
            }
        }
        return idx;
    }
}

// 方法二：
class Solution {
    public int findPeakElement(int[] nums) {
        int n = nums.length;
        int idx = (int) (Math.random() * n);

        while (!(compare(nums, idx - 1, idx) < 0 && compare(nums, idx, idx + 1) > 0)) {
            if (compare(nums, idx, idx + 1) < 0) {
                idx += 1;
            } else {
                idx -= 1;
            }
        }

        return idx;
    }

    // 辅助函数，输入下标 i，返回一个二元组 (0/1, nums[i])
    // 方便处理 nums[-1] 以及 nums[n] 的边界情况
    public int[] get(int[] nums, int idx) {
        if (idx == -1 || idx == nums.length) {
            return new int[]{0, 0};
        }
        return new int[]{1, nums[idx]};
    }

    public int compare(int[] nums, int idx1, int idx2) {
        int[] num1 = get(nums, idx1);
        int[] num2 = get(nums, idx2);
        if (num1[0] != num2[0]) {
            return num1[0] > num2[0] ? 1 : -1;
        }
        if (num1[1] == num2[1]) {
            return 0;
        }
        return num1[1] > num2[1] ? 1 : -1;
    }
}

//方法三：优化方法二中的二分查找
class Solution {
public:
    int findPeakElement(vector<int>& nums) {
        int n = nums.size();

        // 辅助函数，输入下标 i，返回一个二元组 (0/1, nums[i])
        // 方便处理 nums[-1] 以及 nums[n] 的边界情况
        auto get = [&](int i) -> pair<int, int> {
            if (i == -1 || i == n) {
                return {0, 0};
            }
            return {1, nums[i]};
        };

        int left = 0, right = n - 1, ans = -1;
        while (left <= right) {
            int mid = (left + right) / 2;
            if (get(mid - 1) < get(mid) && get(mid) > get(mid + 1)) {
                ans = mid;
                break;
            }
            if (get(mid) < get(mid + 1)) {
                left = mid + 1;
            }
            else {
                right = mid - 1;
            }
        }
        return ans;
    }
};
```

#### **9、第一个错误的版本（ LeetCode 278 ）**

**题目：** 你是产品经理，目前正在带领一个团队开发新的产品。不幸的是，你的产品的最新版本没有通过质量检测。由于每个版本都是基于之前的版本开发的，所以错误的版本之后的所有版本都是错的。假设你有 n 个版本 [1, 2, ..., n]，你想找出导致之后所有版本出错的第一个错误的版本。你可以通过调用 bool isBadVersion(version) 接口来判断版本号 version 是否在单元测试中出错。实现一个函数来查找第一个错误的版本。你应该尽量减少对调用 API 的次数。

**示例：** 

    输入：n = 5, bad = 4
    输出：4
    解释：
    调用 isBadVersion(3) -> false 
    调用 isBadVersion(5) -> true 
    调用 isBadVersion(4) -> true
    所以，4 是第一个错误的版本。

**题解：** 

```java
public class Solution extends VersionControl {
    public int firstBadVersion(int n) {
        int low = 1;
        int high = n;
        while(low < high){
            int mid = low + (high - low) / 2 ;
            if (isBadVersion(mid)) {
                high = mid;
            }else{
                low = mid + 1;
            }
        }
        return low;
    }
}
```

#### **10、山脉数组的峰顶索引（ LeetCode 852 ）**

**题目：** 符合下列属性的数组 arr 称为 山脉数组 ：

    arr.length >= 3
    存在 i（0 < i < arr.length - 1）使得：
        arr[0] < arr[1] < ... arr[i-1] < arr[i]
        arr[i] > arr[i+1] > ... > arr[arr.length - 1]

给你由整数组成的山脉数组 arr ，返回任何满足 arr[0] < arr[1] < ... arr[i - 1] < arr[i] > arr[i + 1] > ... > arr[arr.length - 1] 的下标 i 。

**示例：** 输入：arr = [0,1,0]  输出：1

**题解：** 

```java
class Solution {
    public int peakIndexInMountainArray(int[] arr) {
        int n = arr.length;
        int left = 1, right = n - 2, ans = 0;
        while (left <= right) {
            int mid = (left + right) / 2;
            if (arr[mid] > arr[mid + 1]) {
                ans = mid;
                right = mid - 1;
            } else {
                left = mid + 1;
            }
        }
        return ans;
    }
}
```

#### **11、有效的完全平方数（ LeetCode 367 ）**

**题目：** 给定一个 正整数 num ，编写一个函数，如果 num 是一个完全平方数，则返回 true ，否则返回 false 。

进阶：不要 使用任何内置的库函数，如  sqrt 。

**示例：** 输入：num = 16  输出：true

**题解：** 

```java
class Solution {
    public boolean isPerfectSquare(int num) {
        int left = 0, right = num;
        while (left <= right) {
            int mid = (right - left) / 2 + left;
            long square = (long) mid * mid;
            if (square < num) {
                left = mid + 1;
            } else if (square > num) {
                right = mid - 1;
            } else {
                return true;
            }
        }
        return false;
    }
}
```

#### **12、2 的幂（ LeetCode 231 ）**

**题目：** 给你一个整数 n，请你判断该整数是否是 2 的幂次方。如果是，返回 true ；否则，返回 false 。如果存在一个整数 x 使得 n == 2x ，则认为 n 是 2 的幂次方。

**示例：** 
输入：n = 1 输出：true   解释：20 = 1

**题解：** 

```java
class Solution {
    public boolean isPowerOfTwo(int n) {
        return n > 0 && (n & (n - 1)) == 0;
    }
}
```

#### **13、比特位计数（ LeetCode 338 ）**

**题目：** 给你一个整数 n ，对于 0 <= i <= n 中的每个 i ，计算其二进制表示中 1 的个数 ，返回一个长度为 n + 1 的数组 ans 作为答案。

**示例：** 

    输入：n = 2
    输出：[0,1,1]
    解释：
    0 --> 0
    1 --> 1
    2 --> 10

**题解：** 

```java
class Solution {
    public int[] countBits(int n) {
        int[] bits = new int[n + 1];
        for(int i = 1; i <= n; i++){
        // 这道题的本质在这里，每个下标与前一个下标进行与运算，然后取出该下标的元素再加1
            bits[i] = bits[(i & (i - 1))] + 1;
        }
        return bits;
    }
}
```

#### **14、位 1 的个数（ LeetCode 191 ）**

**题目：** 编写一个函数，输入是一个无符号整数（以二进制串的形式），返回其二进制表达式中数字位数为 '1' 的个数（也被称为汉明重量）。

**示例：** 

    输入：00000000000000000000000000001011
    输出：3
    解释：输入的二进制串 00000000000000000000000000001011 中，共有三位为 '1'。

**题解：** 

```java
public class Solution {
    public int hammingWeight(int n) {
        int ret = 0;
        while(n != 0){
            n &= n-1;
            ret ++;
        }
        return ret;
    }
}
```

#### **15、只出现一次的数字 II（ LeetCode 137 ）**

**题目：** 给你一个整数数组 nums ，除某个元素仅出现 一次 外，其余每个元素都恰出现 三次 。请你找出并返回那个只出现了一次的元素。

**示例：** 输入：nums = [2,2,3,2]  输出：3

**题解：** 

```java
// 方法一：
class Solution {
    public int singleNumber(int[] nums) {
        int ans = 0;
        for (int i = 0; i < 32; ++i) {
            int total = 0;
            for (int num: nums) {
                total += ((num >> i) & 1);
            }
            if (total % 3 != 0) {
                ans |= (1 << i);
            }
        }
        return ans;
    }
}

// 方法二：（原理复杂，难度大，背）
class Solution {
    public int singleNumber(int[] nums) {
        int a = 0, b = 0;
        for (int num : nums) {
            b = ~a & (b ^ num);
            a = ~b & (a ^ num);
        }
        return b;
    }
}
```

#### **16、只出现一次的数字 III（ LeetCode 260 ）**

**题目：** 给定一个整数数组 nums，其中恰好有两个元素只出现一次，其余所有元素均出现两次。 找出只出现一次的那两个元素。你可以按 任意顺序 返回答案。

**示例：** 

    输入：nums = [1,2,1,3,2,5]  
    输出：[3,5]
    解释：[5, 3] 也是有效的答案。

**题解：** 

```java
class Solution {
    public int[] singleNumber(int[] nums) {
        int xorsum = 0;
        for (int num : nums) {
            xorsum ^= num;
        }
        // 防止溢出
        int lsb = (xorsum == Integer.MIN_VALUE ? xorsum : xorsum & (-xorsum));
        int type1 = 0, type2 = 0;
        for (int num : nums) {
            if ((num & lsb) != 0) {
                type1 ^= num;
            } else {
                type2 ^= num;
            }
        }
        return new int[]{type1, type2};
    }
}
```

#### **17、最大单词长度乘积（ LeetCode 318 ）**

**题目：** 给你一个字符串数组 words ，找出并返回 length(words[i]) * length(words[j]) 的最大值，并且这两个单词不含有公共字母。如果不存在这样的两个单词，返回 0 。

**示例：** 

    输入：words = ["abcw","baz","foo","bar","xtfn","abcdef"]
    输出：16 
    解释：这两个单词为 "abcw", "xtfn"。

**题解：** 

```java
class Solution {
    public int maxProduct(String[] words) {
        int length = words.length;
        int[] masks = new int[length];
        for (int i = 0; i < length; i++) {
            String word = words[i];
            int wordLength = word.length();
            for (int j = 0; j < wordLength; j++) {
                masks[i] |= 1 << (word.charAt(j) - 'a');
            }
        }
        int maxProd = 0;
        for (int i = 0; i < length; i++) {
            for (int j = i + 1; j < length; j++) {
                if ((masks[i] & masks[j]) == 0) {
                    maxProd = Math.max(maxProd, words[i].length() * words[j].length());
                }
            }
        }
        return maxProd;
    }
}
```

#### **18、汉明距离（ LeetCode 461 ）**

**题目：** 两个整数之间的 汉明距离 指的是这两个数字对应二进制位不同的位置的数目。给你两个整数 x 和 y，计算并返回它们之间的汉明距离。

**示例：** 

    输入：x = 1, y = 4
    输出：2
    解释：
    1   (0 0 0 1)
    4   (0 1 0 0)
        ↑   ↑
    上面的箭头指出了对应二进制位不同的位置。

**题解：** 

```java
//方法一：
class Solution {
    public int hammingDistance(int x, int y) {
        return Integer.bitCount(x ^ y);
    }
}
// 方法二：
class Solution {
    public int hammingDistance(int x, int y) {
        int s = x ^ y, ret = 0;
        while (s != 0) {
            s &= s - 1;
            ret++;
        }
        return ret;
    }
}
```

#### **19、岛屿数量（ LeetCode 200）**

**题目：** 给你一个由 '1'（陆地）和 '0'（水）组成的的二维网格，请你计算网格中岛屿的数量。岛屿总是被水包围，并且每座岛屿只能由水平方向和/或竖直方向上相邻的陆地连接形成。此外，你可以假设该网格的四条边均被水包围。

**示例：** 

    输入：grid = [
    ["1","1","1","1","0"],
    ["1","1","0","1","0"],
    ["1","1","0","0","0"],
    ["0","0","0","0","0"]
    ]
    输出：1

**题解：** 

```java
// 方法一：深度优先搜索
class Solution {
    void dfs(char[][] grid, int r, int c) {
        int nr = grid.length;
        int nc = grid[0].length;

        if (r < 0 || c < 0 || r >= nr || c >= nc || grid[r][c] == '0') {
            return;
        }

        grid[r][c] = '0';
        dfs(grid, r - 1, c);
        dfs(grid, r + 1, c);
        dfs(grid, r, c - 1);
        dfs(grid, r, c + 1);
    }

    public int numIslands(char[][] grid) {
        if (grid == null || grid.length == 0) {
            return 0;
        }

        int nr = grid.length;
        int nc = grid[0].length;
        int num_islands = 0;
        for (int r = 0; r < nr; ++r) {
            for (int c = 0; c < nc; ++c) {
                if (grid[r][c] == '1') {
                    ++num_islands;
                    dfs(grid, r, c);
                }
            }
        }

        return num_islands;
    }
}

// 方法二：广度优先搜索
class Solution {
    public int numIslands(char[][] grid) {
        if (grid == null || grid.length == 0) {
            return 0;
        }

        int nr = grid.length;
        int nc = grid[0].length;
        int num_islands = 0;

        for (int r = 0; r < nr; ++r) {
            for (int c = 0; c < nc; ++c) {
                if (grid[r][c] == '1') {
                    ++num_islands;
                    grid[r][c] = '0';
                    Queue<Integer> neighbors = new LinkedList<>();
                    neighbors.add(r * nc + c);
                    while (!neighbors.isEmpty()) {
                        int id = neighbors.remove();
                        int row = id / nc;
                        int col = id % nc;
                        if (row - 1 >= 0 && grid[row-1][col] == '1') {
                            neighbors.add((row-1) * nc + col);
                            grid[row-1][col] = '0';
                        }
                        if (row + 1 < nr && grid[row+1][col] == '1') {
                            neighbors.add((row+1) * nc + col);
                            grid[row+1][col] = '0';
                        }
                        if (col - 1 >= 0 && grid[row][col-1] == '1') {
                            neighbors.add(row * nc + col-1);
                            grid[row][col-1] = '0';
                        }
                        if (col + 1 < nc && grid[row][col+1] == '1') {
                            neighbors.add(row * nc + col+1);
                            grid[row][col+1] = '0';
                        }
                    }
                }
            }
        }

        return num_islands;
    }
}
```

#### **20、N 皇后（ LeetCode 51 ）**

**题目：** 按照国际象棋的规则，皇后可以攻击与之处在同一行或同一列或同一斜线上的棋子。n 皇后问题 研究的是如何将 n 个皇后放置在 n×n 的棋盘上，并且使皇后彼此之间不能相互攻击。给你一个整数 n ，返回所有不同的 n 皇后问题 的解决方案。每一种解法包含一个不同的 n 皇后问题 的棋子放置方案，该方案中 'Q' 和 '.' 分别代表了皇后和空位。

**示例：** 

    输入：n = 4
    输出：[[".Q..","...Q","Q...","..Q."],["..Q.","Q...","...Q",".Q.."]]
    解释：如上图所示，4 皇后问题存在两个不同的解法。

**题解：** 

```java
// 方法一：
class Solution {
    public List<List<String>> solveNQueens(int n) {
        List<List<String>> solutions = new ArrayList<List<String>>();
        int[] queens = new int[n];
        Arrays.fill(queens, -1);
        Set<Integer> columns = new HashSet<Integer>();
        Set<Integer> diagonals1 = new HashSet<Integer>();
        Set<Integer> diagonals2 = new HashSet<Integer>();
        backtrack(solutions, queens, n, 0, columns, diagonals1, diagonals2);
        return solutions;
    }

    public void backtrack(List<List<String>> solutions, int[] queens, int n, int row, Set<Integer> columns, Set<Integer> diagonals1, Set<Integer> diagonals2) {
        if (row == n) {
            List<String> board = generateBoard(queens, n);
            solutions.add(board);
        } else {
            for (int i = 0; i < n; i++) {
                if (columns.contains(i)) {
                    continue;
                }
                int diagonal1 = row - i;
                if (diagonals1.contains(diagonal1)) {
                    continue;
                }
                int diagonal2 = row + i;
                if (diagonals2.contains(diagonal2)) {
                    continue;
                }
                queens[row] = i;
                columns.add(i);
                diagonals1.add(diagonal1);
                diagonals2.add(diagonal2);
                backtrack(solutions, queens, n, row + 1, columns, diagonals1, diagonals2);
                queens[row] = -1;
                columns.remove(i);
                diagonals1.remove(diagonal1);
                diagonals2.remove(diagonal2);
            }
        }
    }

    public List<String> generateBoard(int[] queens, int n) {
        List<String> board = new ArrayList<String>();
        for (int i = 0; i < n; i++) {
            char[] row = new char[n];
            Arrays.fill(row, '.');
            row[queens[i]] = 'Q';
            board.add(new String(row));
        }
        return board;
    }
}

//方法二：
class Solution {
    public List<List<String>> solveNQueens(int n) {
        int[] queens = new int[n];
        Arrays.fill(queens, -1);
        List<List<String>> solutions = new ArrayList<List<String>>();
        solve(solutions, queens, n, 0, 0, 0, 0);
        return solutions;
    }

    public void solve(List<List<String>> solutions, int[] queens, int n, int row, int columns, int diagonals1, int diagonals2) {
        if (row == n) {
            List<String> board = generateBoard(queens, n);
            solutions.add(board);
        } else {
            int availablePositions = ((1 << n) - 1) & (~(columns | diagonals1 | diagonals2));
            while (availablePositions != 0) {
                int position = availablePositions & (-availablePositions);
                availablePositions = availablePositions & (availablePositions - 1);
                int column = Integer.bitCount(position - 1);
                queens[row] = column;
                solve(solutions, queens, n, row + 1, columns | position, (diagonals1 | position) << 1, (diagonals2 | position) >> 1);
                queens[row] = -1;
            }
        }
    }

    public List<String> generateBoard(int[] queens, int n) {
        List<String> board = new ArrayList<String>();
        for (int i = 0; i < n; i++) {
            char[] row = new char[n];
            Arrays.fill(row, '.');
            row[queens[i]] = 'Q';
            board.add(new String(row));
        }
        return board;
    }
}
```

#### **21、子集（ LeetCode 78 ）**

**题目：** 给你一个整数数组 nums ，数组中的元素 互不相同 。返回该数组所有可能的子集（幂集）。解集 不能 包含重复的子集。你可以按 任意顺序 返回解集。

**示例：** 

    输入：nums = [1,2,3]
    输出：[[],[1],[2],[1,2],[3],[1,3],[2,3],[1,2,3]]

**题解：** 

```java
// 方法一：
class Solution {
    List<Integer> t = new ArrayList<Integer>();
    List<List<Integer>> ans = new ArrayList<List<Integer>>();

    public List<List<Integer>> subsets(int[] nums) {
        int n = nums.length;
        for (int mask = 0; mask < (1 << n); ++mask) {
            t.clear();
            for (int i = 0; i < n; ++i) {
                if ((mask & (1 << i)) != 0) {
                    t.add(nums[i]);
                }
            }
            ans.add(new ArrayList<Integer>(t));
        }
        return ans;
    }
}
// 方法二：
class Solution {
    List<Integer> t = new ArrayList<Integer>();
    List<List<Integer>> ans = new ArrayList<List<Integer>>();

    public List<List<Integer>> subsets(int[] nums) {
        dfs(0, nums);
        return ans;
    }

    public void dfs(int cur, int[] nums) {
        if (cur == nums.length) {
            ans.add(new ArrayList<Integer>(t));
            return;
        }
        t.add(nums[cur]);
        dfs(cur + 1, nums);
        t.remove(t.size() - 1);
        dfs(cur + 1, nums);
    }
}
```

#### **22、组合总和 II（ LeetCode 39 ）**

**题目：** 给定一个候选人编号的集合 candidates 和一个目标数 target ，找出 candidates 中所有可以使数字和为 target 的组合。candidates 中的每个数字在每个组合中只能使用 一次 。注意：解集不能包含重复的组合。 

**示例：** 

    输入: candidates = [10,1,2,7,6,1,5], target = 8,
    输出:
    [
    [1,1,6],
    [1,2,5],
    [1,7],
    [2,6]
    ]

**题解：** 

```java
class Solution {
    List<int[]> freq = new ArrayList<int[]>();
    List<List<Integer>> ans = new ArrayList<List<Integer>>();
    List<Integer> sequence = new ArrayList<Integer>();

    public List<List<Integer>> combinationSum2(int[] candidates, int target) {
        Arrays.sort(candidates);
        for (int num : candidates) {
            int size = freq.size();
            if (freq.isEmpty() || num != freq.get(size - 1)[0]) {
                freq.add(new int[]{num, 1});
            } else {
                ++freq.get(size - 1)[1];
            }
        }
        dfs(0, target);
        return ans;
    }

    public void dfs(int pos, int rest) {
        if (rest == 0) {
            ans.add(new ArrayList<Integer>(sequence));
            return;
        }
        if (pos == freq.size() || rest < freq.get(pos)[0]) {
            return;
        }

        dfs(pos + 1, rest);

        int most = Math.min(rest / freq.get(pos)[0], freq.get(pos)[1]);
        for (int i = 1; i <= most; ++i) {
            sequence.add(freq.get(pos)[0]);
            dfs(pos + 1, rest - i * freq.get(pos)[0]);
        }
        for (int i = 1; i <= most; ++i) {
            sequence.remove(sequence.size() - 1);
        }
    }
}
```

#### **23、括号生成（ LeetCode 22 ）**

**题目：** 数字 n 代表生成括号的对数，请你设计一个函数，用于能够生成所有可能的并且 有效的 括号组合。

**示例：** 

    输入：n = 3
    输出：["((()))","(()())","(())()","()(())","()()()"]

**题解：** 

```java
class Solution {
    public List<String> generateParenthesis(int n) {
        List<String> ans = new ArrayList<String>();
        backtrack(ans, new StringBuilder(), 0, 0, n);
        return ans;
    }

    public void backtrack(List<String> ans, StringBuilder cur, int open, int close, int max) {
        if (cur.length() == max * 2) {
            ans.add(cur.toString());
            return;
        }
        if (open < max) {
            cur.append('(');
            backtrack(ans, cur, open + 1, close, max);
            cur.deleteCharAt(cur.length() - 1);
        }
        if (close < open) {
            cur.append(')');
            backtrack(ans, cur, open, close + 1, max);
            cur.deleteCharAt(cur.length() - 1);
        }
    }
}
```

#### **24、火柴拼正方形（ LeetCode 437 ）**

**题目：** 给定一个二叉树的根节点 root ，和一个整数 targetSum ，求该二叉树里节点值之和等于 targetSum 的 路径 的数目。

路径 不需要从根节点开始，也不需要在叶子节点结束，但是路径方向必须是向下的（只能从父节点到子节点）

**示例：** 

    输入：root = [10,5,-3,3,2,null,11,3,-2,null,1], targetSum = 8
    输出：3
    解释：和等于 8 的路径有 3 条，如图所示。

**题解：** 

```java
// 深度优先
class Solution {
    public int pathSum(TreeNode root, int targetSum) {
        if (root == null) {
            return 0;
        }

        int ret = rootSum(root, targetSum);
        ret += pathSum(root.left, targetSum);
        ret += pathSum(root.right, targetSum);
        return ret;
    }

    public int rootSum(TreeNode root, int targetSum) {
        int ret = 0;

        if (root == null) {
            return 0;
        }
        int val = root.val;
        if (val == targetSum) {
            ret++;
        } 

        ret += rootSum(root.left, targetSum - val);
        ret += rootSum(root.right, targetSum - val);
        return ret;
    }
}

// 前缀和class Solution {
    public int pathSum(TreeNode root, int targetSum) {
        Map<Long, Integer> prefix = new HashMap<Long, Integer>();
        prefix.put(0L, 1);
        return dfs(root, prefix, 0, targetSum);
    }

    public int dfs(TreeNode root, Map<Long, Integer> prefix, long curr, int targetSum) {
        if (root == null) {
            return 0;
        }

        int ret = 0;
        curr += root.val;

        ret = prefix.getOrDefault(curr - targetSum, 0);
        prefix.put(curr, prefix.getOrDefault(curr, 0) + 1);
        ret += dfs(root.left, prefix, curr, targetSum);
        ret += dfs(root.right, prefix, curr, targetSum);
        prefix.put(curr, prefix.getOrDefault(curr, 0) - 1);

        return ret;
    }
}
```

#### **25、接雨水 II（ LeetCode 407 ）**

**题目：** 给你一个 m x n 的矩阵，其中的值均为非负整数，代表二维高度图每个单元的高度，请计算图中形状最多能接多少体积的雨水。

**示例：** 

    输入: heightMap = [[1,4,3,1,3,2],[3,2,1,3,2,4],[2,3,3,2,3,1]]
    输出: 4
    解释: 下雨后，雨水将会被上图蓝色的方块中。总的接雨水量为1+2+1=4。

**题解：** 

```java
// 最小堆
class Solution {
    public int trapRainWater(int[][] heightMap) {
        if (heightMap.length <= 2 || heightMap[0].length <= 2) {
            return 0;
        }
        int m = heightMap.length;
        int n = heightMap[0].length;
        boolean[][] visit = new boolean[m][n];
        PriorityQueue<int[]> pq = new PriorityQueue<>((o1, o2) -> o1[1] - o2[1]);

        for (int i = 0; i < m; ++i) {
            for (int j = 0; j < n; ++j) {
                if (i == 0 || i == m - 1 || j == 0 || j == n - 1) {
                    pq.offer(new int[]{i * n + j, heightMap[i][j]});
                    visit[i][j] = true;
                }
            }
        }
        int res = 0;
        int[] dirs = {-1, 0, 1, 0, -1};
        while (!pq.isEmpty()) {
            int[] curr = pq.poll();
            for (int k = 0; k < 4; ++k) {
                int nx = curr[0] / n + dirs[k];
                int ny = curr[0] % n + dirs[k + 1];
                if (nx >= 0 && nx < m && ny >= 0 && ny < n && !visit[nx][ny]) {
                    if (curr[1] > heightMap[nx][ny]) {
                        res += curr[1] - heightMap[nx][ny];
                    }
                    pq.offer(new int[]{nx * n + ny, Math.max(heightMap[nx][ny], curr[1])});
                    visit[nx][ny] = true;
                }
            }
        }
        return res;
    }
}

// 广度优先搜索
class Solution {
    public int trapRainWater(int[][] heightMap) {
        int m = heightMap.length;
        int n = heightMap[0].length;
        int[] dirs = {-1, 0, 1, 0, -1};
        int maxHeight = 0;

        for (int i = 0; i < m; ++i) {
            for (int j = 0; j < n; ++j) {
                maxHeight = Math.max(maxHeight, heightMap[i][j]);
            }
        }
        int[][] water = new int[m][n];
        for (int i = 0; i < m; ++i) {
            for (int j = 0; j < n; ++j){
                water[i][j] = maxHeight;      
            }
        }  
        Queue<int[]> qu = new LinkedList<>();
        for (int i = 0; i < m; ++i) {
            for (int j = 0; j < n; ++j) {
                if (i == 0 || i == m - 1 || j == 0 || j == n - 1) {
                    if (water[i][j] > heightMap[i][j]) {
                        water[i][j] = heightMap[i][j];
                        qu.offer(new int[]{i, j});
                    }
                }
            }
        } 
        while (!qu.isEmpty()) {
            int[] curr = qu.poll();
            int x = curr[0];
            int y = curr[1];
            for (int i = 0; i < 4; ++i) {
                int nx = x + dirs[i], ny = y + dirs[i + 1];
                if (nx < 0 || nx >= m || ny < 0 || ny >= n) {
                    continue;
                }
                if (water[x][y] < water[nx][ny] && water[nx][ny] > heightMap[nx][ny]) {
                    water[nx][ny] = Math.max(water[x][y], heightMap[nx][ny]);
                    qu.offer(new int[]{nx, ny});
                }
            }
        }

        int res = 0;
        for (int i = 0; i < m; ++i) {
            for (int j = 0; j < n; ++j) {
                res += water[i][j] - heightMap[i][j];
            }
        }
        return res;
    }
}
```

#### **26、组合（ LeetCode 77 ）**

**题目：** 给定两个整数 n 和 k，返回范围 [1, n] 中所有可能的 k 个数的组合。 

**示例：** 

    输入：n = 4, k = 2
    输出：
    [
    [2,4],
    [3,4],
    [2,3],
    [1,2],
    [1,3],
    [1,4],
    ]

**题解：** 

```java
class Solution {
    List<Integer> temp = new ArrayList<Integer>();
    List<List<Integer>> ans = new ArrayList<List<Integer>>();

    public List<List<Integer>> combine(int n, int k) {
        List<Integer> temp = new ArrayList<Integer>();
        List<List<Integer>> ans = new ArrayList<List<Integer>>();
        // 初始化
        // 将 temp 中 [0, k - 1] 每个位置 i 设置为 i + 1，即 [0, k - 1] 存 [1, k]
        // 末尾加一位 n + 1 作为哨兵
        for (int i = 1; i <= k; ++i) {
            temp.add(i);
        }
        temp.add(n + 1);

        int j = 0;
        while (j < k) {
            ans.add(new ArrayList<Integer>(temp.subList(0, k)));
            j = 0;
            // 寻找第一个 temp[j] + 1 != temp[j + 1] 的位置 t
            // 我们需要把 [0, t - 1] 区间内的每个位置重置成 [1, t]
            while (j < k && temp.get(j) + 1 == temp.get(j + 1)) {
                temp.set(j, j + 1);
                ++j;
            }
            // j 是第一个 temp[j] + 1 != temp[j + 1] 的位置
            temp.set(j, temp.get(j) + 1);
        }
        return ans;
    }
}
```

#### **27、组合总和 II（ LeetCode 216 ）**

**题目：** 找出所有相加之和为 n 的 k 个数的组合，且满足下列条件：

    只使用数字1到9
    每个数字 最多使用一次 

返回 所有可能的有效组合的列表 。该列表不能包含相同的组合两次，组合可以以任何顺序返回。

**示例：** 

    输入: k = 3, n = 7
    输出: [[1,2,4]]
    解释:
    1 + 2 + 4 = 7
    没有其他符合的组合了。

**题解：** 

```java
// 方法一：
class Solution {
    List<Integer> temp = new ArrayList<Integer>();
    List<List<Integer>> ans = new ArrayList<List<Integer>>();

    public List<List<Integer>> combinationSum3(int k, int n) {
        for (int mask = 0; mask < (1 << 9); ++mask) {
            if (check(mask, k, n)) {
                ans.add(new ArrayList<Integer>(temp));
            }
        }
        return ans;
    }

    public boolean check(int mask, int k, int n) {
        temp.clear();
        for (int i = 0; i < 9; ++i) {
            if (((1 << i) & mask) != 0) {
                temp.add(i + 1);
            }
        }
        if (temp.size() != k) {
            return false;
        }
        int sum = 0;
        for (int num : temp) {
            sum += num;
        }
        return sum == n;
    }
}
// 方法额二
class Solution {
    List<Integer> temp = new ArrayList<Integer>();
    List<List<Integer>> ans = new ArrayList<List<Integer>>();

    public List<List<Integer>> combinationSum3(int k, int n) {
        dfs(1, 9, k, n);
        return ans;
    }

    public void dfs(int cur, int n, int k, int sum) {
        if (temp.size() + (n - cur + 1) < k || temp.size() > k) {
            return;
        }
        if (temp.size() == k) {
            int tempSum = 0;
            for (int num : temp) {
                tempSum += num;
            }
            if (tempSum == sum) {
                ans.add(new ArrayList<Integer>(temp));
                return;
            }
        }
        temp.add(cur);
        dfs(cur + 1, n, k, sum);
        temp.remove(temp.size() - 1);
        dfs(cur + 1, n, k, sum);
    }
}
```

#### **28、分割回文串（ LeetCode 131 ）**

**题目：** 给你一个字符串 s，请你将 s 分割成一些子串，使每个子串都是 回文串 。返回 s 所有可能的分割方案。回文串 是正着读和反着读都一样的字符串。

**示例：** 

    输入：s = "aab"
    输出：[["a","a","b"],["aa","b"]]

**题解：** 

```java
// 方法一：
class Solution {
    boolean[][] f;
    List<List<String>> ret = new ArrayList<List<String>>();
    List<String> ans = new ArrayList<String>();
    int n;

    public List<List<String>> partition(String s) {
        n = s.length();
        f = new boolean[n][n];
        for (int i = 0; i < n; ++i) {
            Arrays.fill(f[i], true);
        }

        for (int i = n - 1; i >= 0; --i) {
            for (int j = i + 1; j < n; ++j) {
                f[i][j] = (s.charAt(i) == s.charAt(j)) && f[i + 1][j - 1];
            }
        }

        dfs(s, 0);
        return ret;
    }

    public void dfs(String s, int i) {
        if (i == n) {
            ret.add(new ArrayList<String>(ans));
            return;
        }
        for (int j = i; j < n; ++j) {
            if (f[i][j]) {
                ans.add(s.substring(i, j + 1));
                dfs(s, j + 1);
                ans.remove(ans.size() - 1);
            }
        }
    }
}
// 方法二：
class Solution {
    int[][] f;
    List<List<String>> ret = new ArrayList<List<String>>();
    List<String> ans = new ArrayList<String>();
    int n;

    public List<List<String>> partition(String s) {
        n = s.length();
        f = new int[n][n];

        dfs(s, 0);
        return ret;
    }

    public void dfs(String s, int i) {
        if (i == n) {
            ret.add(new ArrayList<String>(ans));
            return;
        }
        for (int j = i; j < n; ++j) {
            if (isPalindrome(s, i, j) == 1) {
                ans.add(s.substring(i, j + 1));
                dfs(s, j + 1);
                ans.remove(ans.size() - 1);
            }
        }
    }

    // 记忆化搜索中，f[i][j] = 0 表示未搜索，1 表示是回文串，-1 表示不是回文串
    public int isPalindrome(String s, int i, int j) {
        if (f[i][j] != 0) {
            return f[i][j];
        }
        if (i >= j) {
            f[i][j] = 1;
        } else if (s.charAt(i) == s.charAt(j)) {
            f[i][j] = isPalindrome(s, i + 1, j - 1);
        } else {
            f[i][j] = -1;
        }
        return f[i][j];
    }
}
```

#### **29、全排列（ LeetCode 46 ）**

**题目：** 给定一个不含重复数字的数组 nums ，返回其 所有可能的全排列 。你可以 按任意顺序 返回答案。

**示例：** 

    输入：nums = [1,2,3]
    输出：[[1,2,3],[1,3,2],[2,1,3],[2,3,1],[3,1,2],[3,2,1]]

**题解：** 

```java
class Solution {
    public List<List<Integer>> permute(int[] nums) {
        List<List<Integer>> res = new ArrayList<List<Integer>>();

        List<Integer> output = new ArrayList<Integer>();
        for (int num : nums) {
            output.add(num);
        }

        int n = nums.length;
        backtrack(n, output, res, 0);
        return res;
    }

    public void backtrack(int n, List<Integer> output, List<List<Integer>> res, int first) {
        // 所有数都填完了
        if (first == n) {
            res.add(new ArrayList<Integer>(output));
        }
        for (int i = first; i < n; i++) {
            // 动态维护数组
            Collections.swap(output, first, i);
            // 继续递归填下一个数
            backtrack(n, output, res, first + 1);
            // 撤销操作
            Collections.swap(output, first, i);
        }
    }
}
```

#### **30、电话号码的字母组合（ LeetCode 17 ）**

**题目：**

**示例：** 

**题解：** 

```java

```

#### **31、复原ip地址（ LeetCode 93 ）**

**题目：**

**示例：** 

**题解：** 

```java

```


#### **33、全排列Ⅱ（ LeetCode 47 ）**

**题目：**

**示例：** 

**题解：** 

```java

```

#### **34、重新安排行程（ LeetCode 332 ）**

**题目：**

**示例：** 

**题解：** 

```java

```

#### **35、解数独（ LeetCode 37 ）**

**题目：**

**示例：** 

**题解：** 

```java

```
#### **36、快排**

**题目：**

**示例：** 

**题解：** 

```java

```
#### **37、归并**

**题目：**

**示例：** 

**题解：** 

```java

```
#### **38、堆排**

**题目：**

**示例：** 

**题解：** 

```java

```
#### **39、冒泡排序**

**题目：**

**示例：** 

**题解：** 

```java

```

#### **40、选择排序**

**题目：**

**示例：** 

**题解：** 

```java

```

#### **41、计数排序**

**题目：**

**示例：** 

**题解：** 

```java

```

### **六、前缀和计算**
#### **1、区域和检索 - 数组不可变（ LeetCode 303 ）**

**题目：** 给定一个整数数组  nums，处理以下类型的多个查询:计算索引 left 和 right （包含 left 和 right）之间的 nums 元素的 和 ，其中 left <= right
```
实现 NumArray 类：

    NumArray(int[] nums) 使用数组 nums 初始化对象
    int sumRange(int i, int j) 返回数组 nums 中索引 left 和 right 之间的元素的 总和 ，包含 left 和 right 两点（也就是 nums[left] + nums[left + 1] + ... + nums[right] )
```
**示例：** 
```
输入：
["NumArray", "sumRange", "sumRange", "sumRange"]
[[[-2, 0, 3, -5, 2, -1]], [0, 2], [2, 5], [0, 5]]
输出：
[null, 1, -1, -3]

解释：
NumArray numArray = new NumArray([-2, 0, 3, -5, 2, -1]);
numArray.sumRange(0, 2); // return 1 ((-2) + 0 + 3)
numArray.sumRange(2, 5); // return -1 (3 + (-5) + 2 + (-1)) 
numArray.sumRange(0, 5); // return -3 ((-2) + 0 + 3 + (-5) + 2 + (-1))
```

**题解：** 

```java
public class LeetCode303 {
    // 先定义一个前缀和数组
    private int[] prevNums;
    public LeetCode303(int[] nums) {
        // 在构造函数中，完成前缀数组的初始化和计算
        prevNums = new int[nums.length + 1];
        // 合理利用数组的下标。
        // 前缀和数组中，从1下标开始的结果，等于前缀和数组前一个下标i的值加上原本数组这个下标i的值
        for(int i = 1; i < nums.length; i++) {
            prevNums[i] = prevNums[i-1] + nums[i-1];
        }
    }
    public int sumRange(int left, int right) {
        // 最后求区间的和时，就是右标+1的值减去左标的值
        return prevNums[right + 1] - prevNums[left];
    }
}
```
#### **2、二维区域和检索 - 矩阵不可变（ LeetCode 304 ）**

**题目：** 给定一个二维矩阵 matrix，以下类型的多个请求：

    计算其子矩形范围内元素的总和，该子矩阵的 左上角 为 (row1, col1) ，右下角 为 (row2, col2) 。
```
实现 NumMatrix 类：

    NumMatrix(int[][] matrix) 给定整数矩阵 matrix 进行初始化
    int sumRegion(int row1, int col1, int row2, int col2) 返回 左上角 (row1, col1) 、右下角 (row2, col2) 所描述的子矩阵的元素 总和 。
```

**示例：** 
```
输入: 
["NumMatrix","sumRegion","sumRegion","sumRegion"]
[[[[3,0,1,4,2],[5,6,3,2,1],[1,2,0,1,5],[4,1,0,1,7],[1,0,3,0,5]]],[2,1,4,3],[1,1,2,2],[1,2,2,4]]
输出: 
[null, 8, 11, 12]

解释:
NumMatrix numMatrix = new NumMatrix([[3,0,1,4,2],[5,6,3,2,1],[1,2,0,1,5],[4,1,0,1,7],[1,0,3,0,5]]);
numMatrix.sumRegion(2, 1, 4, 3); // return 8 (红色矩形框的元素总和)
numMatrix.sumRegion(1, 1, 2, 2); // return 11 (绿色矩形框的元素总和)
numMatrix.sumRegion(1, 2, 2, 4); // return 12 (蓝色矩形框的元素总和)
```

**题解：** 

```java
class NumMatrix {

    // 定义前缀和数组
    private int[][] preSum;

    public NumMatrix(int[][] matrix) {
        // 取出矩阵的长、宽
        int m = matrix.length, n = matrix[0].length;
        // 循环遍历
        if(m == 0 || n == 0){
            return;
        }
        preSum = new int[m + 1][n + 1];
        for(int i = 1; i <= m; i++) {
            for(int j = 1; j <= n; j++){
                // 计算每个子矩阵 [0, 0, i, j] 的元素和
                preSum[i][j] = preSum[i-1][j] + preSum[i][j-1] + matrix[i - 1][j - 1] - preSum[i-1][j-1];
            }
        }
    }
    
    // 计算子矩阵 [x1, y1, x2, y2] 的元素和
    public int sumRegion(int x1, int y1, int x2, int y2) {
        // 目标矩阵之和由四个相邻矩阵运算获得
        return preSum[x2+1][y2+1] - preSum[x1][y2+1] - preSum[x2+1][y1] + preSum[x1][y1];
    }
}

/**
 * Your NumMatrix object will be instantiated and called as such:
 * NumMatrix obj = new NumMatrix(matrix);
 * int param_1 = obj.sumRegion(row1,col1,row2,col2);
 */
```

### **七、字符串系列**

#### **1. 最长公共前缀**

**题目：** 编写一个函数来查找字符串数组中的最长公共前缀。如果不存在公共前缀，返回空字符串 ""。

**示例：** 
```
输入：strs = ["flower","flow","flight"]
输出："fl"
```
**题解：** 

```java

```

### **八、剑指 Offer 系列**

#### 剑指offer 03.数组中重复的数字

**题目：**找出数组中重复的数字。在一个长度为 n 的数组 nums 里的所有数字都在 0～n-1 的范围内。数组中某些数字是重复的，但不知道有几个数字重复了，也不知道每个数字重复了几次。请找出数组中任意一个重复的数字。

**示例：** 

```
输入：
[2, 3, 1, 0, 2, 5, 3]
输出：2 或 3 
```

**题解：** 

```java
public class Offer3 {

    /**
     * 长度为 n 的数组 nums 里的所有数字都在 0～n-1 的范围内，如果数组中有重复的元素，那么数组元素的索引和值是一对多的关系，
     * 没有重复的话，元素和索引就是一对一的关系
     * 思路：
     * 遍历数组 nums ，设索引初始值为 i=0 :
     *     若 nums[i]=i： 说明此数字已在对应索引位置，无需交换，因此跳过；
     *     若 nums[nums[i]]=nums[i]： 代表索引 nums[i]处和索引 i处的元素值都为 nums[i]，即找到一组重复值，返回此值 nums[i] ；
     *     否则： 交换索引为 i 和 nums[i] 的元素值，将此数字交换至对应索引位置。
     *
     * 若遍历完毕尚未返回，则返回 −1 。
     *
     * @param nums
     * @return
     */
    public static int findRepeatNumber(int[] nums) {
        int i = 0;
        while (i < nums.length) {
            if(nums[i] == i) {
                i ++;
                continue;
            }

            // 如果当前下标的元素 和 以这个元素为下标的位置处的元素相同，则表示有重复元素
            if(nums[i] == nums[nums[i]]){
                return nums[i];
            }

            int temp = nums[i];
            nums[i] = nums[temp];
            nums[temp] = temp;
        }
        return -1;
    }

    public static void main(String[] args) {
        int[] nums = new int[]{2,3,1,0,2,5,3};
        System.out.println(findRepeatNumber(nums));
    }

}
```

#### **剑指 Offer 04. 二维数组中的查找**

**题目：** 在一个 n * m 的二维数组中，每一行都按照从左到右递增的顺序排序，每一列都按照从上到下递增的顺序排序。请完成一个高效的函数，输入这样的一个二维数组和一个整数，判断数组中是否含有该整数。

**示例：** 

```
[
    [1,   4,  7, 11, 15],
    [2,   5,  8, 12, 19],
    [3,   6,  9, 16, 22],
    [10, 13, 14, 17, 24],
    [18, 21, 23, 26, 30]
]

给定 target = 5，返回 true。
给定 target = 20，返回 false。
```

**题解：** 

```java
public class Offer4 {

    /**
     * 数组中每行，每列都遵循由小到大排列，因此我们从矩阵最左下角开始遍历，并于目标值比较
     * 当 matrix[i][j] > target 时，说明这一行都会比所给目标值大，因此执行 i-- ，即消去第 i 行元素；
     * 当 matrix[i][j] < target 时，说明这一列都比所给目标值小，执行 j++ ，即消去第 j 列元素；
     * 当 matrix[i][j] = target 时，返回 truetruetrue ，代表找到目标值。
     *
     * @param matrix
     * @param target
     * @return
     */
    public static boolean findNumberIn2DArray(int[][] matrix, int target) {
        // 初始坐标
        int i = matrix.length - 1;
        int j = 0;
        while(i >= 0 && j <= matrix[0].length - 1){
            int current = matrix[i][j];
            if(current > target) {
                i--;
                continue;
            }else if(current < target){
                j++;
                continue;
            }else{
                return true;
            }
        }
        return false;
    }

    public static void main(String[] args) {
        int[][] arr = new int[][]{
                {1,   4,  7, 11, 15},
                {2,   5,  8, 12, 19},
                {3,   6,  9, 16, 22},
                {10, 13, 14, 17, 24},
                {18, 21, 23, 26, 30}
        };
        System.out.println(findNumberIn2DArray(arr, 5));
        System.out.println(findNumberIn2DArray(arr, 20));
    }
}
```

#### **剑指 Offer 05. 替换空格(这题没啥意义)**

**题目：** 请实现一个函数，把字符串 s 中的每个空格替换成"%20"。
**示例：** 

    输入：s = "We are happy."
    输出："We%20are%20happy."

**题解：** 

```java
public class Offer5 {

    public static String replaceSpace(String s) {
        return s.replaceAll(" ", "%20");
    }

    public static String replaceSpace2(String s) {
        StringBuffer sb = new StringBuffer();
        for (char ch: s.toCharArray()) {
            if(ch == ' '){
                sb.append("%20");
            }else{
                sb.append(ch);
            }
        }
        return sb.toString();
    }

    public static void main(String[] args) {
       String s = "We are happy.";
        System.out.println(replaceSpace(s));
        System.out.println(replaceSpace2(s));
    }
}
```

#### **剑指 Offer 06. 从尾到头打印链表**

**题目：** 输入一个链表的头节点，从尾到头反过来返回每个节点的值（用数组返回）。

**示例：** 

    输入：head = [1,3,2]
    输出：[2,3,1]

**题解：** 

```java
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode(int x) { val = x; }
 * }
 */
class Solution {
   /**
     * 逆序打印链表，借助栈即可完成，因为栈是天然的先进后出型
     * @param head
     * @return
     */
    public static int[] reversePrint(ListNode head) {
        Stack<ListNode> stack = new Stack<>();

        ListNode cur = head;
        while(cur != null) {
            stack.push(cur);
            cur = cur.next;
        }

        int size = stack.size();
        int[] ret = new int[size];
        for (int i = 0; i < size; i++) {
            ret[i] = stack.pop().val;
        }
        return ret;
    }

    public static void main(String[] args) {
        ListNode node = new ListNode(1);
        ListNode node2 = new ListNode(3);
        ListNode node3 = new ListNode(2);
        node.next = node2;
        node2.next = node3;
        System.out.println(Arrays.toString(reversePrint(node)));
    }
}
```

#### **剑指 Offer 07. 重建二叉树**

**题目：** 输入某二叉树的前序和中序，构建二叉树，假设输入的序列都不含重复的数字。

**示例：** 

```
Input: preorder = [3,9,20,15,7], inorder = [9,3,15,20,7]
Output: [3,9,20,null,null,15,7]
```

**题解：** 
```java
public class Offer7 {

    int[] preOrder;
    // 定义缓存表，用来存储中序遍历的内容
    HashMap<Integer, Integer> dict = new HashMap<>();
    /**
     * 前序遍历性质： 节点按照 [ 根节点 | 左子树 | 右子树 ] 排序。
     * 中序遍历性质： 节点按照 [ 左子树 | 根节点 | 右子树 ] 排序。
     * 以题目示例为例：
     *         前序遍历划分 [ 3 | 9 | 20 15 7 ]
     *         中序遍历划分 [ 9 | 3 | 15 20 7 ]
     *
     * 根据以上性质，可得出以下推论：
     *     前序遍历的首元素 为 树的根节点 node 的值。
     *     在中序遍历中搜索根节点 node 的索引 ，可将 中序遍历 划分为 [ 左子树 | 根节点 | 右子树 ] 。
     *     根据中序遍历中的左（右）子树的节点数量，可将 前序遍历 划分为 [ 根节点 | 左子树 | 右子树 ]
     * @param preorder
     * @param inorder
     * @return
     */
    public TreeNode buildTree(int[] preorder, int[] inorder) {
        this.preOrder = preorder;
        // 将中序存放在字典中
        for (int i = 0; i < inorder.length; i++) {
            dict.put(inorder[i], i);
        }
        return recur(0, 0, inorder.length - 1);
    }

    /**
     * 重新建立二叉树
     * @param root
     * @param left
     * @param right
     * @return
     */
    public TreeNode recur(int root, int left, int right) {
        if(left > right) return null;
        // 前序序列中，0下标作为根节点
        TreeNode node = new TreeNode(preOrder[root]);
        // 查看根节点在中序遍历中的索引i
        int i  = dict.get(preOrder[root]);
        // 递归划分，重点就是这个划分区间。左子树的根为root+1,区间在left， i - 1
        node.left = recur(root + 1, left, i - 1);
        // 右子树的根为 root + (i - left) + 1，区间在 i + 1, right
        node.right = recur(root + i - left + 1, i + 1, right);
        return node;
    }

    public void postOrder(TreeNode node) {
        if (node == null) {
            return;
        }
        postOrder(node.left);
        postOrder(node.right);
        System.out.println(node.val);
    }


    public static void main(String[] args) {
        int[] preorder = {3, 9, 20, 15, 7};
        int[] inorder = {9, 3, 15, 20, 7};
        TreeNode treeNode = new Offer7().buildTree(preorder, inorder);
        new Offer7().postOrder(treeNode);
    }
}
```

#### **剑指 Offer 09. 用两个栈实现队列**

**题目：** 用两个栈实现一个队列。队列的声明如下，请实现它的两个函数 appendTail 和 deleteHead ，分别完成在队列尾部插入整数和在队列头部删除整数的功能。(若队列中没有元素，deleteHead 操作返回 -1 )
**示例：** 

    输入：
    ["CQueue","appendTail","deleteHead","deleteHead"]
    [[],[3],[],[]]
    输出：[null,null,3,-1]

**题解：** 

```java
class CQueue {

    LinkedList<Integer> inStack, outStack;

    public CQueue() {
        // 进栈，用来保存进栈的数据
        inStack = new LinkedList<>();
		// 出栈，用来保存逆序的进栈
        outStack = new LinkedList<>();
    }
    
    public void appendTail(int value) {
        // 添加数据，只需要将数据添加到进栈尾中
        inStack.addLast(value);
    }
    
    public int deleteHead() {
        // 先查出栈有没有数，有数，直接出就行
        if(!outStack.isEmpty()){
            return outStack.removeLast();
        }
        if(inStack.isEmpty()) {
            return -1;
        }
        // 进栈不为空，将进栈的数据逆序存放在出栈中
        while(!inStack.isEmpty()){
            outStack.addLast(inStack.removeLast());
        }
        return outStack.removeLast();
    }
}

/**
 * Your CQueue object will be instantiated and called as such:
 * CQueue obj = new CQueue();
 * obj.appendTail(value);
 * int param_2 = obj.deleteHead();
 */
```

#### **剑指 Offer 10-1. 斐波那契数列**

**题目：** 写一个函数，输入 n ，求斐波那契（Fibonacci）数列的第 n 项（即 F(N)）。斐波那契数列的定义如下：

```
F(0) = 0,   F(1) = 1
F(N) = F(N - 1) + F(N - 2), 其中 N > 1.
```

**实例：**

```
输入：n = 2
输出：1
```

**题解：** 
```java

```

#### **剑指 Offer 10- II. 青蛙跳台阶问题**

**题目：** 输入一颗二叉搜索树，将该二叉搜索树转换成一个排序的循环双链表。要求不能创建任何新的结。只能调整树中节点指针的指向。

**题解：** 
```java

```

#### **剑指 Offer 36. 二叉搜索树与双向链表**

**题目：** 输入一颗二叉搜索树，将该二叉搜索树转换成一个排序的循环双链表。要求不能创建任何新的结。只能调整树中节点指针的指向。

**题解：** 
```java

```

#### **剑指 Offer 11. 旋转数组的最小数字**

**题目：** 把一个数组最开始的若干个元素搬到数组的末尾，我们称之为数组的旋转。给你一个可能存在 重复 元素值的数组 numbers ，它原来是一个升序排列的数组，并按上述情形进行了一次旋转。请返回旋转数组的最小元素。例如，数组 [3,4,5,1,2] 为 [1,2,3,4,5] 的一次旋转，该数组的最小值为 1。注意，数组 [a[0], a[1], a[2], ..., a[n-1]] 旋转一次 的结果为数组 [a[n-1], a[0], a[1], a[2], ..., a[n-2]] 。

**示例：** 

    输入：numbers = [3,4,5,1,2]
    输出：1

**题解：** 

```java
class Solution {
    public int minArray(int[] numbers) {
        int low = 0;
        int high = numbers.length - 1;
        while (low < high) {
            int pivot = low + (high - low) / 2;
            if (numbers[pivot] < numbers[high]) {
                high = pivot;
            } else if (numbers[pivot] > numbers[high]) {
                low = pivot + 1;
            } else {
                high -= 1;
            }
        }
        return numbers[low];
    }
}
```

#### **剑指 Offer 12. 矩阵中的路径**

**题目：** 给定一个 m x n 二维字符网格 board 和一个字符串单词 word 。如果 word 存在于网格中，返回 true ；否则，返回 false 。单词必须按照字母顺序，通过相邻的单元格内的字母构成，其中“相邻”单元格是那些水平相邻或垂直相邻的单元格。同一个单元格内的字母不允许被重复使用。例如，在下面的 3×4 的矩阵中包含单词 "ABCCED"（单词中的字母已标出）。
|1|2|3|4|
|--|--|--|--|
|A|B|C|E|
|S|F|C|S|
|A|D|E|E|
**示例：** 

    输入：board = [["A","B","C","E"],["S","F","C","S"],["A","D","E","E"]], word = "ABCCED"
    输出：true

**题解：** 

```java
// 回溯
class Solution {
    public boolean exist(char[][] board, String word) {
        int h = board.length, w = board[0].length;
        boolean[][] visited = new boolean[h][w];
        for (int i = 0; i < h; i++) {
            for (int j = 0; j < w; j++) {
                boolean flag = check(board, visited, i, j, word, 0);
                if (flag) {
                    return true;
                }
            }
        }
        return false;
    }

    public boolean check(char[][] board, boolean[][] visited, int i, int j, String s, int k) {
        if (board[i][j] != s.charAt(k)) {
            return false;
        } else if (k == s.length() - 1) {
            return true;
        }
        visited[i][j] = true;
        int[][] directions = {{0, 1}, {0, -1}, {1, 0}, {-1, 0}};
        boolean result = false;
        for (int[] dir : directions) {
            int newi = i + dir[0], newj = j + dir[1];
            if (newi >= 0 && newi < board.length && newj >= 0 && newj < board[0].length) {
                if (!visited[newi][newj]) {
                    boolean flag = check(board, visited, newi, newj, s, k + 1);
                    if (flag) {
                        result = true;
                        break;
                    }
                }
            }
        }
        visited[i][j] = false;
        return result;
    }
}
```

#### **剑指 Offer 18. 删除链表的节点**

**题目：** 给定单向链表的头指针和一个要删除的节点的值，定义一个函数删除该节点。返回删除后的链表的头节点。

**示例：** 

    输入: head = [4,5,1,9], val = 5
    输出: [4,1,9]
    解释: 给定你链表中值为 5 的第二个节点，那么在调用了你的函数之后，该链表应变为 4 -> 1 -> 9.

**题解：** 

```java
class Solution {
    public ListNode deleteNode(ListNode head, int val) {
        if(head.val == val) return head.next;
        ListNode pre = head, cur = head.next;
        while(cur != null && cur.val != val) {
            pre = cur;
            cur = cur.next;
        }
        if(cur != null) pre.next = cur.next;
        return head;
    }
}
```

#### **剑指 Offer 21. 调整数组顺序使奇数位于偶数前面**

**题目：** 输入一个整数数组，实现一个函数来调整该数组中数字的顺序，使得所有奇数在数组的前半部分，所有偶数在数组的后半部分。

**示例：** 

    输入：nums = [1,2,3,4]
    输出：[1,3,2,4] 
    注：[3,1,2,4] 也是正确的答案之一。

**题解：** 

```java
class Solution {
    public int[] exchange(int[] nums) {
        int i = 0, j = nums.length - 1, tmp;
        while(i < j) {
            while(i < j && (nums[i] & 1) == 1) i++;
            while(i < j && (nums[j] & 1) == 0) j--;
            tmp = nums[i];
            nums[i] = nums[j];
            nums[j] = tmp;
        }
        return nums;
    }
}
```

#### **剑指 Offer 22. 链表中倒数第k个节点**

**题目：** 输入一个链表，输出该链表中倒数第k个节点。为了符合大多数人的习惯，本题从1开始计数，即链表的尾节点是倒数第1个节点。例如，一个链表有 6 个节点，从头节点开始，它们的值依次是 1、2、3、4、5、6。这个链表的倒数第 3 个节点是值为 4 的节点。

**示例：** 

    给定一个链表: 1->2->3->4->5, 和 k = 2.
    返回链表 4->5.

**题解：** 

```java
// 方法一：顺序查找
class Solution {
    public ListNode getKthFromEnd(ListNode head, int k) {
        int n = 0;
        ListNode node = null;

        for (node = head; node != null; node = node.next) {
            n++;
        }
        for (node = head; n > k; n--) {
            node = node.next;
        }

        return node;
    }
}
// 方法二：双指针
class Solution {
    public ListNode getKthFromEnd(ListNode head, int k) {
        ListNode fast = head;
        ListNode slow = head;

        while (fast != null && k > 0) {
            fast = fast.next;
            k--;
        }
        while (fast != null) {
            fast = fast.next;
            slow = slow.next;
        }

        return slow;
    }
}
```

#### **剑指 Offer 24. 反转链表**

**题目：** 给定单链表的头节点 head ，请反转链表，并返回反转后的链表的头节点。

**示例：** 

    输入：head = [1,2,3,4,5]
    输出：[5,4,3,2,1]

**题解：** 

```java
// 方法一：迭代
class Solution {
    public ListNode reverseList(ListNode head) {
        ListNode prev = null;
        ListNode curr = head;
        while (curr != null) {
            ListNode next = curr.next;
            curr.next = prev;
            prev = curr;
            curr = next;
        }
        return prev;
    }
}

// 方法二：递归
class Solution {
    public ListNode reverseList(ListNode head) {
        if (head == null || head.next == null) {
            return head;
        }
        ListNode newHead = reverseList(head.next);
        head.next.next = head;
        head.next = null;
        return newHead;
    }
}
```

#### **剑指 Offer 25. 合并两个排序的链表**

**题目：** 输入两个递增排序的链表，合并这两个链表并使新链表中的节点仍然是递增排序的。

**示例：** 

    输入：1->2->4, 1->3->4
    输出：1->1->2->3->4->4

**题解：** 

```java
// 递归
class Solution {
    public ListNode mergeTwoLists(ListNode l1, ListNode l2) {
        if (l1 == null) {
            return l2;
        } else if (l2 == null) {
            return l1;
        } else if (l1.val < l2.val) {
            l1.next = mergeTwoLists(l1.next, l2);
            return l1;
        } else {
            l2.next = mergeTwoLists(l1, l2.next);
            return l2;
        }
    }
}

// 迭代
class Solution {
    public ListNode mergeTwoLists(ListNode l1, ListNode l2) {
        ListNode prehead = new ListNode(-1);

        ListNode prev = prehead;
        while (l1 != null && l2 != null) {
            if (l1.val <= l2.val) {
                prev.next = l1;
                l1 = l1.next;
            } else {
                prev.next = l2;
                l2 = l2.next;
            }
            prev = prev.next;
        }

        // 合并后 l1 和 l2 最多只有一个还未被合并完，我们直接将链表末尾指向未合并完的链表即可
        prev.next = l1 == null ? l2 : l1;

        return prehead.next;
    }
}
```

#### **剑指 Offer 26. 树的子结构**

**题目：** 输入两棵二叉树A和B，判断B是不是A的子结构。(约定空树不是任意一个树的子结构)

B是A的子结构， 即 A中有出现和B相同的结构和节点值。

例如:
给定的树 A:

        3
        / \
     4   5
    / \
    1   2

给定的树 B：

      4 
     /
    1

返回 true，因为 B 与 A 的一个子树拥有相同的结构和节点值。

**示例：** 

    输入：A = [1,2,3], B = [3,1]
    输出：false

**题解：** 

```java
class Solution {
    public boolean isSubStructure(TreeNode A, TreeNode B) {
        return (A != null && B != null) && (recur(A, B) || isSubStructure(A.left, B) || isSubStructure(A.right, B));
    }
    boolean recur(TreeNode A, TreeNode B) {
        if(B == null) return true;
        if(A == null || A.val != B.val) return false;
        return recur(A.left, B.left) && recur(A.right, B.right);
    }
}
```

#### **剑指 Offer 30. 包含min函数的栈**

**题目：** 定义栈的数据结构，请在该类型中实现一个能够得到栈的最小元素的 min 函数在该栈中，调用 min、push 及 pop 的时间复杂度都是 O(1)。

**示例：** 

    MinStack minStack = new MinStack();
    minStack.push(-2);
    minStack.push(0);
    minStack.push(-3);
    minStack.min();   --> 返回 -3.
    minStack.pop();
    minStack.top();      --> 返回 0.
    minStack.min();   --> 返回 -2.

**题解：** 

```java
// 辅助栈
class MinStack {
    Deque<Integer> xStack;
    Deque<Integer> minStack;

    public MinStack() {
        xStack = new LinkedList<Integer>();
        minStack = new LinkedList<Integer>();
        minStack.push(Integer.MAX_VALUE);
    }

    public void push(int x) {
        xStack.push(x);
        minStack.push(Math.min(minStack.peek(), x));
    }

    public void pop() {
        xStack.pop();
        minStack.pop();
    }

    public int top() {
        return xStack.peek();
    }

    public int min() {
        return minStack.peek();
    }
}
```

#### **剑指 Offer 32 - I. 从上到下打印二叉树**

**题目：** 从上到下打印出二叉树的每个节点，同一层的节点按照从左到右的顺序打印。

**示例：** 
例如:
给定二叉树: [3,9,20,null,null,15,7],

    3
    / \
    9  20
        /  \
        15   7

返回：[3,9,20,15,7]

**题解：** 

```java
class Solution {
    public int[] levelOrder(TreeNode root) {
        if(root == null) return new int[0];
        Queue<TreeNode> queue = new LinkedList<>(){{ add(root); }};
        ArrayList<Integer> ans = new ArrayList<>();
        while(!queue.isEmpty()) {
            TreeNode node = queue.poll();
            ans.add(node.val);
            if(node.left != null) queue.add(node.left);
            if(node.right != null) queue.add(node.right);
        }
        int[] res = new int[ans.size()];
        for(int i = 0; i < ans.size(); i++)
            res[i] = ans.get(i);
        return res;
    }
}
```

#### **剑指 Offer 32 - II. 从上到下打印二叉树 II**

**题目：** 从上到下按层打印二叉树，同一层的节点按从左到右的顺序打印，每一层打印到一行。

**示例：** 
例如:
给定二叉树: [3,9,20,null,null,15,7],

     3
    / \
    9  20
        /  \
       15   7

返回其层次遍历结果：

    [
    [3],
    [9,20],
    [15,7]
    ]

**题解：** 

```java
// 层序遍历
class Solution {
    public List<List<Integer>> levelOrder(TreeNode root) {
        Queue<TreeNode> queue = new LinkedList<>();
        List<List<Integer>> res = new ArrayList<>();
        if(root != null) queue.add(root);
        while(!queue.isEmpty()) {
            List<Integer> tmp = new ArrayList<>();
            for(int i = queue.size(); i > 0; i--) {
                TreeNode node = queue.poll();
                tmp.add(node.val);
                if(node.left != null) queue.add(node.left);
                if(node.right != null) queue.add(node.right);
            }
            res.add(tmp);
        }
        return res;
    }
}
```

#### **剑指 Offer 32 - III. 从上到下打印二叉树 III**

**题目：** 请实现一个函数按照之字形顺序打印二叉树，即第一行按照从左到右的顺序打印，第二层按照从右到左的顺序打印，第三行再按照从左到右的顺序打印，其他行以此类推。

**示例：** 

例如:
给定二叉树: [3,9,20,null,null,15,7],

     3
    / \
    9  20
        /  \
       15   7

返回其层次遍历结果：

    [
    [3],
    [20,9],
    [15,7]
    ]

**题解：** 

```java
// 层序遍历 + 双端队列
class Solution {
    public List<List<Integer>> levelOrder(TreeNode root) {
        Queue<TreeNode> queue = new LinkedList<>();
        List<List<Integer>> res = new ArrayList<>();
        if(root != null) queue.add(root);
        while(!queue.isEmpty()) {
            LinkedList<Integer> tmp = new LinkedList<>();
            for(int i = queue.size(); i > 0; i--) {
                TreeNode node = queue.poll();
                if(res.size() % 2 == 0) tmp.addLast(node.val); // 偶数层 -> 队列头部
                else tmp.addFirst(node.val); // 奇数层 -> 队列尾部
                if(node.left != null) queue.add(node.left);
                if(node.right != null) queue.add(node.right);
            }
            res.add(tmp);
        }
        return res;
    }
}
```

#### **剑指 Offer 33. 二叉搜索树的后序遍历序列**

**题目：** 输入一个整数数组，判断该数组是不是某二叉搜索树的后序遍历结果。如果是则返回 true，否则返回 false。假设输入的数组的任意两个数字都互不相同。

**示例：** 
参考以下这颗二叉搜索树：

         5
        / \
      2   6
     / \
    1   3
    
    输入: [1,6,3,2,5]
    输出: false

**题解：** 

```java
// 方法一：递归分治
class Solution {
    public boolean verifyPostorder(int[] postorder) {
        return recur(postorder, 0, postorder.length - 1);
    }
    boolean recur(int[] postorder, int i, int j) {
        if(i >= j) return true;
        int p = i;
        while(postorder[p] < postorder[j]) p++;
        int m = p;
        while(postorder[p] > postorder[j]) p++;
        return p == j && recur(postorder, i, m - 1) && recur(postorder, m, j - 1);
    }
}

// 方法二：辅助单调栈
class Solution {
    public boolean verifyPostorder(int[] postorder) {
        Stack<Integer> stack = new Stack<>();
        int root = Integer.MAX_VALUE;
        for(int i = postorder.length - 1; i >= 0; i--) {
            if(postorder[i] > root) return false;
            while(!stack.isEmpty() && stack.peek() > postorder[i])
                root = stack.pop();
            stack.add(postorder[i]);
        }
        return true;
    }
}
```
#### **剑指 Offer 36. 二叉搜索树与双向链表**

**题目：** 输入一颗二叉搜索树，将该二叉搜索树转换成一个排序的循环双链表。要求不能创建任何新的结。只能调整树中节点指针的指向。

**题解：** 
```java

```

#### **剑指 Offer 41. 数据流中的中位数**

**题目：** 如何得到一个数据流中的中位数？如果从数据流中读出奇数个数值，那么中位数就是所有数值排序之后位于中间的数值。如果从数据流中读出偶数个数值，那么中位数就是所有数值排序之后中间两个数的平均值。

例如，[2,3,4] 的中位数是 3，[2,3] 的中位数是 (2 + 3) / 2 = 2.5

设计一个支持以下两种操作的数据结构：

    void addNum(int num) - 从数据流中添加一个整数到数据结构中。
    double findMedian() - 返回目前所有元素的中位数。

**示例：** 

    输入：["MedianFinder","addNum","addNum","findMedian","addNum","findMedian"]
    [[],[1],[2],[],[3],[]]
    输出：[null,null,null,1.50000,null,2.00000]

**题解：** 

```java
class MedianFinder {
    Queue<Integer> A, B;
    public MedianFinder() {
        A = new PriorityQueue<>(); // 小顶堆，保存较大的一半
        B = new PriorityQueue<>((x, y) -> (y - x)); // 大顶堆，保存较小的一半
    }
    public void addNum(int num) {
        if(A.size() != B.size()) {
            A.add(num);
            B.add(A.poll());
        } else {
            B.add(num);
            A.add(B.poll());
        }
    }
    public double findMedian() {
        return A.size() != B.size() ? A.peek() : (A.peek() + B.peek()) / 2.0;
    }
}
```

#### **剑指 Offer 42. 连续子数组的最大和**

**题目：** 输入一个整型数组，数组中的一个或连续多个整数组成一个子数组。求所有子数组的和的最大值。要求时间复杂度为O(n)。

**示例：** 输入: nums = [-2,1,-3,4,-1,2,1,-5,4] 输出: 6

解释: 连续子数组 [4,-1,2,1] 的和最大，为 6。

**题解：** 

```java
class Solution {
    public int maxSubArray(int[] nums) {
        int pre = 0, maxAns = nums[0];
        for (int x : nums) {
            pre = Math.max(pre + x, x);
            maxAns = Math.max(maxAns, pre);
        }
        return maxAns;
    }
}
```

#### **剑指 Offer 45. 把数组排成最小的数**

**题目：** 输入一个非负整数数组，把数组里所有数字拼接起来排成一个数，打印能拼接出的所有数字中最小的一个。

**示例：** 输入: [10,2] 输出: "102"

**题解：** 

```java
// 方法一：快速排序
class Solution {
    public String minNumber(int[] nums) {
        String[] strs = new String[nums.length];
        for(int i = 0; i < nums.length; i++)
            strs[i] = String.valueOf(nums[i]);
        quickSort(strs, 0, strs.length - 1);
        StringBuilder res = new StringBuilder();
        for(String s : strs)
            res.append(s);
        return res.toString();
    }
    void quickSort(String[] strs, int l, int r) {
        if(l >= r) return;
        int i = l, j = r;
        String tmp = strs[i];
        while(i < j) {
            while((strs[j] + strs[l]).compareTo(strs[l] + strs[j]) >= 0 && i < j) j--;
            while((strs[i] + strs[l]).compareTo(strs[l] + strs[i]) <= 0 && i < j) i++;
            tmp = strs[i];
            strs[i] = strs[j];
            strs[j] = tmp;
        }
        strs[i] = strs[l];
        strs[l] = tmp;
        quickSort(strs, l, i - 1);
        quickSort(strs, i + 1, r);
    }
}

// 方法二：内置sort
class Solution {
    public String minNumber(int[] nums) {
        String[] strs = new String[nums.length];
        for(int i = 0; i < nums.length; i++)
            strs[i] = String.valueOf(nums[i]);
        Arrays.sort(strs, (x, y) -> (x + y).compareTo(y + x));
        StringBuilder res = new StringBuilder();
        for(String s : strs)
            res.append(s);
        return res.toString();
    }
}
```

#### **剑指 Offer 46. 把数字翻译成字符串**

**题目：** 给定一个数字，我们按照如下规则把它翻译为字符串：0 翻译成 “a” ，1 翻译成 “b”，……，11 翻译成 “l”，……，25 翻译成 “z”。一个数字可能有多个翻译。请编程实现一个函数，用来计算一个数字有多少种不同的翻译方法。

**示例：** 

    输入: 12258
    输出: 5
    解释: 12258有5种不同的翻译，分别是"bccfi", "bwfi", "bczi", "mcfi"和"mzi"

**题解：** 

```java
// 方法一：
class Solution {
    public int translateNum(int num) {
        String s = String.valueOf(num);
        int a = 1, b = 1;
        for(int i = 2; i <= s.length(); i++) {
            String tmp = s.substring(i - 2, i);
            int c = tmp.compareTo("10") >= 0 && tmp.compareTo("25") <= 0 ? a + b : a;
            b = a;
            a = c;
        }
        return a;
    }
}

// 方法二：
class Solution {
    public int translateNum(int num) {
        int a = 1, b = 1, x, y = num % 10;
        while(num != 0) {
            num /= 10;
            x = num % 10;
            int tmp = 10 * x + y;
            int c = (tmp >= 10 && tmp <= 25) ? a + b : a;
            b = a;
            a = c;
            y = x;
        }
        return a;
    }
}
```

#### **剑指 Offer 47. 礼物的最大价值**

**题目：** 在一个 m*n 的棋盘的每一格都放有一个礼物，每个礼物都有一定的价值（价值大于 0）。你可以从棋盘的左上角开始拿格子里的礼物，并每次向右或者向下移动一格、直到到达棋盘的右下角。给定一个棋盘及其上面的礼物的价值，请计算你最多能拿到多少价值的礼物？

**示例：** 输入: [ [1,3,1],  [1,5,1],   [4,2,1] ]   输出: 12

解释: 路径 1→3→5→2→1 可以拿到最多价值的礼物

**题解：** 

```java
class Solution {
    public int maxValue(int[][] grid) {
        int row = grid.length;
        int column = grid[0].length;
        //dp[i][j]表示从grid[0][0]到grid[i - 1][j - 1]时的最大价值
        int[][] dp = new int[row + 1][column + 1];
        for (int i = 1; i <= row; i++) {
            for (int j = 1; j <= column; j++) {
                dp[i][j] = Math.max(dp[i - 1][j], dp[i][j - 1]) + grid[i - 1][j - 1];
            }
        }
        return dp[row][column];
    }
}
```

#### **剑指 Offer 50. 第一个只出现一次的字符**

**题目：** 在字符串 s 中找出第一个只出现一次的字符。如果没有，返回一个单空格。 s 只包含小写字母。

**示例：** 输入：s = "abaccdeff"    输出：'b'

**题解：** 

```java
// 方法一：
class Solution {
    public char firstUniqChar(String s) {
        char res = ' ';
        int resIndex = Integer.MAX_VALUE;
        for(char i = 'a';i <= 'z';i++){
            int index = s.indexOf(i);
            if (index == -1){
                continue;
            }

            if (index != s.lastIndexOf(i)) {
                continue;
            }

            if (index < resIndex) {
                resIndex = index;
                res = i;
            }

        }
        return res;
    }
}
// 方法二：
class Solution {
    public char firstUniqChar(String s) {
        Map<Character, Integer> position = new HashMap<Character, Integer>();
        Queue<Pair> queue = new LinkedList<Pair>();
        int n = s.length();
        for (int i = 0; i < n; ++i) {
            char ch = s.charAt(i);
            if (!position.containsKey(ch)) {
                position.put(ch, i);
                queue.offer(new Pair(ch, i));
            } else {
                position.put(ch, -1);
                while (!queue.isEmpty() && position.get(queue.peek().ch) == -1) {
                    queue.poll();
                }
            }
        }
        return queue.isEmpty() ? ' ' : queue.poll().ch;
    }

    class Pair {
        char ch;
        int pos;

        Pair(char ch, int pos) {
            this.ch = ch;
            this.pos = pos;
        }
    }
}
```

#### **剑指 Offer 51. 数组中的逆序对**

**题目：** 在数组中的两个数字，如果前面一个数字大于后面的数字，则这两个数字组成一个逆序对。输入一个数组，求出这个数组中的逆序对的总数。

**示例：** 输入: [7,5,6,4]   输出: 5

**题解：** 

```java
// 方法一：分治法
public class Solution {
    public int reversePairs(int[] nums) {
        int len = nums.length;

        if (len < 2) {
            return 0;
        }

        int[] copy = new int[len];
        for (int i = 0; i < len; i++) {
            copy[i] = nums[i];
        }

        int[] temp = new int[len];
        return reversePairs(copy, 0, len - 1, temp);
    }

    private int reversePairs(int[] nums, int left, int right, int[] temp) {
        if (left == right) {
            return 0;
        }

        int mid = left + (right - left) / 2;
        int leftPairs = reversePairs(nums, left, mid, temp);
        int rightPairs = reversePairs(nums, mid + 1, right, temp);

        if (nums[mid] <= nums[mid + 1]) {
            return leftPairs + rightPairs;
        }

        int crossPairs = mergeAndCount(nums, left, mid, right, temp);
        return leftPairs + rightPairs + crossPairs;
    }

    private int mergeAndCount(int[] nums, int left, int mid, int right, int[] temp) {
        for (int i = left; i <= right; i++) {
            temp[i] = nums[i];
        }

        int i = left;
        int j = mid + 1;

        int count = 0;
        for (int k = left; k <= right; k++) {

            if (i == mid + 1) {
                nums[k] = temp[j];
                j++;
            } else if (j == right + 1) {
                nums[k] = temp[i];
                i++;
            } else if (temp[i] <= temp[j]) {
                nums[k] = temp[i];
                i++;
            } else {
                nums[k] = temp[j];
                j++;
                count += (mid - i + 1);
            }
        }
        return count;
    }
}

// 方法二：离散化树状数组
class Solution {
    public int reversePairs(int[] nums) {
        int n = nums.length;
        int[] tmp = new int[n];
        System.arraycopy(nums, 0, tmp, 0, n);
        // 离散化
        Arrays.sort(tmp);
        for (int i = 0; i < n; ++i) {
            nums[i] = Arrays.binarySearch(tmp, nums[i]) + 1;
        }
        // 树状数组统计逆序对
        BIT bit = new BIT(n);
        int ans = 0;
        for (int i = n - 1; i >= 0; --i) {
            ans += bit.query(nums[i] - 1);
            bit.update(nums[i]);
        }
        return ans;
    }
}

class BIT {
    private int[] tree;
    private int n;

    public BIT(int n) {
        this.n = n;
        this.tree = new int[n + 1];
    }

    public static int lowbit(int x) {
        return x & (-x);
    }

    public int query(int x) {
        int ret = 0;
        while (x != 0) {
            ret += tree[x];
            x -= lowbit(x);
        }
        return ret;
    }

    public void update(int x) {
        while (x <= n) {
            ++tree[x];
            x += lowbit(x);
        }
    }
}
```

#### **剑指 Offer 53 - I. 在排序数组中查找数字（这题比较废物）**

**题目：** 统计一个数字在排序数组中出现的次数。
**示例：** 输入: nums = [5,7,7,8,8,10], target = 8    输出: 2
**题解：** 二分查找法

```java
class Solution {
    public int search(int[] nums, int target) {
        int leftIdx = binarySearch(nums, target, true);
        int rightIdx = binarySearch(nums, target, false) - 1;
        if (leftIdx <= rightIdx && rightIdx < nums.length && nums[leftIdx] == target && nums[rightIdx] == target) {
            return rightIdx - leftIdx + 1;
        } 
        return 0;
    }

    public int binarySearch(int[] nums, int target, boolean lower) {
        int left = 0, right = nums.length - 1, ans = nums.length;
        while (left <= right) {
            int mid = (left + right) / 2;
            if (nums[mid] > target || (lower && nums[mid] >= target)) {
                right = mid - 1;
                ans = mid;
            } else {
                left = mid + 1;
            }
        }
        return ans;
    }
}
```

#### **剑指 Offer 53 - II. 0～n-1中缺失的数字**

**题目：** 一个长度为n-1的递增排序数组中的所有数字都是唯一的，并且每个数字都在范围0～n-1之内。在范围0～n-1内的n个数字中有且只有一个数字不在该数组中，请找出这个数字。

**示例：** 输入: [0,1,3]  输出: 2

数组nums中有n−1个数，在这n−1个数的后面添加从0到n−1的每个整数，则添加了n个整数，共有2n−1个整数。在 2n−1个整数中，缺失的数字只在后面 n个整数中出现一次，其余的数字在前面 n−1个整数中（即数组中）和后面n个整数中各出现一次，即其余的数字都出现了两次。

根据出现的次数的奇偶性，可以使用按位异或运算得到缺失的数字。按位异或运算 ⊕ 满足交换律和结合律，且对任意整数 x 都满足 x⊕x=0 和 x⊕0=x。

由于上述 2n−1个整数中，缺失的数字出现了一次，其余的数字都出现了两次，因此对上述 2n−1个整数进行按位异或运算，结果即为缺失的数字。

**题解：** 

```java
// 主要是考察二分法
class Solution {
    public int missingNumber(int[] nums) {
      
    }
}

```

#### **剑指 Offer 54. 二叉搜索树的第k大节点**

**题目：** 给定一棵二叉搜索树，请找出其中第 k 大的节点的值。

**思路：**求 “二叉搜索树第 kkk 大的节点” 可转化为求 “此树的中序遍历倒序的第 kkk 个节点”。

递归解析：

    终止条件： 当节点 rootrootroot 为空（越过叶节点），则直接返回；
    递归右子树： 即 dfs(root.right)dfs(root.right)dfs(root.right) ；
    三项工作：
        提前返回： 若 k=0k = 0k=0 ，代表已找到目标节点，无需继续遍历，因此直接返回；
        统计序号： 执行 k=k−1k = k - 1k=k−1 （即从 kkk 减至 000 ）；
        记录结果： 若 k=0k = 0k=0 ，代表当前节点为第 kkk 大的节点，因此记录 res=root.valres = root.valres=root.val ；
    递归左子树： 即 dfs(root.left)dfs(root.left)dfs(root.left) ；

**题解：** 

```java
class Solution {
    int res, k;
    public int kthLargest(TreeNode root, int k) {
        this.k = k;
        dfs(root);
        return res;
    }
    void dfs(TreeNode root) {
        if(root == null) return;
        dfs(root.right);
        if(k == 0) return;
        if(--k == 0) res = root.val;
        dfs(root.left);
    }
}
```

#### **剑指 Offer 55 - I. 二叉树的深度**

**题目：** 输入一棵二叉树的根节点，求该树的深度。从根节点到叶节点依次经过的节点（含根、叶节点）形成树的一条路径，最长路径的长度为树的深度。

**题解：** 

```java
// 方法一：深度优先搜索
class Solution {
    public int maxDepth(TreeNode root) {
        if (root == null) {
            return 0;
        } else {
            int leftHeight = maxDepth(root.left);
            int rightHeight = maxDepth(root.right);
            return Math.max(leftHeight, rightHeight) + 1;
        }
    }
}

// 方法二：广度优先算法
class Solution {
    public int maxDepth(TreeNode root) {
        if (root == null) {
            return 0;
        }
        Queue<TreeNode> queue = new LinkedList<TreeNode>();
        queue.offer(root);
        int ans = 0;
        while (!queue.isEmpty()) {
            int size = queue.size();
            while (size > 0) {
                TreeNode node = queue.poll();
                if (node.left != null) {
                    queue.offer(node.left);
                }
                if (node.right != null) {
                    queue.offer(node.right);
                }
                size--;
            }
            ans++;
        }
        return ans;
    }
}
```

#### **剑指 Offer 58 - II. 左旋转字符串**

**题目：** 给你一个字符串 s，由若干单词组成，单词前后用一些空格字符隔开。返回字符串中 最后一个 单词的长度。单词 是指仅由字母组成、不包含任何空格字符的最大子字符串。

**示例：** 输入：s = "Hello World"   输出：5

解释：最后一个单词是“World”，长度为5。

**题解：** 

```java
class Solution {
    public int lengthOfLastWord(String s) {
        int count = 0;  
       if(s.length() == 0){
            return 0;
       }
       for(int i = s.length()-1 ; i >= 0 ; i--){    
           if(s.charAt(i) == ' ' && count!=0){      
               return count;
           }else if(s.charAt(i) == ' '){    
               count = 0;
           }else{
               count++;                        
           }
       }
        return count;
    }
}
```

#### **剑指 Offer 61. 扑克牌中的顺子**

**题目：** 从若干副扑克牌中随机抽 5 张牌，判断是不是一个顺子，即这5张牌是不是连续的。2～10为数字本身，A为1，J为11，Q为12，K为13，而大、小王为 0 ，可以看成任意数字。A 不能视为 14。

**示例：** 输入: [1,2,3,4,5]   输出: True

**题解：** 

```java
class Solution {
    public boolean isStraight(int[] nums) {
        Set<Integer> numSet = new HashSet<>();
        int max = 0 , min = 14;
        for(int num : nums){
            if(num == 0 ){
                continue;
            }
            max = Math.max(max, num);
            min = Math.min(min, num);
            if(numSet.contains(num)){
                return false;
            }
            numSet.add(num);
        }
        return max - min < 5;
    }
}
```

#### **剑指 Offer 66. 构建乘积数组**

**题目：** 给定一个数组 A[0,1,…,n-1]，请构建一个数组 B[0,1,…,n-1]，其中 B[i] 的值是数组 A 中除了下标 i 以外的元素的积, 即 B[i]=A[0]×A[1]×…×A[i-1]×A[i+1]×…×A[n-1]。不能使用除法。

**示例：** 输入: [1,2,3,4,5]   输出: [120,60,40,30,24]

**题解：** 

```java
class Solution {
    public int[] productExceptSelf(int[] a) {
        int length = a.length;
        int[] answer = new int[length];

        // answer[i] 表示索引 i 左侧所有元素的乘积
        // 因为索引为 '0' 的元素左侧没有元素， 所以 answer[0] = 1
        answer[0] = 1;
        for (int i = 1; i < length; i++) {
            answer[i] = a[i-1] * answer[i-1];
        }

        // R 为右侧所有元素的乘积
        // 刚开始右边没有元素，所以 R = 1
        int R = 1;
        for (int i = length - 1; i >= 0; i--) {
            // 对于索引 i，左边的乘积为 answer[i]，右边的乘积为 R
            answer[i] = answer[i] * R;
            // R 需要包含右边所有的乘积，所以计算下一个结果时需要将当前值乘到 R 上
            R *= a[i];
        }
        return answer;
    }
}

```

## **章节二：CodeTop前300题（除去章节一中已经拥有的）**
#### **1、无重复字符的最长子串（ LeetCode 3）频度484**

**题目：** 

**示例：** 

**题解：** 

```java

```
#### **2、至少有 K 个重复字符的最长子串（ LeetCode 395）**

**题目：** 

**示例：** 

**题解：** 

```java

```
#### **3、K 个不同整数的子数组（ LeetCode 992）**

**题目：** 

**示例：** 

**题解：** 

```java

```

#### **4、下一个排列（ LeetCode 31）**

**题目：** 

**示例：** 

**题解：** 

```java

```

#### **5、最长有效括号（ LeetCode 32）**

**题目：** 

**示例：** 

**题解：** 

```java

```


#### **6、最长有效括号（ LeetCode 32）**

**题目：** 

**示例：** 

**题解：** 

```java

```


#### **7、旋转图像（ LeetCode 48）**

**题目：** 

**示例：** 

**题解：** 

```java

```



#### **8、字母异位词分组 （LeetCode 49）**

**题目：** 

**示例：** 

**题解：** 

```java

```


#### **9、颜色分类 （LeetCode 75）**

**题目：** 

**示例：** 

**题解：** 

```java

```


#### **10、最小覆盖子串（ LeetCode 76）**

**题目：** 

**示例：** 

**题解：** 

```java

```


#### **11、单词搜索（ LeetCode 79）**

**题目：** 

**示例：** 

**题解：** 

```java

```


#### **12、柱状图中最大的矩形（ LeetCode 84）**

**题目：** 

**示例：** 

**题解：** 

```java

```


#### **13、最大矩形（ LeetCode 85）**

**题目：** 

**示例：** 

**题解：** 

```java

```


#### **14、最长有效括号（ LeetCode 32）**

**题目：** 

**示例：** 

**题解：** 

```java

```


#### **18、最长连续序列（ LeetCode 128）**

**题目：** 

**示例：** 

**题解：** 

```java

```


#### **19、只出现一次的数字（ LeetCode 136）**

**题目：** 

**示例：** 

**题解：** 

```java

```


#### **20、排序链表（ LeetCode 148）**

**题目：** 

**示例：** 

**题解：** 

```java

```

#### **21、乘积最大子数组（ LeetCode 152）**

**题目：** 

**示例：** 

**题解：** 

```java

```

#### **22、多数元素（ LeetCode 141）**

**题目：** 

**示例：** 

**题解：** 

```java

```

#### **23、课程表（ LeetCode 207）**

**题目：** 

**示例：** 

**题解：** 

```java

```

#### **24、实现 Trie (前缀树)（ LeetCode 208）**

**题目：** 

**示例：** 

**题解：** 

```java

```

#### **25、最长有效括号（ LeetCode 141）**

**题目：** 

**示例：** 

**题解：** 

```java

```

#### **26、数组中的第K个最大元素（ LeetCode 215）**

**题目：** 

**示例：** 

**题解：** 

```java

```

#### **27、最大正方形（ LeetCode 221）**

**题目：** 

**示例：** 

**题解：** 

```java

```

#### **28、最长有效括号（ LeetCode 141）**

**题目：** 

**示例：** 

**题解：** 

```java

```

#### **29、翻转二叉树（ LeetCode 226）**

**题目：** 

**示例：** 

**题解：** 

```java

```

#### **30、除自身以外数组的乘积（ LeetCode 226）**

**题目：** 

**示例：** 

**题解：** 

```java

```

#### **31、搜索二维矩阵 II（ LeetCode 240）**

**题目：** 

**示例：** 

**题解：** 

```java

```

#### **32、移动零（ LeetCode 283）**

**题目：** 

**示例：** 

**题解：** 

```java

```

#### **32、寻找重复数（ LeetCode 287）**

**题目：** 

**示例：** 

**题解：** 

```java

```

#### **33、删除无效的括号（ LeetCode 301）**

**题目：** 

**示例：** 

**题解：** 

```java

```

#### **34、戳气球（ LeetCode 312）**

**题目：** 

**示例：** 

**题解：** 

```java

```

#### **35、打家劫舍 III（ LeetCode 337）**

**题目：** 

**示例：** 

**题解：** 

```java

```

#### **35、前 K 个高频元素（ LeetCode 347）**

**题目：** 

**示例：** 

**题解：** 

```java

```

#### **36、字符串解码（ LeetCode 394）**

**题目：** 

**示例：** 

**题解：** 

```java

```

#### **37、打家劫舍 III（ LeetCode 337）**

**题目：** 

**示例：** 

**题解：** 

```java

```

#### **38、除法求值（ LeetCode 399）**

**题目：** 

**示例：** 

**题解：** 

```java

```

#### **39、根据身高重建队列（ LeetCode 406）**

**题目：** 

**示例：** 

**题解：** 

```java

```

#### **40、路径总和 III（ LeetCode 437）**

**题目：** 

**示例：** 

**题解：** 

```java

```

#### **41、找到字符串中所有字母异位词（ LeetCode 438）**

**题目：** 

**示例：** 

**题解：** 

```java

```

#### **42、找到所有数组中消失的数字（ LeetCode 448）**

**题目：** 给你一个含 n 个整数的数组 nums ，其中 nums[i] 在区间 [1, n] 内。请你找出所有在 [1, n] 范围内但没有出现在 nums 中的数字，并以数组的形式返回结果。

**示例：** 
```
输入：nums = [4,3,2,7,8,2,3,1]
输出：[5,6]
```

**题解：** 

```java
class Solution {
    public List<Integer> findDisappearedNumbers(int[] nums) {
        int n = nums.length;
        // 这种解法有一定的局限性，只能是1到n的，如果数组中存在0，这种解法会有问题
        for(int num : nums) {
            int x = (num - 1) % num;
            nums[x] += n;
        }
        List<Integer> list = new List<>();
        for(int i = 0; i < n; i++) {
            if(nums[i] <= n) {
                list.add(i + 1)；
            }
        }
        return list;
    }
}
```

#### **42题的变型Ⅰ、无重复数字的丢失的数字（ LeetCode 268） **

**题目：** 给定一个包含 [0, n] 中 n 个数的数组 nums ，找出 [0, n] 这个范围内没有出现在数组中的那个数，有重复的。

**示例：** 
```
输入：nums = [3,0,1]
输出：2
解释：n = 3，因为有 3 个数字，所以所有的数字都在范围 [0,3] 内。2 是丢失的数字，因为它没有出现在 nums 中。
```

**题解：** 
```java
class Solution {
    public int missingNumber(int[] nums) {
        // 直接将数组进行排序，因为数组中每个数字仅有一个
        Arrays.sort(nums);
        int n = nums.length;
        // 遍历数组，第一个出现下标和所在位置上的值不相等时候，就是返回这个下标
        for (int i = 0; i < n; i++) {
            if (nums[i] != i) {
                return i;
            }
        }
        return n;
    }
}

```

#### **42题的变型Ⅱ、有重复数字的丢失数字**

**题目：** 给你一个整数的数组 nums ，其中 nums[i] 在区间 [0, n] 内。请你找出所有在 [1, n] 范围内但没有出现在 nums 中的数字，并以数组的形式返回结果。

**示例：** 
```
输入：nums = [0,4,4,3,2,7,8,2,3]
输出：[1,5,6]
```

**题解：** 
```java
import java.util.ArrayList;
import java.util.Arrays;
import java.util.HashSet;
import java.util.List;

public class Test {

    public static void main(String[] args) {
        int[] nums = new int[]{0, 1, 0, 5, 4, 4, 3, 7, 4, 10};
        List<Integer> list = missingNumber(nums);
        System.out.println(Arrays.toString(list.toArray()));
    }

    
    public static List<Integer> missingNumber(int[] nums) {
        // 用hashset将数据存储一遍，同时过滤相同的元素
        HashSet<Integer> set = new HashSet<>();
        for (int num : nums) {
            set.add(num);
        }
        // 排序，取最后一个，即最大的数max，可以将nums[i] 在区间 [0, n] 内 看作是nums[i] 在区间 [0, max] 
        Arrays.sort(nums);
        int max = nums[nums.length - 1];

        List<Integer> list = new ArrayList<>();

        // 从0开始遍历，然后不在set里的。把它放到list中。
        for (int i = 0; i <= max; i++) {
            if(!set.contains(i)){
                list.add(i);
            }
        }
        return list;
    }
}
```

#### **43、汉明距离（ LeetCode 461）**

**题目：** 

**示例：** 

**题解：** 

```java

```

#### **45、二叉树的直径（ LeetCode 543）**

**题目：** 

**示例：** 

**题解：** 

```java

```

#### **46、和为 K 的子数组（ LeetCode 560）**

**题目：** 

**示例：** 

**题解：** 

```java

```

#### **47、最短无序连续子数组（ LeetCode 581）**

**题目：** 

**示例：** 

**题解：** 

```java

```

#### **48、和为 K 的子数组（ LeetCode 560）**

**题目：** 

**示例：** 

**题解：** 

```java

```

#### **49、合并二叉树（ LeetCode 617）**

**题目：** 

**示例：** 

**题解：** 

```java

```

#### **50、任务调度器（ LeetCode 627）**

**题目：** 

**示例：** 

**题解：** 

```java

```

#### **51、找出数组中的所有孤独数字（ LeetCode 2150）**

**题目：** 

**示例：** 

**题解：** 

```java

```

#### **52、找出数组排序后的目标下标（ LeetCode 2089）**

**题目：** 

**示例：** 

**题解：** 

```java

```

#### **53、找出数组中的幸运数（ LeetCode 1394）**

**题目：** 

**示例：** 

**题解：** 

```java

```

