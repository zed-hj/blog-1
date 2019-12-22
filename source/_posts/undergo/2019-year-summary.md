
---
title: 步入正轨的一年

date: 2019/12/20 23:39:40

categories:
    - 经历
---

![](http://cdn.talei.me/image/%E7%A7%92%E9%80%9F%E4%BA%94%E5%8E%98%E7%B1%B3/1238.jpg)

<!--more-->

<iframe frameborder="no" border="0" marginwidth="0" marginheight="0" width=330 height=86 src="//music.163.com/outchain/player?type=2&id=477342444&auto=1&height=66"></iframe>

## 概述

经历了近半年的 **小黑屋式** 开发, 中午晚上吃完饭射飞镖活动一下后接着干。晚上最早九点, 最晚第二天凌晨七点下班, 下午三点上班, 晚上十一点再下班, 一天下班两次...痛并快乐着。

BOSS 专注产品原型、架构、以及给我们梳理业务, 技术评审。我们则根据原型设计领域模型, 在脑海里走一遍业务流程看是否走得通, 大致没问题时再一起评审, 暴露出细节问题再调整修复。领域模型评审通过后再跟前端协商接口, 写好接口文档后跟前端核实一遍, 评审走通流程没问题后前后端按照接口文档各自并行开发(前端 Mock 数据), 有条不絮的进行开发。 

在此期间负责跨境电商系统中的商品域, 经历了千万级数据量的索引优化, 也因为缺乏经验没在一开始建立好唯一约束而导致出现两百万重复数据... 以及在错误的时机初始化消息消费者而拖垮系统的正常启动等巨坑。学习到了产品从原型到落地的整条流程, 以及如何在人员不足的情况下充分利用开源项目(工具)敏捷开发, 用 [Jhipster](https://www.jhipster.tech/) 脚手架搭建微服务基础设施([uaa](https://www.jhipster.tech/using-uaa/), [gateway](https://www.jhipster.tech/api-gateway/)), 再用 [JDL](https://start.jhipster.tech/jdl-studio/) 设计领域模型, 生成 CURD 代码,用 Docker + K8s 进行项目部署, 配合阿里云云效使用, 减轻部署成本, 让开发可以专注自身负责领域的业务逻辑实现, 使产品得以快速落地。这段时间使我得到了非常大的锤炼(BOSS说相当于阿里的日常), 一直按部就班的每天早晨打开 Tower 规划今天的任务, 遇到复杂业务时画图梳理, 语雀 写 API 文档, 跟 BOSS 开会梳理新需求, 跟前端联调接口。

长时间的加班加点终使我疲惫了, 于是在七月份来到了广州开始新的工作...

新的工作人员更少, 在之前的工作里架构是 BOSS 搭好(Jhipster通用微服务架构), K8s、阿里云流水线都搭好, 我只需要创建 feature 分支, 实现功能, Review 代码, 点一下运行流水线即可, 一切环境都是搭好的, 而现在则需要我自己来动手搭建了, 跟前公司一样使用了 Spring Cloud(Jhipster) + (Docker + 阿里云K8s) 这套技术栈。

![](http://cdn.talei.me/blog/undergo/2019/pipeline-list.jpg)

![](http://cdn.talei.me/blog/undergo/2019/pipeline-detailed.jpg)

![](http://cdn.talei.me/blog/undergo/2019/k8s-home.jpg)

为了减轻频繁部署带来的工作量, 必须借助 K8s 的力量, 但是自己搭建又需要处理很多异常情况, 为了进度采用了阿里云的K8s服务(这也导致了我对K8s的搭建不熟悉)。

k8s 查看日志很方便, 不管有多少个节点, 日志都能合并查看, 下面是一段检索 uaa 异常的命令 :

```shell
kubectl logs -f --tail=200 -lk8s-app=uaa | -C20 'Exception'
```


## 收获

### 智六

大厂的研发流程, 从出原型评审到部署上线、基本的数据库优化、接口耗时排查、性能优化...

技术方面有：Jhipster、JDL、Spring Cloud、Reids、RocketMQ、Docker、K8s...

业务逻辑梳理清楚再动手写代码, easy

### 舱舱

Jhipster 搭建通用技术架构。

阿里云云效、流水线、托管版K8s搭建 CI/CD 环境。

前端项目容器化部署，静态资源托管到七牛云、阿里云OSS。

Mysql、Redis、RocketMQ 全部使用阿里云产品(我们没有运维)。

以此为基石在此之上专注业务逻辑实现。

## 不足

- 感觉很多东西还是只停留在会用的层面上, 没有去深究。
- 生活上太单调了
- 并没有坚持锻炼身体...

## 展望

静下心来沉淀, 编码只是很小一部分, 要注重设计思想, hold住复杂的业务场景。

纯粹的编码是非常枯燥的, 要从中享受到设计的乐趣。

- 沉淀技术深度
- 学习 React 及 Antd UI 库, 达到能快速开发管理系统的程度
- 学习胶水语言 Python
- 学习常见的加密算法
- 培养一两个兴趣爱好, 如吉他、摄影等
- 年年都有的 #锻炼身体~


阅读:

- [高性能Mysql](https://book.douban.com/subject/23008813/)
- [Kubernetes in Action](https://book.douban.com/subject/30418855/)
- [领域驱动设计](https://book.douban.com/subject/5344973/)
- [中国近代史(徐中约)(Hong Kong)](https://drive.google.com/drive/folders/0B7o-Es_WO2PbazJSVWhkY3Y4Q3M)
