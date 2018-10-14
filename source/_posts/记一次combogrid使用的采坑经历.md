---
title: 记一次combogrid使用的采坑经历
date: 2017-11-27 17:15:00
tags: [js,EasyUI,Combo,Datagrid,Combogrid,问题,前端]
categories: [技术,学习,前端,js,EasyUI,Combo,Datagrid,Combogrid,问题]
---

#### 缘由

项目需要combogrid来实现下拉列表，并且实现select下拉框刷选下拉列表内容的功能。查了官方文档，好像没有介绍combogrid添加toolbar的例子，但是看到combogrid继承了combo和datagrid，那好，应该是可以使用。

#### 问题

combogrid成功添加了toolbar实现了相关功能。但是问题来了。

1. 当我切换到别的页面，再切回来时，发现toolbar的select框选择无效了，筛选的参数是上一次最后操作的参数，只能刷新整个浏览器页面才能生效。
2. 当我切换到别的页面时，按F12输入`$("#tb")`竟然生效，id为tb的div竟然成了全局div。当输入`$("[id=tb]").length`是返回的结果不只是1，会随着切换次数的增加加1。
3. 当我连着初始化combogrid两次是，toolbar竟然出现了一模一样的两个。

为什么出现这种问题，由于本人前端水平有限，一时半会也找不出答案，只能简单暴力地解决问题！

#### 解决方案

1. 初始化两次combogrid，使得出现两处toolbar。如果只是初始化一次那么toolbar只是出现一次，但是`[id=tb]").length`的长度还是会大于1，实际上可删除的id为tb的div只有一处，那么再删掉就没意义了。

2. 连续两次初始化完之后，接着加入以下逻辑

   ```javascript
      init : function() {
   	// 初始化账期控件，注意：必须初始化两次
   	initComboBoGrid();
   	initComboBoGrid();
   	$("[id=tb]").each(function(index,element){
   		if(0 != index){
   			element.remove();
   		}
   	});
   	$("#tb").show();
   }
   ```

没办法，只能简单暴力解决！