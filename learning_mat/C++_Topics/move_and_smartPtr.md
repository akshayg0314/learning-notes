# Move Semantics & Smart Pointers — Explained Simply

---

## Part 1: Move Semantics

### The Problem Move Semantics Solves

Imagine you have a big box of books (a `std::vector` with 1 million elements).
Without move semantics, every time you pass this box around, C++ makes a
**photocopy of every single book**. That's extremely slow and wasteful.

Move semantics says: *"Don't copy the books. Just hand over the box itself."*

---

### 1.1 Rvalue References (`&&`) — Identifying Temporary Objects

In C++, every expression is either an **lvalue** or an **rvalue**:

- **lvalue** = something with a **name** and a **lasting address** (you can point to it).
- **rvalue** = something **temporary** that is about to die (no lasting name).

```cpp
int a = 10;       // 'a' is an lvalue  (has a name, lives on)
                  // '10' is an rvalue  (temporary, no name)

int b = a + 5;    // 'a + 5' is an rvalue (temporary result, about to disappear)
```

**`&&` is the rvalue reference.** It's a way to "catch" a temporary before it dies:

```cpp
int&& ref = 42;   // ref binds to the temporary '42'
                   // Now we can use 'ref' to access it
```

**Real-world analogy:**
Think of an rvalue like a delivery package sitting at your door.
Nobody owns it yet — it's temporary. An `&&` reference lets you grab it
before it gets thrown away.

---

### 1.2 `std::move` — Casting to Allow Resource "Stealing"

`std::move` does NOT actually move anything. It simply says:

> *"I don't need this object anymore. Treat it as a temporary so someone can steal its guts."*

It **casts an lvalue into an rvalue reference**.

```cpp
#include <iostream>
#include <string>
#include <utility>  // for std::move

int main() {
    std::string original = "Hello, World!";

    // WITHOUT move: copies the string (slow for big strings)
    std::string copy = original;
    std::cout << "original: " << original << "\n";  // "Hello, World!" (still there)
    std::cout << "copy:     " << copy << "\n";       // "Hello, World!"

    // WITH move: steals the string's internal data
    std::string stolen = std::move(original);
    std::cout << "original: " << original << "\n";   // "" (empty! resources were stolen)
    std::cout << "stolen:   " << stolen << "\n";     // "Hello, World!"
}
```

**Output:**
```
original: Hello, World!
copy:     Hello, World!
original:
stolen:   Hello, World!
```

**Analogy:**
- Copy = Photocopy a document (original stays intact).
- Move = Hand the document to someone else (your hands are now empty).

---

### 1.3 Move Constructor — Transferring Ownership

A **move constructor** takes ownership of another object's resources
instead of copying them. The key steps are:

1. **Steal** the other object's pointers/data.
2. **Set the old object to a safe empty state** (usually `nullptr`).

```cpp
#include <iostream>

class MyBuffer {
    int* data;
    int size;

public:
    // Normal Constructor — allocates memory
    MyBuffer(int sz) {
        size = sz;
        data = new int[sz];
        std::cout << "Constructed: allocated " << sz << " ints\n";
    }

    // Copy Constructor — EXPENSIVE (copies all data)
    MyBuffer(const MyBuffer& other) {
        size = other.size;
        data = new int[other.size];
        for (int i = 0; i < size; i++)
            data[i] = other.data[i];
        std::cout << "Copied: duplicated " << size << " ints\n";
    }

    // Move Constructor — CHEAP (just swaps pointers)
    MyBuffer(MyBuffer&& other) noexcept {
        data = other.data;
        size = other.size;
        other.data = nullptr;   // <-- Step 2: old object is now empty
        other.size = 0;
        std::cout << "Moved: stole pointer, zero copies!\n";
    }

    // Destructor
    ~MyBuffer() {
        delete[] data;
        std::cout << "Destroyed\n";
    }
};

int main() {
    MyBuffer buf1(1000000);               // Allocates 1 million ints

    MyBuffer buf2 = buf1;                 // COPY — slow, duplicates 1M ints
    MyBuffer buf3 = std::move(buf1);      // MOVE — fast, just swaps a pointer
}
```

**Output:**
```
Constructed: allocated 1000000 ints
Copied: duplicated 1000000 ints
Moved: stole pointer, zero copies!
Destroyed
Destroyed
Destroyed
```

**What happened inside the move constructor:**

