---
layout: post
title:  "SWIG-2-script-lang"
date:   2016-03-18 21:10:03
categories: tool
tags:  swig script-language c/c++
---

* content
{:toc}

This blog provides a brief overview of:
 1. script language extension programming
 2. mechanisms by which script lang interpreters access C and C++ code
 
C/C++ can be used for maximal performance and complicated systems programming tasks.  
Scripting languages can be used for rapid prototyping, interactive debugging, scripting, 
and access to high-level data structures such associative arrays. 

## 4.2 How does a script lang talk to C?
By extending the interpreter, it is usually possible to add new commands and variables.

To do this, most lang define a special API for adding new commands. Furthermore, a special foreign function interface 
defines how these new commands are supposed to hook into the interpreter.

Typically, when you add a new command to a scripting interpreter you need to do two things:
 - first you need to write a special "wrapper" function that serves as the glue between the 
interpreter and the underlying C function. 
 - Then you need to give the interpreter information about the wrapper by providing 
details about the name of the function, arguments, and so forth.

### 4.2.1 Wrapper functions
Suppose you have an ordinary C function like this :
```c++
int fact(int n) {
    if (n <= 1) return 1;
    else return n*fact(n-1);
}
```

A wrapper function for it must do three things :
 - Gather function arguments and make sure they are valid.
 - Call the C function.
 - Convert the return value into a form recognized by the scripting language.
 
As an example, the Tcl wrapper function for the fact() function above example might look like the following :
```c++
int wrap_fact(ClientData clientData, 
              Tcl_Interp *interp,
              int argc, 
              char *argv[]) 
{
    int result;
    int arg0;
    if (argc != 2) 
    {
        interp->result = "wrong # args";
        return TCL_ERROR;
    }
    arg0 = atoi(argv[1]);
    result = fact(arg0);
    sprintf(interp->result,"%d", result);
    return TCL_OK;
}
```
---
Once you have created a wrapper function, the final step is to tell the scripting language about the new function. 
This is usually done in an initialization function called by the language when the module is loaded. 

For example, adding the above function to the Tcl interpreter requires code like the following :
```c++
int Wrap_Init(Tcl_Interp *interp)
{
    Tcl_CreateCommand(interp, 
    "fact", wrap_fact, 
    (ClientData) NULL,
    (Tcl_CmdDeleteProc *) NULL);
    return TCL_OK;
}
```
When executed, Tcl will now have a new command called "fact" that you can use like any other Tcl command.

Although the process of adding a new function to Tcl has been illustrated, the procedure is almost identical 
for Perl and Python. Both require special wrappers to be written and both need additional initialization code.
Only the specific details are different.

### 4.2.2 Variable linking
ariable linking refers to the problem of mapping a C/C++ global variable to a variable in the scripting language 
interpreter. For example, suppose you had the following variable: `double Foo = 3.5;`

evaluating a variable such as $Foo might implicitly call the get function. Similarly, typing $Foo = 4 would call the 
underlying set function to change the value.

### 4.2.3 Constants
In many cases, a C program or library may define a large collection of constants. For example:
```c++
#define RED
0xff0000
#define BLUE 0x0000ff
#define GREEN 0x00ff00
```
To make constants available, their values can be stored in scripting language variables such as $RED, $BLUE, and $GREEN.
Virtually all scripting languages provide C functions for creating variables so installing constants is usually a trivial 
exercise.

### 4.2.4 Structures and classes





