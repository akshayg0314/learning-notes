# Virtual Destructors (and why "Virtual Constructors" don't exist)

## Part 1: The Virtual Destructor Problem

**Rule:** if you have even *one* `virtual` function in a class, you **MUST** have a `virtual` destructor.

### The Problem: Deleting a Child through a Parent Pointer

Imagine you have a `Dog` (Child) which is a type of `Animal` (Parent).
If you point to a `Dog` using an `Animal*` pointer and then `delete` it, only the **Animal** part gets destroyed. The **Dog** part (and its toys) leaks!

### Example WITHOUT Virtual Destructor (The Bug)

```cpp
#include <iostream>

class Animal {
public:
    Animal() { std::cout << "Animal created\n"; }
    
    // Any virtual function usually implies we will use polymorphism
    virtual void makeSound() { std::cout << "Generic sound\n"; }

    // PROBLEM: Non-virtual destructor!
    ~Animal() { std::cout << "Animal destroyed\n"; } 
};

class Dog : public Animal {
    int* toys;
public:
    Dog() { 
        toys = new int[100]; // Dog allocates memory
        std::cout << "Dog allocated toys\n"; 
    }
    ~Dog() { 
        delete[] toys;       // Dog cleans up
        std::cout << "Dog destroyed & toys deleted\n"; 
    }
};

int main() {
    // Upcasting: Storing a Dog in an Animal pointer
    Animal* myPet = new Dog(); 
    
    // DANGER: We delete using the ANIMAL pointer
    delete myPet; 
}
```

**Output (The Bug):**
```
Animal created
Dog allocated toys
Animal destroyed    <-- ERROR! Dog destructor NEVER called! Toys leaked!
```

---

## Part 2: The Solution — Virtual Destructor

By adding `virtual` to the parent's destructor, you tell C++:
> *"When you delete this, check if it's actually a Dog (or Cat) first, and call THAT destructor."*

### Example WITH Virtual Destructor (The Fix)

```cpp
class Animal {
public:
    Animal() { std::cout << "Animal created\n"; }
    
    // FIX: Virtual destructor
    virtual ~Animal() { std::cout << "Animal destroyed\n"; } 
};

// ... Dog class stays the same ...

int main() {
    Animal* myPet = new Dog();
    delete myPet; // Safe now!
}
```

**Output (Correct):**
```
Animal created
Dog allocated toys
Dog destroyed & toys deleted   <-- Success! Child destructor called first
Animal destroyed               <-- Then Parent destructor
```

---

## Part 3: Why No "Virtual Constructor"?

You **cannot** have a `virtual` constructor in C++.

**Why?**
To call a virtual function, you need an object with a **vtable** (the hidden lookup table for virtual functions).
But the constructor's job IS to build the object and the vtable!
You can't use the vtable before it exists. It's a "chicken and egg" problem.

### The Workaround: "Virtual Constructor Idiom" (Clone)

If you need to create a copy of an object but you only have a pointer to its Parent (e.g., `Animal*`), you use a virtual `clone()` function.

```cpp
class Animal {
public:
    virtual Animal* clone() const = 0; // Pure virtual
    virtual ~Animal() {}
};

class Dog : public Animal {
public:
    // "Virtual Constructor" equivalent
    Dog* clone() const override { return new Dog(*this); }
};

int main() {
    Animal* original = new Dog();
    
    // I don't know it's a Dog, but I can copy it!
    Animal* copy = original->clone(); 
}
```

---

## Summary

| Concept | Rule | Why? |
|-------------------------|-------------------------------|-------------------------------------------------------------------------|
| **Virtual Destructor**  | **REQUIRED** for base classes | Ensures Child destructors run when deleting via Parent* (avoids leaks). |
| **Virtual Constructor** | **IMPOSSIBLE**                | Object doesn't exist yet to do virtual lookup.                          |
| **Virtual Copy**        | Use `clone()`                 | Common pattern to duplicate an object when you only know its base type. |
