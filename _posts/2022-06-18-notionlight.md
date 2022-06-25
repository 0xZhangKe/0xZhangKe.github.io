---
layout: post
category: computer
title: "用 Compose 做了个开源的轻量级的 Notion 客户端 NotionLight，现已上架 Google Play。"
author: ZhangKe
date:   2022-06-18 23:15:03 +0800
---

迫于 Notion 的客户端比较慢，而且操作路径有点长，如果想当做快速笔记或者 TODO 来用还是不太够。

正好前段时间因为疫情在家待了三个月没出门，打算学学 Compose，所以顺便 :) 用 Notion 的 API 卷了个简单快速的客户端出来。

既然是当做快速笔记以及 TODO 来用，那内容的组织形式就是按照列表来的，会把 Notion 中的每个能被识别的内容块映射成列表中一个条目展开显示。每个 Notion 页面对应 NotionLight 中的一个 TAB。授权后自己选择将 Notion 中的对应的页面添加进来。

目前支持对内容快的添加、修改及删除操作，也支持 Android Shortcut 快速添加内容，可以说非常快速了。

主要技术栈：Compose+Kotlin+Coroutines+Retrofit.

Google Play: [https://play.google.com/store/apps/details?id=com.zhangke.notionlight](https://play.google.com/store/apps/details?id=com.zhangke.notionlight)

GitHub( 各位大佬点一点 star 吧): [https://github.com/0xZhangKe/NotionLight](https://github.com/0xZhangKe/NotionLight)

下面是截图：

![](/assets/img/post/notionlight/main_setting.png)

![](/assets/img/post/notionlight/op.png)
