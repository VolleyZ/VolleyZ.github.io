---
title: OpenGL
grammar_cjkRuby: true
header-img: "img/post-bg-e2e-ux.jpg"
catalog: true
layout:  post
tags:
    - OpenGL
---

---
## OpenGL初探
OpenGL是一套用于渲染的图形API。


* **OpenGL上下文(context)**    
在应用程序调用OpenGL指令之前，首先都要创建一个上下文。这个上下文是一个非常庞大的状态机，保存了OpenGL中的各种状态。

* **OpenGL状态机(context)**   
状态机描述了一个对象在其生命周期内所经历的各种状态、状态间的转变、发生转变的动因、条件及转变中所执行的活动。
或者说，状态机是一种行为，说明对象在其生命周期中响应时间所经历的状态序列以及对那些状态事件的响应。
因此具有以下特点：
1.有记忆功能，能记住其当前的状态；
2.可以接收输入，根据输入的内容和自己的原先状态，修改自己当前状态，并且可以有对应输出；
3.当进入特殊状态（停机状态）的时候，变不再接收输入，停止工作


