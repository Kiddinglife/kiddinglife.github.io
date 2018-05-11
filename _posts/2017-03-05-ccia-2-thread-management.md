---
layout: post
title:  "CCIA-2-basic-thread-mangement"
date:   2017-03-05 21:06:05
categories: read
tags:  thread c++
---

* content
{:toc}

## 2.1 线程管理的基础 Basic Thread Mangement
每个程序至少有一个线程：执行main()函数的线程，其余线程有其各自的入口函数. 线程与原始线程(以main()为入口函数的线程)同时运行。如同main()函数执行完会退出一样，当线程执行完入口函数后，线程也会退出. 在为一个线程创建了一个 std::thread 对象后，需要等待这个线程结束。   
Every C++ program has at least one thread, which is started by the C++ runtime: the thread running main(). Your program can then launch additional threads that have another function as the entry point. These threads then run concurrently with each other and with the initial thread. Just as the program exits when the program returns from main(), when the specified entry point function returns, the thread exits. As you’ll see, if you have a std::thread object for a thread, you can wait for it to finish。

### 2.1.1 Launching a thread 启动线程
使用C++线程库启动线程，可以归结为构造 std::thread 对象:      
Starting a thread using the C++ Thread Library always boils down to constructing a std::thread object:
```c++
class background_task
{
public:
void operator()() const
{
do_something();
do_something_else();
}
};
background_task f;
std::thread my_thread(f);
```

有件事需要注意，当把函数对象传入到线程构造函数中时，如果你传递了一个临时变量，而不是一个命名的变量；C++编译器会将其解析为函数声明，而不是类型对象的定义。    
One thing to consider when passing a function object to the thread constructor is to avoid what is dubbed “C++’s most vexing parse.” If you pass a temporary rather Basic thread management 17 than a named variable, then the syntax can be the same as that of a function declaration, in which case the compiler interprets it as such, rather than an object definition.  

For example,
```c++
std::thread my_thread(background_task());
```
这里相当与声明了一个名为my_thread的函数，这个函数带有一个参数(函数指针指向没有参
数并返回background_task对象的函数)，返回一个 std::thread 对象的函数，而非启动了一个线程。
declares a function my_thread that takes a single parameter (of type pointer to a function taking no parameters and returning a background_task object) and returns a std::thread object, rather than launching a new thread. 


使用在前面命名函数对象的方式，或使用多组括号①，或使用新统一的初始化语法②，可以避免这个问题。
You can avoid this by naming your function object as shown previously, by using an extra set of parentheses, or
by using the new uniform initialization syntax, for example:
```c++
// 1
std::thread my_thread((background_task())); 
// 2
std::thread my_thread{background_task()}; 
// 3
std::thread my_thread([]{
do_something();
do_something_else();
});
```

启动了线程，你需要明确是要等待线程结束，还是让其自主运行。如果 std::thread 对象销毁之前还没有做出决定，程序就会终止( std::thread 的析构函数会调用 std::terminate() )。因此，即便是有异常存在，也需要确保线程能够正确的加入(joined)或分离(detached).    
It’s therefore imperative that you ensure that the thread is correctly joined or detached, even in the presence of exceptions。

如果不等待线程，就必须保证线程结束之前，可访问的数据得有效性。  
If you don’t wait for your thread to finish, then you need to ensure that the data accessed by the thread is valid until the thread has finished with it.  
```c++
//函数已经结束，线程依旧访问局部变量 
//A function that returns while a thread still has access to local variables
struct func
{
int& i;
func(int& i_) : i(i_) {}
void operator() ()
{
   for (unsigned j=0 ; j<1000000 ; ++j)
   {
      // 1. 潜在访问隐患：悬空引用 Potential access to dangling reference
      do_something(i); 
   }
}
};
void oops()
{
   int some_local_state=0;
   func my_func(some_local_state);
   std::thread my_thread(my_func);
   // 2. 不等待线程结束 do not wait for thread to finish
   my_thread.detach(); 
} // 3. 函数退出时新线程可能还在运行  New thread might still be running when func exits
```
处理这种情况的常规方法 common way to handle this scenario:  
1 数据复制到线程中，而非共享数据    
make the thread function self-contained and copy the data into the thread rather than sharing the data。     
2 使用一个能访问局部变量的函数去创建线程是一个糟糕的主意(除非十分确定线程会在函数完成前结束，例如使用join see 2.1.2).  
repace `my_thread.detach()` with `my_thread.join()`;

