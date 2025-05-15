---
layout: post
title: 记一个压力测试中遇到的问题
date: 2025-05-15 10:13
description: performance-testing
tags: skynet network
categories: tech
---

## 前置信息

    最近在做游戏服压测的事情，发生一起诡异的案件：当瞬间请求达到5000左右时，会出现部分连接被关掉的情况。
    压测工具的实现大概是这样的：有一个总控 skynet 进程接收控制请求，总控进程控制自己或者从属的所有 slave 进程。比如创建机器人请求，则总控在 slave 进程中分布式创建机器人，每个机器人创建一个服务。因此，所有的机器人并行执行命令，以达到模拟玩家行为的目的。

## 历程

    首先怀疑压测工具的实现。调查后发现连接请求确实发出去，大部分请求顺利进入后续行为，小部分请求(约10%)连接很快被close。
    接着怀疑被压测服的承载能力，发现压测服中被 close 的连接在被压测服中并没有被成功open。（补充信息：并非 EMFILE 问题，该问题会有日志提示且已经被解决。）于是怀疑连接在底层就已经被默默抛弃。在 lua 代码中，可能的原因是 maxconnection 设置问题，经检查还没到上线。于是怀疑 c 层问题，进入调试模式，断点可能的无提示的close代码。但是都没有能进入断点。
    接着抓包验证目前获得的信息是否真实，发现大部分连接能够三次握手，少部分连接在发起握手请求并被通知连接建立完毕后（收到 [syn,ack]），被 fin 。

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/net_pack_1.png" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    一次连接的通信过程
</div>
    接下来怀疑是操作系统问题，同时发现内存已经接近满了，也许是 oom，但没有崩溃，且 oom 日志也告诉我这是无稽之谈。这时同事提到，可能是 somaxconn 设置问题，一切才明朗起来。

## backlog 与 somaxconn

    在linux的实现中，tcp 的三次握手用到了两个队列，一个是接收 sync 时（ sync 队列，也称半连接队列），一个是接收 ack 时（ accpet 队列，也称全连接队列），sync队列的长度由 net.ipv4.tcp_max_syn_backlog 控制；accpet队列长度由 min(backlog, net.core.somaxconn) 控制，其中 backlog 由用户 listen 时传入。当队列堆满，即机器消化连接的速度不够时，就会出现连接被丢弃的情形。
    可以通过设置 somaxconn、增加负载进程或者客户端重试/延缓发送机制解决或绕开该问题。
