---
title: 无障碍使用Google搜索
updated: 2017/11/04 21:38:25
categories: 
- Tools
tags: [工具]
---

### 简单介绍
可能很多人和笔者一样，自己有购买过VPN，访问某些网站和应用的时候会把VPN打开， 如果VPN有流量限制，访问完之后还要将VPN关掉。如果你用的VPN不稳定，可能还需要多次更换主机；而笔者在自己日常工作中发现，几乎有90%的需求都是用Google搜索，有时候想迅速搜到想要的结果，以解决纠缠了很久的问题。这种时候，如果能够无障碍的快速访问Google，那简直是再高效不过的了。今天介绍一种无需VPN快速使用Google搜索的方法。
[https://nova.rambler.ru](https://nova.rambler.ru) 是一个俄罗斯的搜索引擎网站，它内部将搜索操作重定向到Google去完成的，因此，它更像是一个代理，其搜索结果也就和Google的一模一样。借助这个网站，我们可以做一些简单的定制，在不需要VPN的情况下，简单快速的使用Google搜索。

<!-- more -->

### 如何使用
1. 打开浏览器，访问 [https://nova.rambler.ru/search?query=test&lang=en](https://nova.rambler.ru/search?query=test&lang=en) 进入Rambler的搜索页面，这里，我们用“test”作为搜索关键字。
2. Chrome浏览器下在页面右下角，点击按钮 **Search Settings** 打开搜索定制对话框，可以选择单页显示搜索结果排序方法、结果条数，以及搜索过滤政策；  
    {% asset_img search-settings.png 搜索设置 %}	
3. 设置为浏览器默认搜索引擎（以Chrome为例，其他浏览器类似）：进入搜索引擎设置窗口（Chrome中访问[chrome://settings/searchEngines](chrome://settings/searchEngines) 即可），点击按钮 **添加** 来新增一个搜索引擎，在 **网址** 一栏输入该URL：[https://nova.rambler.ru/search?query=%s&lang=en](https://nova.rambler.ru/search?query=%s&lang=en) ；将新增的搜索引擎设置为默认搜索引擎。这样，在浏览器输入拦输入关键字就可以完成搜索了。  
    {% asset_img set-default-searchengine.png 配置默认搜索引擎 %}	

### 更进一步
##### 搜索结果页面去广告
由于该搜索引擎结果页面中带有广告，且结果条目预览宽度较窄，Chrome浏览器里，我们可以安装Stylish插件，然后做如下简单的CSS定制，即可去除广告，且让搜索结果预览更多的文字。
	google_ads_frame1 { display: none; }
	.l-main, .l-null { width: 950px; }
	.l-aside { display: none; }
这是去除广告之后的页面效果：  
    {% asset_img search-results-wo-ads.png 去除广告 %}	

##### 手机浏览器上使用Google搜索
手机端浏览器，可以访问[https://m.search.rambler.ru/search?query=test](https://m.search.rambler.ru/search?query=test)来进行搜索，但是貌似手机端的语言请求参数 `lang=en` 并不生效；支持自定义搜索引擎的浏览器，可以使用[https://m.search.rambler.ru/search?query=%s](https://m.search.rambler.ru/search?query=test)作为默认索索URL；
