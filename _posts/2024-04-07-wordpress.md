---
layout: post
category: computer
title: "WordPress 搭建个人博客指南"
author: ZhangKe
date:   2024-04-07 21:43:30 +0800
---

# WordPress 搭建个人博客指南

WordPress 是一套免费开源的网站管理框架，目前全球 43% 的网站都是使用 WordPress 搭建，具备丰富的插件和主题模版，而且使用简单，新手可以快速搭建一个完备的网站，今天介绍如何使用 WordPress 搭建一个个人博客网站。

# 域名

首先需要购买一个域名，去哪里买都行，阿里云可以使用支付宝付款，国外有些平台也可以使用支付宝，不过大部分都是 PayPal 和信用卡。

价格根据不同的域名差别很大，我在阿里云买的 [zhangke.space](http://zhangke.space) 十年才 188 块钱，四舍五入等于不要钱，同样的域名不同平台差别也很大，可以货比三家。

# 服务器

服务器这个也根据自己需求买就行了，想折腾的可以自己买 VPS 然后一点点开始搭建，想省事的也可以直接买虚拟主机，因为 WordPress 体量很大，用户群体非常多，所以很多商家都提供针对 WordPress 的预制服务，可以直接买个 WordPress 虚拟主机，买来就能用，毫无技术含量，基本上付钱就行，剩下的全帮你搞好了。

我买的 SiteGround 家的虚拟主机，最便宜的 3 刀一个月。

SiteGround 购买前会先问你有没有域名，没有的话他们也提供购买域名服务，可以直接捆绑购买，不过我没这么选，如果已经买好域名到这里直接选择 Existing Domain 然后输入自己的域名即可。

然后 SiteGround 购买时时没有 PayPal 入口的，需要联系客服要 PayPal 连接才行（不懂为什么要这样）。

# 域名解析

服务器买好之后你会得到一个服务器的 IP 地址，以 SiteGround 家的为例，买好之后进入该服务器的 DashBoard 能看到各种信息。

![](/assets/img/post/wordpress/Untitled.png)

红框部分就是 IP 地址，拿到这个 IP 地址之后需要去购买域名的平台配置一下。

以阿里云为例，进入域名控制台，点击对应的域名右侧的解析按钮，添加记录即可。

![](/assets/img/post/wordpress/Untitled1.png)

这样域名解析就配置好了。

然后，我们上面 SiteGround 截图中 IP 地址红框位置的下面还有个 Name Servers，这个是域名服务器，我们还需要将这个 DNS 服务器也配置到阿里云后台。

点击域名右侧的管理按钮，点击左侧工具栏中的 DNS 管理 → DNS 修改，把 SiteGround 提供的两个 DNS 服务器添加进去就行了。

# 安装/配置 WordPress

安装过程挺简单的，买好之后进入该服务器的控制台，左边有 WordPress 工具栏，进入点击安装，然后按需选择就行了，安装完成后选择左边的 WordPress → Install&Manage，点击这个按钮进入 WordPress 管理页面。

![](/assets/img/post/wordpress/Untitled2.png)

这个时候 WordPress 就已经安装好了，管理页面是一个固定的路径，以后可以这么进入。

[https://yourdomain.com/wp-admin/admin.php](https://yourdomain.com/wp-admin/admin.php?page=siteground-dashboard.php)

你上面安装 WordPress 流程中会要求你创建 WordPress 的账户密码，此时用这个账号登陆就可以进入管理后台了，管理后台可以添加内容、改变主题、统计信息等。

## 常规设置

首先可以设置一下时区和语言，日期格式等等，按照自己的喜好设置。

![](/assets/img/post/wordpress/Untitled3.png)

网站的 Logo，标题之类的也都是在这里设置的。

## 主题

然后打开左侧的工具栏中的外观挑选主题，WordPress 并不单纯是给博客用的，它支持很多类型的网站，因此这里的主题种类也非常丰富，为了挑选方便，可以从这个网址打开博客主题的列表。

[https://wordpress.com/themes/filter/blog](https://wordpress.com/themes/filter/blog)

挑选好之后进入管理后台的主题页面搜索选好的名字，然后安装并激活就行了。

大部分主题都是可以自定义的，但是自定义程度根据不同的主题也有所不同。

进入博客主页，已登陆状态下顶部会有一栏管理员才能看到的管理顶部栏。

![](/assets/img/post/wordpress/Untitled4.png)

点击编辑站点进入编辑页面，这个页面略微复杂，自己玩一玩大概就能玩明白要这么设置，也不是很好介绍。里面功能很强大，比如你想在页面某个区域添加最近发布文章的列表，就可以在左上角加号的按钮中找到对应的列表添加。

## 添加文章

WordPress 有一些插件可以从原本的 WordPress 中导入文章，或者从 RSS 导入文章，但是没有从 GitHubPage 导入文章的插件，RSS 导入的格式也都错误，只能手动导入了。

点击管理后台左侧工具栏中的文章→新增文章进入添加文章页面，然后编辑就行了，编辑完设置时间、分类等信息然后发布即可。

文章是跟主题无关的，即使切换了主题文章仍然存在。

## 配置 SSL

不配置 SSL 的话打开你的网站会提示连接不安全，虚拟主机一般都提供了直接设置的方法。

首先进入虚拟主机的管理后台（不是 WordPress 的管理后台），点击左侧工具栏的 Security，选择其中的 SSL Manager，下面的选择对应的域名和 Let's Encrypt，再点击 GET，等一会就好了。

![](/assets/img/post/wordpress/Untitled5.png)

然后进入 WordPress 管理后台，点击左侧工具栏的设置，下面红框部分改成 https。

![](/assets/img/post/wordpress/Untitled6.png)

然后就好了，此时再打开自己的网站就不会提示不安全了。

## 数据统计

数据统计 SiteGround 本身的管理后台能看到一部分，工具栏中的 Statistics 就是数据统计，但不是很全，如果感觉不够用还可以去 WordPress 中安装 WP Statistics 插件。

直接在插件中搜索 WP Statistics 安装启用，然后左侧工具栏中会多一个统计的按钮，点开就能看到各种统计信息了。

## RSS/Atom

WordPress 是默认支持 RSS 和 Atom 协议的，并且两个都支持，不需要额外设置，只需要在首页地址后面加上 rss 或者 atom 路径就行了。

---

以上就是 WordPress 新手入门教程，按照上面的步骤就可以搭建一个非常完备且漂亮的网站出来了。也欢迎大家关注我的网站或者订阅 RSS：[https://zhangke.space/](https://zhangke.space/) 。