### 2.1.2 等待线程完成 Waiting for a thread to complete
join()是简单粗暴的等待线程完成或不等待。当你需要对等待中的线程有更灵活的控制时，比
如，看一下某个线程是否结束，或者只等待一段时间(超过时间就判定为超时)。想要做到这
些，你需要使用其他机制来完成，比如条件变量和期待(futures).    
If you need more fine-grained control over waiting for a thread, such as to
check whether a thread is finished, or to wait only a certain period of time, then you
have to use alternative mechanisms such as condition variables and futures。

调用join()的行为，还清理了线程相关的存储部分，这样 std::thread 对象将不再与已经
完成的线程有任何关联。这意味着，只能对一个线程使用一次join();一旦已经使用过
join()， std::thread 对象就不能再次加入了，当对其使用joinable()时，将返回否（false）。    
The act of calling join() also cleans up any storage associated
with the thread, so the std::thread object is no longer associated with the now finished
thread; it isn’t associated with any thread. This means that you can call
join() only once for a given thread; once you’ve called join(), the std::thread
object is no longer joinable, and joinable() will return false.

#### 2.1.2.1 特殊情况下的等待  Waiting in exceptional circumstances
如前所述，需要对一个还未销毁的 std::thread 对象使用join()或detach()。如果想要分离一
个线程，可以在线程启动后，直接使用detach()进行分离。如果打算等待对应线程，则需要细
心挑选调用join()的位置。当在线程运行之后产生异常，在join()调用之前抛出，就意味着很这
次调用会被跳过.   
you need to ensure that you’ve called either join() or detach() before a std::thread object is destroyed.  
If you’re detaching a thread, you can usually call detach() immediately after the thread has been started, so this isn’t a
problem. But if you’re intending to wait for the thread, you need to pick carefully the place in the code where you call join(). This means that the call to join() is liable to be skipped if an exception is thrown after the thread has been started but before the call to join().

Solotion is shown below:
```c++
// solution 1 add join() in all exit palces
struct func; 
void f()
{
   int some_local_state=0;
   func my_func(some_local_state);
   std::thread t(my_func);
try
{
   do_something_in_current_thread();
}
catch(...)
{
   t.join(); // 1
   throw;
}
t.join(); // 2
}

// solution 2 is using RAII to wait for a thread to complete 
// it is easier than s0lution 1
class thread_guard
{
   std::thread& t;
   public:
   explicit thread_guard(std::thread& t_):
   t(t_){}
   ~thread_guard()
   {
      if(t.joinable())
      {
         t.join();
      }
   }
   thread_guard(thread_guard const&)=delete;
   thread_guard& operator=(thread_guard const&)=delete;
};
struct func;
void f()
{
   int some_local_state=0;
   func my_func(some_local_state);
   std::thread t(my_func);
   thread_guard g(t);
   do_something_in_current_thread();
}
```

### 2.1.3 后台运行线程 Running threads in the background
如果线程分离，那么就不可能有 std::thread 对象能引用它，分离线程
的确在后台运行，所以分离线程不能被加入. C++运行库保证，当线程退出时，
相关资源的能够正确回收，后台线程的归属和控制C++运行库都会处理.  
if a thread becomes detached, it isn’t possible to obtain a
std::thread object that references it, so it can no longer be joined. Detached threads
truly run in the background; ownership and control are passed over to the C++ Runtime
Library, which ensures that the resources associated with the thread are correctly
reclaimed when the thread exits.

这种线程的特点就是长时间运行；线程的生命周期可能会从某一个应用起始到结束，可能会在后台监视文件系统，还有可能对缓存进行清理，亦或对数据结构进行优化.  
Such threads are typically long-running; they may well run for almost the entire lifetime of
the application, performing a background task such as monitoring the filesystem,
clearing unused entries out of object caches, or optimizing data structures.

