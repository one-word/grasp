## 笔试题：旋转链表
给你一个链表的头节点 head ，旋转链表，将链表每个节点向右移动 k 个位置。

1 2 3 4 5
==> 5 1 2 3 4
==> 4 5 1 2 3

```
ListNode* rotateRight(ListNode* head, int k) {
        if (k == 0 || head == nullptr || head->next == nullptr) {
            return head;
        }
        int n = 1;
        ListNode* iter = head;
        while (iter->next != nullptr) {
            iter = iter->next;
            n++;
        }
        int add = n - k % n;
        if (add == n) {
            return head;
        }
        iter->next = head;
        while (add--) {
            iter = iter->next;
        }
        ListNode* ret = iter->next;
        iter->next = nullptr;
        return ret;
    }
}
```

