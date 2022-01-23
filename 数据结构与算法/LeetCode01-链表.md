# LeetCode01-链表

# 1.解题技巧

链表问题都会涉及遍历,核心是通过"画图举例"确定遍历的"三要素";

* 三要素
  * 指针的初始值: `p = head or ...`;
  * 遍历的结束条件:`p == null or p.next == null`;
  * 遍历的核心逻辑:...;

* 特殊情况:是否要对头节点,尾节点,空链表等做特殊处理?

* 引入虚拟节点:是否可以通过添加虚拟节点简化编程?

* 引入尾指针:是否可以引入尾指针简化编程?

 # 2.移除链表元素(简单)

https://leetcode-cn.com/problems/remove-linked-list-elements/

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210829163617728.png" alt="image-20210829163617728" style="zoom:50%;" />

~~~java
class Solution {
    public ListNode removeElements(ListNode head, int val) {
           if(head == null) return null;
           ListNode p = head;
           //新链表中的哨兵节点
           ListNode newHead = new ListNode(-1,null);
           ListNode tail = newHead;
           while(p!=null){
               //这里借助一个tmp
               ListNode temp = p.next;
               if(p.val != val){
                    tail.next = p;
                    tail = p; 
                    p.next = null;    
               }
               p = temp;
           }
           return newHead.next;            
    }
}
~~~

# 3.链表的中间结点(简单)

https://leetcode-cn.com/problems/middle-of-the-linked-list/

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210829163643515.png" alt="image-20210829163643515" style="zoom:50%;" />

~~~java
class Solution {
    public ListNode middleNode(ListNode head) {
        //if(head == null) return null;这一行可以省略,因为题目提示头结点为head的非空单链表
        ListNode fast = head;
        ListNode slow = head;
        while(fast != null && fast.next != null){
            fast = fast.next.next;
            slow = slow.next;
        }
      return slow;
    }
}
~~~

这种题可以遍历2次,也可以通过快慢指针的方式;

针对快慢指针,分两种情况来看,情况1是奇数个节点,情况2是偶数个节点;通过画图可以知道

![image-20210829172233448](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210829172233448.png)

# 4.删除排序链表中的重复元素(简单)

https://leetcode-cn.com/problems/remove-duplicates-from-sorted-list/

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210829172616043.png" alt="image-20210829172616043" style="zoom:50%;" />

~~~java
class Solution {
    public ListNode deleteDuplicates(ListNode head) {
        if(head == null) return null;
        ListNode newHead = head;
        ListNode newTail = head;
        ListNode p = head.next;
        int temVal = head.val;
        while(p!=null){     
            ListNode temp = p.next;
            if(p.val != temVal){
                newTail.next = p;
                newTail = p;
                temVal = p.val;
            }
            newTail.next = null;
            p = temp;

        }
        return newHead;
    }
}
~~~

# 5.合并两个排序的链表(中等)

https://leetcode-cn.com/problems/he-bing-liang-ge-pai-xu-de-lian-biao-lcof/

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210829172804763.png" alt="image-20210829172804763" style="zoom:50%;" />

~~~java
class Solution {
    public ListNode mergeTwoLists(ListNode l1, ListNode l2) {
        //特殊情况处理;
      	if(l1==null) return l2;
      	if(l2==null) return l1;
        ListNode newHead = new ListNode(-1);
        ListNode newTail = newHead;
        ListNode p = l1;
        ListNode q = l2;
        while(p!=null&&q!=null){
            if(p.val<=q.val){
                newTail.next = p;
                newTail = p;
                p = p.next;
            }else{
                newTail.next = q;
                newTail = q;
                q = q.next;
            }
        }
        if(p==null){
            newTail.next = q;
        }else{
            newTail.next = p;
        }
        return newHead.next;
    }    
}
~~~

![image-20210829173729274](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210829173729274.png)

# 6.两数相加(中等)

https://leetcode-cn.com/problems/add-two-numbers/

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210829175313177.png" alt="image-20210829175313177" style="zoom:50%;" />

