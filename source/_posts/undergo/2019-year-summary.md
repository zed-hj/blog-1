
---
title: 步入正轨的一年

date: 2019/12/31 19:39:40

categories:
    - 经历
---

![](http://cdn.talei.me/image/%E7%A7%92%E9%80%9F%E4%BA%94%E5%8E%98%E7%B1%B3/1238.jpg)

<!--more-->

<iframe frameborder="no" border="0" marginwidth="0" marginheight="0" width=330 height=86 src="//music.163.com/outchain/player?type=2&id=477342444&auto=1&height=66"></iframe>

## 概述

### One

今年上半年长沙几乎都在下雨... 天气又冷又潮湿... 非常烦人。

从18年年底开始筹备新项目, 要求春节放假前上线初版(勉强上线, 年后`for true: fix bug`)... 非常自然的进入了平均 9106 工作制。 由于我在此之前没有做过微服务系统, 此时面临新的技术栈、业务知识..., 一切都需要学习, 我就像块海棉一样, 疯狂的吸收着技术上、业务(领域)上的知识。

工作上强度很大, 经常是饭点吃完饭回办公室休息一小会后就接着干, 经常晚上十一二点都在梳理业务, 偶尔是一两点才回家。春节前发版那天下班了两次, 通宵工作到了隔天的早晨七点, 然后睡到下午三点又去公司上班, 晚上十一点再下班... 虽然工作强度大, 但我丝毫没有觉得不适, 因为这段时间的锤炼确确实实的使我升华了。

从 BOSS 身上学到了很多:

- **技术:** Jhipster、JDL、Spring Cloud、Reids、RocketMQ、Docker、K8s...

- **设计思想:** 领域设计的思想, 一个人(Team)负责一个领域, 只需要了解消化自身领域的知识, 和在必要的时候跟其他领域(Team)协调交互, 从开发人员分配时就自然形成了低耦合高内聚。

**敏捷开发流程** 

根据原型设计领域模型, 脑海中走一遍业务流程(走到某一步某张表有没有数据,数据呈什么状态), 然后拿出领域模型一起技术评审, BOSS 把关提出疑问和建议。过了之后再跟前端确定接口, 写好接口文档后跟前端核实, 然后前后端并行开发。

开发时创建特性分支去开发新功能, 合并到开发分支时要先让 BOSS 把关 Review。发布部署时先发一个节点, 验证之后再发其他节点, 一旦出现问题马上回滚...

经历了千万级数据量的索引优化, 也因为粗心大意没在一开始建立好唯一约束而导致出现两百多万重复数据, 导致删数据时需要依赖中间件存储ID... 以及在错误的时机初始化消息消费者而拖垮系统的正常启动等巨坑, 踩了不少坑。

一直按部就班的每天早晨打开 Tower 规划今天的任务, 遇到复杂业务时画图梳理, 语雀 写 API 文档, 跟 BOSS 开会讨论新需求梳理业务流程, 跟前端联调接口...

### Two

在六月份我回学校拿了毕业证, 为了寻求更好的发展来到了广州, 七月份在广州开始了新工作。 

新的工作人员更少, 在之前的工作里架构是 BOSS 搭好(Jhipster通用微服务架构), K8s、阿里云流水线都搭好, 说到底我只需要创建 feature 分支, 实现功能, Review 代码, 点一下运行流水线自动部署即可, 一切环境都是搭好的, 而现在... 啥都没有了。

业务人员比较少, 系统比较复杂, 我习惯性的引入了跟前公司一样的技术栈: Spring Cloud + Docker + K8s。

![](http://cdn.talei.me/blog/undergo/2019/pipeline-list.jpg)

![](http://cdn.talei.me/blog/undergo/2019/pipeline-detailed.jpg)

![](http://cdn.talei.me/blog/undergo/2019/k8s-home.jpg)

使用了阿里云的K8s服务, 我们人员比较少, 没有运维, 目前只需要会用就好了。

## 收获

- Spring Cloud、Spring Security(OAuth2)、Spring Data JPA、Websocket(Stomp)
- Redis、RocketMq、MongoDB
- Docker、K8s

阿里云云效、流水线、托管版K8s搭建 CI/CD 环境

前端容器化部署, 静态资源托管到七牛云、阿里云OSS

SpringSecurity OAuth2 集成短信验证码、微信扫码登录...

## 不足

- 感觉很多东西还是只停留在会用的层面上, 没有去深究。
- 并不能很好的区分工作和生活
- 生活上太单调了
- 并没有坚持锻炼身体...

## 展望

### 技术

静下心来沉淀, 编码只是很小一部分, 要注重设计思想, hold住复杂的业务场景。

纯粹的编码是非常枯燥的, 要从中享受到设计的乐趣。

- 沉淀技术深度
- ~~学习 React 及 Antd UI 库~~ 深入学习 Vue 及 Element UI 库, 达到能快速开发管理系统的程度
- 学习胶水语言 Python
- 学习常见的加密算法

### 生活

- 性格太急躁, 心理活动都写在脸上了, 必须跟周围几位大佬学习学习
- 分清楚工作和生活, 拿钱办好事即可
- 培养一两个兴趣爱好, 如吉他、摄影等
- 年年都有的 #锻炼身体~

### 阅读

- [高性能Mysql](https://book.douban.com/subject/23008813/)
- [Kubernetes in Action](https://book.douban.com/subject/30418855/)
- [领域驱动设计](https://book.douban.com/subject/5344973/)
- [中国近代史(徐中约)(Hong Kong)](https://drive.google.com/drive/folders/0B7o-Es_WO2PbazJSVWhkY3Y4Q3M)
