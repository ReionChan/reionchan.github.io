---
layout: post
title: 基于 DDD & COLA 的分布式微服务实践
categories: Java DDD COLA MicroService Arch
excerpt: DDD 业务建模，COLA 工程实现
image: https://raw.githubusercontent.com/ReionChan/PhotoRepo/refs/heads/master/arch/DDD_COLA.jpg
description: 采用 DDD 思想建模，使用 COLA 整洁架构的分布式微服务样例工程
keywords: microservice DDD COLA arch
licences: apache, cc
author: Reion Chan
repo: msia-cola-ddd-arch
---

![DDD-COLA](https://raw.githubusercontent.com/ReionChan/PhotoRepo/refs/heads/master/arch/DDD_COLA.jpg)

## MVC 现状

&emsp;&emsp;在业务相对简单、变动不频繁时，MVC 凭借其简单清晰的视图与数据模型分离特性，能够使开发团队快速的上手完成既定的业务开发。然而，随着市场需求不断变化，业务通常也需要不断调整变化来适应市场变化，这就对系统的架构设计提出了更高的要求。

&emsp;&emsp;MVC 分层架构中，对业务的逻辑处理、技术手段实现等等都只被粗粒度的封装在 Service 层。随着业务系统的不断变化及扩张，这将导致该层的臃肿。同时，由于业务逻辑和解决业务问题的技术手段相互掺杂，很难适应现今不断变化的业务需求。

&emsp;&emsp;在此背景下，一种能够将业务逻辑与技术基础设施分离，以业务为导向能够快速响应业务变更的架构孕育而生，这就是本文接下来要详细介绍的基于业务领域的设计。

## DDD 介绍

### 概念

&emsp;&emsp;DDD （Domain-Drive Design）领域驱动设计是 Eric Evans 在 2004 年提出的一种软件设计方法和理念，其主要的思想是，利用确定的业务模型来指导业务与应用的设计和实现。主张开发人员与业务人员持续地沟通和模型的持续迭代式演化，以保证业务模型与代码实现的一致性，从而实现有效管理业务复杂度，优化软件设计的目的。

<center><img src="https://raw.githubusercontent.com/ReionChan/PhotoRepo/refs/heads/master/arch/concept-map-hd.png"/> </center>

<center>DDD 基本概念图（来源 <a ref="https://domain-driven-design.org">domain-driven-design</a> ) </center>

&emsp;&emsp;DDD 把所需解决的问题拆解成不同领域，使用模型驱动设计方法对领域进行建模，从而得到可以被用来进行软件工程落地实现的领域模型。此外，它指导软件设计者使用分层架构的逻辑进行软件设计，最终得到目标软件。

### 分层设计

&emsp;&emsp;关于 DDD 请参考 DDD 概念 [^1] 网站的详细介绍，本文着重关注其在软件设计架构上的指导建议。DDD 推荐采用分层架构的形式将不同的功能实现拆分到不同层级进行实现，每个层级包含独立的职责，多个层次协同合作来提供完整的业务功能。DDD 采用四层的分层模型，分别为：接入层、应用层、领域层、基础设施层。

<center><img src="https://raw.githubusercontent.com/ReionChan/PhotoRepo/refs/heads/master/arch/ddd-4-layer.png"/> </center>

<center>DDD 四层架构图（来源 <a ref="https://juejin.cn/post/7256795117675216952#heading-6">掘金-京东云开发者</a> ) </center>

* 接入层

  &emsp;**&emsp;负责系统的输入输出，只关心沟通协议，不关心业务相关数据的校验。**它对应用数据透明，只关心数据格式而非内容，在大部分单体系统中，接入层往往被框架实现。例如：Spring Boot 框架中就将接入层的 HTTP 协议进行封装，开发人员只关注基于 HTTP 协议的接口设计而非协议本身。而在分布式系统中，接入层通常由网关实现。

* 应用层

  &emsp;&emsp;**负责组织业务场景，编排业务，隔离场景对领域层的差异。**它将统筹编排不同领域服务来达成一个完整的业务，而具体业务逻辑由各个领域服务层实现，它只负责接入层与领域层之间对象的转换、业务一致性的事务控制及业务权限相关的处理。

* 领域层

  &emsp;&emsp;**负责具体的业务逻辑、规则，为应用层提供无差别的服务能力。**它是实际处理业务的地方，对应用层提供无差别的服务与能力。它不关心业务场景，只关心当前上下文的业务完整性，故而强一致性的事务被当做聚合纳入本层。该层做业务规则的验证、数据权限控制。

