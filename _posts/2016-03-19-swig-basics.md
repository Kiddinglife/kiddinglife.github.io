---
layout: post
title:  "SWIG-3-Basics"
date:   2016-03-19 20:10:03
categories: tool
tags:  swig script-language c/c++
---

* content
{:toc}

## 5.2 Wrapping Simple C Declarations
For example, consider the following interface file:
```c++
%module example
%inline %{
extern double sin(double x);
extern int strcmp(const char *, const char *);
extern int Foo;
%}
#define STATUS 50
#define VERSION "1.1"
```
When SWIG creates an extension module, these declarations are accessible as scripting language functions, variables, and constants respectively. For example, in python:
```python
>>> example.sin(3)
5.2335956
>>> example.strcmp('Dave', 'Mike')
-1
>>> print example.cvar.Foo
42
>>> print example.STATUS
50
>>> print example.VERSION
1.1
```
Whenever possible, SWIG creates an interface that closely matches the underlying C/C++ code. However, due to subtle differences between languages, run-time environments, and semantics, it is not always possible to do so. 

### 5.2.1 Basic Type Handling
In order to build an interface, SWIG has to convert C/C++ datatypes to equivalent types in the target language:

#### 5.2.1.1 Integral
When an integral value is converted from C, a cast is used to convert it to the representation in the target language. Thus, a 16 bit short in C may be promoted to a 32 bit integer. When integers are converted in the other direction, the value is cast back into the original C type. If the value is too large to fit, _it is silently truncated._

bool datatype is cast to and from an integer value of 0 and 1 unless the target language provides a special boolean type.    

unsigned/signed char are special cases that are handled as small 8-bit integers. Normally, the char datatype is mapped as a one-character ASCII string.      

_As a rule of thumb, the int datatype and all variations of char and short datatypes are safe to use. For unsigned int and long datatypes, you will need to carefully check the correct operation of your program after it has been wrapped with SWIG._
 
#### 5.2.1.2 Float
SWIG recognizes the following floating point types : float and double. Floating point numbers are mapped to and from the natural representation of floats in the target language. This is almost always a C _double_.

#### 5.2.1.3 Unicode:
For those scripting languages that provide Unicode support, Unicode strings are often available in an 8-bit representation such as UTF-8 that can be mapped to the char * type (in which case the SWIG interface will probably work). 

### 5.2.2 Global Variables
Whenever possible, SWIG maps C/C++ global variables into scripting language variables. 

Whenever the scripting language variable is used, the underlying C global variable is accessed。However, the way to access global variables in different script langs are different. 

Finally, if a global variable has been declared as const, it only supports read-only access. 

For example:
```c++
%module example
double foo;
```
results in a scripting language variable like this:
```c++
# Tcl
set foo [3.5]                   ;# Set foo to 3.5
puts $foo                       ;# Print the value of foo
# Python
cvar.foo = 3.5                  # Set foo to 3.5
print cvar.foo                  # Print value of foo
# Perl
$foo = 3.5;                     # Set foo to 3.5
print $foo, "\n";               # Print value of foo
# Ruby
Module.foo = 3.5               # Set foo to 3.5
print Module.foo, "\n"         # Print value of foo
```

### 5.2.3 Constants  
Constants can be created using #define, enumerations, or a special %constant directive. The following interface file shows a few valid constant declarations :
```c++
#define I_CONST       5               // An integer constant
#define PI            3.14159         // A Floating point constant
#define S_CONST       "hello world"   // A string constant
#define NEWLINE       '\n'            // Character constant
enum boolean {NO=0, YES=1};
enum months {JAN, FEB, MAR, APR, MAY, JUN, JUL, AUG,
             SEP, OCT, NOV, DEC};
%constant double BLAH = 42.37;
#define PI_4 PI/4
#define FLAGS 0x04 | 0x08 | 0x40
```

#### 5.2.3.1 Special cases
1.SWIG will not create constants for macros unless the value can be completely determined by the preprocessor.
```c++
#define EXTERN extern
EXTERN void foo();
```
what would the value of a constant called EXTERN would be?  

2.for the same conservative reasons even a constant with a simple cast will be ignored, such as
```c++
#define F_CONST (double) 5 // A floating point constant with cast
```

3.For enumerations, it is critical that the original enum definition be included somewhere in the interface file (either in a header file or in the %{ %} block) because it needs the original enumeration declaration in order to get the correct enum values as assigned by the C compiler.

#### 5.2.3.2 A brief word about const
Typically, %constant is only used when you want to add constants to the scripting language interface that are not defined in the original header file.

If the right-most const occurs after all other type modifiers (such as pointers), then the variable is const. Otherwise, it is not. Here are some examples of const declarations:
```c++
const char a;           // A constant character
char const b;           // A constant character (the same)
char *const c;          // A constant pointer to a character
const char *const d;    // A constant pointer to a constant character
```
Here is an example of a declaration that is not const:  
```c++
const char *e; 
```
In this case, the pointer e can change --- it's only the value being pointed to that is read-only.

### 5.2.5 A cautionary tale of char *
When strings are passed from a scripting language to a C char *, the pointer usually points to string data stored inside the interpreter.
```c++
char *strcat(char *s, const char *t)
```
your application will perhaps crashes with a segmentation fault or other memory related problem. This is because s refers to some internal data in the target language---data that you shouldn't be touching.






