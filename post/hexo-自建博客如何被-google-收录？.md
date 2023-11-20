---
title: hexo 自建博客如何被 google 收录？
date: 2023-07-04 15:30:24
tags:
- hexo
---



## 测试站点是否被 google 收录

测试方法：打开 google 主页，在搜索框键入 `site:blog.wukaige.com`

未被收录：

![image-20230704153416648](https://gitee.com/gitduk/typora-image/raw/master/image/image-20230704153416648.png)



被收录：
![image-20230704153457622](https://gitee.com/gitduk/typora-image/raw/master/image/image-20230704153457622.png)



## 安装 hexo-generator-sitemap 插件

hexo-generator-sitemap 可自动在根目录生成 sitemap.xml 文件

安装： `npm install hexo-generator-sitemap --save`

在 <u>_config.yaml</u> 文件中添加：

```
# https://github.com/hexojs/hexo-generator-sitemap
sitemap:
  path: 
    - sitemap.xml
    - sitemap.txt
  rel: false
  tags: true
  categories: true
```



## 提交站点 URL 到 Google Search Console

打开 [Google Search Console](https://www.google.com/webmasters/tools/home#utm_source=en-wmxmsg&utm_medium=wmxmsg&utm_campaign=bm&authuser=0) 并添加网站资源

![image-20230704153836019](https://gitee.com/gitduk/typora-image/raw/master/image/image-20230704153836019.png)



在下面添加你的博客 url 链接，并点击 CONTINUE，根据提示授权 Google 访问你的网站。

![image-20230704153924857](https://gitee.com/gitduk/typora-image/raw/master/image/image-20230704153924857.png)



这里填入自动生成的 **sitemap.xml** ，状态为 Success 代表添加成功。

![image-20230704154016411](https://gitee.com/gitduk/typora-image/raw/master/image/image-20230704154016411.png)



## 测试 Google Search Console 是否配置完成

在这里填入你的博客地址：

![image-20230704155013177](https://gitee.com/gitduk/typora-image/raw/master/image/image-20230704155013177.png)



下面是你的网站状态：

![image-20230704155054178](https://gitee.com/gitduk/typora-image/raw/master/image/image-20230704155054178.png)



URL is not on Google，问题不大，可能是刚刚收录还没来得及抓取你的网站。

点击右上角的 **TEST LIVE URL** 测试配置是否成功：

![image-20230704155337684](https://gitee.com/gitduk/typora-image/raw/master/image/image-20230704155337684.png)

出现这个页面代表配置没问题。



## 参考

- https://zhuanlan.zhihu.com/p/129022264

