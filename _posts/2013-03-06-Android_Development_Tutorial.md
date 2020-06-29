---
layout: post
tag: Android
date: '\[2013-03-06 三 22:15\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-2.org'
title: Android 开发指南笔记
---

以下是我学习 Android Development Tutorial 的笔记

什么是 Android
==============

Android 操作系统
----------------

-   Android SDK(Software Development Kit) 提供了所有开发 Android 程序
    需要用到的工具, 包括一个编译器, 调试器, 和一个模拟器.

Google Play(Android 市场)
-------------------------

安全和权限
==========

Android 中的安全概念
--------------------

-   Android 系统会为每个程序创建一个独立的用户和组 ID,
    其它程序不能访问属于 其它程序的文件.
-   如果想要在不同程序间共享数据, 必须严格的通过一些服务来做.

Android 中的权限概念
--------------------

-   Android 预定义了一些权限, 但是不同的程序可以定义额外的权限
-   需要的权限在 AndroidManifest.xml 中申明
-   权限有不同的级别, 有的权限北系统自动的授予, 有的被自动的拒绝
-   大多数权限都是在程序安装前被用户授予的.
-   如果用户拒绝授予一个权限, 那么该程序将不能被安装, 只有安装的时候
    才进行权限的检查, 安装后不能修改权限.

Android 应用程序和任务
======================

应用程序
--------

Android 由不同的组件和额外的资源组成, android 系统中有四大组件(Activity,
Service, BroadcastReceiver, ContentProvider)

任务
----

Android 程序的组件可以连接到其他程序的组件, 从而创建任务

Android 用户接口组件
====================

描述 Android 中用户接口组件相关的最重要的组件

Activity
--------

Activity 代表 Android 程序的可视化界面, activities 使用 view 和
fragments 创建 用户界面.

Android 程序一般由多个 Activity 组成, 各 activities 之间没有直接的关联.
必须有一个 activity 被指定为主 activity, 它是程序启动时首先显示的界面.
每个 activity 都可以随意启动其它的 activity. 每当一个 activity 被启动,
则前一个 activity 就被停止.

Fragments
---------

一个 activity 可能包含一个或多个 fragment(或者不包含). 它们协同工作组成
一个连贯的 UI 界面. 例如, 一个 activity 包含左右两个 fragment, 左侧的
fragment 包含了一个列表(新闻题目), 点击每个新闻题目的时候, 右侧的
fragment 就会显示这 条新闻的详尽信息.

[没有 fragments 的 activity](fragment-0.gif)

[包含 fragments 的 activity](fragment.gif)

Views and layout manager
------------------------

Views 是用户接口的组件(例如按钮或者文本框), Views 的基类是
android.view.View, Views 具有控制显示和行为属性.

Layout manager 负责排列其他 views, 他的基类是 android.view.ViewGroup,
继承自 View class.

Device configuration specific layouts
-------------------------------------

Activity 的 user interface 是由 XML 文件定义的(称为 layout files),
为不同的 设备配置定义不同的 layout files 是有可能的,
比如不同的设备定义不同的长宽.

其它 Android 组件
=================

Intents
-------

Intents 是一个异步信息的载体, 可以描述想要执行的操作以及用于这个操作的
数据和其它属性. Intents 允许应用程序向其它组件发起请求.

如何解决哪个组件应该接收 intent 消息? 对于没有指定目标组件名字的 intent,
这个处理过程包括按照 intent filters 匹配每个潜在的目标对象

Services
--------

Services 在后台执行任务. Services 可以和其它组件进行交互, 通过
notification 框架通知用户.

ContentProvider
---------------

提供了结构化的数据的使用. Content providers 是从一个进程连接另一个进程中
的数据的标准接口, 用 sqlite 来存储, 访问数据.

ContentProvider 分为系统的和自定义的, 系统的也就是例如联系人,
图片等数据.

BroadcastReceiver
-----------------

broadcast receivers 用来接收系统信息和 intents. 当指定的事件发生时候,
broadcast receiver 被系统调用.

例如, 注册了当系统启动完毕是调用 broadcast receiver, 或者手机状态改变
时(有人打电话给你)调用 broadcast receiver.

(HomeScreen) Widgets
--------------------

Widgets 主要用来在 homescreen 上和用户交互.

主要时显示一些数据, 允许用户通过它执行一些操作. 例如, 一个小 widget
能显示 总的 email 的信息, 如果用户点击它, 马上就启动一个 email 程序.

Live Wallpapers
---------------

允许用户创建动态的屏幕背景.

Android 开发工具
================

Android SDK
-----------

Android SDK(Software Development Kit) 包含所有创建, 编译, 打包 Android
程序的工具.

