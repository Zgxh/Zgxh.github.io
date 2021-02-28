---
title: Tex与论文技巧
date: 2020-04-24 22:48:40
tags: 科研
categories: 科研
---

## tex 引文引用网址

首先tex文件导包 `\usepackage{url}`

其次bib文件中写入

```tex
@Misc{timmurphy.org,
author = {Murphy, Timothy I},
title = {Line Spacing in LaTeX documents},
howpublished = {\url{http://timmurphy.org/2009/07/22/line-spacing-in-latex-documents/}},
note = {Accessed April 4, 2010}
}
```

## PPT 画图的导出问题

把某页ppt导出成pdf格式的图片
在pptx文件中，文件 - 导出 - 导出pdf - 选项（当前页）。

导出时选择PDF，会保存成矢量图
不要选择Adobe PDF，否则图片会失真

## pdf 矢量图的裁剪问题

tex编辑软件内集成了一个叫做 `pdfcrop` 的工具， 可以自动裁剪pdf使之适应图片大小.

在文件目录下打开终端，调用指令：

`pdfcrop name1.pdf name2.pdf`

## pdf结果图中添加箭头等额外元素

例如：HNE里对降维结果pdf中缺点指向的箭头添加。
使用编辑pdf功能 -> 插入图像。
图像来源可以是用ppt的元素组合，另存为png图像。