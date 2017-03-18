---
layout: post
title:  "CCIA-3-shared-data-1"
date:   2017-03-12 10:02:02
categories: read
tags:  thread c++
---

* content
{:toc}

## 3.1 Problems with sharing data between threads
If all shared data is read-only, there’s no problem. However, if data is shared between threads, and **one or more threads** start modifying the data, there’s a lot of potential for trouble.

One concept that’s widely used to help programmers reason about their code is that of invariants—statements that are always true about a particular data structure, These invariants are often broken during an update, especially if the data structure is of any complexity or the update requires modification of more than one value.

Consider a doubly linked list, the steps in deleting an entry from such a list are shown below:
 1. Identify the node to delete (N). a
 2. Update the link from the node prior to N to point to the node after N. b
 3. Update the link from the node after N to point to the node prior to N. c
 4. Delete node N. d

if one thread is reading the doubly linked list while another is removing a node, it’s quite possible for the reading thread to see the list with a node only partially removed (because only one of the links has been changed, as in step b of figure 3.1), so the **invariant is broken**. 

The consequences of this broken invariant can vary; if the other thread is just reading the list items from left to right in the diagram, it will skip the node being. On the other hand, if the second thread is trying to delete the rightmost node in the diagram, it might end up permanently corrupting the data structure and
eventually crashing the program. this is an example of one of the most common causes of bugs in concurrent code: **a race condition**.

### 3.1.1 Race Condition
In concurrency, a race condition is anything where the outcome depends on the **relative ordering** of execution of operations on two or more threads; the threads race to perform their **respective operations**.

the term race condition is usually used to mean a **problematic race condition**; **benign race conditions** aren’t so interesting and aren’t a cause of bugs. The C++ Standard also defines the term data race to mean the specific type of race condition that arises because of concurrent modification to a single object (see section 5.1.2 for details); data races cause the dreaded undefined behavior and it is problemic race condition.  

Take steps in deleting an entry from such a list above as example. Assume thread a operation is to visit each element and thread b operation is to delete element. There are 3 outcomes with diferent relative-ordering executions:
 1.  benign race condition outcome: thread a reads deleting element -> thread b deletes left link step b-> thread b deletes right link step c 
 2.  problemic race condition outcome: thread b deletes left link step b -> thread a reads deleting element -> thread b deletes right link step c 
 3.  benign race condition outcome: thread a reads deleting element -> thread b deletes right link step c ->  thread a reads deleting element  

### 3.1.2 Problematic race conditions
Problematic race conditions typically occur where completing an operation requires modification of two or more distinct pieces of data, such as the two link pointers in the above example. 

what happens is that the operation must access two separate pieces of data, these must be modified in separate instructions, and another thread could potentially access the data structure when only one of them has been completed.

Because race conditions are generally timing sensitive, they can often disappear entirely when
the application is run under the debugger, because the debugger affects the timing
of the program, even if only slightly.

### 3.1.3 Avoid Problematic race conditions
 -  The simplest option is to wrap your data structure with a protection mechanism, to ensure that only the thread actually performing a modification can see the intermediate states where the invariants are broken. From the point of view of other threads accessing that data structure,such modifications either haven’t started or have completed.
 - Another option is to modify the design of your data structure and its invariants so
that modifications are done as a series of indivisible changes, each of which preserves
the invariants. This is generally referred to as **lock-free** programming and is difficult to get
right.
 - Another way of dealing with race conditions is to handle the updates to the data
structure as a transaction, just as updates to a database are done within a transaction.
The required series of data modifications and reads is stored in a transaction log and
then committed in a single step.

The most basic mechanism for protecting shared data provided by the C++ Standard is the mutex, so we’ll look at that.

## 3.2 Protecting shared data with mutexes
Before accessing a shared data structure, you lock the mutex associated with that data, and when you’ve finished accessing the data structure, you unlock the mutex.  

Mutexes are the most general of the data-protection mechanisms available in C++,
but they’re not a silver bullet:
 - it’s important to structure your code to protect the right data (see section 3.2.2)
 - avoid race conditions inherent in your interfaces (see section 3.2.3)
 - deadlock (see section 3.2.4)
 - protecting either too much or too little data (see section 3.2.8)

### 3.2.1 Use mutex
n C++, you create a mutex by constructing an instance of std::mutex , lock it with a call to the member function lock() , and unlock it with a call to the member function unlock().  

