---
layout: post
title: Double linked list
date: 2019-03-22
author: Huang Chihsiao
header-img: img/green.jpg
catalog: true
tags: linkedlist, linked list
keywords: linkedlist,linked list
---

**民國108年3月17日**

**說明:**文章關於二元樹的算法根據 "Data Structures and Algorithms:Annotated Reference with Examples" 這本書的僞代碼通過C++實現而得，此外本文用於說明的插圖也出自此書。

**1.雙向鏈表結構**
```c++
typedef struct Node {
    int data;
    struct Node *prev;   //指向前一個node的指標
    struct Node *next;  //指向後一個node的指標
} linkedList;
```
節點node是鏈表的基本單元，一個節點包含三個要素：數據，前指標，後指標。圖示如下,圖出自"Data Structures and Algorithms:
Annotated Reference with Examples"，第十五頁：
![ ](https://github.com/huangwenshan1999/huangwenshan1999.github.io/raw/master/post_img/double-linked-list/double_linked_list_p15.png  "Double linked list")

##二、雙向鏈表的操作
**1.後插法**
新的節點每次都添加到鏈表的尾端。
```c++
void append(linkedList **head, int value) {
    auto *newBlock = new(linkedList);
    newBlock->data = value;
    if (*head == nullptr) {
        std::cout << "double linked list is empty" << std::endl;
        *head = newBlock;
        newBlock->next = nullptr;
    } else {
        auto *p = *head;
        while (p->next != nullptr) {
            p = p->next;
        }
        p->next = newBlock;
        newBlock->prev = p;
        newBlock->next = nullptr;
    }
}
```
**2.前插法**
新節點每次都在鏈表頭添加
```c++
void insertFront(linkedList **head, int value) {
    auto newNode = new(linkedList);
    newNode->data = value;
    newNode->next = *head;
    newNode->prev = nullptr;

    if (*head != nullptr) {
        (*head)->prev = newNode;
    }
    //移動head,把newNode作爲新的head
    *head = newNode;
}
```
**3.在特定節點前插入**
此插入法需要一個輔助函式來幫助尋找要插入的位置,此處的輔助函式是getNode()完成
```c++
linkedList *getNode(linkedList *head, int value) {
    while (head != nullptr && head->data != value) {
        head = head->next;
    }
    //返回包含value值的節點的地址。
    return head;
}

void insertBefore(linkedList **head, linkedList *nextNode, int value) {
    if (nextNode == nullptr) {
        std::cout << "Given nextNode is null" << std::endl;
        return;
    }
    auto *newNode = new(linkedList);
    newNode->data = value;
    //newNode在nextNode之前插入，需要把nextNode的前節點地址賦值給newNode->prev,即讓newNode指向nextNode的前驅。
    newNode->prev = nextNode->prev;
    //newNode的下一個節點就是nextNode，,因爲newNode在nextNode前插入
    newNode->next = nextNode;
    //nextNode的前節點是newNode,因爲newNode在nextNode前插入
    nextNode->prev = newNode;
    if (newNode->prev != nullptr) {
    //newNode不是head節點，告訴newNode的前一個節點，它的下一個節點是newNode
        newNode->prev->next = newNode;
    } else {
    //若newNode前節點爲空，即表明newNode就是head節點，此時的插入操作相當於前插法,讓新插入的newNode變成head節點。
        *head = newNode;
    }
}
```
**4.移除節點**
作用：移除包含value的節點
```c++
bool remove(linkedList **head, int value) {
    if (*head == nullptr) {
        std::cout << "double linked list is empty" << std::endl;
        return false;
    }
    linkedList *n = (*head);
    if (value == n->data) {
        if (n->next == nullptr) {
            std::cout << "There are just one block in the linked list" << std::endl;
            delete *head;
        } else if (n->next != nullptr) {
            std::cout << "We are removing head node" << std::endl;
            *head = n->next;
            delete (*head)->prev;
        }
        return true;
    }
    while (n->next != nullptr && n->data != value) {
        n = n->next;
    }
    if (n->next == nullptr) {
        if (n->data != value) {
            std::cout << "The item doesn't in the linked list" << std::endl;
            return false;
        } else if (n->data == value) {
            std::cout << "Value to be removed is in the tail of linked list" << std::endl;
            delete n;
            n->prev->next = nullptr;
            return true;
        }
    } else if (n->next != nullptr) {
        std::cout << "The item to remove is somewhere in between head and tail" << std::endl;
        n->prev->next = n->next;
        n->next->prev = n->prev;
        delete n;
        return true;
    }
}
```
**5.reverseTraverse**
作用：從表尾循環到表頭(時間複雜度O(n^2))，比單向鏈表的反向遊歷簡單很多
```c++
linkedList *getLastNode(linkedList *head) {
    while (head != nullptr && head->next != nullptr) {
        head = head->next;
    }
    return head;
}

linkedList *reverseTraverse(linkedList *head) {
    if (head == nullptr) {
        std::cout << "The linked list is empty" << std::endl;
        return nullptr;
    } else {
        auto *tail = getLastNode(head);
        while (tail != nullptr && tail->prev != nullptr) {
            tail = tail->prev;
        }
        return tail;
    }
}
```
反向歷遍如下圖所示，圖出自”Data Structures and Algorithms:
Annotated Reference with Examples”，第十七頁：
![ ](https://github.com/huangwenshan1999/huangwenshan1999.github.io/raw/master/post_img/double-linked-list/double_linked_list_p17.png  "reverse traverse")