```
BEFORE std::move(buf1):
    buf1.data ──────► [ 1, 2, 3, ... 1000000 ]    (heap memory)

AFTER std::move(buf1):
    buf1.data ──────► nullptr                       (empty, safe to destroy)
    buf3.data ──────► [ 1, 2, 3, ... 1000000 ]    (same heap memory, no copy!)
```

---
---

## Part 2: Smart Pointers (Memory Management)

### The Problem Smart Pointers Solve

With raw pointers (`new`/`delete`), you can easily:
- Forget to `delete` → **Memory leak**
- `delete` twice → **Crash**
- Use after `delete` → **Undefined behavior**

Smart pointers **automatically manage memory** using RAII
(Resource Acquisition Is Initialization) — when the smart pointer dies,
it cleans up the memory for you.

---

### 2.1 `std::unique_ptr` — Exclusive Ownership

`unique_ptr` is the **most common** smart pointer. It ensures only **one** pointer owns the object.
- **Fast:** Zero overhead (same size as a raw pointer).
- **Safe:** Auto-deletes when it goes out of scope.
- **Movable:** You can transfer ownership, but you cannot copy it.

```cpp
#include <iostream>
#include <memory>
#include <utility> // for std::move

class Dog {
public:
    std::string name;
    Dog(std::string n) {
        name = n;
        std::cout << name << " born!\n";
    }
    ~Dog() { std::cout << name << " destroyed!\n"; }
    void bark() { std::cout << name << " says Woof!\n"; }
};

int main() {
    // 1. Create a unique_ptr
    std::unique_ptr<Dog> dog1 = std::make_unique<Dog>("Buddy");
    dog1->bark();

    // 2. Transer ownership to dog2
    // std::unique_ptr<Dog> dog2 = dog1;     // ERROR! Cannot copy unique_ptr
    std::unique_ptr<Dog> dog2 = std::move(dog1); // OK! Ownership moved

    if (!dog1) std::cout << "dog1 is empty now.\n";
    dog2->bark();

} // main ends -> dog2 goes out of scope -> Buddy destroyed!
```

**Output:**
```
Buddy born!
Buddy says Woof!
dog1 is empty now.
Buddy says Woof!
Buddy destroyed!
```
---

### 2.2 `std::shared_ptr` — Shared Ownership with Reference Counting

Multiple `shared_ptr` instances can **share** the same object.
An internal **reference counter** tracks how many `shared_ptr`s point to it.
When the counter reaches **zero**, the object is automatically deleted.

```cpp
#include <iostream>
#include <memory>

class Dog {
public:
    std::string name;
    Dog(std::string n) {
        name = n;
        std::cout << name << " born!\n";
    }
    ~Dog(){ std::cout << name << " destroyed!\n"; }
};

int main() {
    std::shared_ptr<Dog> ptr1 = std::make_shared<Dog>("Buddy");
    std::cout << "ref count: " << ptr1.use_count() << "\n";  // 1

    {
        std::shared_ptr<Dog> ptr2 = ptr1;   // share ownership
        std::cout << "ref count: " << ptr1.use_count() << "\n";  // 2

        std::shared_ptr<Dog> ptr3 = ptr1;   // share again
        std::cout << "ref count: " << ptr1.use_count() << "\n";  // 3

    }   // ptr2 and ptr3 go out of scope → ref count drops to 1

    std::cout << "ref count: " << ptr1.use_count() << "\n";  // 1

}   // ptr1 goes out of scope → ref count drops to 0 → Dog is destroyed!
```

**Output:**
```
Buddy born!
ref count: 1
ref count: 2
ref count: 3
ref count: 1
Buddy destroyed!
```

**How it looks in memory:**

```
  ptr1 ─────┐
             ▼
  ptr2 ────► [Control Block]  ───► [Dog: "Buddy"]
             │ ref_count = 3 │
  ptr3 ─────┘
```

---

### 2.3 Reference Counting — The Control Block

Every `shared_ptr` points to a **Control Block** which stores:

```
┌─────────────────────────────┐
│        CONTROL BLOCK        │
├─────────────────────────────┤
│  strong_count (shared_ptr)  │ ← how many shared_ptrs exist
│  weak_count   (weak_ptr)    │ ← how many weak_ptrs exist
│  pointer to actual object   │ ← the real data
│  deleter (optional)         │ ← custom cleanup function
└─────────────────────────────┘
```

- When `strong_count` reaches 0 → the **object** is deleted.
- When `weak_count` also reaches 0 → the **control block** itself is deleted.

