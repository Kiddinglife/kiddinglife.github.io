---
layout: post
title:  "SWIG-4: Wrap C++"
date:   2016-03-21 19:10:03
categories: tool
tags:  swig script-language c/c++
---

* content
{:toc}

This chapter describes SWIG's support for wrapping C++. As a prerequisite, 
you should first read the chapter _SWIG-3: Wrap ANSI C Type_ to see how SWIG wraps ANSI C. 
Support for C++ builds upon ANSI C wrapping and that material will be useful 
in understanding this chapter.

## 6.1 Introduction of C++ Wrapping
SWIG only provides support for a subset of C++ features. Fortunately, this is now a rather large subset.
there is no semantically obvious (or automatic ) way to map many of its advanced features into other 
languages. As a simple example:
 - consider the problem of wrapping C++ multiple inheritance to a target language with no such support. 
 - Similarly, the use of overloaded operators and overloaded functions can be problematic when no such 
 capability exists in a target language.

In contrast, C++ has increasingly relied upon generic programming and templates for much of its 
functionality. Although templates are a powerful feature, they are largely orthogonal to the whole notion 
of binary components and libraries. For example, an STL vector does not define any kind of binary object 
for which SWIG can just create a wrapper.

## 6.2 Approach of C++ Wrapping
To wrap C++, SWIG uses a layered approach to code generation. 

_At the lowest level,_ SWIG generates a collection of procedural ANSI-C style wrappers. These wrappers take 
care of basic type conversion, type checking, error handling, and other low-level details of the C++ binding. 

These wrappers are also sufficient to bind C++ into any target language that supports built-in procedures. 
In some sense, you might view this layer of wrapping as providing a C library interface to C++.

_On top of the low-level procedural (flattened) interface,_ SWIG generates _proxy classes_ that provide a natural 
object-oriented (OO) interface to the underlying code. The proxy classes are typically written in the target language 
itself. For instance, in Python, a real Python class is used to provide a wrapper around the underlying C++ object.

SWIG tries to maintain a very strict and clean separation between the implementation of your C++ application 
and the resulting wrapper code.it does not play sneaky tricks with the C++ type system, it doesn't mess with your class 
hierarchies, and it doesn't introduce new semantics. 

__NOTE:__ _Although this approach might not provide the most seamless integration with C++ (not most efficiently), 
it is safe, simple, portable, and debuggable._

Some of this chapter focuses on the low-level procedural interface to C++ that is used as the foundation for 
all language modules. 

__NOTE:__ Keep in mind that the target languages also provide the high-level OO interface via proxy classes. 
More detailed coverage can be found in the documentation for each target language.

## 6.3 Supported C++ features
SWIG currently supports most C++ features including the following:
 - Classes
 - Constructors and destructors
 - Virtual functions
 - Public inheritance (including multiple inheritance)
 - Static functions
 - Function and method overloading
 - Operator overloading for many standard operators
 - References
 - Templates (including specialization and member templates)
 - Pointers to members
 - Namespaces
 - Default parameters
 - Smart pointers
 
The following C++ features are not currently supported:
 - Overloaded versions of certain operators (new, delete, etc.)
 
__NOTE:__ As a rule of thumb, SWIG should not be used on raw C++ source files, _use header files only._

## 6.4 Command line options and compilation
When wrapping C++ code, it is critical that SWIG be called with the `-c++` option. 

When compiling and linking the resulting wrapper file, it is normal to use the `C++` compiler. For example:
```shell
$ swig -c++ -tcl example.i
$ c++ -fPIC -c example_wrap.cxx 
$ c++ example_wrap.o $(OBJS) -o example.so
```
Unfortunately, the process varies slightly on each platform. Make sure you refer to the documentation on each 
target language for further details. 

## 6.5 Proxy classes
if you're building a Python module, each C++ class is wrapped by a Python proxy class.  
Or if you're building a Java module, each C++ class is wrapped by a Java proxy class.

### 6.5.1 Construction of proxy classes
roxy classes are always constructed as an extra layer of wrapping that uses low-level accessor functions. 
To illustrate, suppose you had a C++ class like this:
```shell
class Foo {
  public:
    Foo();
    ~Foo();
    int  bar(int x);
    int  x;
};
```
the real proxy class is written in the target language. For example, in Python, 
the proxy might look roughly like this:
```python
class Foo:
    def __init__(self):
        self.this = new_Foo()
    def __del__(self):
        delete_Foo(self.this)
    def bar(self, x):
        return Foo_bar(self.this, x)
    def __getattr__(self, name):
        if name == 'x':
            return Foo_x_get(self.this)
        # ...
    def __setattr__(self, name, value):
        if name == 'x':
            Foo_x_set(self.this, value)
        # ...
```
__NOTE:__ the low-level accessor functions are always used by the proxy classes. 
Whenever possible, proxies try to take advantage of language features that 
are similar to C++. This might include operator overloading, exception handling, 
and other features.