The following listing shows how to protect a list that can be accessed by multiple threads using a std::mutex , along with std::lock_guard . Both of these are declared in the <mutex> header.  
```c++
#include <list>
#include <mutex>
#include <algorithm>
std::list<int> some_list; //  1
std::mutex some_mutex; //  2
void add_to_list(int new_value)
{
    std::lock_guard<std::mutex> guard(some_mutex); //  3
    some_list.push_back(new_value);
}
bool list_contains(int value_to_find)
{
    std::lock_guard<std::mutex> guard(some_mutex); //  4
    return std::find(some_list.begin(), some_list.end(), value_to_find) != some_list.end();
}
```
there’s a single global variable _1_, and it’s protected with a corresponding global instance of std::mutex _2_.
The use of `std::lock_guard<std::mutex>` in `add_to_list()` _3_ and again in `list_contains()` _4_ means that the accesses in these functions are mutually exclusive: list_contains() will never see the list partway
through a modification by `add_to_list()`.

If all the member functions of he class lock the mutex before accessing any other data members and unlock it when done, the data is nicely protected from all comers.

Well, that’s not quite true, as the astute among you will have noticed: if one of the member functions returns a pointer or reference to the protected data, then it doesn’t matter that the member functions all lock the mutex in a nice orderly fashion, because you’ve just blown a big hole in the protection. 

*Any code that has access to that pointer or reference can now access (and potentially modify) the protected data without locking the mutex.*

Protecting data with a mutex therefore requires careful interface design, to
ensure that the mutex is locked before there’s any access to the protected data and
that there are no backdoors.

### 3.2.2 Structure code to protect right shared data
At one level, checking for stray pointers or references is easy; as long as none of the member functions return a pointer or reference to the protected data to their caller either via their return value or via an out parameter, the data is safe.

those functions that aren’t under your control might store the pointer or reference in a place where it can later be used without the protection of the mutex. Particularly dangerous in this regard are functions that are supplied at runtime via a function argument or other means, as in the next listing.
```c++
class some_data
{
        int a;
        std::string b;
    public:
        void do_something();
};
class data_wrapper
{
    private:
        some_data data;
        std::mutex m;
    public:
        template<typename Function>
        void process_data(Function func)
        {
            std::lock_guard<std::mutex> l(m);
            func(data); //  1  Pass “protected” data to user-supplied function
        }
};
some_data* unprotected;
void malicious_function(some_data& protected_data)
{
    unprotected = &protected_data;
}
data_wrapper x;
void foo()
{
    x.process_data(malicious_function); //  2 Pass in a malicious function
    unprotected->do_something(); //  3 Unprotected access to protected data
}
```
you have a guideline to follow, which will help you in these cases:  
*Don’t pass pointers and references to protected data outside the scope of the lock, whether by
returning them from a function, storing them in externally visible memory, or passing them as
arguments to user-supplied functions.*

Although this is a common mistake when trying to use mutexes to protect shared
data, it’s far from the only potential pitfall. As you’ll see in the next section, **it’s still
possible to have race conditions, even when data is protected with a mutex**.

### 3.2.3 Spotting race conditions inherent in interfaces
Consider a stack:
```c++
template<typename T,typename Container=std::deque<T> >
class stack
{
    public:
    explicit stack(const Container&);
    explicit stack(Container&& = Container());
    template <class Alloc> explicit stack(const Alloc&);
    template <class Alloc> stack(const Container&, const Alloc&);
    template <class Alloc> stack(Container&&, const Alloc&);
    template <class Alloc> stack(stack&&, const Alloc&);
    bool empty() const;
    size_t size() const;
    T& top();
    T const& top() const;
    void push(T const&);
    void push(T&&);
    void pop();
    void swap(stack&&);
};
```
In order for a thread to safely pop a node:     
 - firstly you need to ensure that  you’re preventing concurrent accesses to three nodes:  
the node being deleted (node2) and the nodes on either side (node 1 and 3). 
 - secondly you need to ensure that  you’re preventing concurrent accesses to three interfaces:  
empty(),top() and pop().

think about the reason for this whole problem in term of _**multi-level-invariant**_ shown below:  
```c++
// to maitain level 1 invarirant, lock concurrent accesss to stack varaibles inside each interface
// to maitain level 2 invariant, lock concurrent accesss to stack interfaces inside caller scope
stack<int> s;
if(!s.empty()) //1
{
    int const value=s.top(); //2
    s.pop(); //3
    do_something(value);
}
```
calling top() on an empty stack is undefined behavior. With a shared stack object, this call sequence is no longer
safe, because there might be a call to pop() from another thread that removes the last element in between the call to 
empty() and the call to top()._This is yet another race condition and far more insidious than the undefined behavior of 
the empty() / top() race;

