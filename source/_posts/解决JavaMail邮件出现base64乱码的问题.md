---
title: 解决JavaMail邮件出现base64乱码的问题
date: 2018-04-26 17:25:36
tags: [Java,JavaMail,base64]
categories: [技术,学习,Java]
---

### 问题

本机进行邮件发送没出现问题，但是放到某些linux环境中则出现邮件内容是现实base编码的问题：

```
------=_Part_3_2039075268.1524573999534
Content-Type: text/html; charset=utf-8
Content-Transfer-Encoding: base64
 
PHA+5ZGK6K2m5L+h5oGv5aaC5LiL77yaPGJyPumhueebriAmbmJzcDsgJm5ic3A7ICZuYnNwOyAm
bmJzcDsgJm5ic3A7ICZuYnNwOyAmbmJzcDvkv6Hmga88YnI+5ZGK6K2m5a6e5L2TIO+8mndpbmRv
d3Nf57qz566h5rWL6K+VPGJyPuWRiuitpuWQjeensCDvvJromZrmi5/mnLrnm67lvZXkuI3kuIDo
h7Q8YnI+5ZGK6K2mSUQgJm5ic3A7ICZuYnNwOyDvvJpjbS1hbGFybS0xMDxicj7lkYrorabmj4/o
v7Ag77ya6Jma5ouf5py6ICB3aW5kb3dzX+e6s+euoea1i+ivleW3sue7j+iiq+enu+iHs+aWh+S7
tuWkuSB2bSE8YnI+5ZGK6K2m57qn5YirIO+8muS4pemHjeWRiuitpjxicj7lkYrorabml7bpl7Qg
77yaMjAxOC0wNC0yNCAyMDo0NToyOTxicj48YnI+5aaC5pyJ55aR6Zeu77yM6K+35LiO57O757uf
566h55CG5ZGY6IGU57O744CCPC9wPg==
------=_Part_3_2039075268.1524573999534--
```



一开始以为是编码没设置好，但是本地测试或是在某些linux环境中测试都没问题。

<!--more-->



### 问题解决

后来通过网上寻找答案，发现其实是环境中引入了无用的包，使得包冲突导致，连接如下：

[javamail 发送邮件 内容乱码问题的解决](http://yanghengtao.iteye.com/blog/1924276)

[spring mail 发送邮件，没有主题，没有收件人，显示乱码问题](http://jeyke.iteye.com/blog/1441548)

引入的多余jar包如下：
![image](https://note.youdao.com/yws/public/resource/92d9bb9b2b48a12e9ca04faaf8368cd6/xmlnote/67976E0A5CDE450E8E981492CCE25620/9342)

*解决方法*：将此jar包删除即可！