# Linked List — Explained Simply (C++)

---

## 1. What Is a Linked List?

An array stores elements in **one continuous block** of memory.
A linked list stores elements in **separate scattered boxes**, each box pointing
to the next one.

```
  ARRAY:  (all elements sit side-by-side in memory)
  ┌─────┬─────┬─────┬─────┬─────┐
  │  10 │  20 │  30 │  40 │  50 │
  └─────┴─────┴─────┴─────┴─────┘
  0x100  0x104  0x108  0x10C  0x110     ← addresses are consecutive


  LINKED LIST:  (elements are scattered, connected by pointers)

  head
   │
   ▼
  ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
  │ data: 10 │     │ data: 20 │     │ data: 30 │     │ data: 40 │
  │ next: ───┼────►│ next: ───┼────►│ next: ───┼────►│ next:NULL │
  └──────────┘     └──────────┘     └──────────┘     └──────────┘
  addr: 0x200      addr: 0x500      addr: 0x120      addr: 0x800
                   ↑ scattered!      ↑ not in order!
```

**Each box is called a "Node".**
Each node holds two things:
1. **data** — the actual value (10, 20, etc.)
2. **next** — a pointer (arrow) to the next node

The last node's `next` is `NULL` (meaning "no more nodes").

---

## 2. struct / class in C++

### What Is a struct?

A struct is just a way to **bundle multiple variables together** into one custom type.

```cpp
// WITHOUT struct — messy, hard to manage
int student1_age = 20;
string student1_name = "Raj";
float student1_gpa = 8.5;

int student2_age = 22;
string student2_name = "Priya";
float student2_gpa = 9.1;
// imagine doing this for 100 students... nightmare!


// WITH struct — clean, organized
struct Student {
    int age;
    string name;
    float gpa;
};

Student s1 = {20, "Raj", 8.5};
Student s2 = {22, "Priya", 9.1};
// everything about one student is in ONE variable
```

### struct vs class in C++

```
  In C++, struct and class are ALMOST IDENTICAL.
  The ONLY difference:

  ┌─────────────────────────────────────────────────────────┐
  │  struct  →  members are PUBLIC by default               │
  │  class   →  members are PRIVATE by default              │
  └─────────────────────────────────────────────────────────┘

  That's it. That's the whole difference.

  struct Student {
      int age;          // PUBLIC (anyone can access)
      string name;      // PUBLIC
  };

  class Student {
      int age;          // PRIVATE (only this class can access)
      string name;      // PRIVATE
  public:
      void setAge(int a) { age = a; }   // you need this to access
  };

  FOR LINKED LISTS: we use struct because we WANT everything public.
  It's simpler. No need for getters/setters for a simple node.
```

### Building a Node struct for Linked List (C++)

```cpp
struct Node {
    int data;       // the value stored in this node
    Node* next;     // pointer to the next node

    // constructor — makes creating nodes easy
    Node(int val) {
        data = val;
        next = nullptr;
    }
};

// Creating a node:
Node* n = new Node(10);   // creates a node with data=10, next=nullptr

// What it looks like in memory:
//  ┌──────────────┐
//  │ data: 10     │
//  │ next: nullptr│
//  └──────────────┘
```

## 5. Difference Between `Node` and `Node*`

This is a **crucial** concept. Let's go slow.

### `Node` = the actual box (the whole object)

```
  Node a(10);      // creates a REAL Node object on the stack

  Stack memory:
  ┌──────────────────┐
  │  a               │
  │  ├─ data: 10     │  ← the actual data lives HERE
  │  └─ next: nullptr│
  └──────────────────┘

  'a' IS the node. It contains the data directly.
  sizeof(a) = 12 or 16 bytes (data + pointer + padding)
```

### `Node*` = a sticky note with the address of a box

```
  Node* p = new Node(10);   // p is just an address, the node is somewhere else

  Stack:                          Heap:
  ┌────────────┐                  ┌──────────────────┐
  │  p = 0x500 │ ────────────────►│  data: 10        │
  │  (8 bytes) │                  │  next: nullptr   │
  └────────────┘                  └──────────────────┘
  p is SMALL                      the actual node is HERE
  (just an address)               (on the heap)

  p does NOT contain the data.
  p just POINTS to where the data is.
  sizeof(p) = 8 bytes (always, regardless of what's in the node)
```
### The `.` vs `->` shorthand

