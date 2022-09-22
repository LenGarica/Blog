# 面向大厂编程总结——企业真题版

## **一、美团**
## **二、字节**
## **三、阿里**
## **四、大疆**
### 4.1 简单难度
#### 1. LeetCode1 两数之和
**题目：** 给定一个整数数组 nums 和一个整数目标值 target，请你在该数组中找出 和为目标值 target  的那 两个 整数，并返回它们的数组下标。你可以假设每种输入只会对应一个答案。但是，数组中同一个元素在答案里不能重复出现。你可以按任意顺序返回答案。

**示例：** 
```
输入：nums = [2,7,11,15], target = 9
输出：[0,1]
解释：因为 nums[0] + nums[1] == 9 ，返回 [0, 1] 。
```

**题解：** 
```java
// 使用哈希表将值作为key，对应的下标作为value 
class Solution {
    public int[] twoSum(int[] nums, int target) {
        Map<Integer, Integer> map = new HashMap<>();
        // 循环遍历
        for(int i=0; i< nums.length; i++){
          // 如果map中存放了目标-当前固定值，则直接返回两个数的下标数组即可
          if(map.containsKey(target - nums[i])){
            return new int[]{map.get(target - nums[i]), i};
          }
          // 将所有所给数字和其对应的下标存入map中
          map.put(nums[i],i);
        }
      return new int[0];
    }
}
```

#### 2. LeetCode69 x的平方根
**题目：** 给你一个非负整数 x ，计算并返回 x 的 算术平方根。由于返回类型是整数，结果只保留 整数部分 ，小数部分将被 舍去 。注意：不允许使用任何内置指数函数和算符，例如 pow(x, 0.5) 或者 x ** 0.5 。

**示例：** 
```
示例 1：
输入：x = 4
输出：2

示例 2：
输入：x = 8
输出：2
解释：8 的算术平方根是 2.82842..., 由于返回类型是整数，小数部分将被舍去。
```

**题解：** 
```java
// 二分法解决 ，因为对任何x来说，他的平方根一定小于等于它本身，因此可以使用二分查找进行判断
class Solution {
    public int mySqrt(int x) {
      int l = 0, r = x, ans = -1;
      while(l <= r){
        int mid = l + (r-l)/2;
        if((long) mid * mid <= x){
          ans = mid;
          l = mid + 1;
        }else {
          r = mid - 1;
        }
      }
      return ans;
    }
}
```
#### 3. LeetCode206. 反转链表
**题目：** 给定一个链表，请将其反转

**示例：** 
```
输入：head = [1,2,3,4,5]
输出：[5,4,3,2,1]
```

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
    public ListNode reverseList(ListNode head) {
        // 递归法
        // return reverseList1(head);
        // 迭代法
        return reverseList2(head);
    }

    // 递归法
    public ListNode reverseList1(ListNode head){
        // 首先写递归函数出口
        if(head == null || head.next == null) {
            return head;
        }
        // 递归入口
        ListNode result = reverseList1(head.next);
        // 反转
        head.next.next = head;
        // 断链
        head.next = null;
        return result;

    }

     // 迭代法
    public ListNode reverseList2(ListNode head){

        // 需要三个指针，分别指向前驱、后继和当前指针
        ListNode pre = null;
        ListNode cur = head;

        while(cur != null){
            ListNode next = cur.next;
            // 反转
            cur.next = pre;
            pre = cur;
            cur = next;
        }
        return pre;
    }
}
```

#### 4. LeetCode234. 回文链表
**题目：** 给你一个单链表的头节点 head ，请你判断该链表是否为回文链表。如果是，返回 true ；否则，返回 false 。

**示例：** 
```
输入：head = [1,2,2,1]
输出：true
```

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
    public boolean isPalindrome(ListNode head) {
        // 首先判断所给链表是否满足条件
        if(head == null || head.next == null){
            return true;
        }

        // 定义快慢指针
        ListNode fast = head, slow = head;
        ListNode pre = null;
        // 循环遍历，让前面的链表反转
        while(fast != null && fast.next != null){
            ListNode next = slow.next;
            // 走两步
            fast = fast.next.next;
            // 反转
            slow.next = pre;
            pre = slow;
            slow = next;
        }

        // 判断奇数个node时，slow没有到应该到的位置
        if(fast != null) {
            slow = slow.next;
        }

        // 比较相同
        while(pre != null && slow!=null){
            if(pre.val != slow.val){
                return false;
            }
            pre = pre.next;
            slow = slow.next;
        }
        return true;
    }
}
```


#### 5. LeetCode1550. 存在连续三个奇数的数组
**题目：** 给你一个整数数组 arr，请你判断数组中是否存在连续三个元素都是奇数的情况：如果存在，请返回 true ；否则，返回 false 。

**示例：** 
```
输入：arr = [2,6,4,1]
输出：false
解释：不存在连续三个元素都是奇数的情况。

输入：arr = [1,2,34,3,4,5,7,23,12]
输出：true
解释：存在连续三个元素都是奇数的情况，即 [5,7,23] 。
```

**题解：** 
```java
class Solution {
    public boolean threeConsecutiveOdds(int[] arr) {
        for(int i = 0; i < arr.length - 2; i++){
            if((arr[i] & 1)!=0 && (arr[i+1] & 1) !=0 && (arr[i+2] & 1 )!= 0)         {
                return true;
            }
        }
        return false;
    }
}
```

#### 6. 重复元素



#### 7. 重复元素Ⅱ



### 4.2 中等难度
#### 1. 剑指offer35
**题目：** 

**示例：** 
```
```

**题解：** 
```java

```

### 4.3 困难难度
#### 1. LeetCode1675 数组最小偏移量
**题目：** 

**示例：** 
```
```

**题解：** 
```java

```

## **5.百度**