---
title: "【Obsidian】win+ipad+andriod利用坚果云三端同步"
author: "Tweakzx"
description: 
date: 2023-02-04T15:07:34+08:00
image: https://obsidian.md/images/screenshot-1.0-hero-combo.png
math: 
license: 
hidden: false
comments: true
draft: false
categories: Obsidian
tags: 
    - 笔记
---

# 利用坚果云实现Obsidian的多端同步

## win端下载插件Remotely Save

- 启用obsidian的第三方插件
- 浏览社区插件市场并安装Remotely Save
- 安装完成后如图所示

![image-20230204151516303](https://cdn.jsdelivr.net/gh/Tweakzx/ImageHost@main/img/202302041515869.png)

## 注册坚果云

- 进入账户信息->安全选项->第三方应用管理
- 点击添加应用，配置名称生成密码，名称自己设置，可以是Obsidian等等
- 记得复制生成的密码

## win端同步设置

- 打开Remotely Save的设置页面
- 选择远程服务 Webdav
- 填写
  - 坚果云的服务器地址：https://dav.jianguoyun.com/dav/
  - 自己的账户名
  - 刚刚生成的密码
- 检查可否连接
- 重启插件生效
- 单击侧边栏的↺同步文件

![image-20230204152135645](https://cdn.jsdelivr.net/gh/Tweakzx/ImageHost@main/img/202302041521784.png)

## ipad端与Android端同步设置

- 创建与win端的同步仓库**同名**的仓库
- 进入设置安装Remotely Save插件

- 打开电脑端设置->第三方插件->Remotely Save->导入导出部分设置
- 在导出侧边点击生成QR code
- 打开ipad的相机 / Android手机的扫一扫功能
- 扫描这个QRcode导入配置信息即可

## 自定义设置以及其他

- 建议打开ipad和安装端的启动后同步
  - 步骤：基本设置->启动后运行一次->启动后第1s运行一次
  - 我个人电脑端为主要进行读写操作，其他端为读操作
- 可以打开同步配置文件夹
  - 步骤：进阶设置->同步配置文件夹->打开
  - 打开后会同步.obsidian文件夹，Obsidian的设置也会在各个平台同步，不过我觉得没有必要
- 安卓端扫码比较费劲， 但是多试几次也可以成功