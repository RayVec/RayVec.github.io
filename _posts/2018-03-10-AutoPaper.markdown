---
layout:     post
title:      "论文格式化"
subtitle:   "实验室项目的稳定版"
date:       2018-03-10
author:     "RAY"
header-img: "img/post-bg-ios9-web.jpg"
catalog: true
tags:
    - Java
    - 文档
---


>
这是我目前正在研究的一个关于论文格式化的项目，功能是将随意排版的论文转换成标准的论文格式，使用的框架是Java的POI，前前后后研究了将近四个月，用到的是编译原理和模式识别的思想。目前已经完成了主要的功能，为了提高精确度还需要使用深度学习技术，还没有涉猎。

具体的实现过程这里不做过多的介绍了，主要是利用POI对一个word文档进行一层层的解析，最终构建成一个解析树，从而重铸文章的结构。

项目[地址](https://github.com/RayVec/AutoPaper)