---

### 2.4 The Circular Reference Problem — The "Forever Trap"

This is the dangerous situation where two objects hold `shared_ptr`
to each other. Their reference counts **never reach zero**, so they
**never get deleted** → **Memory Leak!**

```cpp
#include <iostream>
#include <memory>

class Dog {
public:
    std::string name;
    std::shared_ptr<Dog> bestFriend;   // <── PROBLEM: shared_ptr to another Dog

    Dog(std::string n) {
        name = n;
        std::cout << name << " born!\n";
    }
    ~Dog() { std::cout << name << " destroyed!\n"; }
};

int main() {
    auto dog1 = std::make_shared<Dog>("Buddy");
    auto dog2 = std::make_shared<Dog>("Rex");

    // They become best friends (point to each other)
    // 1. Buddy points to Rex -> Rex ref_count = 2 (1 from main, 1 from Buddy)
    dog1->bestFriend = dog2; 
    
    // 2. Rex points to Buddy -> Buddy ref_count = 2 (1 from main, 1 from Rex)
    dog2->bestFriend = dog1; 

    // main() ends → dog1 and dog2 go out of scope:
    // - Buddy's ref_count drops: 2 → 1  (Rex still holds a shared_ptr to Buddy)
    // - Rex's ref_count drops:   2 → 1  (Buddy still holds a shared_ptr to Rex)
    //
    // NEITHER reaches 0 → NEITHER gets destroyed → MEMORY LEAK!
}
```

**Output (notice: NO "destroyed" messages!):**
```
Buddy born!
Rex born!
```

### 2.5 `std::weak_ptr` — The Observer That Breaks Cycles

`weak_ptr` is the solution. It can **observe** a `shared_ptr` object
**without increasing the reference count**.

Think of it like:
- `shared_ptr` = "I **own** this" (counts as an owner)
- `weak_ptr` = "I **know about** this, but I don't own it" (doesn't count)

**Fix the circular reference:**

```cpp
#include <iostream>
#include <memory>

class Dog {
public:
    std::string name;
    std::weak_ptr<Dog> bestFriend;   // <── FIX: weak_ptr instead of shared_ptr

    Dog(std::string n) { 
        name = n;
        std::cout << name << " born!\n"; 
    }
    ~Dog() { std::cout << name << " destroyed!\n"; }
};

int main() {
    auto dog1 = std::make_shared<Dog>("Buddy");
    auto dog2 = std::make_shared<Dog>("Rex");

    dog1->bestFriend = dog2;     // weak_ptr does NOT increase Rex's ref count!
    dog2->bestFriend = dog1;     // weak_ptr does NOT increase Buddy's ref count!

    // To USE a weak_ptr, you must "lock" it first (converts to shared_ptr temporarily)
    if (auto friendPtr = dog1->bestFriend.lock()) {
        std::cout << dog1->name << "'s best friend is " << friendPtr->name << "\n";
    } else {
        std::cout << "Friend no longer exists!\n";
    }

}   // main exits:
    // dog1 ref_count: 1 → 0 → DESTROYED! (Buddy dies)
    // dog2 ref_count: 1 → 0 → DESTROYED! (Rex dies)
```

**Output (notice: both ARE destroyed now!):**
```
Buddy born!
Rex born!
Buddy's best friend is Rex
Buddy destroyed!
Rex destroyed!
```
---

## Quick Reference Summary

```
+-------------------+---------------------------------------+
| Concept           | What It Does                          |
+-------------------+---------------------------------------+
| Rvalue (&&)       | Catches temporary objects             |
|                   | before they disappear                 |
+-------------------+---------------------------------------+
| std::move         | Casts lvalue to rvalue so its         |
|                   | resources can be stolen               |
+-------------------+---------------------------------------+
| Move Constructor  | Steals another object's pointers      |
|                   | and sets the old one to nullptr       |
+-------------------+---------------------------------------+
| shared_ptr        | Multiple owners, ref-counted,         |
|                   | auto-deletes when count = 0           |
+-------------------+---------------------------------------+
| weak_ptr          | Observes without owning,              |
|                   | does not increase ref count           |
+-------------------+---------------------------------------+
| Circular Ref      | Two shared_ptrs pointing at each      |
|                   | other — neither can reach 0           |
+-------------------+---------------------------------------+
| Control Block     | Internal struct tracking              |
|                   | strong_count and weak_count           |
+-------------------+---------------------------------------+
```

