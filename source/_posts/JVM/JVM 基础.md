---
title: JVM 基础
date: 2020-9-26 9:59:50
tags: JVM
categories: JVM
mathjax: true
---

# JVM 基础

## 1. Java 程序的执行流程

- java 程序首先经过编译器 javac 的编译，然后交给虚拟机 java。

- 字节码的运行过程：
  - 通过类装载子系统来加载数据到运行时数据区
  - 通过字节码执行引擎来执行代码

![image](https://note.youdao.com/yws/public/resource/bfce0e3d92cf4516094fe684a07f9b39/xmlnote/2C0FE3ED9578470CA9B239DB2558AA9A/8832)