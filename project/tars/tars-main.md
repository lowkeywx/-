# 项目简介

- [tars](https://github.com/TarsCloud/Tars.gi) 是一个基于tars和tup协议的远程调用框架（libservant）。且在远程调用库的基础上衍生出一系列的服务器，从而实现了服务发布和治理的微服务框架。
- 本次阅读及分析的源码为c++ 版本的libServant库， 即远程调用部分的代码。
  
# 源码分析的主要结构

分析主体分成两个部分：
- [客户端部分](./tars-client.md)
- [服务端部分](./tars-server.mn)
  
每个部分都会从主要类及功能、初始化、驱动体系和消息走向为主题进行分析。