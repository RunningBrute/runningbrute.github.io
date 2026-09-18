---
title: C vs C++ - Objects
date: 2026-09-11
categories: [c, cpp]
tags: [c, cpp, oop]
---

Recently I have noticed more and more interest in using C++ in the embedded world and, as a consequence, moving away from C to some extent.

Because of that, more and more Embedded C developers are facing the choice of moving to modern C++. One of the first challenges during such a transition is understanding the differences between writing code that performs the same tasks in these two languages.

This is where the idea for this post came from. I wanted to create two example programs that do the same thing, but are written in an idiomatic way for C and C++. This allows us to see the differences and some possible solutions to common problems.

As a starting point, I will use well-known C constructs to explain what a class/object actually is, what operations we can perform on it and how this compares to common patterns used in C++.

## Creating a new class

C does not provide object-oriented programming mechanisms like C++. This does not mean, however, that we cannot write object-oriented code in C.

As a starting point, I would therefore like to use the so-called object pattern in C to simulate a class from C++.

```c
typedef struct House
{
} House;
```

The code above describes more or less the same thing as an empty class in C++.

```cpp
class House
{
};
```

In both cases we have a new type representing our `House` object.

## State

Now let's add a field containing information about the area of our house.

```c
typedef struct House
{
    size_t area;
} House;
```

In C++, classes make their members private by default, so if we want to access the field from outside the object, we have to put it in the `public` section.

```cpp
class House
{
public:
    std::size_t area;
};
```

At this point both implementations expose the state of our object directly.

## Encapsulation

But what if we don't want anyone to have direct access to our state?

In C++ we simply make the field private.

```cpp
class House
{
private:
    std::size_t area;
};
```

The equivalent approach in C is to only provide a declaration of the type in the `.h` file and define the actual structure in the `.c` file.

```c
// House.h

typedef struct House House;
```

```c
// House.c

struct House
{
    size_t area;
};
```

Now the implementation of `House` is hidden from the user of the API.

This is usually referred to as an **opaque type**. When we use it through a pointer, we can also talk about an opaque pointer.

The important difference is that C does not provide constructors or destructors for us. We have to design an API that manages the lifetime of our object manually.

## Constructor and destructor

C++ allows us to define constructors and destructors. They help us manage the lifetime of an object.

A constructor can take parameters that are used to initialize our object. A destructor is called when the object's lifetime ends and can be used to release resources owned by the object, such as dynamically allocated memory.

```cpp
class House
{
public:
    House(const std::size_t area) : area(area) {}
    ~House() {}

private:
    std::size_t area;
};
```

In C, the equivalent of a constructor can be a function that creates an instance of our structure using dynamic memory allocation and returns a pointer to the newly created structure.

We have to remember that in C we are responsible for releasing the memory ourselves. We therefore have to explicitly call the `destroyHouse()` function.

```c
// House.h

typedef struct House House;

House* createHouse(size_t area);
void destroyHouse(House* house);
```

```c
// House.c

struct House
{
    size_t area;
};

House* createHouse(size_t area)
{
    House* newHouse = malloc(sizeof(*newHouse));

    if (!newHouse)
        return NULL;

    newHouse->area = area;

    return newHouse;
}

void destroyHouse(House* house)
{
    free(house);
}
```

The important thing here is that the lifetime of the object is now controlled by our API.

```c
House* house = createHouse(120);

/* use house */

destroyHouse(house);
```

In C++ the same thing can be expressed much more directly:

```cpp
House house(120);

/* use house */
```

When `house` goes out of scope, its destructor is called automatically.

This is one of the first places where the difference between C and C++ becomes quite visible.

## Methods

A class becomes even more interesting when we start adding methods.

```cpp
class House
{
public:
    House(const std::size_t area) : area(area) {}

    std::size_t getArea() const
    {
        return area;
    }

    void setCount(std::size_t count)
    {
        this->count = count;
    }

private:
    std::size_t area;
    std::size_t count;
};
```

