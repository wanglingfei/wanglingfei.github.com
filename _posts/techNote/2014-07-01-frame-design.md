---
layout:     post
title:      Javascript框架设计 学习笔记
category: techNote
description: Javascript框架设计 学习笔记
tags: Javascript 框架  学习笔记
---

>第一章：种子模块

种子模块也叫核心模块，是框架的最先执行的部分。包括：对象扩展，数组化，类型判断，简单的事件绑定与卸载，无冲突处理，模块加载与domReady。

**1、命名空间**
	
定义一个全局变量作为命名空间，然后对它进行扩展，去jQuery的jQuery。
	
**2、对象扩展**

即 jQuery=>extend or mixin 	

**3、数组化**

浏览器下存在许多类数组对象，如function内的argument。为了享受纯数组的便捷方法，处理他们前都会做一下转换。

	通常用[].slice.call,能将具有length属性的对象转成数组
	[].slice.call(arguments)相当于Array.prototype.slice.call(arguments)。
	但旧版IE下的HTMLCollection、NodeList不是Object子类，如上方法会导致IE异常，需要在IE下单独实现一个slice方法（各大框架做法均不同，此处仅记容易理解的作者的mass框架的方法）,就是把对象遍历进一个数组里。
	$.slice = window.dispatchEvent ? function(nodes,start,end){
		return {}.slice.call(nodes,start,end);
	} : function(nodes,start,end){
		var ret = {},n = nodes.length;
		if( end === void 0 || typeof end ==="number" && isFinite(end)){
			start = parseInt(start,10) || 0;
			end = end == void 0 ? n :parseInt(end,10);
			if(start <0){
				start += n;
			}
			if(end > n){
				end = n;
			}
			if(end < 0){
				end += n;
			}
			for(var i= start;i<end;++i){
				ret[i-start] = node[i];
			}
		}
		return ret;
	}

**4、类型的判定**

一般来说就用Object.prototype.toString.call(要测定的对象如：Array)就可以了，结果是[object Array]。剩下的window对象，Document,Arguments,NodeList对象单独判定。

**5、domReady**

一般我们的js逻辑会写在window.onload回调中，以防DOM树还没建完就开始对节点进行操作，导致出错，但是有时页面图片等资源过多，window.onload就迟迟不能触发。

domReady其实是一种名为"DOMContentLoaded"事件的别称。此处的domReady指做过兼容性处理的domReady机制。策略：（1）对于支持DOMContentLoaded的用DOMContentLoaded事件。（2）对于旧版本IE用Hack。

Hack原理如下：
	
	1、由于在DOM未建完之前调用元素doScoll抛出错误。所以就不断setTimeout去try{window.document.documentElement.doScoll('left');}直到成功之后就代表load成功，执行用户回调。
	2、用户也有可能在执行这个js的时候dom已经ready了，所以只需判断window.document.onreadystatechange事件回调里window.readyState是否为"complete"就可以了。不需要上面的操作了。
	3、也可以通过创建一个script标签并添加deffer属性，因为指定deffer属性的script会在dom树建立完后触发，所以可以在这个脚本load成功之时执行用户的业务逻辑。
	4、总结一下mass框架的思路就是：先判断Document.readyState没有complete没有就去绑定监听document.onreadystatechange的事件继续等待complete，或者如果符合W3C标准，就去监听DOMContentLoaded。
	这个是要等待异步回调的，那这段时间可以去判断一下scroll函数。总之，用各种各样的方式去试探，一旦成功，立即改变状态，使得其他判断方式失效，最后触发用户回调！实在不行还有window.onload!

**6、无冲突处理**

即多库共存。使用时先引入其他框架，然后jQuery，之后执行noConflict()

	noConflict原理：
	//先将别人的命名空间保存在一个内部变量里
	var _jQuery = window.jQuery,_$=window.$;
	jQuery.extend({
		noConflict:function(deep){
			//jQuery执行完了，调用noConflict时，再把人家对象还回去。
			window.$ = _$;
			if(deep)
				window.jQuery = _jQuery;
			//返回jQuery对象就可以随便赋值给其他变量了，如wlf =$.noConflict();
			//wlf('#id')这样使用就可以啦。
				return jQuery;
		}
	});

>第二章：模块加载系统

**1、AMD规范**
即异步模块定义。

	define('xxx',['aaa','bbb'],function(aaa,bbb){})
	xxx为模块ID，第二个参数为模块依赖。

**2、加载器所在路径的探知**	