如果不是后台线程，但是若能确定该线程什么时候结束，或者是"发后即忘"(fire andforget)的任务的也可以使用detached线程。
At the other extreme, it may make sense to use a detached thread where there’s another
mechanism for identifying when the thread has completed or where the thread is
used for a “fire and forget” task.
```c++        
void edit_document(std::string const& filename)
{
   open_document_and_display_gui(filename);
   while(!done_editing())
   {
   user_command cmd=get_user_input();
   if(cmd.type==open_new_document)
   {
      std::string const new_name=get_filename_from_user();
      std::thread t(edit_document,new_name); // 1
      t.detach(); // 2
   }
   else
   {
      process_user_input(cmd);
   }
}
}
``` 

## 2.2 向线程函数传递参数 Passing arguments to a thread function 
需要注意的是，默认参数要拷贝到线程独立内存中，即使参数是引用的形式。  
it’s important to bear in mind that by default the arguments are copied into internal
storage, where they can be accessed by the newly created thread of execution,
even if the corresponding parameter in the function is expecting a reference。
```c++ 
void f(int i, std::string const& s);
std::thread t(f, 3, "hello");
```
代码创建了一个调用f(3, "hello")的线程。注意，函数f需要一个 std::string 对象作为第二个
参数，但这里使用的是字符串的字面值，也就是 char const * 类型。之后，在线程的上下文
中完成字面值向 std::string 对象的转化。需要特别要注意，当指向动态变量的指针作为参数
传递给线程的情况，代码如下：  
This creates a new thread of execution associated with t, which calls f(3,”hello”).
Note that even though f takes a std::string as the second parameter, the string literal
is passed as a char const* and converted to a std::string only in the context of
the new thread. This is particularly important when the argument supplied is a
pointer to an automatic variable, as follows:  
```c++        
void f(int i,std::string const& s);
void oops(int some_param)
{
char buffer[1024]; // 1
sprintf(buffer, "%i",some_param);
std::thread t(f,3,buffer); // 2
t.detach();
}
```
这种情况下，buffer②是一个指针变量，指向本地变量，通过std::thread对象构造函数copy到thread内部成员中②。然而，oops函数有很大的可能，会在thread将字面值转化成 std::string 对象之前exit，从而导致线程的一些未定义行为。解决方案就是将字面值转化为 std::string 对象，然后传递给thread，从而拷贝buffer到thread中去.   
In this case, it’s the pointer to the local variable buffer B that’s passed through to the
new thread c, and there’s a significant chance that the function oops will exit before
the buffer has been converted to a std::string on the new thread, thus leading to
undefined behavior. The solution is to cast to std::string before passing the buffer
to the std::thread constructor.(thread's ctor is inviked on the new thread execution).
```c++        
void f(int i,std::string const& s);
void not_oops(int some_param)
{
char buffer[1024];
sprintf(buffer,"%i",some_param);
// 使用std::string，避免悬垂指针 Using std::string avoids dangling pointer
std::thread t(f,3,std::string(buffer)); 
// poesudo code in constructor of thread
// std::string internal_str = std::string(buffer); // value-copy
t.detach();
}
```
不过，也有相反的情况：你想要的就是引用拷贝 而非默认的值拷贝，你需要更新你传递的引用，例如：    
It’s also possible to get the reverse scenario: the object is copied, and what you
wanted was a reference. This might happen if the thread is updating a data structure
that’s passed in by reference, for example:
```c++   
void update_data_for_widget(widget_id w,widget_data& data); // 1
void oops_again(widget_id w)
{
   widget_data data;
   std::thread t(update_data_for_widget,w,data); // 2
   display_status();
   t.join();
   process_widget_data(data); // 3
}
```
当线程调用
update_data_for_widget函数时，传递给函数的参数是data变量内部拷贝的引用，而非数据本
身的引用。因此，当线程结束时，内部拷贝数据将会在数据更新阶段被销毁，且 process_widget_data将会接收到没有修改的data变量③。
使用 std::bind ，就可以解决这个问题，使用 std::ref 将参数转换成引用的形式。这种情况下，可将线程的调用，改成以下形式：  
Consequently, when the
thread finishes, these updates will be discarded as the internal copies of the supplied arguments are destroyed, and process_widget_data will be passed an unchanged data d rather than a correctly updated version. you need to wrap the arguments that really need to be references in std::ref. In this case, if you change the thread invocation to：
```c++
std::thread t(update_data_for_widget,w,std::ref(data));
```

## 2.3 Transferring ownership of a thread 转移线程所有权
假设要写一个函数，该函数会创建一个运行在后台的新线程， 并且会返回该线程的所有权给函数调用者；
或完全与之相反：创建一个线程，并将所有权通过参数传递给另外一个函数，并且在该函数作用域内join该线程。
总之，新线程的所有权都需要转移。 明执行线程的所有权可以在 std::thread 实例中移动，下面将展示一个例
子。例子中，创建了两个执行线程，并且在 std::thread 实例之间(t1,t2和t3)转移所有权：   
the ownership of a particular thread of execution can be moved between std::thread instances, as in the following example. The example 26 CHAPTER 2 Managing threads shows the creation of two threads of execution and the transfer of ownership of those threads among three std::thread instances, t1, t2, and t3:
```c++ 
void some_function();
void some_other_function();

/* 1 a new thread is started and associated with t1. */
std::thread t1(some_function);

/* 2
 * ownership is then transferred over to t2 when t2 is constructed, 
 * by invoking std::move() to explicitly move ownership. 
 * At this point, t1 no longer has an associated thread of execution; 
 * the thread running some_function is now associated with t2.
 */
std::thread t2=std::move(t1);

/* 3
 * Then, a new thread is started and associated with a temporary std::thread
 * object d. The subsequent transfer of ownership into t1 doesn’t require a call to std::
 * move() to explicitly move ownership, because the owner is a temporary object—moving
 * from temporaries is automatic and implicit.
 */
t1=std::thread(some_other_function); 

/* 4
 * t3 is default constructed e, which means that it’s created without any associated
 * thread of execution. Ownership of the thread currently associated with t2 is transferred
 * into t3 f, again with an explicit call to std::move(), because t2 is a named object. After
 * all these moves, t1 is associated with the thread running some_other_function, t2 has no
 * associated thread, and t3 is associated with the thread running some_function.
 */
std::thread t3; 
t3=std::move(t2); 

/* 5
 * The final move g transfers ownership of the thread running some_function back
 * to t1 where it started. But in this case t1 already had an associated thread (which was
 * running some_other_function), so std::terminate() is called to terminate the
 * program. This is done for consistency with the std::thread destructor. You saw in
 * section 2.1.1 that you must explicitly wait for a thread to complete or detach it before
 * destruction, and the same applies to assignment: you can’t just “drop” a thread by
 * assigning a new value to the std::thread object that manages it.
 */
t1=std::move(t3); // 赋值操作将使程序崩溃 This assignment will terminate program!
```
The move support in std::thread means that ownership can readily be transferred out of a function:
```c++
// 函数返回 std::thread 对象 Returning a std::thread from a function
std::thread f()
{
   void some_function();
   return std::thread(some_function);
}
std::thread g()
{
   void some_other_function(int);
   std::thread t(some_other_function,42);
   return t;
}
```

Likewise, if ownership should be transferred into a function, it can just accept an
instance of std::thread by value as one of the parameters, as shown here:
```c++
// 传递 std::thread 对象到函数 pass in a std::thread to a function
void f(std::thread t);
void g()
{
void some_function();
// tempory thread instance and use std::move implicitely
f(std::thread(some_function));
std::thread t(some_function);
// named thread instance t and so has to use std::move explicitely
f(std::move(t));
}
```

```c++
class scoped_thread
{
std::thread t;
public:
explicit scoped_thread(std::thread t_): // 1
t(std::move(t_))
{
if(!t.joinable()) // 2
throw std::logic_error(“No thread”);
}
~scoped_thread()
{
t.join(); // 3
}
scoped_thread(scoped_thread const&)=delete;
scoped_thread& operator=(scoped_thread const&)=delete;
};
struct func; 
void f()
{
int some_local_state;
scoped_thread t(std::thread(func(some_local_state))); // 4
do_something_in_current_thread();
}
```

std::thread 对象的容器，如果这个容器是移动敏感的(比如，标准中的 std::vector<> )，那么移动操作同样适用于这些容器.  
The move support in std::thread also allows for containers of std::thread objects, if those containers are move aware (like the updated std::vector<>).
```c++
void do_work(unsigned id);
void f()
{
std::vector<std::thread> threads;
for(unsigned i=0; i < 20; ++i)
{
threads.push_back(std::thread(do_work,i)); // 产生线程 spawn threads
}
std::for_each(threads.begin(),threads.end(),
std::mem_fn(&std::thread::join)); // 对每个线程调用join() call join on each thread
}
```

## 2.4 运行时决定线程数量
将 std::thread 放入 std::vector 是向线程自动化管理迈出的第一步：并非为这些线程创建独
立的变量，并且将他们直接加入，可以把它们当做一个组。创建一组线程(数量在运行时确定)，可使得这一步迈的更大.  
Putting std::thread objects in a std::vector is a step toward automating the management of those threads. You can take this a step further by creating a dynamic number of threads determined at runtime.
```c++
template<typename Iterator,typename T>
struct accumulate_block
{
void operator()(Iterator first,Iterator last,T& result)
{
result=std::accumulate(first,last,result);
}
};
template<typename Iterator,typename T>
T parallel_accumulate(Iterator first,Iterator last,T init)
{
unsigned long const length=std::distance(first,last);
if(!length) // 1
return init;
unsigned long const min_per_thread=25;
unsigned long const max_threads=
(length+min_per_thread-1)/min_per_thread; // 2
unsigned long const hardware_threads=
std::thread::hardware_concurrency();
unsigned long const num_threads= // 3
std::min(hardware_threads != 0 ? hardware_threads : 2, max_threads);
unsigned long const block_size=length/num_threads; // 4
std::vector<T> results(num_threads);
std::vector<std::thread> threads(num_threads-1); // 5
Iterator block_start=first;
for(unsigned long i=0; i < (num_threads-1); ++i)
{
Iterator block_end=block_start;
std::advance(block_end,block_size); // 6
threads[i]=std::thread( // 7
accumulate_block<Iterator,T>(),
block_start,block_end,std::ref(results[i]));
block_start=block_end; // 8
}
accumulate_block<Iterator,T>()(
block_start,last,results[num_threads-1]); // 9
std::for_each(threads.begin(),threads.end(),
std::mem_fn(&std::thread::join)); // 10
return std::accumulate(results.begin(),results.end(),init); // 11
}
```

## 2.5 识别线程 Identifying 
First, the identifier for a thread can be obtained from its associated std::thread object by calling the get_id() member function. returns a defaultconstructed std::thread::id object, which indicates “no any thread.”.  
线程标识类型是 std::thread::id ，可以通过两种方式进行检索。第一种，可以通过调用 std::thread 对象的成员函数 get_id() 来直接获取。如果 std::thread 对象没有与任何执行线程相关联， get_id() 将返回 std::thread::type 默认构造值，这个值表示“没有线程”。第二种，当前线程中调用 std::this_thread::get_id().

threadsstd::thread::id 类型对象提供相当丰富的对比操作；比如，提供为不同的值进行排序. 标准库也提供 std::hash<std::thread::id> 容器，所以 std::thread::id 也可以作为无序容器的键值。
```c++
std::thread::id master_thread;
void some_core_part_of_algorithm()
{
if(std::this_thread::get_id()==master_thread)
{
do_master_thread_work();
}
do_common_work();
}
```
std::thread::id 可以作为一个线程的通用标识符，当标识符只与语义相关(比如，数组的索
引)时，就需要想其他办法了（例如之前例子中只用循环中的index作为线程的标识）。  
The idea is that std::thread::id will suffice as a generic identifier for a thread in
most circumstances; it’s only if the identifier has semantic meaning associated with it
(such as being an index into an array) that alternatives should be necessary.