Each method in C++ allows us to perform some kind of operation on the object's state.

There is no magic involved here, though. We can implement a very similar approach in C using functions that receive a pointer to our structure.

```c
// House.h

typedef struct House House;

House* createHouse(size_t area);
void destroyHouse(House* house);

size_t getArea(const House* house);
void setCount(House* house, size_t count);
```

```c
// House.c

struct House
{
    size_t area;
    size_t count;
};

size_t getArea(const House* house)
{
    if (!house)
        return 0;

    return house->area;
}

void setCount(House* house, size_t count)
{
    if (!house)
        return;

    house->count = count;
}
```

As we can see in the example above, we have to explicitly pass a pointer to our `House` object to every function that operates on it.

This is somewhat similar to `self` in Python.

There is also no automatic null checking in C, so our API has to define what should happen when a `NULL` pointer is passed.

## Ownership

So far our object does not own any interesting resources.

Let's change that by adding dynamically allocated memory.

```cpp
class House
{
public:
    House(const std::size_t area) : area(area), data(new char[1024]) {}
    
    ~House()
    {
        delete[] data;
    }

private:
    std::size_t area;
    char* data;
};
```

Now things become much more interesting.

Our `House` object owns a dynamically allocated memory region. This means that copying the object requires some thought.

A simple copy of the pointer would create two objects pointing to the same memory.

```text
House 1 ──────┐
              ├────> [ data ]
House 2 ──────┘
```

Both objects would then believe that they own the same resource. When both destructors try to release it, we would end up with a double delete.

This is where copy and move operations become important.

## Copy and move in C++

C++ provides special member functions that allow us to define what happens when an object is copied or moved.

For our `House` class we can explicitly implement both operations.

```cpp
#include <algorithm>
#include <cstddef>

class House
{
public:
    House(std::size_t area) : area(area), data(new char[1024]){}

    ~House()
    {
        delete[] data;
    }

    House(const House& other) : area(other.area), data(new char[1024])
    {
        std::copy(other.data, other.data + 1024, data);
    }

    House(House&& other) noexcept : area(other.area), data(other.data)
    {
        other.data = nullptr;
    }

    House& operator=(const House& other)
    {
        if (this == &other)
            return *this;

        char* newData = new char[1024];

        std::copy(other.data, other.data + 1024, newData);

        delete[] data;

        area = other.area;
        data = newData;

        return *this;
    }

    House& operator=(House&& other) noexcept
    {
        if (this == &other)
            return *this;

        delete[] data;

        area = other.area;
        data = other.data;

        other.data = nullptr;

        return *this;
    }

private:
    std::size_t area;
    char* data;
};
```

The copy constructor creates a completely new memory region and copies the data into it.

```cpp
House house1(120);
House house2 = house1;
```

The result is:

```text
house1 ──────> [ data A ]

house2 ──────> [ data B ]
```

Both objects own their own memory.

Move works differently.

```cpp
House house1(120);
House house2 = std::move(house1);
```

Instead of copying the data, we transfer the pointer.

```text
before:

house1 ──────> [ data ]


after:

house1 ──────> nullptr
house2 ──────> [ data ]
```

The original object still exists after the move. It is simply left in a valid but moved-from state.

This is the important difference between copying and moving.

## Copy and move in C

C does not have copy constructors, move constructors or move semantics as language features.

If we want similar behavior, we have to design it ourselves as part of our API.

For example:

```c
// House.h

typedef struct House House;

House* createHouse(size_t area);
void destroyHouse(House* house);

House* copyHouse(const House* source);
House* moveHouse(House* source);
```

Our structure can look like this:

```c
// House.c

struct House
{
    size_t area;
    size_t count;

    char* data;
    size_t dataSize;
};
```

The copy operation needs to allocate a new resource and copy the data.