1、一般用node.src,在IE下js的绝对路径返回不准确。需要做兼容性处理，用node.readyState === 'interactive'判断，即过滤几个js脚本，找出正在当前解析的那个node。

2、还可以用Error对象。当发生脚本错误的时候，一般会给出错误的事件对象，利用这个对象可以得知出错的js的文件路径等信息，如下：

	try{
		a.b.c();//故意报错
	}catch(e){
		if(e.fileName){//firefox
			return e.fileName;
		}else if(e.sourceURL){//safafi
			return e.souceURL;
		}
	}

**3、require方法**

	作用是当依赖加载完毕，执行 用户回调。
	1）、取得依赖 列表的第一个ID，转换为URL。
	2）、没加载过的模块加载。
	3）、创建script标签，绑定onerror，onload，onreadychange等事件判定加载成功与否，然后添加src并插入DOM树，开始下载。虽然获取js文件是异步的，但并不在此等待下载成功，让浏览器并发下载。
	4）、将模块的URL，依赖列表等构建成一个对象，放到检测队列中，在上面的事件触发时进行检测。检测列表去检测大家都下载好没有，没下载好就一直检测。

seajs是CMD规范，在执行该模块的时候才加载依赖，它的require原理：
	
	debug版的：
	define(factory) 中的 factory 函数。原理是，当将 js 文件加载回来后，执行的仅是 define(factory) 函数，factory 则还未执行。执行 define 时，会扫描 factory.toString() 方法，得到当前模块依赖的文件，下载好好，再执行 factory 函数， 这样就实现了提前并行加载，但执行时看起来是同步的。
	如果是打包过后的，就无所谓了，但也是require时去解析的，不提前解析js。
	看了下源码（v3.0.0），流程大概是这样的：
	use->把入口模块entry给依赖->load->fetch（下载文件）->如果entry还没触发onload->entry callback,删entry->exec依赖的模块：factory!!!factory里传的require只需执行exec，不需要下载文件了。

**define方法**

require只加载define过的模块。


>第三章：语言模块

字符串、数组、数值、函数、日期的扩展与修复。

>第四章：浏览器嗅探与特征侦测

判定浏览器、事件的支持侦测、样式的支持侦测

>第五章：类工厂

各个框架（P.js、JS.Class、simple-inheritance、def.js）的类工厂实现及ES5属性描述符新特性（Object.create等，另见后续总结blog）

>第六章：选择器引擎

这里主要是jQuery的Sizzle选择器的实现。

**几个概念：**

	种子集：第一次得到的元素集合叫种子集，Sizzle是从右到左筛选元素，所以种子集里有一部分符合条件的元素。如果从左到右筛选，则种子集相当于一个据点，然后去找它的子节点或兄弟
	结果集：选择器引擎最终返回的元素集合。
	过滤集：选取一组元素后，之后每一个步骤要处理的元素集合都可以称为过滤集。
	选择器群组：一个选择符被关联选择器“,”划分成的每一个大分组。
	选择器组：一个选择器群组被关系选择器划分的第一个小分组。

**涉及的通用函数**

1、isXML

2、contains：判断参数1是否包含参数2。可以用compareDocumentPosition或者cantains两种方式实现。

	DOMElement.contains(DOMNode)
	NodeA.compareDocumentPosition(NodeB)：
	这个方法是 DOM Level 3 specification 的一部分，允许你确定2个 DOM Node 之间的相互位置。这个方法比 .contains() 强大。这个方法的一个可能应用是排序 DOM Node 成一个详细精确的顺序。使用这个方法你可以确定关于一个元素位置的一连串的信息。所有的这些信息将返回一个比特码（Bit，比特，亦称二进制位）。
	 Bits          Number        Meaning 
	000000         0              元素一致 
	000001         1              节点在不同的文档（或者一个在文档之外） 
	000010         2              节点 B 在节点 A 之前 
	000100         4              节点 A 在节点 B 之前 
	001000         8              节点 B 包含节点 A 
	010000         16             节点 A 包含节点 B 
	100000         32             浏览器的私有使用
	通过API返回的数字来确定关系，如返回20，则是16+4，即前面的节点A包含另一个节点 B（+16） 且在B之前（+4）

3、节点排序与去重

