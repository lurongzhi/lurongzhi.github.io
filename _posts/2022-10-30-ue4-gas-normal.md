---
layout: post
title:  "UE4笔记-GAS概述和总结"
date:   2020-10-30 09:00:07
categories: development
excerpt_separator: <!--more-->
description: "整理总结GAS各模块的负责功能和定位，从而理解GAS的设计思想"
image: false
show_sidebar: false
published: true
# canonical_url: https://www.csrhymes.com/development/2019/02/09/getting-started-with-bulma-clean-theme.html
---

学习和使用UE4有半年时间了，UE4给我的最大感觉就是功能庞大而且代码复杂，其中为了适配各种不同的业务场景，有很多代码逻辑分支。因此往往阅读完代码之后往往几天后就很容易忘记了。相信很多人也会和我有一样的遭遇，所以在这里把自己的笔记写下来，希望加深自己的记忆同时，也能给打算学习和深入了解UE4的同学作为一些参考。

<!--more-->

# 概述
GAS(**Gameplay Ability System**)是UE4官方自带的一个技能框架系统，框架整合了游戏中的战斗技能需求，涵盖了技能管理、技能逻辑、Buff、表现等功能，同时还提供了预测先行的功能。

网上有很多关于GAS的参考文章，大部分都是介绍GAS如何去使用，而很少有文章会提到为什么。为什么GAS要这样设计，其中有哪些模块负责了什么功能，如果我想要做拓展应该怎么修改。

这篇笔记基本不会设计到具体代码逻辑或者蓝图使用，更多是从框架设计层面上去理解GAS的思想。相信在此之后再去看代码，会有更加清晰的思路。

# 系统模块和功能
## ASC
我们在设计一个模块的时候，往往希望将对外接口封装整理起来到一个对象上，设计模式上叫外观模式(Facade)，这个对象也被称为外观类。这样的好处是对外部使用者来说，这个模块是个高内聚的模块。__AbilitySystemComponent__ (以下简称ASC)就是一个这样的组件，是整合了GAS功能接口的一个对象。我们只需要给角色赋值一个ASC就能够使用这个技能框架。

ASC的功能包括了GameplayAbility的管理以及使用，存放Attribute属性、以及GameplayCue的管理等等，基本所有涉及到技能的东西都在ASC组件里面，可谓是集大成者。至于上面说的这些名词，莫慌待会会一个个细说，一个整体的类结构图是这样的：

![GAS类图](/img/UEGAS/GAS_class_diagram.png)


## GameplayAbility
__GameplayAbility__ (简称 __GA__ )作为一个技能实例的对象，提供Activate、Deactivate等接口。使用技能之前，要先通过GiveAbility将技能注册到ASC中。关于技能的逻辑，可以直接写在Ability上，或者是调用封装好的GameplayTask，也可以直接调用GameplayEffect来修改属性。

可以看到在上面类图上，ASC并没有直接引用GA，而是先引用了GameplayAbilitySpec的东西。这个Spec到底是什么来头？我个人觉得主要有两个作用：
- UE中的技能带有等级的概念，不同等级的技能有不同的属性。由Spec记录着level信息
- 技能实例类型分为Non Instance、Instanced Per Actor、Instanced Per Execution，分别代表所有角色使用同一个技能对象、每个角色一个技能对象以及每次激活生成一个新的技能对象。

不过按照常规思路来说，技能等级一般会放到Attribute属性那一层去做，UE4放到Spec这种设计个人感觉是不太合理的，显得模块职责划分不怎么清晰。

## GameplayTask
GameplayTask可以理解为一些通用化的技能逻辑，封装起来以便于复用。比如说冲刺逻辑、播放特定动画、生成召唤物等等。一个Task可以理解为技能编辑器中的一个节点，通过整合不同的节点来串联出一个技能逻辑。

同时，封装成GameplayTask还有一个好处，就是网络游戏中的同步可以以Task为单位。并且还能为每个Task自定预测逻辑，处理技能先行表现、避免Redo、以及处理回滚(Undo)。关于预测方面内容，不是这次的重点就不多说了~

## GameplayEffect
按照MVC的思想，系统可以划分为控制逻辑->属性修改->表现内容。UE4将对属性的所有修改封装成了一个类:__GameplayEffect__（简称 __GE__），也就是所有关于属性的修改都要通过GE来执行，这样。GE本身是不带运行时数据的，运行时的数据存放在 __GameplayEffectContext__ 中，比如说是谁放技能，技能击中了谁。

## AttributeSet
顾名思义，就是角色战斗属性声明的地方。虽然是这样，但是实际上保存数据的地方并不在AttributeSet对象，而是在ASC中的SpawnedAttributes容器里面，并且由ASC序列化和网络同步。

## GameplayCue
__GameplayeCue__ 为MVC模型中的View层，则表现层。其激活和删除均通过以GameplayTag作为Key映射到具体的CUE对象上。CUE由ASC管理，但是网络同步中ASC只同步Tag和Param，而不是直接同步CUE对象。CUE的使用，在上面提到的所有技能逻辑的地方都可能发生，包括GA、Task、GE。

CUE对象的实例化分为两种，一种是Static，则同一个类型的CUE只会使用其CDO对象，这类的CUE是不能够带有数据的；另一种是Actor，每次激活CUE都会根据一个Key值进行生成和复用，同一个Key值使用到的CUE对象为同一个。这个Key值的规则也分几种，可选为Source、Target、Instigator这三种的组合。

GameplayCue很好地做到了逻辑和表现分离，但是CUE对象的实例化个人感觉是太复杂了，明明可以做成是每次激活都是一个CUE对象，这样状态数据管理会简单很多。UE的思想可能是考虑到内存占用和生成新对象的消耗，但是这个问题完全可以由对象池解决，CUE使用完之后放回到池子里，被使用就取出来就行了。

# 总结
GAS里各对象的职责划分比较细致，并且大部分有明确的定位，很好地做到了逻辑、数据和表现的分离，并且还提供了预测功能。但是对于一个技能系统来说，GAS的代码量实在太多太复杂了，比如单单一个修改属性的需求，调用栈就有十几行。并且由于要适配各种模式下的运行，Standalone、DedicateServer,客户端和服务端均跑同一份代码，再加上本地预测功能，代码就显得分支很多，带来阅读的困难。只能说要研究透彻GAS还是任重道远。

# 参考阅读

[深入GAS架构设计](https://www.bilibili.com/video/BV1zD4y1X77M/?spm_id_from=333.999.0.0&vd_source=b76b5435b32df225d989897641a20b55) ,虚幻官方对于GAS设计的介绍

[GASDocumentation](https://github.com/BillEliot/GASDocumentation_Chinese) ，GitHub上关于GAS的总结，很详细可以作为工具书参考