### 6.5.2 Resource management in proxies
### 6.5.2.1 Issues brought from proxies
A major issue with proxies concerns the memory management of wrapped 
objects. Consider the following C++ code:
```c++
class Foo 
{
    public:
        Foo();
        ~Foo();
        int bar(int x);
        int x;
};
class Spam 
{
    public:
        Foo *value;
        ...
};
```
Consider some script code that uses these classes:
```python
f = Foo()               # Creates a new Foo 1
s = Spam()           # Creates a new Spam 1
s.value = f            # Stores a reference to f inside s 2
g = s.value           # Returns stored reference 3
g = 4                    # Reassign g to some other value 4
del f                     # Destroy f 5
```
1.When objects are created in the script, the objects are wrapped by newly 
created proxy classes. That is, there is both a new proxy class instance and 
a new instance of the underlying C++ class.In this example, both `f` and `s` 
are created in this way. 

2.However, the statement s.value is rather curious. `s.value` is a script 
representations of `Foo*`. when executed, a pointer to f is set to Foo* value inside  
the underlying C++ object of `s` . This means that the scripting proxy class 
AND another C++ class share a reference to the same object. As a result,
now the underlying C++ object of `s` stores a reference to the underlying 
C++ object of `f`.

3.To make matters even more interesting, consider the statement `g = s.value`. 
When executed, this creates a new proxy class g that provides a wrapper 
around the C++ object stored in `s.value`.

In general, there is no way to know where this object came from---it could 
have been created by the script, but it could also have been generated internally. 

In this particular example, the assignment of g results in a second proxy 
class for `Foo`. In other words, a reference to `Foo` is now shared by two proxy 
objects (`f` and `g`) and a C++ object(`Spam`).

4.In the statement, g=4, the variable g is reassigned. In many languages, 
this makes the old value of g available for garbage collection. Therefore, 
this causes one of the proxy classes to be destroyed. 

5.Later on, the statement del f destroys the other proxy class. 

_Of course, there is still a reference to the original object stored inside another C++ 
object. What happens to it? Is the object still valid?_

### 6.5.2.1 Solution is ownership transfer
To answer this problem, we must talk anout proxy classes that provide an API
for controlling ownership. In C++ pseudocode, ownership control might 
look roughly like this:
```c++
class FooProxy {
  public:
    Foo    *self;
    int     thisown;

    FooProxy() {
      self = new_Foo();
      thisown = 1;       // Newly created object
    }
    ~FooProxy() {
      if (thisown) delete_Foo(self);
    }
    ...
    // Ownership control API
    void disown() {
      thisown = 0;
    }
    void acquire() {
      thisown = 1;
    }
};

class FooPtrProxy: public FooProxy {
public:
  FooPtrProxy(Foo *s) {
    self = s;
    thisown = 0;
  }
};

class SpamProxy {
  ...
  FooProxy *value_get() {
    return FooPtrProxy(Spam_value_get(self));
  }
  void value_set(FooProxy *v) {
    Spam_value_set(self, v->self);
    v->disown();
  }
  ...
};
```
Looking at this code, there are a few central features:
 1. Each proxy class keeps an extra flag to indicate ownership. C++ objects 
 are only destroyed if the ownership flag is set.
 2. When new objects are created in the target language, the ownership 
 flag is set.
 3. When a reference to an internal C++ object is returned, it is wrapped
  by a proxy class, but the proxy class does not have ownership.
 4. In certain cases, ownership is adjusted. For instance, when a value is 
 assigned to the member of a class, ownership is lost.
 5. Manual ownership control is provided by special disown() and acquire() 
 methods.

Given the tricky nature of C++ memory management, it is impossible for 
proxy classes to automatically handle every possible memory management 
problem. However, proxies do provide a mechanism for manual control 
that can be used (if necessary) to address some of the more tricky memory 
management problems.

__NOTE:__ Language specific details on proxy classes are contained in the chapters 
describing each target language. This chapter has merely introduced the 
topic in a very general way.

## 6.6 Simple C++ wrapping
The following code shows a SWIG interface file for a simple C++ class:
```c++
%module list
%{
        #include "list.h"
%}

// Very simple C++ example for linked list
class List 
{
    public:
        List();
        ~List();
        int  search(char *value);
        void insert(char *);
        void remove(char *);
        char *get(int n);
        static void print(List *l);
        
        int  length;
};
```
To generate wrappers for this class, SWIG first reduces the class to a 
collection of _low-level C-style accessor functions_ which are then used 
by the proxy classes.

### 6.6.1 C-style Constructors and destructors
C++ constructors and destructors are translated into accessor functions 
such as the following :
```c++
List * new_List(void) {
  return new List;
}
void delete_List(List *l) {
  delete l;
}
```
## 6.6.2 Default ctors, copy ctors and implicit dtors
_Following the C++ rules for implicit constructor and destructors, 
SWIG will automatically assume there is one even when they are not 
explicitly declared in the class interface._

