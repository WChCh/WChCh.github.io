---
title: Hyper-V之WMI、J-Interop
date: 2017-11-03 16:57:55
tags: [Hyper-V,虚拟化]
categories: [技术,学习,Hyper-V]
---

### 概述

经过调研发现要通过调用WMI接口来实现对Hyper-V的管理，由于之前没有相关的经验所以通过网上搜索来理解什么是WMI。

关于使用何种编程语言调用WMI，微软官网给出的是封装很友好的c#和powershell脚本接口，但目前公司的开发语言主要为Java，所以经过调研是能使用J-Interop。J-Interop是一个基于Java语言的WMI封装库，但是用方式还是太过于原始，使用上不是很友好，还得需要进一步封装，当然这一部分不在本文的编写范围内。

### 什么是WMI？

官网介绍是：

Windows Management Instrumentation (WMI) is the Microsoft implementation of Web-Based Enterprise Management (WBEM), which is an industry initiative to develop a standard technology for accessing management information in an enterprise environment. WMI uses the Common Information Model (CIM) industry standard to represent systems, applications, networks, devices, and other managed components. CIM is developed and maintained by the Distributed Management Task Force (DMTF). [About WMI](https://msdn.microsoft.com/en-us/library/aa384642(v=vs.85).aspx)

以下WMI相关的内容转载自：<http://blog.csdn.net/ilovepxm/article/details/6690224>

<!--more-->

### WMI的组成？

严格说来，WMI由四部分组成：

1. 公共信息模型对象管理器——CIMOM
2. 公共信息模型——CIM
3. WMI提供程序
4. WMI脚本对象库
   其中其第1、2、3三个部分，在使用中很少加以区别，我们一般统称为CIM库。
   用WMI脚本对象库访问cim库。
   本质：是一个带很多分支的树状数据库，是一项服务！

### WMI能做什么？

利用WMI可以高效地管理远程和本地的计算机.WMI是WBEM模型的一种实现。

WBEM模型最关键的部分是它的数据模型（或描述和定义对象的方式）、编码规范（Encoding Specification），以及在客户端和服务器端之间传输数据的模式。
CIM即WBEM的数据模型！

CIM是一个用来命名计算机的物理和逻辑单元的标准的命名系统（或称为命名模式），例如硬盘的逻辑分区、正在运行的应用的一个实例，或者一条电缆。

WBEM模型：基于web的企业管理，为管理企业环境开发一套标准接口集。

WMI作为一种规范和基础结构，通过它可以访问、配置、管理和监视几乎所有的Windows资源，比如用户可以在远程计算机器上启动一个进程；设定一个在特定日期和时间运行的进程；远程启动计算机；获得本地或远程计算机的已安装程序列表；查询本地或远程计算机的Windows事件日志等等。

WMI通过提供一致的模型和框架，所有的 Windows 资源均被描述并公开给外界。最好的一点是，系统管理员可以使用 WMI 脚本库创建系统管理脚本，从而管理任何通过 WMI 公开的 Windows 资源！

### 什么是J-Interop？

官网介绍如下：

j-Interop is a Java Open Source library (under LGPL) that implements the DCOM wire protocol (MSRPC) to enable development of Pure, Bi-Directional, Non-Native Java applications which can interoperate with any COM component.

The implementation is itself purely in Java and does not use Java Native Interface (JNI) to provide COM access. This allows the library to be used from any Non-Windows platform.

It comes with pre-implemented packages for automation. This includes support for IDispatch, ITypeInfo, and ITypeLib. For more flexibility (in the cases where automation is not supported), it provides an API set to directly invoke operations on a COM server.

Another important feature is allowing full access and manipulation (C-R-U-D) of the Windows Registry in a platform independent manner.
The implementation has been tested on all advanced Windows and Fedora platform(s) and displays upward compatibility from JRE 1.3.1.

通过J-Interop就可以实现对WMI的访问。