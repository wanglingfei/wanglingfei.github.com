---
layout:     post
title:      angularjs  知识点笔记
category: techNote
description: angularjs  知识点笔记
tags: angularjs  
---

最近新公司用了一段时间的Angular，嘛，我以前还是有点儿抵触Angular的，害怕，不过用起来还是很爽的，毕竟它确实节省了很多开发时间，基本不操作Dom了，然而总觉得HTML加了这么多逻辑，有点儿别扭。。。而且，Angular还有很多很多的东西没搞懂，或者很多代码还有很多优化空间。工程上就不说了，太多可做的了。。。一步一步改进吧。

先记录一下这些日子看Angular的一些知识点吧，一点点补充。

1、controller在每次切换或刷新页面的时候，Angular 会清空当前的 controller，而service可以用来永久保存应用的数据，并且这些数据可以在不同的 controller 之间使用。

Angular 提供了3种方法来创建并注册service。

	Factory
	Service
	Provider

1) 用 Factory 就是创建一个对象，为它添加属性，然后把这个对象返回出来。 把 service 传进 controller 之后，在 controller 里这个对象里的属性就可以通过 factory 使用了。

2) Service 是用"new"关键字实例化的。因此，你应该给"this"添加属性，然后 service 返回"this"。你把 service 传进 controller 之后，在controller里 "this" 上的属性就可以通过 service 来使用了。一旦了解了new关键字的作用，就知道Services和Factories几乎一样

3) Providers 是唯一一种你可以传进 .config() 函数的 service。当你想要在 service 对象启用之前，先进行模块范围的配置，那就应该用 provider。Provider是你可以传递到应用程序的app.config部分唯一的服务。


2、stateProviders路由

resolve实例化之后，controller才会实例化

3、ng只有在指定事件触发后，才进入 $digest cycle ： - DOM事件，譬如用户输入文本，点击按钮等。( ng-click ) - XHR响应事件 ( $http ) - 浏览器Location变更事件 ( $location ) - Timer事件( $timeout , $interval ) - 执行 $digest() 或 $apply()。可以手动调用$apply触发。


4、Angularjs的生命周期

编译阶段 ：遍历整个HTML并根据js中的指令定义处理页面上声明的指令。一个指令一旦编译完成，就可以通过编译函数进行访问，这个编译函数返回模版函数，其中含有完整的解析树。最后，模版函数被传递给编译后的DOM树中每个指令定义规则中指定的链接函数。

compile返回一个对象或者汗水，负债对模版DOM进行转换，

link链接函数负责将作用域和DOM进行链接

－－－－－－－－－－－

2016.2.19 挤挤放个知识点

js标签要加＋async defer。script标签放在body底部，做与不做async或者defer处理，都不会影响首屏时间，但影响DomContentLoad和load的时间，进而影响依赖他们的代码的执行的开始时间。



参考资料：

[AngularJS 之 Factory vs Service vs Provider](http://www.oschina.net/translate/angularjs-factory-vs-service-vs-provider)

[学习 ui-router 系列文章索引](http://bubkoo.com/2014/01/02/angular/ui-router/guide/index/)