```cpp
  Node a(10);          // a is a Node (object)
  a.data = 20;         // use DOT to access members

  Node* p = &a;        // p is a pointer to a
  (*p).data = 20;      // dereference first, then dot — ugly
  p->data = 20;        // SAME THING, but cleaner!

  //  p->data   is just shorthand for   (*p).data
  //  "follow the arrow, then access the member"
```

---

## 6. Array → Linked List Conversion

### The idea

```
  Given: int arr[] = {10, 20, 30, 40, 50};

  Build:
  head → [10|*] → [20|*] → [30|*] → [40|*] → [50|NULL]
```

### C++ Code

```cpp
#include <iostream>
using namespace std;

struct Node {
    int data;
    Node* next;
    Node(int val) : data(val), next(nullptr) {}
};

// Convert array to linked list
Node* arrayToLL(int arr[], int n) {
    if (n == 0) return nullptr;

    Node* head = new Node(arr[0]);   // first element becomes head
    Node* curr = head;

    for (int i = 1; i < n; i++) {
        curr->next = new Node(arr[i]);  // create new node, link it
        curr = curr->next;              // move curr forward
    }
    return head;
}

int main() {
    int arr[] = {10, 20, 30, 40, 50};
    Node* head = arrayToLL(arr, 5);
    return 0;
}
```

## 7. Traversal in Linked List

Traversal = **visiting every node one by one**, from head to the end.

You can't do `list[3]` like arrays. You MUST start from the head and
follow the arrows.

### C++ Code

```cpp
void printList(Node* head) {
    Node* curr = head;       // start at the beginning

    while (curr != nullptr) {   // while we haven't reached the end
        cout << curr->data << " → ";
        curr = curr->next;      // move to the next node
    }
    cout << "NULL" << endl;
}

// Usage:
// printList(head);
// Output: 10 → 20 → 30 → 40 → 50 → NULL
```
### Why we use `curr` and DON'T move `head`

```
  ⚠️  MISTAKE (never do this):

  while (head != nullptr) {
      cout << head->data;
      head = head->next;     // you just LOST the head!
  }
  // head is now NULL. You can never access the list again!

  ✅  CORRECT:
  Use a separate variable (curr) to walk. Keep head untouched.
```

---

## 8. Length of a Linked List

Count how many nodes there are. Same pattern as traversal — walk through
and count.

### C++ Code

```cpp
int getLength(Node* head) {
    int count = 0;
    Node* curr = head;

    while (curr != nullptr) {
        count++;               // count this node
        curr = curr->next;     // move to next
    }
    return count;
}

// Usage:
// head → [10]──►[20]──►[30]──►[40]──►[50]──►NULL
// getLength(head) returns 5
```

## 9. Search an Element in Linked List

Walk through the list, check each node's data. If found, return `true`
(or the position). If you reach NULL without finding it, return `false`.

### C++ Code: Does it exist? (true/false)

```cpp
bool search(Node* head, int target) {
    Node* curr = head;

    while (curr != nullptr) {
        if (curr->data == target)   // found it!
            return true;
        curr = curr->next;          // not here, check next
    }
    return false;   // reached the end, not found
}

```

---

## 10. Deletion in Linked List

Delete a node that contains a given value.

**Two cases to handle:**

```
  CASE 1: Deleting the HEAD node
  ──────────────────────────────
  Before:  head → [10] → [20] → [30] → NULL
  Delete 10:
           head now points to the NEXT node
  After:   head → [20] → [30] → NULL
           [10] is freed (deleted from memory)


  CASE 2: Deleting a MIDDLE or LAST node
  ───────────────────────────────────────
  Before:  head → [10] → [20] → [30] → NULL
  Delete 20:
           the node BEFORE 20 skips over it
  After:   head → [10] → [30] → NULL
                    ↑ next now points to [30], skipping [20]
           [20] is freed
```

### C++ Code

```cpp
Node* deleteele(Node* head, int ele) {
    if (head == nullptr) return nullptr;     // empty list, nothing to delete

    // Case 1: element is at the head
    if (head->data == ele) {
        Node* newhead = head->next;          // save the next node
        delete head;                          // free the old head
        return newhead;                       // new head is the next node
    }

    // Case 2: element is somewhere in the middle/end
    Node* tmp = head;                         // tmp trails behind curr (previous node)
    Node* curr = head;
    while (curr != nullptr) {
        if (curr->data == ele) {
            tmp->next = tmp->next->next;      // skip over curr (unlink it)
            delete curr;                       // free the deleted node
            break;
        }
        tmp = curr;                            // move tmp to curr's position
        curr = curr->next;                     // move curr forward
    }
    return head;
}
```

