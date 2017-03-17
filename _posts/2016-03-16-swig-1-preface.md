---
layout: post
title:  "swig-1-preface"
date:   2016-03-16 19:12:03
categories: tool
tags:  swig script-language c/c++
---

* content
{:toc}

## 1.8 Backwards compatibility
If you are a previous user of SWIG, don't expect SWIG to provide complete backwards compatibility. 

If you need to work with different versions of SWIG and backwards compatibility is an issue, you can use the SWIG_VERSION preprocessor symbol which holds the version of SWIG being executed. 

SWIG_VERSION is a hexadecimal integer such as 0x010311 (corresponding to SWIG-1.3.11). This can be used in an interface file to define different typemaps, take advantage of different features etc:
```c++
#if SWIG_VERSION >= 0x010311
/*Use some fancy new feature */
#endif
```

## 1.12 Installation
If you checked the code out via Git, you will have to run ./autogen.sh before ./configure.
### 1.12.1 Windows installation
The Windows distribution is called swigwin and includes a prebuilt SWIG executable, swig.exe and so ypu do not need to compile it.
### 1.12.2 Unix installation
To build and install SWIG, simply type the following:
```shell
$ ./configure
$ make
$ make install
```
By default SWIG installs itself in /usr/local. If you need to install SWIG in a different location or in your home directory, use the --prefix option to ./configure. For example:
```shell
$ ./configure --prefix=/home/yourname/projects
$ make
$ make install
```
Note: the directory given to --prefix must be an absolute pathname. Do not use the ~ shell-escape to refer to your home directory. SWIG won't work properly if you do this.

### 1.12.4 Testing
if you want to test SWIG after building it, a check can be performed on Unix operating systems. Type the following: ```$ make -k check```  
you can use parallel make to speed up the check as it does take quite some time to run, for
example: ```$ make -j2 -k check```

Even if the check fails, there is a pretty good chance SWIG still works correctly --- you will just have to mess around with one of the examples and some makefiles to get it to work. 

Some tests may also fail due to missing dependency packages, eg PCRE or Boost, but this will require careful
analysis of the configure output done during configuration.

### 1.12.5 Examples
The Examples directory also includes Visual C++ project 6 (.dsp) files for building some of the examples on Windows. Later versions of Visual Studio will convert these old style project files into a current solution file.





