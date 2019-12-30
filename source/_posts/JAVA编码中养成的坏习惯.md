---
title: JAVA编码中养成的坏习惯
author: meng@xy
avatar: https://i.loli.net/2019/12/30/ekxtSEDmwrsBqnU.png
authorLink: vertx.top
authorAbout: 研发工程师 | 角马科技（大连）有限公司
authorDesc: 指尖下的幽灵-异世界的伴侣-希望永绝之地-生命沉寂之岭
categories: 
  - 技术
  - 随想
date: 2019-12-12 22:16:01
comments: true
tags: 
 - web
 - 悦读
keywords: JAVA
description: JAVA编码中养成的坏习惯
photos: https://i.loli.net/2019/12/30/STyfbRIFCWnM6Vs.jpg
---
> 在设计及开发中，通常会由于各种原因，致使代码设计和开发中的无序。这里简单说一下碰到的几种简单情况。

### 1. 服务调用写到了循环中
``` java
    public List<MemberDetailDto> queryMembers(List<String> memberIds) {
        return memberIds.stream().map(memberId -> {
            MemberDetailDto memberDetailDto = memberService.queryOneMember(memberId);
            // ...
            return memberDetailDto;
        }).collect(Collectors.toList());
    }
```
以上这种写法并不算好，容易导致后续的性能问题，甚至如果后期在维护中，`queryOneMember`方法操作复杂话，可能导致各种异常问题发生。当然如果业务中 `memberIds` 的量是可控的，一般不会产生发生问题。

``` java
final String tenantId = "传入参数";
List<CardInfoDto> data = mbrMemberCards.stream().filter(Objects::nonNull).map(mbrMemberCard -> {
    CardInfoDto cardInfoDto = MbrMemberCardMapper.INSTANCE.mapToCardInfoDto(mbrMemberCard);
    TenantDto tenantDto = tenTenantApi.findOneTenant(tenantId);
    cardInfoDto.setLogo(tenantDto.getLogo());
    return cardInfoDto;
}).collect(Collectors.toList());
```
而如果是上面这种写法，那么就有点蠢了。`TenantDto tenantDto = tenTenantApi.findOneTenant(tenantId);` 这个在循环中每次都会调用，是没有意义的，可以在循环中调用一次即可。

通过代码历史查到，这些是同一个人写的。这就是一种代码编写的习惯，如果养成了在循环中调用服务的习惯，那么可以预见垃圾代码的成片出现。

### 2.服务领域划分不清楚
同样一个例子。
``` java
final String tenantId = "传入参数";
List<CardInfoDto> data = mbrMemberCards.stream().filter(Objects::nonNull).map(mbrMemberCard -> {
    CardInfoDto cardInfoDto = MbrMemberCardMapper.INSTANCE.mapToCardInfoDto(mbrMemberCard);
    TenantDto tenantDto = tenTenantApi.findOneTenant(tenantId);
    cardInfoDto.setLogo(tenantDto.getLogo()); //商户Logo
    return cardInfoDto;
}).collect(Collectors.toList());
```
我们可以猜测出，此代码片段是用于查询会员卡列表的，那么为什么在循环中存在代码 `cardInfoDto.setLogo(tenantDto.getLogo()); //商户Logo`，这样代码是用来增加字段“商户Logo”。从领域模型来看， “商户Logo”与“会员卡”没有直接从属关系，那么为什么会如此去设计呢？

通过GIT的提交历史中，我看到此“商户Logo”字段是后来增加的，为了在客户端展示会员卡的归属商户的Logo。根据业务上，每个商户的客户端都是隔离的，而且在客户端启动时也初始化了商户的基本信息，但是由于Logo字段是后续增加的，而没有初始化到客户端，才导致客户端中需要展示“商户Logo”时就找不到了。

而后续的开发人员为了解决一时的需要，就采取了“你啥时候要这个字段，我就给你在接口中增加这个字段”这种设计心理。

最后导致，客户端做接口对接时，在文档里寻找需要的字段。一个反人类的服务诞生了。我们JAVA开发工程师经常被问这样一类问题“商户Logo不应个在查询商户信息的接口中吗？”

### 3.不遵循设计原则，协作中的混乱

一个敏捷的团队还是比较注重沟通和协作的，但是团队中总会发生意想不到的事情。

例如，类似功能的接口在系统中存在多个。今张三添加一个查询商户的接口，明天李四增加一个查询商户的接口，导致在系统层面的混乱。特别是目前的微服务架构。

导致混乱的根本原因，还是没有遵循设计原则。简单的说，手机客户端需要一个查询商户的接口，张三认领了此功能的开发，然后完成。后来，需求发生了变化，WEB客户端需要在商户信息中增加商户下的渠道信息，李四认领了这个功能的开发，然后李四重新写了一个查询商户信息的接口也提交测试了。

于是，后台服务的接口越来越多，系统间交互也越来越复杂。此时，已经没有人能够阻止系统继续混乱下去了。

> 以上只是简单的举个例子，从个人到团队总会存在各种各样的问题。从个人角度出发，编码算是技术活，每个人的习惯可能不一样，但是尽量培养好的习惯，`checkstyle`、`findbug` 等工具都可以发现代码中的问题，争取从业务级别的代码向系统级别的代码进化。 从团队角度，在牛逼的管理方法都不如大家都积极的去促进系统的优化发展。唯心的一句话，“队伍不是靠管起来的，而是心往一处看，劲往一处使”。
