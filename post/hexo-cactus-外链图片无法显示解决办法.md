---
title: hexo cactus 外链图片无法显示解决办法
date: 2023-06-29 18:01:51
tags:
- hexo
---



## 修复外链

cactus 主题目录结构



<img src="https://gitee.com/gitduk/typora-image/raw/master/image/image-20230629180457583.png" alt="image-20230629180457583"  />



在 <u>/layout/_partial/head.ejs</u> 文件中加一行代码

````
<meta name="referrer" content="no-referrer" />
````



![image-20230629181145010](https://gitee.com/gitduk/typora-image/raw/master/image/image-20230629181145010.png)



## 参考链接

- https://zhuanlan.zhihu.com/p/577256660
