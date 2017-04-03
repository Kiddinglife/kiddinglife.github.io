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

Some of this chapter focuses on the low-level procedural interface to C++ that is used as the foundation for all language modules. 

__NOTE:__ Keep in mind that the target languages also provide the high-level OO interface via proxy classes. More detailed coverage can be found in the documentation for each target language.

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
```c++
$ swig -c++ -tcl example.i
$ c++ -fPIC -c example_wrap.cxx 
$ c++ example_wrap.o $(OBJS) -o example.so
```
Unfortunately, the process varies slightly on each platform. Make sure you refer to the documentation on each 
target language for further details. 















