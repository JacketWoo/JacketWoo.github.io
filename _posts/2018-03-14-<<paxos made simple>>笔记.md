---
layout: post_layout
title: 《paxos made simple》笔记
time: 2018年03月14日 星期三
location: 北京
published: true
excerpt_separator: "##"
---


本文我认为主要是讨论了paxos关于如何解决一致性协议中的safety部分，对termination（liveness）和faulty tolerance没有涉及太多。

## 1 一致性协议的safety

- Only a value that has been proposed may be chosen
- Only a single value is chosen, and
- A process never learns that a value has been chosen unless it actually
  has been

针对某个条目的多个提案内容，最多只有一个可以被选中，并且只有被选中的提案内容才可以扩散到一致性系统中的其他成员

## 2 实现

> *-* proposers： 提案发起者，发起某个提案供acceptors投票
>
> *-* accepters：投票者，对某个提案进行投票
>
> *-* learners: 学习者，获知哪个提案被通过了

### 2.1 paxos算法

单节点是解决safety问题的最简单的方法，但是有单点宕机的风险。因此才有了分布式系统。当针对一个条目，有多个提案被发起时，为了保证只有一个提案被通过，通常采用的是“大部分方法”，即当一个提案被大部分的成员批准了才允许通过。

分布式系统的成员，可能会等待所有的提案都到达时，然后对某个提案投票；但是当某个条目只有一个提案被发起时，为了保证这个提案可以被及时的通过（主要是为了保证一致性算法的liveness属性），paxos规定：

**P1. An acceptor must accept the first proposal that it receives.**

基于P1可能会产生另一个问题，如果有多个提案在某个极小的时间段内被发起，那么每个提案的支持者可能都会沦为“小部分”，这样将没有一个提案被通过。为了保证有提案可以获得大多数的投票，一个办法是让每个投票者允许给不止一个提案投票。但是，这样又引入了另一个问题，可能会有不止一个提案获得大多数投票，从而被通过，这样就丧失了safety。为此，paxos 为每一个提案赋予一个编号，确定它们的逻辑顺序，然后规定：

**P2. If a proposal with value v is chosen, then every higher-numbered proposal that is chosen has value v.**

这样就保证了基于某个条目的众多**提案内容**中，只有一个会被选中。P2定义的太抽象了，我们做进一步的限定:

**P2a. If a proposal with value v is chosen, then every higher-numbered proposal accepted by any acceptor has value v.**

> 证明：给定P2a是可以保证P2的，当某个提案被通过后，每个被投票的更高编号提案的内容都和这个被通过提案的内容相同，说明任何被通过的更高编号提案的内容都与这个提案的内容相同，也就是P2

为了实现方便，对P2a作进一步限定

**P2b. If a proposal with value v is chosen, then every higher-numbered proposal issued by any proposer has value v**

> 证明：显然P2b可以保证P2a

P2b的实现还是比较抽象，直观的实现比较困难，为此，paxos进行了迂回，采用了另外一种办法来保证P2b，提出了：

**P2c. For any v and n, if a proposal with value v and number n is issued,then there is a set S consisting of a majority of acceptors such that either (a) no acceptor in S has accepted any proposal numbered less than n, or (b) v is the value of the highest-numbered proposal among all proposals numbered less than n accepted by the acceptors in S**

> *论文里的证明没有看懂，这里是自己证明的，应该殊途同归*
>
> 证明：在P2c的条件下。若
>
> a. 某个编号为m的内容为v的提案被通过了
>
> =》存在一个大多数的acceptors集合，他们批准了编号为m内容为v的提案
>
> =》当n=m+1时，任何一个大多数的acceptors集合中，所批准的编号小于n的提案的最大编号为m，
>
> =》根据P2c，则n=m+1时，被发起的编号为n的提案应该为编号m所对应的的提案，内容为v
>
>  
>
> b. 基于上，当n=m+2时，任何一个大多数的acceptors集合中，所批准的编号小于n的提案的最大编号为m或者m+1，然而编号m+1和m对应的提案的内容都为v
>
> =》根据P2c，则n=m+2时，被发起编号为n的提案的内容为v
>
>  
>
> c. 同理地推，根据P2c，当n>m时，任何发起的编号为n的提案的内容都为v
>
>  
>
> 综合a、b、c，可以说明，满足P2c的情况下，可以保证P2b

P2c中，提案到达acceptor的时间点和确定提案内容的时间点，之间是有一个时间差的，在这两个时间点上，P2c的a和b可能会不一样，这样P2c就不能满足了。所以Paxos将一个提案的发起分为两部分：

```
1. A proposer chooses a new proposal number n and sends a request to each member of some set of acceptors, asking it to respond with:
   (a) A promise never again to accept a proposal numbered less than n, and
   (b) The proposal with the highest number less than n that it has accepted, if any.
2. If the proposer receives the requested responses from a majority ofthe acceptors, then it can issue a proposal with number n and valuev, where v is the value of the highest-numbered proposal among the responses, or is any value selected by the proposer if the respondersreported no proposals.
```