* 基础设施层

  &emsp;&emsp;**负责具体的技术实现，例如：存储、通知、外部系统适配集成等，它对业务保持透明。**它不仅包含常用组件，还包括对组件、外部系统的适配。

### 分层演进

&emsp;&emsp;如上图所示的 DDD 松散分层架构中，DDD 的资源库用来获取或持久化聚合，资源库与领域层的聚合为强关联，理应放在领域层。然而，由于资源库的接口的实现需要依赖于基础设施层的持久化实现，势必会将基础设施层所选择的具体持久化技术引入领域层，导致领域层与技术实现细节强绑定，引入了其职责之外的不稳定因素。若将资源库接口的实现放入基础设施层，那么底层的基础设施层将依赖其上层的领域层，这种依赖倒转是违背了分层架构的原则。[^2]

&emsp;&emsp;打破分层架构原则，利用依赖倒置原则来确保领域层的独立与稳定，不但没有违背 DDD 的主旨思想，相反它使得 DDD 以业务为模型来驱动指导应用设计的思想更加存粹。系统中所有其他层的实现都以领域为中心，基于这种领域层依赖倒转原则诞生了很多不同的架构设计，例如：六边形架构、COLA 架构。**它们都提倡以业务为核心，解耦外部依赖，分离业务复杂度和技术复杂度。**

&emsp;&emsp;像六边形架构、COLA 架构这种独立于框架、方便测试、隔离不同技术实现的架构都被归纳为整洁架构。本文接下来着重介绍 COLA 架构在实践落地 DDD 设计思想中的各种技术细节。

## COLA 介绍

### 概念

&emsp;&emsp;COLA（Clean Object-Oriented and Layered Architecture）[^3] 整洁面向对象分层架构，由阿里巴巴主导开源。它通过定义良好的应用结构、提供最佳应用架构实践、治理应用复杂度，从而降低系统熵值。目前已经发展到 5.0 版本，其定义的应用架构如下图：

<center><img src="https://raw.githubusercontent.com/ReionChan/PhotoRepo/refs/heads/master/arch/cola-arch.png"/> </center>

<center>COLA 架构图（来源 <a ref="https://github.com/alibaba/COLA">COLA Github</a> ) </center>

### 分层设计

* 适配器层（Adapter 层）

  &emsp;&emsp;**负责不同接入客户端的适配，概念类似 DDD 的接入层。** 它是领域的对外的数据输入口，对不同终端的不同协议的适配，此外外系统的消息接收、任务调度等触发对本领域的调用都被认做事一种数据输入而放在此层。

* 应用层（App 层）

  &emsp;&emsp;**负责处理外部请求，隔离不同业务场景差异，组织编排不同领域服务完成完整业务。**它参考 DDD 关于事件风暴的建模手段，采用   CQRS 原则，对外部的请求抽象成命令、查询两种类型，分配给执行器处理。此层的服务可以跨域编排不同的领域层服务，完成一个应用级业务逻辑，同时层次是开放的，可以饶过领域层直接访问基础设施层。

* 领域层（Domain 层）

  &emsp;&emsp;**负责具体领域上下文封装，形成领域服务能力，支撑应用层业务实现。**该层兼容 DDD 中有关领域层的要求与功能，包括不限于：领域服务、领域事件、策略及规格。同时定义领域网关接口，利用依赖倒置原则结构对基础设施层的依赖。

* 基础设施层（Infrastructure 层）

  &emsp;&emsp;**负责领域网关接口及其他技术细节实现。**它将业务领域对外部的依赖进行隔离解耦。

* 客户端 SDK 开发套件（Client SDK）

  &emsp;&emsp;**负责暴露领域对外部客户端的 API 及传输对象。**

### 通用组件

&emsp;&emsp;COLA 架构提供了一些非常有用的通用组件，合理利用这些模式组件能进一步简化代码，使得业务代码更整洁清晰，提升研发效率。例如：使用 `cola-component-catchlog-starter` AOP 组件来减少多余的异常处理模版代码、使用 `cola-component-statemachine` 状态机组件管控限制聚合实体的状态流转等等。

## DDD & COLA 实践



## 参考资料

[^1]:[DDD 概念参考](https://domain-driven-design.org/zh/ddd-concept-reference.html)
[^2]:[DDD  架构为什么应该首选六边形架构？](https://juejin.cn/post/7256795117675216952)
[^3]: [COLA](https://github.com/alibaba/COLA)
