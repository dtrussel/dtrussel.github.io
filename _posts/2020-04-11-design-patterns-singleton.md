---
layout: post
title:  "C++ Design Patterns: Singleton - the Classic"
thumbnail: assets/images/cpp-logo.svg
categories: [cpp, design patterns]
---

The singleton is one of the simplest object-oriented C++ patterns. Probably
due to its simplicity, it is also an often misused one. It is easy to implement your
own, and therefore one might tend to use it a bit too often as a design choice.
When should you use a singleton then? The answer is quite obvious, when you need a
unique global object. However remember, global variables are usually frowned upon
and you shouldn't treat a singleton any other way. Avoid global variables as much
as you can, but in some cases they are valid design decisions.

![SingletonMeme](https://pics.me.me/singleton-global-variable-same-but-different-39479078.png){: .center-image }

Global variables are considered bad practice, because they make it very hard
to reason about code locally. The state of a global object might be changed
anywhere in the program and therefore you usually cannot make any assumption
about its state when looking at it in a local scope. However the singleton
can be used to address two categories of design problems.  
The first one is to represent physical objects or resources that are unique in 
the context of the program. E.g. imagine you are designing software that will
run on a single car. So a `Car` object might be a singleton, since the software
will never have to deal with fleet of cars (of course you could also envision a
software that controls several cars, where you would not make `Car` a singleton).   
The second category is objects that are global by design (without representing
any physical object). E.g. you might decide to implement a resource manager
as a singleton. There might be several instances of the resource itself, but the 
manager, which keeps track of the - limited - number of instances is a unique
global object.
Or as another example, loggers are often implemented as singletons since you want
your whole program to log to the same instance.

So in short. Do not think if you should use a singleton or not, but instead think if
your design should enforce that a certain object is global and unique.

So if after some consideration you still decided to go ahead with a singleton,
let's have a look on how to implement one.

For the sake of simplicity we will consider the associated data of the singleton
to be an `int`.

## Quick and Dirty
In the header:
```cpp
struct Singleton {
  int& get() { return value_; }
private:
  int value_ = 0;
};

extern Singleton global_instance;
```
In the .cpp file:
```cpp
Singleton global_instance;
```
But that really is just a global object. Nothing prevents the user from creating
another `Singleton` instance.
So let us change this slightly into the static singleton.

## The Static Singleton
```cpp
struct Singleton {
  int& get() { return value_; }
private:
  inline static int value_ = 0;
};
```
(Note that we used the nice C++17 feature of initializing an [inline static](https://en.cppreference.com/w/cpp/language/static) data member.)
Now the user might still create several instances of the singleton, but they are
really nothing else than handles to the same static data member.

E.g. user code
```cpp
Singleton one;
++one.get();
...
Singleton another;
int value = another.get();
```
And `one` and `another` will return references to the same data instance.

Of course you can take that a step further and make the `get` function also static
to make the singleton nature of your class more clear (since not all your classes
which are singletons will have Singleton in the name).

```cpp
struct Singleton {
  static int& get() { return value_; }
private:
  Singleton() = delete;
  inline static int value_ = 0;
};
```
Which will make the usage look like this:
```cpp
++Singleton::get();
...
int value = Singleton::get();
```
And thus hopefully a bit more readable and clear.

Now all is fine and well. Everything is static and the singleton is initialized
before the program i.e. `main()` starts executing and is destroyed after it ends. But
what if code in another static object uses our singleton? The order of 
initialization of static objects is generally undefined (implementation dependent).
The standard only guarantees that all static objects defined in the same file
will be initalized in order of their declaration. But as soon as they are spread
out over several files, we do have a problem. E.g. imagine a singleton logger
that makes use of singleton memory manager.
How can we make sure the memory manager is initalized when we use it in the logger
in the static part of our code?

## The Meyers` Singleton
Named after [Scott Meyers](https://en.wikipedia.org/wiki/Scott_Meyers), this 
implementation of a singleton defers its initialization to its first use. Thus
solving the above mentioned problem of undefined static object initialization
order.

```cpp
struct Singleton {
  static Singleton& instance() {
    static Singleton inst;
    return inst;
  }
  int& get() { return value_; }
private:
  Singleton() = default;
  ~Singleton() = default;
  Singleton(const Singleton&) = delete;
  Singleton& operator=(const Singleton&) = delete;
  int value_ = 0;
};
```
The private constructor prevents the program from directly constructing the object.
Instead the initialization will take place only on the first call of the static
`instance()` member function. Since this is the only way to access the singleton,
it is guaranteed to be initialzed on the first usage.

The usage would look like this:
```cpp
++Singleton::instance().get();
...
int value = Singleton::instance().get();
```

Looking at the `instance()` function you might have realized that we return a 
reference of a local variable (which is usually a really bad idea). However it
is a reference to a [static local variable](https://en.cppreference.com/w/cpp/language/storage_duration#Static_local_variables), hence only one instance of this local
variable exists in the entire program and therefore returning its reference is
perfectly fine. And unlike a file static object, static local variables
are initialized the first time they are used.

However there is one downside. There is a performance overhead. Every time the
`instance()` function is called, there is an implicit check to see if the static
 variable is already initialized. So if you repeatedly need to access the singleton,
it is best to store a reference to the returned instance. E.g.:
```cpp
Singleton& inst = Singleton::instance();
for (auto i = 0; i < N; ++i) {
  do_something(inst.get());
}
```

## The Pimpl Singleton
Sometimes it is desired to have a clear seperation between interface and
implementation. The pointer to implementation (pimpl) idiom exposes only the
interface in the header file, and the implementation is hidden inside a class
inside the .cpp file. The actual singleton class then only holds a pointer to 
that implementation class (we will actually store a reference instead of pointer,
but let's call it pimpl anyway).

So a Pimpl Singleton looks like this.
In the header:
```cpp
struct SingletonImpl; // Forward declare
struct Singleton {
  Singleton();
  int& get();
private:
  static SingletonImpl& impl();
  SingletonImpl& impl_;
};
```

In the .cpp file:
```cpp
// User code will not mind changes here as long as the interface stays the same
struct SingletonImpl {
  int value_ = 0;
};

Singleton::Singleton() : impl_(impl()) {}

int& Singleton::get() { return impl().value_; }

SingletonImpl& Singleton::impl(){
 static SingletonImpl instance;
 return instance;
}
```
Each singleton instance also saves a reference to the implementation instance to
avoid the performance overhead mentioned in the previous section.
(There is also a small overhead of an indirection).

## How about thread safety?
Until now we ignored the thread safety of the singletons. But while the here presented
singleton instances are thread safe, because they are static, the associated data
members are no different than any other shared variable. When used in a multi-threaded
context the programmer is responsible to use it in a thread safe manner, e.g.
protecting them by a mutex.

For a more in-depth view into the subject I highly recommend the book referenced
below. I hope now you know how to implement your singleton, but remember to think
twice before deciding to chose a global variable in disguise.

## References 
* Hands-On Design Patterns with C++ by Fedor G. Pikus (ISBN 978-1-78883-256-4)

