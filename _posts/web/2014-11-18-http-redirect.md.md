---
title: http重定向原理
layout: post
categories: [技术研究]
tags: [http, django]
description: http重定向原理,django示例
---

在web开发中，我们经常会碰到需要将页面重定向的需求。

譬如接入某些开放平台的登录，用户登录我们自己的网站， 然后我们需要将页面重定向到开放平台的登录页面，然后再由开放平台回调我们预先设置好的回调地址(callback_url)。

不论你是使用什么语言（java、 php、 python等等）， 这些语言内部通常会提供一些函数或这方法，帮你做重定向的逻辑， 通常情况下看起来是这样：

	#php
	<?
		Header("HTTP/1.1 303 See Other");
		Header("Location: $redirect_url");
	?> 

	#python django
	def Redirect():
		response = http.HttpResponse()
		response['status_code'] = 302
		response['Location'] = redirect_url
		return response

	#java
	HttpServletResponse.sendRedirect("redirect_url")

很多时候，我们都知道这样做能帮我们到达目的， 但是很少透过这些现象去了解事情的本质。下边重点介绍下重定向的原理。

我们知道，web开发中最常用的就是http协议了， http是一种请求-响应协议， 当浏览器收到http响应时， 会根据响应(http response)中的状态码采取相应的操作。最常见的几种状态码有：200， 404， 500。 而http中， 有一类重定向状态码： 3xx， 这类状态码指示需要用户代理(浏览器)采取更进一步的行为来完成请求， 主要看下301， 302这两个状态码的具体含义：

	301 Moved Permanently
	所请求的资源已经指定到一个新的永久URI， 后续的任何对该资源的引用都应该使用所
	返回的URL之一。（新url会自动缓存，后续所有对源url的请求都会直接重定向到新url）

	302 Found
	所请求的资源临时存在于不同的URI。由于重定向随时会改变， 客户端应该在将来的请求
	中继续使用原url。（通过302指示的url不会自动缓存，除非通过Cache-Control、Expire头部域指示）

所以， 上述重定向的场景， 在我们调用基础的函数/方法的时候， 它帮助设置了http response的响应码和头部Location域。 不同的是， 301是永久重定向， 302是临时重定向。
		  

-EOF-