1）compareBoundaryPoints(how,sourceRange):
该方法将比较当前范围的边界点和指定的sourceRange的边界点，并返回一个值，声明它们在源文档中的相对位置。参数how指定了比较两个范围的哪个边界点。该参数的合法值和它们的含义如下：

	Range.START_TO_START - 比较两个 Range 节点的开始点
	Range.END_TO_END - 比较两个 Range 节点的结束点
	Range.START_TO_END - 用 sourceRange 的开始点与当前范围的结束点比较
	Range.END_TO_START - 用 sourceRange 的结束点与当前范围的开始点比较

如果当前范围的指定边界点位于sourceRange指定的边界点之前，则返回-1。如果指定的两个边界点相同，则返回0。如果当前范围的边界点位于sourceRange指定的边界点之后，则返回 1。

2）兼容旧版本标准浏览器与XML文档：不断向上获取要比较顺序的节点的父节点，直到HTML元素连同最初的那个节点，组成两个数组，然后每次去数组最后的元素进行比较，如果相同就去掉，一直到去掉不相同为止，最后用nextSibling结束。

3）通过sort排序：
	
	1、取出元素节点的sourceIndex的值，转换成一个String对象。
	2、将元素节点附在String对象上。
	3、用String对象组成数组。
	4、用原生的sort对String对象进行排序
	5、在排好序的String数组中，按序取出元素节点，即可得到排好序的结果集。
	p.s 因为sourceIndex，所以这种方法只能用在IE或早期Opera
	高级浏览器可以直接nodes.sort(比较顺序结果)排序