~~~java
class Solution {
    public ListNode addTwoNumbers(ListNode l1, ListNode l2) {
        ListNode newHead = new ListNode(-1,null);
        ListNode newTail = newHead;
        ListNode p = l1;
        ListNode q = l2;
        int temp = 0;
        while(p!=null||q!=null){
            if(q==null){
                q = new ListNode(0,null);
            }else if(p == null){
                p = new ListNode(0,null);
            }
            int result = p.val+q.val;
            int val = (result+temp)%10;
            ListNode newNode = new ListNode(val,null);
            newTail.next = newNode;
            newTail = newNode;
            //if((result+temp)>=10){
                //temp = 1;
            //}else{
                //temp = 0;
            //} 
          	//上述代码优化为:
            temp = result+temp/10;
            q = q.next;
            p = p.next;
        }
        if(temp == 1){
            ListNode resultTail = new ListNode(1,null);
            newTail.next = resultTail;
            newTail = resultTail;
        }
        return newHead.next;
    }
}
~~~

# 7.反转链表(中等)**

https://leetcode-cn.com/problems/reverse-linked-list/

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210829175349423.png" alt="image-20210829175349423" style="zoom:50%;" />

~~~java
class Solution {
    public ListNode reverseList(ListNode head) {
        if(head == null) return null;
        ListNode p = head;
        ListNode newHead = null;
        while(p!=null){
            ListNode temp = p.next;
            p.next = newHead;
            newHead = p;
            p = temp;
        }
        return newHead;
    }
}
~~~

![image-20210829190134954](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210829190134954.png)

# 8.回文链表(中等)**

https://leetcode-cn.com/problems/palindrome-linked-list/

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210829190340899.png" alt="image-20210829190340899" style="zoom:50%;" />

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210829220210485.png" alt="image-20210829220210485" style="zoom:50%;" />

~~~java
class Solution {
    public boolean isPalindrome(ListNode head) {
        if(head.next == null) return true;
        //回文列表直接转化为数组去判断
        ListNode mid = getMidNode(head);
        ListNode newRightLink = flip(head,mid);
        ListNode x = head;
        while(x != null){
            if(x.val != newRightLink.val){
                return false;
            }
            newRightLink = newRightLink.next;
            x = x.next;
        }
        return true;
    }

    private ListNode getMidNode(ListNode head){
        ListNode fast = head;
        ListNode slow = head;
        while(fast != null&&fast.next != null){
            fast = fast.next.next;
            slow = slow.next;
        }
        return slow;
    }

    private ListNode flip(ListNode head,ListNode mid){
        ListNode prev = head;
        while(prev.next!=mid){
            prev = prev.next;
        }
        prev.next = null;
        ListNode newHead = null;
        ListNode p = mid;
        while(p != null){
            ListNode temp = p.next;
            p.next = newHead;
            newHead = p;
            p = temp;
        }
        
        return newHead;
    }
}
~~~

# 9.奇偶链表(中等)

https://leetcode-cn.com/problems/odd-even-linked-list/submissions/

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210829221327466.png" alt="image-20210829221327466" style="zoom:50%;" />

~~~java
class Solution {
    public ListNode oddEvenList(ListNode head) {
        if(head == null) return null;
        if(head.next == null) return head;
        ListNode jhead = new ListNode(-1,null);
        ListNode jtail = jhead;
        ListNode ohead = new ListNode(-1,null);
        ListNode otail = ohead;
        ListNode p = head;
        //while(p != null){            
            //ListNode temp = p.next; 
            //jtail.next = p;
            //jtail = p;
            //jtail.next = null;
            //if(temp == null){
                //break;
            //}
            //p = temp;
            //ListNode temp2 = p.next;
            //otail.next = p;
            //otail = p;
            //otail.next = null;
            //p = temp2;
        //}
        //上述实现不够优雅
        int count = 1;
        while(p!=null){
            ListNode tmp = p.next;
            if(count % 2 == 1){
                p.next = null;
                jtail.next = p;
                jtail = p;
            }else{
                p.next = null;
                otail.next = p;
                otail = p;
            }
            count++;
            p = tmp;
        }
        jtail.next = ohead.next;
        return jhead.next;
    }
}
~~~