Android SDK 还提供了一个模拟器, 你可以用 SDK 创建 AVD(Android virtual
devices) 来 测试 Android 程序.

Android SDK 海包括了一个 debug 工具(adb), 允许连接一个虚拟或者真实的
Android 设备.

Android 开发工具
----------------

Google 提供了 ADT(Android Development Tools) 用来在 Eclipse 上开发
Android 程序. ADT 时一些 Eclipse 的插件组合.

ADT 包含了左右创建, 编译, 调试, 部署 Android 程序 需要的工具. ADT
也允许创建和启动 AVD.

ADT 对资源文件(e.g. layout files)提供了所见即所得功能.

Dalvik 虚拟机
-------------

Android 系统使用一种称为 Dalvik 的虚拟机来运行 java 字节码. Dalvik
使用自己私有的字节码格式.

所以不能直接在 Android 上运行一般的 Java 程序, 必须要转换为 Dalvik
的字节码格式

怎样开发 Android 程序
---------------------

Android 程序使用 Java 开发的. Java 源代码需要首先被 Java 编译器 编译成
Java class 文件.

Android SDK 包含了一个转换 Java chass 到 .dex(Dalvik Executable)的工具
dx. 所有的 class 文件会被转换为 .dex 文件. 在转换中, class
文件里多余的信息会被 优化掉, 这样会节省空间和提高新能. 例如, 不同的
class 文件里面有相同的字符串, 那么 .dex 只会包含一个该字符串的引用.

因此, .dex 文件比相应的 class 文件小很多.

.dex 文件和 Android 程序的资源文件(图片和 XML 文件)被打包进 .apk(Android
Package) 文件, 这是由 aapt(Android Asset Packaging Tool) 工具来完成的.

.apk 文件包含 运行 Android 程序所需要的所有数据, 通过 adb 工具能被部署到
Android 设备上.

ADT(Android Development Tool) 会自动的执行这些步骤, 所以对用户来说,
他们时透明的.

如果使用 ADT 工具, 只需要点击一个按钮, .apk 文件就会被自动的创建和分发.

Resource editors
----------------

ADT 允许开发者通过两种方法自己定义各种样式:

-   通过一个所见即所得编辑器
-   直接修改 XML 文件

两种方法可以自由的切换

Android 应用程序架构
====================

AndroidManifest.xml
-------------------

AndroidManifest.xml 是保存组件和设置程序的文件, e.g. 所有的 activities
和 services 都需要在这个文件里申明

另外, 程序需要的权限也需要在这个文件里面申明.

下面是一个简单的 helloworld 程序的 AndroidManifest.xml:

``` xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.helloworld"
    android:versionCode="1"
    android:versionName="1.0" >

    <uses-sdk
        android:minSdkVersion="8"
        android:targetSdkVersion="17" />

    <application
        android:allowBackup="true"
        android:icon="@drawable/ic_launcher"
        android:label="@string/app_name"
        android:theme="@style/AppTheme" >
        <activity
            android:name="com.example.helloworld.MainActivity"
            android:label="@string/app_name" >
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>

</manifest>
```

`package` 属性定义了是本文见引用的 java 包.

Google Play 需要每个 Android 程序使用它自己唯一的包名, 所以使用
`package` 是一个好习惯.

`android:versionName` 和 `android:versionCode` 表示应用程序的版本

`versionCode` 必须是整数.

`<activity>` 标签定义了一个 activity, 这里指向
`com.example.helloworld.MainActivity`. 这里的 `intent filter`
注册为应用程序启动后, 该 activity 随之启动(action
android:name="android.intent.action.MAIN). 而
"android:name=android.intent.category.LAUNCHER"
表示该程序被添加到应用程序 目录.

`@string/app_name` value refers to resource files which contain the
actual value of the application name. The usage of resource file makes
it easy to provide different resources, e.g. strings, colors, icons, for
different devices and makes it easy to translate applications.

`uses-sdk` 描述了最小的 SDK 版本

Activities and Lifecycle
------------------------

Android 系统控制了程序的生命周期, 任何时候 Android 都可以结束应用程序,
例如有来电的时候. Android 系统提供了一些预定义的方法来管理生命周期,
最重要的是 以下是几个比较重要的方法:

onSaveInstanceState()
:   called after the Activity is stopped. Used to save data so that the
    Activity can restore its states if re-started

onPause()

:   always called if the Activity ends, can be used to release resource
    or save data

onResume()

:   called if the Activity is re-started, can be used to initialize
    fields

参考: ![activity 生命周期管理](/images/posts/Android/activity_lifecycle_thumb1.jpg)

Configuration Change
--------------------

Context
-------

资源
====

使用资源
========
