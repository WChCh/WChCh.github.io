---
title: EasyUI layer.tips位置出错问题解决
date: 2017-11-16 17:11:40
tags: [js,EasyUI,Combo,Datagrid,Combogrid,问题,前端]
categories: [技术,学习,前端,js,EasyUI,Combo,Datagrid,Combogrid,问题]
---

#### EasyUI layer.tips位置出错问题解决

当在划出窗口或是弹出面板中使用layer.tips时，老是出现tips位置错误的问题。感觉是给layer.tips提供的id位置应该不是当前窗口或是面板的上的id。

在同事的建议下直接把当前对象的`this`参数传进来单做id，这样就解决问题了。代码片段如下：

```
operation =  '<a href="javascript:;" class="btn btn-link btn-xs" onmouseover="showTestTooltip(\''
				+ tipStr + "','" + row.id +'\',this);" onmouseout="hideTestTooltip(this);">'
				+ "tip" + '</a>';
		return operation;
```

<!--more-->

```javascript
showTestTooltip : function(tipStr, id, thiz) {
		testTooltip = layer.tips(
                '<font size="2">'+ tipStr + '</font>',
                thiz, {
                    tips : [ 3, '#3595CC' ],
                    time : 0,
                    area : 'auto',
                    maxWidth : 500,
                    zIndex:999999
                });
    },
```

由于本人前端水平有限，不知道为啥这样，但至少能够解决问题，还请大家指教！！！