the use of a mutex internally to protect the stack contents doesn’t prevent it; it’s a consequence of the interface. 
This problem is not unique to a mutex-based implementation; it’s an interface problem, so the race conditions would still
occur with a lock-free implementation.

A more radical change to the interface that combines the calls to top() and pop() under the protection of the mutex can
resolved PRC _but a combined call can lead to issues if the copy constructor for the objects on the stack can throw an exception._

Why? Assume you have `stack<vector<int>>`. If the pop() function was defined to return the value popped, 
as well as remove it from the stack,you have a potential problem: the value being popped is returned to the caller only
after the stack has been modified, but the process of copying the data to return to the caller might throw an exception. 
If this happens, the data just popped is lost; it has been removed from the stack, but the copy was unsuccessful!

So the designers of the std::stack interface helpfully split the operation in two: get the top element ( top() )
and then remove it from the stack ( pop() ), so that if you can’t safely copy the data, it
stays on the stack. If the problem was lack of heap memory, maybe the application can
free some memory and try again.

_Unfortunately, it’s precisely this split that you’re trying to avoid in eliminating the race condition! Thankfully, 
there are alternatives, but they aren’t without cost._

  1. PASS IN A REFERENCE  
pass a reference to a variable in which you wish to receive the popped value as an argument 
in the call to pop():
 ```c++
void pop(std::vector<int>& result);
std::vector<int> result;
some_stack.pop(result);
```

 2. RETURN A POINTER   
return a pointer to the popped item rather than return the item by value.   
The advantage here is that pointers can be freely copied without throwing an exception, 
so you’ve avoided Cargill’s exception problem.    
The disadvantage is that returning a pointer requires a means of managing the memory allocated to the
object, and for simple types such as integers, the overhead of such memory manage-
ment can exceed the cost of just returning the type by value

 3. provide both of 2 and 3  
 Flexibility should never be ruled out, especially in generic code
 
Based on suggestions shown above, this is an thread-safe-stack:
 ```c++
#include <exception>
#include <memory>
#include <mutex>
#include <stack>
struct empty_stack: std::exception
{
    const char*	what() const throw() {};
};
template<typename T>
class threadsafe_stack
{
private:
    std::stack<T> data;
    mutable	std::mutex m;
public:
    threadsafe_stack(){}
    threadsafe_stack(const threadsafe_stack& other)
    {
        // 1 This stack implementation is actually copyable—the copy constructor 
        // locks the mutex in the source object and then copies the internal stack.
        // You do the copy in the constructor body  rather than the member initializer 
        // list in order to ensure that the mutex is held across the copy.
        std::lock_guard<std::mutex> lock(m); 
        data = other.data;	
    }
    threadsafe_stack& operator=(const threadsafe_stack&)=delete;
    void push(T	new_value)
    {
        std::lock_guard<std::mutex> lock(m); 
        data.push(new_value);
    }
    std::shared_ptr<T> pop()
    {
        std::lock_guard<std::mutex> lock(m);     
        //2 Check for empty before trying to pop value
        if(data.empty()) throw empty_stack(); 
        //3 allocate return value before pop it out
        std::shared_ptr<T> const res(std::make_shared<T>(data.top()));
        data.pop();
        return res;
    }
    void pop(T&	value)
    {
        std::lock_guard<std::mutex> lock(m);  
        //4 some threads remove element from backdoor interfaces eg, call data.pop()
        if(data.empty()) throw empty_stack();      
        value=data.top();
        data.pop();
    }
    bool empty() const
    {
        std::lock_guard<std::mutex> lock(m);
        return data.empty();
    }
```

Problems with mutexes can also arise from: 
 1. locking at too large a granularity  
the extreme situation is a single global mutex that protects all shared data.
 2. locking at too small a granularity  
the protection doesn’t cover the entirety of the desired operation (this is problem )

One issue with fine-grained locking schemes is that sometimes you need more than one mutex locked in order to protect 
all the data in an operation.

if you end up having to lock two or more mutexes for a given operation, there’s
another potential problem lurking in the wings: _**deadlock**_. This is almost the opposite
of a race condition: rather than two threads racing to be first, each one is waiting for
the other, so neither makes any progress.

### 3.2.4 Deadlock: the problem and a solution















