
---
title: "使用acme申请https免费证书"
date: 2018-12-20T21:38:52+08:00
lastmod: 2018-12-28T21:41:52+08:00
draft: false
tags: ["https",]
categories: ["docs",]
author: "Dean"
menu: "home"
weight: 50
---  
### 前言

---

​	上次写了一篇[https证书相关的笔记整理](https://juejin.im/post/5be2ab1a51882516d85b40c3),个人觉得有些地方欠妥,这次介绍一个更方便更简单更🐂一点的工具——acme.sh.上次使用的工具是certbot.

两者对比,acme.sh有如下优点:

- acme.sh会自动设置好定时任务.自动更新证书.certbot的更新需要手动设置cron.
- acme.sh可以使用域名解析商提供的 api 自动添加 txt 记录完成验证.简单、高效.
- 安装简单,没有环境依赖.卸载同样简单.

### 安装

---

```shell
# 建议使用root安装,
curl  https://get.acme.sh | sh 
```

该命令会把acme安装在~/.acme.sh路径下,并为你创建一个检查更新证书的定时任务.

因为该工具有个参数reloadcmd可以预设命令,可能会reload nginx服务器等.建议使用root安装.

```shell
#查看定时任务
crontab -l
23 0 * * * "/root/.acme.sh"/acme.sh --cron --home "/root/.acme.sh" > /dev/null
# --home --cron参数解释可用~/.acme.sh/acme.sh -h查看,解释如下
  --home                   Specifies the home dir for acme.sh.指定acme的路径
  --cron                   Run cron job to renew all the certs.定时检查更新证书
```



### 签发证书(Issue a cert)

---

签发证书前,需要验证域名的所有权,[acme支持多种方式验证](https://github.com/Neilpang/acme.sh/wiki/How-to-issue-a-cert),建议使用http和dns验证.

我的个人域名解析使用的是cloudflare的free套餐,且acme文档写明支持cloudflare.所以选择dns验证.

依照[acme文档-how-to-use-dns-api](https://github.com/Neilpang/acme.sh/wiki/dnsapi),

1.登录cloudflare官网获取API key.

```shell
#cloudflare-->个人配置--->API key - Global API Key - view API key
# 拿到API key后,设置如下环境变量.
export CF_Key="sdfsdfsdfljlbjkljlkjsdfoiwje"
export CF_Email="xxxx@sss.com"
```

接下来就可以愉快的申请证书了.

**申请证书命令如下:**

```shell
acme.sh --issue -d glc.im -d *.glc.im --dns dns_cf \ 
--key-file "/etc/nginx/ssl/glc.im/xxxx.key" \ 
--fullchain-file "/etc/nginx/ssl/fullchain.cer" \ 
--reloadcmd "service nginx reload"
```

- glc.im /*.glc.im换成自己的域名
- dns_cf是对应的cloudflare,其他域名解析服务商请参照https://github.com/Neilpang/acme.sh/wiki/dnsapi
- key-file/fullchain-fil 签发证书后,acme会帮你把证书复制到该路径下
- reloadcmd 因为是root安装的acme 此命令可以帮助我重载nginx

### 更多内容

---
- **acme:** https://github.com/Neilpang/acme.sh/wiki

- 如何使githu page跳转到个人域名?

- 如何强制跳转https?