```c
House* copyHouse(const House* source)
{
    if (!source)
        return NULL;

    House* copy = malloc(sizeof(*copy));

    if (!copy)
        return NULL;

    copy->area = source->area;
    copy->count = source->count;
    copy->dataSize = source->dataSize;

    copy->data = malloc(copy->dataSize);

    if (!copy->data)
    {
        free(copy);
        return NULL;
    }

    memcpy(copy->data, source->data, copy->dataSize);

    return copy;
}
```

Now both objects own their own memory:

```text
source ──────> [ data A ]

copy   ──────> [ data B ]
```

For moving, we can design our API to explicitly transfer ownership of the resource.

```c
House* moveHouse(House* source)
{
    if (!source)
        return NULL;

    House* destination = malloc(sizeof(*destination));

    if (!destination)
        return NULL;

    *destination = *source;

    source->data = NULL;
    source->dataSize = 0;

    free(source);

    return destination;
}
```

This transfers the ownership of `data` to the new object.

```text
before:

source ──────> [ data ]


after:

destination ──────> [ data ]
```

The important thing to remember is that this is **not C++ move semantics**. C does not have such a language feature.

We have simply designed an API that provides similar ownership-transfer semantics.

There is also an important difference in the lifetime of the source object.

In C++:

```cpp
House house2 = std::move(house1);
```

`house1` still exists after the move, but is in undefined state.

In our C implementation:

```c
House* house2 = moveHouse(house1);
```

`house1` is destroyed as part of the operation.

This is a design decision of our API, not something provided by the C language.

## Creating objects and managing their lifetime

There is one more difference that is worth looking at. We can compare not only how the object itself is implemented, but also how its lifetime is managed.

In C++ we can create an object directly on the stack:

```cpp
int main()
{
    House house(120);

    // use house

    return 0;
}
```

There is no need to explicitly destroy the object.

When `main()` reaches the end of the scope, the destructor of `house` is called automatically.

This is one of the core ideas behind RAII in C++ - the lifetime of a resource is tied to the lifetime of an object.

We can also return an object from a function:

```cpp
House createHouse()
{
    return House(120);
}

int main()
{
    House house = createHouse();

    return 0;
}
```

Even though `House` is created inside `createHouse()`, we don't have to manually manage its lifetime.

The object can be constructed directly in its final destination thanks to copy elision.

The C version looks quite different.

Because our `House` object is an opaque type, we usually work with it through a pointer:

```c
House* createHouse(size_t area)
{
    House* house = malloc(sizeof(*house));

    if (!house)
        return NULL;

    house->area = area;

    return house;
}
```

The caller is now responsible for destroying the object:

```c
int main(void)
{
    House* house = createHouse(120);

    if (!house)
        return 1;

    /* use house */

    destroyHouse(house);

    return 0;
}
```

The difference becomes quite obvious:

```text
C++:
{
    House house(120);

    // use house

} // destructor is called automatically


C:
{
    House* house = createHouse(120);

    // use house

    destroyHouse(house);
}
```

In C++ the scope determines when the object is destroyed.

In C we have to explicitly define when the object should be destroyed by calling the appropriate function.

This is also why ownership becomes so important in C.

When we pass a `House*` around, we need to know who is responsible for eventually calling `destroyHouse()`.

For example:

```c
void useHouse(House* house)
{
    /* use house */
}
```

Does `useHouse()` own the object?

Should it destroy it?

Or does the caller still own it?

The C language does not answer these questions for us. They are part of the API contract.

In C++ we can express much more of this ownership directly through the type system.

For example, instead of using a raw pointer:

```cpp
House* createHouse();
```

we could return an owning smart pointer:

```cpp
std::unique_ptr<House> createHouse()
{
    return std::make_unique<House>(120);
}
```

Now the ownership is explicit.

```cpp
auto house = createHouse();
```

When `house` goes out of scope, the `House` object is destroyed automatically.

This is a much bigger topic on its own, but it shows where the difference between C and modern C++ starts to become really important.