即分为两个命令：prepare命令和accept命令，分别对应上面的1和2。prepare命令相当于是准备阶段，确定了提案的编号n，acceptors的prepare回应包含了两个信息：a) 不再批准编号小于n的提案；b) 这个acceptor之前批准的小于n的最大提案编号。当搜集到来自大部分acceptors的prepare回应后，就可以依据P2c确定编号为n的提案的内容，然后发送accept命令。

属于preposers部分的算法已经细化很多了，P1的规定也比较的抽象，也可以进一步的喜欢，所以，Paxos对P1进一步细化：

**P1a . An acceptor can accept a proposal numbered n iff it has not responded to a prepare request having a number greater than n.**

> 证明：显然P1是P1a的一种特殊情况。与P2b与P2c的关系类似。



综合以上Proposers和accepters部分的内容，Paxos算法可以表述为以下两个部分：
```
Phase 1. (a) A proposer selects a proposal number n and sends a prepare request with number n to a majority of acceptors.
(b) If an acceptor receives a prepare request with number n greater than that of any prepare request to which it has already responded,then it responds to the request with a promise not to accept any more proposals numbered less than n and with the highest-numbered proposal (if any) that it has accepted.

Phase 2. (a) If the proposer receives a response to its prepare requests(numbered n) from a majority of acceptors, then it sends an accept request to each of those acceptors for a proposal numbered n with a value v, where v is the value of the highest-numbered proposal amongthe responses, or is any value if the responses reported no proposals.
(b) If an acceptor receives an accept request for a proposal numbered n, it accepts the proposal unless it has already responded to a prepare request having a number greater than n.

第一阶段 
(a) proposer发起一个提案编号n的prepare命令到大部分的acceptors
(b) 当accepter收到的prepare命令的提案编号比以往回复的prepare命令的编号都要大时，它回复这个prepare命令，带上的信息包括 i) 该accepter所批准的最大提案编号和提案内容；ii）该accepter将永远不会批准编号小于n的提案
第二阶段
(a) 当proposer搜集到大部分的prepare回应时，他将依据P2c确定编号为n的提案的内容，然后构造accept命令，将它发送给大部分的accepters
(b) 当accepter没有回应过编号大于n的prepare命令时，它就批准收到的编号为n的提案
```
### 2.2 multi-paxos

上面的Paxos算法描述的是针对某个条目的多个提案达成一致，真实的环境下，肯定是需要对某一系列的条目达成一致。大致的做法是为每个条目赋予一个编号，确定各个条目的逻辑顺序，然后为每个条目独立分配一个paxos算法的实例，该实例专门用于服务对应条目提案的一致性。当某个条目之前的所有条目的一致性都达成了之后，这个条目之前的所有所有条目就都可以被应用本地的状态机了，如按照顺序执行数据库操作。举个例子：

假设当前已经达成一致性的条目编号有 0，1，2，4，5。则0，1，2条目所被通过的提案就可以应用到本地的状态机了，而4，5条目暂时还不可以。

***multi-paxos允许，在一定的窗口范围内，多个条目的一致性达成过程，可以同时进行，相比于raft的顺序执行，理论上性能应该更优***

> 由于本文multi-paxos部分只是给了一个可能的实现，并没有正确性的证明，和详细的阐述，如怎么选主，怎么添加/删除成员等，所以这里也不多做展开

### 2.3  其他问题

#### 2.3.1 获知哪个提案被通过

主要有三种方法：

（a）每个accepter都将它批准的提案通知给所有的learners

​	优点：迅速

​	缺点：开销大，num(消息) = num(accepters) * num(learners)

（b） 每个accepter把自己批准的提案信息先告诉某一个learner，然后再由这个learner通知其他learners

​	优点：节省开销

​	缺点：整个扩散过程多增加了一个阶段，并且有单点挂机的风险

（c） 每个accepter把自己批准的提案信息先告诉某一群learners，然后再由这一群learners通知其他的learners

​	这个是对上面a和b的一个折中

（d）由某个proposer再走一次paxos算法。这种办法主要用于消息丢失，导致有的learners没有通知到，或者有节点挂掉的情况。因为有paxos保证了safety，所以结果是，i) 不能达成一致; ii) 被通过的提案是之前已经被通过提案; iii) 原来没有提案被通过，新通过了一个提案

#### 2.3.2 进展

如果有两个proposers不断地的发起提案，可能的一个情况是他们每次新发出的prepare命令的提案编号，都导致另一个proposer已经发起的提案编号失效，让另一个proposer在阶段二不能获得大多数的投票。整个过程一直循环，最终的结果是提案一直不能被通过。

解决的办法是一个段时间内，选出一个proposer，在这段时间内，只有它可以发起提案。