# 10.K个一组翻转链表(困难)\****

https://leetcode-cn.com/problems/reverse-nodes-in-k-group/

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210829221524131.png" alt="image-20210829221524131" style="zoom:50%;" />

# 11.链表中倒数第k个节点

https://leetcode-cn.com/problems/lian-biao-zhong-dao-shu-di-kge-jie-dian-lcof/

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210830212612476.png" alt="image-20210830212612476" style="zoom:50%;" />

思路参考第12题

~~~java
class Solution {
    public ListNode getKthFromEnd(ListNode head, int k) {
        ListNode p = head;
        ListNode q = head;
        for(int i = 0;i<k-1;i++){
            q = q.next;
            if(q == null){
                return head;
            }
        }

        while(q.next != null){
            q = q.next;
            p = p.next;
        }
        return p;
    }
}
~~~

# 12.删除链表的倒数第N个结点(中等)

https://leetcode-cn.com/problems/remove-nth-node-from-end-of-list/

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210830212731990.png" alt="image-20210830212731990" style="zoom:50%;" />

![image-20210830214137920](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210830214137920.png)

~~~java
class Solution {
    public ListNode removeNthFromEnd(ListNode head, int n) {
        if(head == null) return null;
        ListNode prev = new ListNode();
        prev.next = head;
        ListNode p = head;
        ListNode q = head;
        for(int i = 0; i < n-1;i++){
            q = q.next;
            if(q == null) return head;
        }

        while(q.next != null){
            p = p.next;
            q = q.next;
            prev = prev.next;
        }

        prev.next =prev.next.next;
        if(p == head){
            return head.next;
        }
        return head;

    }
}
~~~

# 13.相交链表(简单)

https://leetcode-cn.com/problems/intersection-of-two-linked-lists/

![image-20210830214329967](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210830214329967.png)

~~~java
public class Solution {
    public ListNode getIntersectionNode(ListNode headA, ListNode headB) {
        //暴力求解法
        ListNode newHeadOne = new ListNode(-1);
        ListNode p = newHeadOne;
        newHeadOne.next = headA;
        ListNode newHeadTwo = new ListNode(-1);
        ListNode q = newHeadTwo;
        newHeadTwo.next = headB;
        while(p!=null){
            while(q!=null){
                if(p.next == q.next){
                    return p.next;
                }
                q = q.next;
            }
            q = newHeadTwo;
            p = p.next;
        }
        return null;
    }
}
~~~

上述是时间复杂度最高为O(n^2)的暴力解法

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210830231939187.png" alt="image-20210830231939187" style="zoom:50%;" />

~~~java
public class Solution {
    public ListNode getIntersectionNode(ListNode headA, ListNode headB) {
        ListNode a = headA;
        ListNode b = headB;
        int countA = 1;
        int countB = 1;
        while(a!=null){
            a = a.next;
            countA++;
        }
        while(b!=null){
            b = b.next;
            countB++;
        }
        a = headA;
        b = headB;
        if(countA-countB>=0){
            for(int i = 0;i<countA-countB;i++){
                a = a.next;
            }
        }else{
            for(int i = 0;i<countB-countA;i++){
                b = b.next;
            }
        }
        while(b!=null){
            if(b == a){
                return a;
            }
            b = b.next;
            a = a.next;
        }
        return null;
    }
}
~~~

# 14.环形链表(简单)

https://leetcode-cn.com/problems/linked-list-cycle/

![image-20210830232103310](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210830232103310.png)

~~~java
public class Solution {
    public boolean hasCycle(ListNode head) {
        if(head==null||head.next == null) return false;
        ListNode newQuickHead = new ListNode(-1);
        ListNode newSlowHead = new ListNode(-1);
        newQuickHead.next = head;
        newSlowHead.next = head;
        ListNode q = newQuickHead;
        ListNode p = newSlowHead;
        while(p!=null&&q!=null&&p.next!=null&&q.next!=null){
            if(p.next == q.next.next){
                return true;
            }
            p = p.next;
            q = q.next.next;
        }
        return false;
    }
}
~~~