参见：[js对象数组按属性快速排序](http://www.cnblogs.com/jkisjk/archive/2011/01/28/array_quickly_sortby.html)

4、切割器

用正则表达式将选择器分割成一个数组，如：['div','>','#aaa'，'div']

5、属性选择器对于空白字符、子元素匹配伪类的匹配策略

**Sizzle 引擎**

结构：

	1、Sizzle主函数，包含选择符的分割，内部循环调用主查找函数，主过滤函数，最后是去重过滤。
	2、其他辅助函数，如uniqueSort、matches、matchesSelector
	3、Sizzle.find主查找函数
	4、Sizzle.filter主过滤函数
	5、Sizzle.selectors包含各种匹配用的正则、过滤用的正则、分解用过的正则、预处理函数、过滤函数。
	6、根据浏览器的特征设计makeArray、sortOrder、cantains等方法。
	7、根据浏览器的特症重写Sizzle.selectors中的部分查找函数、过滤函数、查找次序。
	8、若浏览器支持querySelectorAll，用它重写Sizzle，将原来的Sizzle作为后备方案包裹在新的Sizzle里。
	9、其他辅助函数如：isXML，posProgress。

分割选择符，切成选择器组与关系选择器的集合。先通过最右边的选择器得到元素集合作为种子集，复制一分出来作为映射集。关系选择器会让引擎去选取其兄长或者父亲，把这些元素替换到种子集对等的位置上，然后到下一个选择器组时，纯过滤操作。主过滤函数Sizzle.filter会调用Sizzle.selectors下的过滤函数对这些元素进行检测，将不符合的元素替换为false。

总结：先分组如：div.aaa span.bbb > a.ccc 这样的可以分为3组，中间的“” ,> ,+,~是关系选择器，代表了要过滤出父节点，兄弟节点等。分组之后拿到a.ccc,先sizzle.find，找到ccc的集合，然后sizzle.filter过滤出a.ccc,这样最终会有几个组的集合（例子里是三个）。复制一份a.ccc作为一个基本的种子集，通过目录查找它跟前面两组组合的包含关系，去除多余的元素，就完成了。

>第七章 节点模块

**1、节点的创建**

document.createElement，innerHTML，insertAdjacentHTML，createElement

**2、节点的插入、复制、移除、innerHTML、innerText、outerHTML的处理**

**3、奇葩元素节点**

video标签可绑定事件：
	
	{
		'loadstart':'客户端开始请求数据',
	    'progress':'客户端正在请求数据',
	    'suspend':'延迟下载',
	    'abort':'客户端主动终止下载（不是因为错误引起）',
	    'error':'请求数据时遇到错误',
	    'stalled':'网速失速',
	    'play':'开始播放时',
	    'pause':'暂停',
	    'loadedmetadata':'成功获取资源长度',
	    'loadeddata':'loadeddata',
	    'waiting':'等待数据，并非错误',
	    'playing':'开始回放',
	    'canplay':'可以播放，但中途可能因为加载而暂停',
	    'canplaythrough':'可以播放，歌曲全部加载完毕',
	    'seeking':'寻找中',
	    'timeupdate':'播放时间改变',
	    'ended':'播放结束',
	    'ratechange':'播放速率改变',
	    'durationchange':'资源长度改变',
	    'volumechange':'音量改变'
	 }

>第八章：数据缓存系统

4种形态：
	
	属性标记法
	数组索引法
	valueOf 重写法
	weakMap 关联法

>第九章：样式模块

>第十章：属性模块

1、如何区分固有属性和自定义属性

	function isAttribute(attr,host){
		//有些属性是特殊元素才有的，需要用到第二个参数
		host = host || document.createElement('div');
		return host.getAttribute(attr) === null && host[attr] === void 0
	}

2、如何判断浏览器是否区分固有属性与自定义属性

IE6、7不区分固有属性与自定义属性

	var el = document.createElement('div');
	el.setAttribute('className','t');
	console.log(el.className !== 't');

>第十一章 事件系统

**1、onXXX绑定方式的缺陷**
	
	1、对DOM3新增事件或FF某些私有实现，无法支持。
	2、只允许元素每次绑定一个回调，重复绑定会冲掉之前的绑定。
	3、在IE下回调没有参数，在其他浏览器下回调的第一个参数是事件对象。
	4、只能在冒泡阶段可用。

**2、attachEvent的缺陷** 

	1、IE下只支持微软系的事件、DOM3事件一概不能用。
	2、IE下attachEvent回调中的this不是指向被绑定元素，而是windows！
	3、IE下同时绑定多个回调时，回调不是按照绑定顺序依次触发的。
	4、IE下的event事件对象与W3C的存在太多差异了，有的无法对上号，比如currentTarget。
	5、IE还是只支持冒泡阶段

**3、addEventListener的缺陷**

	1、新事件非常不稳定
	2、Firefox既不支持focusin、focus事件，也不支持DOMFocusIn、DOMFocusOut，直接现在也不愿意用mousewheel代替DOMMouseScroll。Chrome不支持mouseenter，mouseleave.
	3、私有前缀标识
	4、第三个参数userCapture，第四个参数是FF专有实现，允许跨文档监听事件。第五个参数只存在于Flash语言的同名方法中。在Flash下，addEventListener的第四个参数用于设置该回调执行顺序，数字大的优先执行。第五个参数用于指定对侦听器函数的引用是弱引用还是正常引用。
	5、事件对象的成员不稳定。
	6、标准浏览器没办法模拟像IE6~8的propertychange事件。

**4、jQuery事件**

事件代理手动模拟冒泡事件。

**5、滚轮事件修复**

mousewheel用于取得滚动距离的属性名为event.wheelDelta，往上滚动一圈为120.往下滚动一圈为-120

**6、oninput事件的兼容性处理**

Firefox、Chrome、IE9，IE10 均支持 oninput 事件，此外所有版本的 IE 均支持 onpropertychange 事件。

oninput 事件在用户输入、退格（backspace）、删除（delete）、剪切（ctrl + x）、粘贴（ctrl + v）及鼠标剪切与粘贴时触发（在 IE9 中只在输入、粘贴、鼠标粘贴时触发）。

onpropertychange 事件在用户输入、退格（backspace）、删除（delete）、剪切（ctrl + x）、粘贴（ctrl + v）及鼠标剪切与粘贴时触发（在 IE9 中只在输入、粘贴、鼠标粘贴时触发）（仅 IE 支持）。

backspace、delete 两个按键的 keyCode 分别为 8、46。

oncut 事件在粘贴（ctrl + v）、鼠标粘贴时触发。

	 if (W3C && DOC.documentMode !== 9) { //IE10+, W3C
        element.addEventListener("input", updateVModel)
        data.rollback = function() {
            element.removeEventListener("input", updateVModel)
        }
    } else {
		var eventArr = ["keyup", "paste", "cut", "change"]
        removeFn = function(e) {
            var key = e.keyCode
            if (key === 91 || (15 < key && key < 19) || (37 <= key && key <= 40))
                return
            updateVModel()
        }
        avalon.each(eventArr, function(i, name) {
            element.attachEvent("on" + name, removeFn)
        });
        data.rollback = function() {
            avalon.each(eventArr, function(i, name) {
                element.detachEvent("on" + name, removeFn)
            })
        }
    }

>第十二章 异步处理

**1、setTimeout与setInterval**

	1、如果回调的执行时间大于间隔时间，那么浏览器会继续执行他们，导致真正的间隔时间会比原来的大一点儿。
	2、他们存在一个最小的始终间隔，在IE6~8中为15.6ms，后来精准到10ms，IE10为4ms，其他浏览器相仿。
	可以加载一个不存在的图片如：img.src="data:,foo"来触发它的onerror事件，从而做到立即执行回调，用这种方式来改善IE下间隔时间过长的问题。
	3、关于零秒延迟，此回调将会放到一个能立即执行的时段进行触发。JS代码大体上是自顶向下执行，中间穿插有关DOM渲染，事件回应等异步代码，它们将组成一个队列，而零秒延迟将会插队。
	4、不写第二个参数，系统自动匹配时间。
	5、标准浏览器与IE10，都支持额外参数，从第三个参数起，作为回调的传参。
	setTimeout(function(){
		alert([].slice.call(arguments));
	},10,1,2,3)
	IE6~9可以用以下代码模拟
	(function(overrideFn){
		window.setTimeout = overrideFn(window.setTimeout)
		window.setInterval = overrideFn(window.setInterval)
	})(function(originalFun){
		return function(code,delay){
			var args=[].slice.call(arguments,2);
			return originalFun(function(){
				if(typeof code ==='string'){
					eval(code);
				}else{
					code.apply(this,args)
				}
			},delay);
		}
	})
	6、setTimeout方法的时间参数若为极端数（如负数，0，极大的正数），最新浏览器会立即执行。

**2、deffered**

基本思路是:在一个deffered执行完，创建一个新的deffered对象，接力下去。

所以异步都执行完再执行回调的这种方法的原理是每个异步回调都执行一个计数方法，当数量跟异步回调个数一样了，说明都异步完成了，于是触发结束方法。

**3、yield**

>第13章：数据交互系统

>第14章：动画引擎

**缓动公式**

**requestAnimationFrame、CSS3transition、CSS3animation**

实现动画的思路：建立一个时间轴的概念，我们再里面插入关键帧，两个关键帧之间就是补间动画，时间轴本质是个数组。它驱动setInterval执行动画，当中计算每一帧之间需要移动、改变的属性。可以调节每帧变化量的大小做出不同的加速度效果，缓动公式可提供现成的几种类型。CSS3效果的动画，就是需要动画的时候加上CSS3样式类，完成删除（已提供动画完成等事件）。

>第15章、插件化

>第16章、MVVM

**属性变化的监听**：观察者模式

**ViewModel**：每个允许监听的属性都配一个方法，方法内部包含这个属性的set，get方法、以及发出通知与收集订阅者的代码。

**綁定**：链接View和ViewModel的纽带。即标签里的指令。

**监控数组与子模板**

>其他知识点：

1、oncompositionstart 、oncompositionend

	复合事件(composition event)是DOM3级事件中新添加的一类事件，用于处理IME的输入序列。IME（Input Method Editor，输入法编辑器）可以让用户输入在物理键盘上找不到的字符。复合事件就是针对检测和处理这种输入而设计的。
	（1）compositionstart：在IME的文本复合系统打开时触发，表示要开始输入了。
	（2）compositionupdate：在向输入字段中插入新字符时触发。
	（3）compositionend：在IME的文本复合系统关闭时触发，表示返回正常键盘的输入状态。

2、"top,left".replace( /[^, ]+/g , function(name) {});//foreach

3、 数组转化为对象的key，值为val或默认为1
		
		var oneObject = function (array, val) {
	        if (typeof array === "string") {
	            array = array.match(rword) || []
	        }
	        var result = {},
                value = val !== void 0 ? val : 1
	        for (var i = 0, n = array.length; i < n; i++) {
	            result[array[i]] = value
	        }
	        return result
	    }

4、Object.getOwnPropertyNames

	返回对象自己的属性的名称。一个对象的自己的属性是指直接对该对象定义的属性，而不是从该对象的原型继承的属性。对象的属性包括字段（对象）和函数。
	和Object.keys的区别，Object.keys只适用于可枚举的属性，而Object.getOwnPropertyNames返回对象自动的全部属性名称

5、hidefocus outline

	hideFocus即隐藏聚焦，具有使对象聚焦失效的功能，其功能相当于： 
	onFocus="this.blur()" 
	它的值是一个布尔值，如hideFocus=true。也可省略赋值直接写hideFocus。 
	你给的代码如果没有hideFocus,那么鼠标点击该超链接，则外面出现一个虚线框，即为聚焦。而使用了hideFocus则不会有虚线框。
	在IE下，需要在标签 a 的结构中加入 hidefocus="true" 属性。
	如：<a href="#" hidefocus="true" title="XX">没有虚线框</a>
	而在FF等浏览器中则相对比较容易，直接给标签 a 定义样式 outline:none; 就可以了，即a {outline:none;}


4、[遍历DOM(NodeIterator和TreeWalker的使用](http://www.cnblogs.com/blog-zwei1989/archive/2012/11/28/2792420.html)
	
5、withCredentials xhr

	默认情况下，标准的跨域请求是不会发送cookie等用户认证凭据的，XMLHttpRequest 2的一个重要改进就是提供了对授信请求访问的支持。
	var xhr = new XMLHttpRequest();
	xhr.open('GET', 'http://www.xxx.com/api');
	xhr.withCredentials = true;
	xhr.onload = onLoadHandler;
	xhr.send();
	请求头，此时已经带上了cookie：
	设置服务端响应头：Access-Control-Allow-Credentials: true
	有一点需要注意，设置了withCredentials为true的请求中会包含远程域的所有cookie，但这些cookie仍然遵循同源策略，所以是访问不了这些cookie的。

6、try{}catch(e){e.stack}//取错误信息,可以用于得知所在的是哪个脚本。

7、[].slice.call(arguments)=》Array.prototype.slice.call(arguments)。能将具有length属性的对象转成数组。

8、/xyz/.test(function(){xyz;})//用于判定函数的toString是否能暴露里面的实现。

9、
	
	var global = (function(){
		return (0, eval)('this'); 
	})();	
	will correctly evaluate to the global object even in strict mode. In non-strict mode the value of this is the global object but in strict mode it is undefined. The expression (1, eval)('this') will always be the global object. The reason for this involves the rules around indirect verses direct eval. Direct calls to eval has the scope of the caller and the string this would evaluate to the value of this in the closure. Indirect evals evaluate in global scope as if they were executed inside a function in the global scope. Since that function is not itself a strict-mode function the global object is passed in as this and then the expression 'this' evaluates to the global object. The expression (1, eval) is just a fancy way to force the eval to be indirect and return the global object.
	A1: (1, eval)('this') is not the same as eval('this') because of the special rules around indirect verse direct calls to eval.
	A2: The original works in strict mode, the modified versions do not.
	As for why the body is this, that is the argument to eval(), that is the code to be executed as a string. It will return the this in the global scope, which is always the global object.
	(0,xx)返回xx。不直接写是为了暴露在全局scope下执行。‘this’严格模式下可以当做全局变量传进去。


参考：[return this || (0,eval)('this');](http://stackoverflow.com/questions/9107240/1-evalthis-vs-evalthis-in-javascript),[(1,eval)('this') vs eval('this') in JavaScript?](http://stackoverflow.com/questions/14119988/return-this-0-evalthis)

10、animation-play-state: running | paused 检索或设置对象动画的状态。
	
11、[WeakMap对象](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/WeakMap):简单的键/值映射.但键只能是对象值,不可以是原始值。

12、window.onpageshow

从浏览器缓存里显示的页面会触发这个方法。如：如果提交表单，完成之后点击后退按钮，浏览器尤其是FF32.0+会完美保存用户的表单状态，要是不想重置表单的话，就可以增加这个方法，因为后退是从浏览器的缓存机制里取得，会触发这个方法。


参考资料：

[contains和compareDocumentPosition 方法来确定是否HTML节点间的关系](http://blog.csdn.net/huajian2008/article/details/3960343)

[js对象数组按属性快速排序](http://www.cnblogs.com/jkisjk/archive/2011/01/28/array_quickly_sortby.html)

[遍历DOM(NodeIterator和TreeWalker的使用](http://www.cnblogs.com/blog-zwei1989/archive/2012/11/28/2792420.html)

[jquery ready方法实现原理 内部原理](http://blog.csdn.net/dyllove98/article/details/9237805)

[onreadystatechange 事件](http://www.cnblogs.com/haogj/archive/2013/01/15/2861950.html)

[return this || (0,eval)('this');](http://stackoverflow.com/questions/9107240/1-evalthis-vs-evalthis-in-javascript)

[(1,eval)('this') vs eval('this') in JavaScript?](http://stackoverflow.com/questions/14119988/return-this-0-evalthis)

[animation-play-state](http://ecd.tencent.com/css3/html/animation/animation-play-state.html)

[WeakMap](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/WeakMap)