---
layout: post
title: High Sierra 版本下Xcode 9 无法注释问题（command + / 失效）
date: 2018-08-14
categories: blog
tags: [技术类]
description: "将mac os版本更新至high Sierra 之后遇见Xcode的代码注释功能失效的问题，具体表现为快捷键失效，且不能从下图所示Editor - Structure处手动添加"
header-img: "img/article3.jpg"
---

将mac os版本更新至High Sierra 之后遇见Xcode的代码注释功能失效的问题，具体表现为**快捷键失效，且不能从下图所示Editor - Structure处手动添加**。

（此前红框部分为灰色不可用状态 ，此处解决后已恢复正常）

![Editor - Structure](http://ovfvfmquv.bkt.clouddn.com/image-20180814104056976.png)



网上找了好多资料，解决方案如下：

1.如果系统版本较低，可采用如下方法（网上大部分是这样说的）

```
sudo /usr/libexec/xpccachectl
```

然而这个方法是历史的产物，在High Sierra的系统中已经没有了这个脚本。

笔者在执行时便会遇见 /usr/libexec/xpccachectl: command not found错误（当然，都没有这个脚本了怎么能执行  (~﹏~) ）

2.Rename is good（非常神奇的方法，如其所说）

打开Finder 中的Application文件夹，将其中的Xcode重命名，比如重命名成Acode，确定更改后打开，如果问题解决则大功告成，最后重新把名字改回来即可。

> 参见Stack Overflow（https://stackoverflow.com/questions/38847530/cant-comment-selection）

#### 总结

归纳一下解决该问题的步骤，由简到难排列为：

```
1.重启Xcode
2.重启电脑
3.尝试 sudo /usr/libexec/xpccachectl
4.重命名Xcode
5.重新下载Xcode
6.升级系统
```

运气好的话到第四步即可解决此Bug，然而本人试了都不行，最后做到了第五步，含泪重装了Xcode解决（5.3个G用校园网下了好久 (..•˘_˘•..)）

<br>

<br>

改编自：

[Stack Overflow中的相关问题1]: https://stackoverflow.com/questions/47101437/
[Stack Overflow中的相关问题2]: https://stackoverflow.com/questions/38847530/cant-comment-selection
[原鸣清的简书]: https://www.jianshu.com/p/4494fb458edb
[okerivy的简书]: https://www.jianshu.com/p/e4b8aff61250

