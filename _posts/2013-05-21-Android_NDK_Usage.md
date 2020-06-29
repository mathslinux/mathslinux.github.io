---
layout: post
tag: Android
date: '\[2013-05-21 二 16:30\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: Android NDK Usage
---

Sometimes we have to write some C/C++ code to deal with something that
java can not do in android development. Such as interacting with
hardware, or reusing some codec library written in C/C++.

Because of above reasons, we need a tool or a method which helps us call
C/C++ code in android(java) code.

The NDK(Native Development Kit) is the tool which fits our requirements.

This page simply describes how to use C/C++ code in android project.

Installing the NDK
==================

Just download the Android NDK from
[here](http://developer.android.com/tools/sdk/ndk/index.html), and unzip
into a anyplace you like, e.g. /opt

Load the library and declare method
===================================

In order to use the functions written with C/C++, we need to load the
library which contains those functions, and declare those functions.

For example, suppose the function we want to call is sayHello(), which
is contained in library libjdemo.so.

Below code show the details:

``` java
package com.example.jdemo;

// A simple JNI interface wrapper
public class JNIWrapper {
    static {
        // Load libjdemo.so, must be called in static block
        System.loadLibrary("jdemo");
    }

    // Declare the C function we want to call
    public static native String sayHello();
}
```

Implement C/C++ code
====================

This step, we are readying to implement the C function declared above.

Create header file
------------------

We can use "javah" utility to create C/C++ header file.

``` bash
$ mkdir jni # this directory is used to place C/C++ source files
$ cd bin/classes && javah com.example.jdemo.JNIWrapper # create header file, note the arguments passed to "javah"
$ cd - && mv bin/classes/com_example_jdemo_JNIWrapper.h jni/demo.h # move the header file to jni directory, for convinent, rename filename to anyone you want
```

Implement the C function
------------------------

Suppose we use demo.c as the C source file, below is its contents:

``` c
#include <stdio.h>
#include <time.h>
#include "demo.h"

/* Return the time information */
JNIEXPORT jstring JNICALL Java_com_example_jdemo_JNIWrapper_sayHello
(JNIEnv *je, jclass jc)
{
    char buf[256];
    time_t tm = time(NULL);
    char *ct = ctime(&tm);

    snprintf(buf, sizeof(buf), "Hello, the time is: %s\n", ct);
    return (*je)->NewStringUTF(je, buf);
}
```

Build library
=============

For building the source file to library using NDK, we need a Makefile
named Android.mk.

``` makefile
LOCAL_PATH := $(call my-dir)

include $(CLEAR_VARS)

LOCAL_MODULE := jdemo # so the library will be named libjdemo.so
LOCAL_SRC_FILES := demo.c # the source files

include $(BUILD_SHARED_LIBRARY)
```

then compile source using ndk-build

``` bash
$ /opt/android-ndk-r8d/ndk-build 
Compile thumb  : jdemo <= demo.c
SharedLibrary  : libjdemo.so
Install        : libjdemo.so => libs/armeabi/libjdemo.so
```

others
======

That's all