In general then:  
 1. If a C++ class does not declare any explicit constructor, 
SWIG will automatically generate a wrapper for one.
 2. If a C++ class does not declare an explicit copy constructor, 
SWIG will automatically generate a wrapper for one if the `%copyctor` is used.
 3. If a C++ class does not declare an explicit destructor, SWIG will 
automatically generate a wrapper for one.
 
_And as in C++, a few rules that alters the previous behavior:_
 1. A default constructor is not created if a class already defines 
 a constructor with arguments.
 2. Default constructors are not generated for classes with pure virtual 
 methods or for classes that inherit from an abstract class, but don't 
 provide definitions for all of the pure methods.
 3. A default constructor is not created unless all base classes support a 
 default constructor.
 4. Default constructors and implicit destructors are not created if a class 
 defines them in a private or protected section.
 5. Default constructors and implicit destructors are not created if any 
 base class defines a non-public default constructor or destructor.
 
To manually disable these, the `%nodefaultctor` and `%nodefaultdtor` feature
flag directives can be used. 

__NOTE:__  that these directives only affects the Implicit generation, 
and they have no effect if the default/copy constructors or destructor 
are explicitly declared in the class interface.

For example:
```c++
%nodefaultctor Foo;  // Disable the default constructor for class Foo.
class Foo {          
// No default constructor is generated, unless one is declared
...
};
class Bar {         
 // A default constructor is generated, if possible
...
};
```
The directive `%nodefaultctor` can also be applied "globally", as in:
```c++
%nodefaultctor; // Disable creation of default constructors
class Foo {     
// No default constructor is generated, unless one is declared
...
};
class Bar {   
public:
  Bar();        
  // The default constructor is generated, since one is declared
};
%clearnodefaultctor; // Enable the creation of default constructors again
```
The corresponding `%nodefaultdtor` directive can be used to disable the 
generation of the default or implicit destructor:
```c++
%nodefaultdtor Foo;   // Disable the implicit/default destructor for class Foo.
class Foo {           
// No destructor is generated, unless one is declared
...
};
```
__Note:__ The `%nodefault` directive/ `-nodefault` options described above, 
which disable both the default constructor and the implicit destructors, 
could lead to memory leaks, and so it is strongly recommended to not use them.

### 6.6.3 When constructor wrappers aren't created
If a class defines a constructor, SWIG normally tries to generate a wrapper 
for it. However, SWIG will not generate a constructor wrapper if it thinks
that it will result in illegal wrapper code. There are really two cases 
where this might show up.

First, SWIG won't generate wrappers for protected or private constructors. 
For example:
```c++
class Foo {
protected:
  Foo();         // Not wrapped.
public:
    ...
};
```  
Next, SWIG won't generate wrappers for a class if it appears to be 
abstract--that is, it has undefined pure virtual methods.
Here are some examples:
```c++
class Bar {
public:
  Bar();               // Not wrapped.  Bar is abstract.
  virtual void spam(void) = 0;
};

class Grok : public Bar {
public:
      Grok();        // Not wrapped. No implementation of abstract spam().
      // but You can use `%feature("notabstract") Foo;` to force wrapping  `Grok();`
};
``` 

### 6.6.4 Copy constructors
If a class defines more than one constructor, its behavior depends on the 
capabilities of the target language. If overloading is supported, the copy 
constructor is accessible using the normal constructor function. 
For example, if you have this:
```c++
class List {
public:
    List();    
    List(const List &);      // Copy constructor
    ...
};
``` 
then the copy constructor can be used in script as follows:
```python
x = List()               # Create a list
y = List(x)              # Copy list x
``` 
If the target language does not support overloading, then the copy 
constructor is available through a special function like this:
```c++
List *copy_List(List *f) {
    return new List(*f);
}
```
then the copy constructor can be used in script as follows:
```python
x = List()               # Create a list
y = copy_List(x)              # Copy list x
``` 

__Note:__ SWIG does not generate a copy constructor wrapper unless one 
is explicitly declared in the class. This differs from the treatment of default
constructors and destructors. However, copy constructor wrappers can 
be generated if using the `copyctor` feature flag. For example:
```c++
%copyctor List;
class List {
public:
    List();    
};
```
Will generate a copy constructor wrapper for List.

### 6.6.5 Member functions
All member functions are roughly translated into accessor functions like this :
```c++
int List_search(List *obj, char *value) {
  return obj->search(value);
}
```
It should be noted that SWIG does not actually create a C accessor 
function in the code it generates. Instead, member access such as 
`obj->search(value)` is directly inlined into the generated wrapper functions. 

### 6.6.6 Static members
Static member functions are called directly without making any special 
transformations. For example, the static member function print(List *l) 
directly invokes List::print(List *l) in the generated wrapper code.





