---

## 11. Insertion in Linked List

Insert a new node at a given position (1-based).

```
  Insert 99 at position 3:

  Before:  head → [10] → [20] → [30] → [40] → NULL
                    pos1    pos2    pos3

  After:   head → [10] → [20] → [99] → [30] → [40] → NULL
                                  ↑ new node inserted here
                                  prev->next = new node
                                  new node->next = curr (old pos3)
```

### C++ Code

```cpp
Node* insertele(Node* head, int pos, int ele) {
    int cnt = 0;
    Node* prev = nullptr;
    Node* curr = head;

    // Special case: insert at position 1 (new head)
    if (pos == 1) {
        Node* newnode = new Node(ele);
        newnode->next = head;                 // new node points to old head
        return newnode;                        // new node IS the new head
    }

    // Walk to the target position
    while (curr != nullptr) {
        cnt++;
        if (cnt == pos) {
            Node* newnode = new Node(ele);
            prev->next = newnode;             // previous node → new node
            newnode->next = curr;             // new node → current node
        }
        prev = curr;
        curr = curr->next;
    }

    // Insert at the very end (pos == length + 1)
    if (cnt + 1 == pos) {
        Node* newnode = new Node(ele);
        prev->next = newnode;                 // last node → new node
    }
    return head;
}
```

---

## 12. Reverse a Linked List

Flip ALL the arrows so the list goes backward.

```
  Before:  head → [10] → [20] → [30] → [40] → NULL

  After:   NULL ← [10] ← [20] ← [30] ← [40] ← head
           (which is the same as:)
           head → [40] → [30] → [20] → [10] → NULL


  HOW? Use 3 pointers: prev, tmp, front

  Step by step:
  ─────────────
  Start:   prev=NULL   tmp=[10]   front=[20]

           NULL    [10] → [20] → [30] → [40] → NULL
           prev    tmp    front

  Step 1:  Flip [10]'s arrow to point to prev (NULL)
           Move all pointers one step right

           NULL ← [10]   [20] → [30] → [40] → NULL
                  prev    tmp    front

  Step 2:  Flip [20]'s arrow to point to prev ([10])

           NULL ← [10] ← [20]   [30] → [40] → NULL
                         prev    tmp    front

  Step 3:  Flip [30]'s arrow

           NULL ← [10] ← [20] ← [30]   [40] → NULL
                                prev    tmp    front

  Step 4:  Flip [40]'s arrow, front becomes NULL → STOP

           NULL ← [10] ← [20] ← [30] ← [40]   NULL
                                        prev    tmp/front

  Return prev → [40] is the new head!
```

### C++ Code

```cpp
Node* reversell(Node* head) {
    if (head == nullptr || head->next == nullptr) return head;  // 0 or 1 node, nothing to reverse

    Node* prev = nullptr;           // starts behind the list
    Node* front = head;             // will move ahead of tmp
    Node* tmp = head;               // current node being processed

    while (front != nullptr) {
        front = front->next;        // save next node before we break the link
        tmp->next = prev;           // FLIP the arrow (point backward)
        prev = tmp;                 // move prev forward
        tmp = front;                // move tmp forward
    }
    return prev;                    // prev is now the new head
}
```

---

## Complete Working Example (All Together)

