---
title: 一个基于github的远程团队协作开发模型
layout: post
categories: 开发
tags: GitHub 开发 模型
author: John He
---

* content
{:toc}

画这张图，一是想给新手们讲清楚git的一些基本操作，二是想描述一下我们现在正在用的基于github的协作开发模型。

"Local of $user"是某位成员的本地环境, 涉及到最基本的git操作, 无须赘述。
"Remote"是github上项目的布局情况, 包括主repo, 和每个成员各自fork的repo, 其中主repo的master随时处于shippable的状态, 通过github的release tag可以随时ship to customer.

![Image of GitHub Dev Model](https://raw.githubusercontent.com/johnhx/johnhx.github.io/master/img/github_base_model.png)