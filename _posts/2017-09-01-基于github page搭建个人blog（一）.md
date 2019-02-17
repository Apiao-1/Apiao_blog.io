---
layout: post
title: 基于github-page搭建个人blog（概要）
date: 2017-09-01
categories: blog
tags: [技术类]
description: "在这一系列博文中，我将具体为大家介绍如何利用github-page搭建个人博客，虽然网上类似的文章很多，可很多内容过于老旧，以致在搭建的时候碰了不少灰，让我决定要把正确的时新的内容带给大家。"
header-img: "img/article3.jpg"
---

因为最近实习结束外加各种杂七杂八的琐事，又要忙着准备开学第一周的数模比赛（嗯，通俗来讲就是很忙：），系列的开篇博文就先说说这个博客的大致架构吧，大家可以先有个概念要做哪些事。


#### 域名
经大多数乎友推荐选择了从[goDaddy](https://sg.godaddy.com/zh?isc=gennbacn29&countrview=1&currencytype=CNY&mkwid=WFSMCUdy&cvosrc=ppc.baidu)购买的顶级域名，包年大概也就￥58的样子，算是比较不错的选择（选购时记得搜索优惠码），不过后来在找服务器的时候发现国内一些公司提供的域名价格会更优惠，比如[腾讯云](https://dnspod.qcloud.com/act/seckill?utm_source=portal&utm_medium=recommend&utm_campaign=recmd3&from=doufu3)、[阿里云](https://wanwang.aliyun.com/?spm=5176.8142029.735711.56.23896dfa2q0NIq)、[百度云](https://cloud.baidu.com/index.html?track=cp:npinzhuan|pf:pc|pp:left|ci:|pu:495)，大家在选购时可以多加比较，域名种类有很多种，比较贵的就是.com/.cn/.net的顶级域名，具体选择可以看个人喜好。

#### 服务器
托管在github-pages上，免费，300M空间，作为个人博客来说足够了，同时作为轻量级的博客系统，配置相对而言较简洁。缺点也比较明显，由于使用Jekyll模板系统，只能发布静态页，无法动态加载，所以如果页面元素过多，加载速度会较慢。

#### 前端
[GoodThingList/GoodJekyllBlogList.md at master · cnfeat/GoodThingList](GoodThingList/GoodJekyllBlogList.md at master · cnfeat/GoodThingList)上有各种风格的模板，有一定前端基础可以自己在模板基础上加以改动。

#### 后台
安装好Jekyll环境后，可直接在本地加以编辑并测试，需注意Liquid语言模板的写法

#### 数据库
github-page无需外接数据库

#### 博客引擎
Jekyll

#### 版本迭代
Git，通过GitHub Desk可在改动后直接发布commit

#### 流量统计
Google Analytics（需翻墙）

#### 图床
//推荐七牛云，10G免费流量

驳回上述⬆️

推荐腾讯云，依然有免费的流量

七牛云恶心的地方是需要绑一个通过备案的域名，它一开始会给你发一个测试域名，所以可以正常使用，但过段时间过期了图床就崩了，把上面的图片重新找回来还废了不少功夫

#### SSL证书加密
因为用了七牛云的图床，顺便就在里边签了TrustAsia的证书，这里还是推荐用 Letsencrypt，知名度更高，不过后来签发下来之后发现github-page无法发布证书，如果想要加密还是需要用 CloudFlare，做一次代理，这个具体操作之后会详细说明（通过 CloudFlare其实不需要额外去注册证书）
![SSL证书.png](https://apiao-1258505467.cos.ap-chengdu.myqcloud.com/blog_pic/SSL%E8%AF%81%E4%B9%A6.png)

---
完成上述内容你的博客就飞速地搭起来啦，总计成本就是买了域名，其余的用免费的均可实现。最花时间的部分其实在于模板的改动，需要花时间去理解模板作者的代码并加以更改，当然好的模板一般在配置文件_config里即可完成大部分元素的更改，但像我这样爱折腾的人总是不能挑出最心仪的模板，在现有基础上更改模板的css、html也就难以避免了。