```cpp
#include <bits/stdc++.h>
using namespace std;

// ─────────────────── Node Definition ───────────────────
struct Node {
    int data;
    Node* next;
    Node(int val) {
        data = val;
        next = nullptr;
    }
};

// ─────────────────── Array to Linked List ──────────────
Node* arytolist(int arr[], int n) {
    if (n == 0) return nullptr;
    Node* head = new Node(arr[0]);        // first element → head
    Node* curr = head;
    for (int i = 1; i < n; i++) {
        curr->next = new Node(arr[i]);    // create & link new node
        curr = curr->next;                // advance curr
    }
    return head;
}

// ─────────────────── Traversal (Print) ─────────────────
void printlist(Node* head) {
    Node* tmp = head;
    while (tmp != nullptr) {
        cout << tmp->data << " ";
        tmp = tmp->next;
    }
    cout << endl;
}

// ─────────────────── Search ───────────────────────────
bool serachinlist(Node* head, int ele) {
    Node* tmp = head;
    while (tmp != nullptr) {
        if (tmp->data == ele) return true;   // found!
        tmp = tmp->next;
    }
    return false;                             // not found
}

// ─────────────────── Deletion ─────────────────────────
Node* deleteele(Node* head, int ele) {
    if (head == nullptr) return nullptr;

    // if head itself is the target
    if (head->data == ele) {
        Node* newhead = head->next;           // next node becomes head
        delete head;                           // free old head
        return newhead;
    }

    // search for element in the rest of the list
    Node* tmp = head;                          // tmp = previous node
    Node* curr = head;                         // curr = node to check
    while (curr != nullptr) {
        if (curr->data == ele) {
            tmp->next = tmp->next->next;       // skip over curr
            delete curr;                        // free deleted node
            break;
        }
        tmp = curr;                             // advance both
        curr = curr->next;
    }
    return head;
}

// ─────────────────── Insertion at Position ─────────────
Node* insertele(Node* head, int pos, int ele) {
    int cnt = 0;
    Node* prev = nullptr;
    Node* curr = head;

    // insert at head (position 1)
    if (pos == 1) {
        Node* newnode = new Node(ele);
        newnode->next = head;                  // new node → old head
        return newnode;                         // new node is new head
    }

    // walk to the target position
    while (curr != nullptr) {
        cnt++;
        if (cnt == pos) {
            Node* newnode = new Node(ele);
            prev->next = newnode;               // prev → new node
            newnode->next = curr;               // new node → curr
        }
        prev = curr;
        curr = curr->next;
    }

    // insert at the very end
    if (cnt + 1 == pos) {
        Node* newnode = new Node(ele);
        prev->next = newnode;
    }
    return head;
}

// ─────────────────── Reverse ──────────────────────────
Node* reversell(Node* head) {
    if (head == nullptr || head->next == nullptr) return head;

    Node* prev = nullptr;                      // will become new head
    Node* front = head;                        // look-ahead pointer
    Node* tmp = head;                          // current node

    while (front != nullptr) {
        front = front->next;                   // save next before breaking link
        tmp->next = prev;                      // flip the arrow backward
        prev = tmp;                            // advance prev
        tmp = front;                           // advance tmp
    }
    return prev;                               // prev = new head
}

// ─────────────────── Main ─────────────────────────────
int main() {
    int arr[] = {10, 20, 30, 40, 50};
    Node* head = arytolist(arr, 5);

    cout << "Original:  ";
    printlist(head);                            // 10 20 30 40 50

    head = deleteele(head, 30);                 // remove 30
    cout << "Delete 30: ";
    printlist(head);                            // 10 20 40 50

    head = insertele(head, 3, 99);              // insert 99 at position 3
    cout << "Insert 99: ";
    printlist(head);                            // 10 20 99 40 50

    head = reversell(head);                     // reverse entire list
    cout << "Reversed:  ";
    printlist(head);                            // 50 40 99 20 10

    cout << "Search 40: " << (serachinlist(head, 40) ? "Found" : "Not Found") << endl;
    cout << "Search 80: " << (serachinlist(head, 80) ? "Found" : "Not Found") << endl;

    return 0;
}
```

**Output:**
```
Original:  10 20 30 40 50
Delete 30: 10 20 40 50
Insert 99: 10 20 99 40 50
Reversed:  50 40 99 20 10
Search 40: Found
Search 80: Not Found
```

---

## Quick Reference — Time Complexity

```
  ┌────────────────────┬───────────────┬───────────────┐
  │  Operation         │  Array        │  Linked List  │
  ├────────────────────┼───────────────┼───────────────┤
  │  Access by index   │  O(1) ✅     │  O(n) ❌     │
  │  Search            │  O(n)         │  O(n)         │
  │  Insert at start   │  O(n) ❌     │  O(1) ✅     │
  │  Insert at end     │  O(1)*        │  O(n)**       │
  │  Insert at middle  │  O(n) ❌     │  O(1)*** ✅  │
  │  Delete at start   │  O(n) ❌     │  O(1) ✅     │
  │  Delete at middle  │  O(n) ❌     │  O(n)         │
  │  Reverse           │  O(n)         │  O(n)         │
  │  Get length        │  O(1) ✅     │  O(n) ❌     │
  └────────────────────┴───────────────┴───────────────┘

  *   if using dynamic array with space left
  **  O(1) if you maintain a tail pointer
  *** O(1) if you already HAVE a pointer to the position
       (finding the position still takes O(n))
```
