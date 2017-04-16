# LeetCode 148. Sort List


今天做美图的笔试题，要求是用快排给单链表排序。虽然这不是最优的排序方式，但既然要求了还是写一写。

---

算法思路：

单链表排序与数组排序思路是一样的，只是两个游标的移动方式不同。

单链表排序的两个游标向同一方向移动，p、q 代指两个游标， p 左侧的都是比枢轴小的，p 到 q 是比枢轴大的，q 右侧的是还未排序的


```java
    private static void quickSort(Node head, Node tail) {
        if (head != tail && head.next != tail) {
            // 至少含有两个元素才排序
            Node pivot = partition(head, tail);
            quickSort(head, pivot);
            quickSort(pivot.next, tail);
        }
    }


    private static Node partition(Node head, Node tail) {
        Node cur = head.next; // 游标 q
        Node mid = head.next; // 游标 p
        Node midPrev = head;
        Node pivot = null;
        int pivotVal = head.val;

        do {
            while (mid.next != tail && mid.val <= pivotVal) {
                // 找到下一个比枢轴大的元素
                midPrev = mid;
                mid = mid.next;
            }
            if (cur == head.next) {
                // q 游标第一次移动前的初始化，保证 q 在 p 右侧
                cur = mid;
            }
            while (cur.next != tail && cur.val > pivotVal) {
                // 找到下一个比枢轴小的元素
                cur = cur.next;
            }
            if (cur.next != tail && mid.next != tail) {
                // 如果两个游标都未到结尾
                swap(cur, mid);
            } else {
                if (cur.next == tail && cur.val < pivotVal) {
                    // 如果 q 游标到结尾了并且该节点的值小于枢轴
                    swap(mid, cur);
                }

                // 将枢轴交换到中间，保证左侧都小于枢轴，右侧都大于枢轴
                if (mid.val < pivotVal) {
                    swap(mid, head);
                    pivot = mid;
                } else {
                    swap(midPrev, head);
                    pivot = midPrev;
                }
            }
            cur = cur.next;
            midPrev = mid;
            mid = mid.next;
        } while (cur != tail && mid != tail);

        return pivot;
    }

    static void swap(Node node1, Node node2) {
        int temp = node1.val;
        node1.val = node2.val;
        node2.val = temp;
    }

class Node {
    public int val;
    public Node next;

    public Node(int data) {
        this.val = data;
    }
}
```


## 参考

1. [单链表的快速排序 ](http://blog.csdn.net/wumuzi520/article/details/8078322)
