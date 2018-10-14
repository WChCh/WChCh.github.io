---
title: 使用easyUI组件combogrid面板相关问题
date: 2017-11-15 17:08:14
tags: [js,EasyUI,Combo,Datagrid,Combogrid,问题,前端]
categories: [技术,学习,前端,js,EasyUI,Combo,Datagrid,Combogrid,问题]
---

#### 问题

使用combogrid组件时，弹出的面板宽度总是与数据框的长度一样，并不是想官方demo那样弹出理想的宽度。各种属性各种设置也没解决问题，网上找了好久也没有找到答案。

#### 解决

后来继续翻官方demo，发现了一个属性，试了一下成功了。属性如下：

```
panelMinWidth : '50%',
```



还得加入`fix:true,`属性，使得表格大小与面板一致
设置值为：`50%`。

#### 下来面板中加入输入框

因为combogrid继承combo以及datagrid，因此可是使用combo以及datagrid的特性。通过”toolbar”属性可以引入输入框形成组合查询。

*注意问题*：在Firefox浏览器中可能会出现数据框加载显示效果问题，可以先默认隐藏输入框，再在列表初始化函数中通过js显示该输入框。

#### 设置默认选中第一行

```javascript
onLoadSuccess:function(){
	//默认选中第一行      
	$('#billingReport_InvoiceTermComboBoxId').combogrid('grid').datagrid('selectRow',0);;
	},
```