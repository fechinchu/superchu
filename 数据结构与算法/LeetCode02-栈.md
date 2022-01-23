# LeetCode02-栈

# 1.用两个栈实现队列(简单)

https://leetcode-cn.com/problems/yong-liang-ge-zhan-shi-xian-dui-lie-lcof/

![image-20211124142835054](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211124142835054.png)

```java
class CQueue {

    private Stack<Integer> stack01;
    private Stack<Integer> stack02;

    public CQueue() {
        stack01 = new Stack<>();
        stack02 = new Stack<>();
    }

    public void appendTail(int value) {
        while (!stack02.isEmpty()) {
            Integer pop = stack02.pop();
            stack01.push(pop);
        }
        stack01.push(value);
    }

    public int deleteHead() {
        while (!stack01.isEmpty()) {
            Integer pop = stack01.pop();
            stack02.push(pop);
        }
        if (stack02.isEmpty()) {
            return -1;
        }
        return stack02.pop();
    }
}
```

# 2.用队列实现栈

