---
title: PBFT的几个细节
categories:
  - 共识算法
tags:
  - PBFT
translate_title: several-details-of-pbft
date: 2019-10-27 14:31:54
---
熟悉PBFT的朋友对"三阶段"不会陌生，但我们今天不谈"三阶段"，只谈PBFT设计的一些细节。
<!--more-->

# 为什么Primary节点不发送prepare消息？
primary发送prepare消息可能会比pre-prepare消息先到达，导致消息失败（个人看法）

# 几个阶段各需要的消息确认数目？
`Prepare阶段`： 需要收集2f个来自不同backup(包括自身的prepare消息，实际收到2f-1)并且与Pre-prepare消息相匹配的的prepare

`Commit阶段`：收到2f+1个（包括自身的commit消息）校验通过的commit

`Reply`: 客户端收到f+1个相同结果的有效回复（校验签名）

`Checkpoint`: 2f+1个相同的checkpoint消息（n,d）相同，则该checkpoint变成stable checkpoint

`ViewChange`: v+1的primary（通过p = （v+1）mod |R|计算）收到2f个有效view-change消息后，广播NewView消息；backup收到f+1个节点发送viewchange报文中的v大于等于当前节点的v，说明当前节点的v是旧的，因此触发viewchange事件来更新当前节点的v。

# 为什么需要commit阶段？

假设我们去掉commit，所有节点收到2f+1(包括自己)的prepare之后就执行操作，会发生什么？

其实如果顺利的话，即使有f个作恶节点，依然有f+1个正常节点所有节点都会收到正确的结果，最后所有的节点都能顺利的达成一致的结论。这样看来似乎我们完全不需要commit吗？

但是如果主节点崩溃发生换主，其中只有一个或几个（不是大多数）已经收到了足够的prepare，其他节点因为网络原因没有收到本应该收到的足够多的prepare（异步网络环境没有任何通信保证，只有最终一定会收到的保证），那么那个执行了操作的节点就悲剧了，这个时候新主发起新一轮共识，sequence跟已经执行的操作一致，那个节点到底执行好还是不执行同样sequence的操作？

那么commit是怎么做到的呢？假设节点收到足够多的prepare进入commit阶段，这个时候发生了一样的换主情形，由于节点还没执行，继续按照新一轮的流程走即可，这个时候sequence不变，但是view改变。

如果已经收到了足够的commit，并且已经执行了操作呢？仿佛陷入了prepare一样的地步...但是实际上因为要产生commit消息，说明2f+1个节点已经prepare了，换主的时候主会去搜集要重放的pre-prepare（2f+1个节点的，必然存在一个诚实节点并且有对应的pre-prepare）,因此会把同样的digest对应的消息view改为自己重发一次，并且注意到commit只需要跟当前的view相同就可以接受，那么实际上commit是对view不敏感的。

简而言之，prepare锁定同一个view下的sequence，commit锁定sequence。

# ViewChange和NewView需要注意的几个参数

<VIEW-CHANGE, v+1, n, C, P, i> 

- n 是节点 i 知道的最后一个 stable checkpoint 的消息序号。
- C 是节点 i 保存的经过 2f+1 个节点确认 stable checkpoint 消息的集合。
- P 是一个保存了 n 之后所有已经达到 prepared 状态消息的集合。

<NEW-VIEW, v+1, V, Q>

- V 是 p1 收到的，包括自己发送的 view-change 的消息集合。
- Q 是 PRE-PREPARE 状态的消息集合，但是这个 PRE-PREPARE 消息是从 PREPARE 状态的消息转换过来的。

总结一下，在 view-change 中最为重要的就是 C，P，Q 三个消息的集合，C 确保了视图变更的时候，stable checkpoint 之前的状态安全。P 确保了视图变更前，已经 PREPARE 的消息的安全。Q 确保了视图变更后 P 集合中的消息安全。回想一下 pre-prepare 和 prepare 阶段最重要的任务是保证，同一个主节点发出的请求在同一个视图（view）中的顺序是一致的，而在视图切换过程中的 C，P，Q 三个集合就是解决这个问题的。

# 异常处理viewchange更换视图的触发条件

在PBFT正常处理的流程的，时常会出现viewchange视图更换（也就是触发更换主节点），视图更换的触发可能有如下原因：

- 没有在规定时间内收到RequestBatch：即从节点收到了广播的交易，但是因为不是主节点无法处理，而且过了指定的时间，还是没有收到主节点发起的共识请求
- pre-prepare阶段没有在规定时间内收到报文请求
- 发起viewchange请求报文后，没有在规定时间完成，重新触发viewchange事件
- 没有收到主节点的心跳
- 收到f+1个节点发送viewchange报文中的v大于等于当前节点的v，说明当前节点的v是旧的，因此触发viewchange事件来更新当前节点的v
- processNewView阶段，如果挑选不出其他节点发的公共的最小的checkpoint，或者assignSequenceNumber=nil（即分配不出Request给seqno），或者收到newview报文的xset和msglist跟自己计算的不一样，以上三种满足任何一种，都会触发viewchange事件
- 在checkpoint中，设置的instance.lastExec%instance.K=0，即满足区块的某个整数倍，则强制viewchange