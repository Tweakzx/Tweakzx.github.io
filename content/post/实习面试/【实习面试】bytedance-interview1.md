---
title:  "【实习面试】字节面试凉经"
author: "Tweak"
date:   2021-11-29T16:52:26+08:00

description: "记录自己的字节面试失败过程"
tags:
  - 面试
  - 字节跳动

categories: 
  - 面试

image: "https://cdn.jsdelivr.net/gh/Tweakzx/ImageHost@main/img/bytedance.jpg"
---



> 时间：2021-11-29下午三点
>
> 方式：飞书视频面试
>
> 岗位：后端开发实习生-业务中台职位

##  为什么会投递字节实习

​		因为中科院的研究所大多不让实习，研二想去实习是一定不行的。但是研一在雁栖湖，课题组在海淀，我研一基本不参与科研，所以想要趁没人管的时候出去实习。因为将来大概率是做软开，所以想找一份后端开发的工作。在校友群里问了一下，有学长给了一个内推的机会，所以就有了这次一面。

##  面试过程

​		面试官上来让我做了自我介绍之后就让我写代码做题。

> 题目一：链表1->2->3->4->5->6 转变成 链表1->6->2->5->3->4 
>
> 大概意思就是将链表平分成两部分，前一部分顺序不变，后一部分反转后插空插入前一部分链表。

​		说实话，自己的代码能力确实有点差，原本打算使用c++来写后来改成了Java，因为我想起Java里有LinkedList这个数据结构。理由有点荒诞哈，其实是我没有意识到要自己定义节点来实现链表，所以在想要写指针的时候就卡住了，完全不知道怎么给LinkedList的链表写指针。然后就尬住了，面试官无奈说那就换一道吧。

> 题目二：判断一个二叉树是否是平衡树

​		题目不是很难，但是自己太久不写代码有些生疏了，写出来的程序不知道为什么没有编译通过。说实话，面试官让我自己实现一个单例来测试程序，其中爆出的各种错误，无一不揭示了我完全没有什么开发经验的事实，例如内部类放错了位置，static的编译问题，还有一个空指针异常。无疑是再次尬死了。

​		之后面试官让我讲一讲自己的项目经历，主要问了一下大三在华为的实习。可惜自己虽然实习了，但实际的工作很少，含金量也不高，面试官兴趣不大。

​		之后就是问我有没有什么想问他的，我已经不想说话了，就没问。

​		草草结束。

##  总结

###  首先还是把这两道题的代码写写吧

####  题目一

```java
/**
 * @author lizhixuan
 * @version 1.0
 * @date 2021/12/1 15:50
 */
public class ReverseList2 {
    public static class ListNode {
        int val;
        ListNode next;
        ListNode() {}
        ListNode(int val) { this.val = val; }
        ListNode(int val, ListNode next) { this.val = val; this.next = next; }
    }
    public static ListNode createInstance(int n){
        ListNode head = new ListNode(1);
        ListNode current = head;
        for (int i = 2; i <= n; i++) {
            current.next = new ListNode(i);
            current = current.next;
        }
        return head;
    }
    public static void printList(ListNode head){
        ListNode current = head;
        while(current.next!=null){
            System.out.print(current.val);
            System.out.print("->");
            current = current.next;
        }
        System.out.println(current.val);
    }
    public static ListNode reverse(ListNode head){
        ListNode pre = null;
        ListNode current = head;
        ListNode next;
        while(current!=null){
            next = current.next;
            current.next = pre;
            pre = current;
            current = next;
        }
       return pre;
    }

    public static ListNode mergeList(ListNode l1,ListNode l2){
        ListNode head = l1;
        ListNode l1Next;
        ListNode l2Next;
        while(l1!=null&&l2!=null){
            l1Next = l1.next;
            l2Next = l2.next;

            l1.next = l2;
            l2.next = l1Next;

            l1 = l1Next;
            l2 = l2Next;
        }
        return head;
    }

    public static ListNode reverseHalf(ListNode head){
        ListNode fast = head;
        ListNode slow = head;
        while(fast != null){
            if(fast.next == null){
                break;
            }
            if(fast.next.next == null){
                break;
            }
            fast = fast.next.next;
            slow = slow.next;
        }
        ListNode half = reverse(slow.next);
        slow.next = null;

        return mergeList(head,half);
    }

    public static void main(String[] args) {
        ListNode head = createInstance(7);
        printList(head);
        head = reverseHalf(head);
        printList(head);
    }
}
```

####  题目二

```java
import java.util.LinkedList;
import java.util.Queue;

/**
 * @author lizhixuan
 * @version 1.0
 * @date 2021/12/1 17:24
 */
public class BalanceCheck {
    public static class TreeNode{
        int val;
        TreeNode left;
        TreeNode right;
        TreeNode(int val){this.val = val;}
    }
    public static TreeNode listToTree(String src){
        src = src.substring(1,src.length()-1);
        String[] strList = src.split(",");

        TreeNode root  ;
        TreeNode result = null;
        Queue<TreeNode> queue = new LinkedList<>();
        for (int i =0 ; i< strList.length ; i++){
            if (i == 0){
                root = new TreeNode(Integer.parseInt(strList[i]));
                result = root;
                queue.add(root);
            }
            if (!queue.isEmpty()){
                root = queue.poll();
            }else {
                break;
            }
            if ( i+1 < strList.length  && !strList[i+1].equals( "null")){
                root.left = new TreeNode(Integer.parseInt(strList[i +1]));
                queue.add(root.left);
            }
            if ( i + 2 < strList.length && !strList[i+2].equals( "null")){
                root.right = new TreeNode(Integer.parseInt(strList[i +2]));
                queue.add(root.right);
            }
            i = i +1;
        }
        return result;
    }
    public static int getHeight(TreeNode root){
        if(root==null){
            return 0;
        }
        return Math.max(getHeight(root.left),getHeight(root.right))+1;
    }
    public static boolean isBalanced(TreeNode root){
        if(root==null){
            return true;
        }
        int leftHeight = getHeight(root.left);
        int rightHeight = getHeight(root.right);
        return Math.abs(leftHeight-rightHeight)<=1 && isBalanced(root.left) && isBalanced(root.right);
    }

    public static void main(String[] args) {
        String tree = "[3,9,20,null,null,15,7]";
        TreeNode root = listToTree(tree);
        System.out.println(isBalanced(root));
    }
}
```

###  一些知识点

- java内部类：
  - 成员内部类中不能存在任何static的变量和方法
  - 成员内部类是依附于外围类的，所以只有先创建了外围类才能够创建内部类

- java的static关键字
  - static变量也称作静态变量，静态变量被所有的对象所共享，在内存中只有一个副本，它当且仅当在类初次加载时会被初始化。
  - 为什么说static块可以用来优化程序性能，是因为它的特性:只会在类加载的时候执行一次。
  - 在C/C++中static是可以作用域局部变量的，但是在Java中切记：static是不允许用来修饰局部变量。

###  反思

- 自己还是缺乏代码经验，代码写不出来，面试官明显十分失望，失去耐心之后多次叹气捂脸，说实话压力还是挺大的，毕竟确实挺丢脸的。以后也要多写一点代码。
- 打算把自己的学习总结下来，于是也有了这篇博客，希望自己可以坚持写博客，说实话，面试完压力挺大，感觉自己就是一个无敌铁废物，写博客写出来感觉好一些了。

