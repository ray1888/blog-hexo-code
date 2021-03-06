---
title: 分布式系统一致性与共识
date: 2019-08-28 21:08:38
tags: distributed-system
---

<!-- ---
title: 
author: Ray Chan(ray1888)
date: '2019-08-20 11:08:38 +0800'
category: 
summary: Distributed System Consistency and Consensus
thumbnail: distributed-system.png
--- -->

# 目录
1. [分布式系统可以提供的若干保证和抽象机制](#PromiseAbstraction)  
   1.1 [共识算法的意义](#1.1)  
2. [如何在分布式系统中做到原子性](#atomic)  
   2.1 [可线性化](#2.1)  
   2.2 [顺序化](#2.2)  
        2.2.1 [顺序、因果、全局序号的关系](#2.2.1)  
        2.2.2 [全局关系的广播](#2.2.2)  
3. [分布式系统的能力边界](#Capability)  
   3.1  [分布式事务](#3.1)  
   3.2  [容错的共识](#3.2)  
4. [ShareNote](#ShareNote)  

# <a id="PromiseAbstraction"><span class="toptitle">分布式系统可以提供的若干保证和抽象机制</span></a>

## <a id="1.1"><span class="secondtitle">共识(分布式一致性)算法的意义</span></a>
对于大多数的多副本的数据库（N>=2)的情况下，它至少达到了最终的一致性。但是因为只是最终，但是到达最终的状态的时间是未知的。  
最终一致性的多副本数据状态（如Aurora、Riak、Cassandra）的不一致对于应用开发者是比较棘手，因为很多底层保证一致的东西需要挪动到应用层上面去添加规则来继续保证。并且导致测试和验证都会变得异常的困难

基于上面的问题，我们可能需要找到一个更加强一致性的模型来进行构建，可以把应用层可能遇到的数据库不一致的问题封装起来。虽然可能会付出其他的代价（如性能下降、容错性差）

### 与事务隔离的级别的差别
共识与事务隔离都有类似与副本的概念（事务隔离是使用乐观锁（MVCC） 共识是使用多副本）  
共识：更加强调是针对延时和故障协调副本之间的关系。  
事务隔离：处理并发事务的各种临界的条件  

# <a id="atomic"><span class="toptitle">如何在分布式系统中做到原子性</span></a>
## <a id="2.1"><span class="secondtitle">可线性化</span></a>
（此部分主要是DDIA第9章节的内容）
定义： 让一个分布式的系统看起来像一个单副本的系统，并且所有的操作都是原子的。  
### 一个非线性化的例子

<!-- ![非线性化图片](/assets/img/posts/non-linearized-example.png)   -->
{% asset_img non-linearized-example.png 非线性化 %}

对于上面这个图的简要描述：  
因为使用了多节点非强一致性的数据库，当Referee修改数据库的情况下，因为集群内同步的时间不一致，导致可能Alice先查的的Slave-1 已经修改了，但是Bob查询的Slave-2的库上面还没有修改，导致Alice和Bob获取到的结果不一致。这种场景对于一些一致性需求比较强的情况下不能接受。

### <span class="thirdtitle">线性化的例子的直觉表达</span>

<b><font color="#FF0000">线性化的简洁但是事实上的描述： 让分布式系统具有寄存器的语义。</font></b>  
对于这个部分，我们详细去探讨一下  
一般我们的对于读写请求并发（在客户端的角度）是这样的:
定义一个前提， 下图中的一个框是指客户端从发出请求到接收到返回值的整个过程  
单个框可以理解为 客户端发出请求-> 服务器端接受并且处理请求-> 服务器端返回结果-> 客户端在应用级别收到返回  
下面可能出现框比较长的情况是在上一篇文章（![分布式系统简介以及其问题](https://ray1888.github.io/distributed-system/2019/08/20/chanllange-of-distributed-system/)）的假设所提及的情况，此处不再复述。  

分布式系统中的寄存器： 线性化数据库中的相同的主键。（此处为X)  
<!-- ![粗粒度线性一致](/assets/img/posts/generate-line.png)   -->
{% asset_img generate-line.png 粗粒度线性一致 %}
此处对于寄存器有两类的操作：
1. read(x)  读取主键为X的值，数据库返回值v
2. write(x,v) 客户端把X的值更新为V，数据库返回值为r(成功或者失败)

对于上图在read和write重合的过程中，read的结果可能有2种情况：  
1. read在write完成之前结束(返回0)  
2. read在write完成之后结束 (返回1)  

但是这个描述的粒度还是比较大，为了把系统线性化的更好的描述，我们添加下面的约束  
<!-- ![细粒度线性一致](/assets/img/posts/more-detail-line.png)   -->
{% asset_img more-detail-line.png 细粒度线性一致 %}
约束为，在写操作的过程中，肯定有某一个时间点，存储上面的x的值会出现冲0到1的跳变。  
对于上图， 客户端A在某个节点能够读到X的值为1，那么因为客户端B的读操作是晚于客户端A读的发生的，因此  
客户端B获取到的值也必然是1（如果不是1，则不是可线性化的系统）。  
<!-- ![线性一致](/assets/img/posts/line-final.png) -->
{% asset_img line-final.png 线性一致 %}
但是上面的讨论颗粒还是太大了，此处我们加入进去存储，以及存储接收到的操作来整体观察线性一致。
我们此处引入一个概念
```
CAS(compare and swap):一个原子的比较设置操作。编程语言中一般都要实现此概念，此处不详细叙述
```
并且添加一个假设：即使客户端可能还没有收到成功的响应，但是分布式的存储已经全部同步成新的值。
上图中每个操作都有竖线，表示可能的执行的时间点，把这些点连接起来，最终结果必须是一个有效的寄存器读写顺序。  
可线性化的要求： 按时间箭头向前移动，不能向后移动。
可以看到上图中，最后客户端A的操作在c的Cas操作完成后，x的值已经变成了4，但是如果发现读的值为2的话，按时间线的理解就是错误了。因为根据可线性化的要求，cas操作已经执行了，并且时间只能向前走的情况下，客户端A的读操作必然是需要返回4的。

```
//可线性化和可串行化的对比
#### 可线性化
对寄存器（单个对象）的最新值保证。他不要求将操作组合到事务中，可能会出现写倾斜的问题（如果不采取手段去解决的问题）
#### 可串行化
是事务的隔离属性。每个事务可以读写多个对象，用于确保事务执行的结果与串行执行相同。即使穿行执行的顺序和实际执行的顺序并不一定相同（可能实际上的执行是并行执行事务，但是对外暴露的结果是一致的就可以）  
```

### <span class="thirdtitle">需要用到线性化的使用场景</span>
1. 加锁与主节点选举
对于主从复制系统的选主问题，可以通过基于Etcd或者ZooKeeper来进行抢锁操作控制选主
2. 约束与唯一性保证
应用中对同一个资源（如用户名）的唯一性的约束。（可以延展到数据表的主键约束）
3. 多信息源的时间依赖
对于一些异步任务系统虽然采用了最终一致性，但是可能也是会因为实现产生数据的依赖问题。

### 如何实现一个线性化系统
#### 对比多种可能使用的方案来确定
在考虑容错的基础上，必须考虑复制的机制。我们可以对比多种复制方案来看看那种可以实现线性化
1. 主从复制。（部分线性化）
因为所有写入操作都是从主节点继续，并且把操作同步到从节点上面。但是问题可能出现在实现的问题上：  
    1. 实时同步可能会出现问题
    2. 可能会因为快照隔离设计出现问题
2. 共识算法（可线性化）
类似于主从复制，但是通过一些手段来防止脑裂和过期的副本
3. 多主复制（不可线性化）
当允许并发写入的时候，如果进行异步复制的话，可能会出现数据的冲突。
4. 无主复制（可能不可线性化）
类似于Dynamo的机制，即使使用了Quroum机制，但是如果选取的Quroum不一定是满足的情况下，可能会出现非线性化的处理

此处提出一个问题，是否只要有Quroum机制（就必定可以支持线性一致呢？）  
答： 不一定。用下面的例子进行解释  

<!-- ![QuroumFail](/assets/img/posts/quroum-fail.png)   -->
{% asset_img quroum-fail.png QuroumFail %}

对于上图中，即使是使用了Quroum，但是没有共识算法的支持，还是可能会出现类似于之前Alice和Bob遇到的情况的问题。但是这个选取的直接是一个Quroum，但是是因为缺少共识，并且网络出现问题才会导致这样的情况出现。

### 线性化的代价

<font color="#FF0000">线性化出现代价的最主要的原因还是因为网络的不确定性。</font>

我们先用主从的架构来讨论这个问题，当网络发生分区的情况，主从不能够继续同步操作，可能会导致从库不可用。
但是这个是我们需要实现线性化所带来的不可用

先引入一个概念：CAP理论
```
CAP : 在同一个时间内，当网络出现分区的情况下，不可能获得兼容可用性和一致性。
```
CAP理论的推理：不要求线性化的应用更能够容忍网络分区。

但是为了线性化，我们可能要牺牲性能和延迟。
那么我们是否有方法可以做到线性化但是减少性能的牺牲呢？

## <a id="2.2"><span class="secondtitle">顺序保证</span></a>
我们刚刚在讨论线性化的时候使用到了寄存器的概念，那么顺序、可线性化、共识之间是否存在的某种关系呢？

### <a id="2.2.1"><span class="thirdtitle">顺序、因果、全局序号的关系</span>
#### <span class="fourth">顺序和因果的关系</span>

因果关系对所发生的事件添加了排序， 一件事情会导致另外一件事情，这些的因果关系依赖链条定义了系统中的因果顺序。如果符合因果关系所规定的顺序，我们称之为因果一致性。（快照隔离，在从数据库中读数据的情况下，查询到的诗句，也可以查到这个数据之前发生了什么的操作事件）

#### <span class="fourth">因果顺序并非全序</span>

全序关系是可以支持两个不同的实体直接进行比较（如自然数集5和13的比较）  
但是部分集合的对比不一定符合全序，集合{a,b} 与 集合{b,c}是无法进行对比的  

因此，提炼到可线性化和因果关系中  
可线性化是存在全序的操作关系，因为暴露对外的行为是与单副本无异，并且每个操作都是原子的。可以分出先后  
因果是偏序的操作关系，对于并发的两个操作无法比较的情况下，就会发生冲突，对于可以比较的情况下，与线性无异。    
那么可以这样说，可线性化一定意味着因果关系，因为可线性化是全序的操作。    
但是可以这样理解，线性化并非是保证因果关系的唯一的途径。我们可以有其他手段去满足因果一致性而避免线性化所带来的问题。    
因果一致性是被认为是不会由于网络延迟而显著影响性能，并且可以对网络故障容错提供容错的最强一致性的模型。  
因为实际上很多的应用所需要的是因果一致性来保证应用的正确性。  

#### <span class="fourth">捕获因果依赖关系</span>
如果只要解决并发操作的先后依赖关系。这里其实只需要由偏序的关系即可。
我们常见的数据库的版本技术就是一个解决这个问题的方案之一
为了确定数据库的因果关系，数据库需要知道应用读取的是哪个版本的数据。

#### <span class="fourth">序列号排序</span>
那么，这样的情况下，我们是否需要显式的跟踪所有的因果关系呢？  
为了性能的考虑，我们可以通过序列号和时间戳（尽量不使用物理时钟）来进行对事件排序，这样在保证性能的同时，也能够保证所有操作在全局的关系。但是要保证每个序列号必须唯一，并且可以比较。  
我们可以按照与因果关系一致的顺序来创建序列号： 如果操作A发生B之前，那么A一定在全序顺序中出现在B之前    
这样的全局排序可以捕获所有的因果信息，并且加强了比因果关系更为严格的顺序性  

那么对于不存在唯一的主节点（多主或者无主），那么我们能怎样生成上面所提及的序列号呢？  
有实践中可以采用以下方法：  
    1. 每个节点单独生成自己的一组序列号。假设有两个节点，一个生成奇数，一个生成偶数  
    2. 把Timestamp添加到操作中（之前生产中有用此种方式）  
    3. 预先分配序列号的区间范围。（A节点分配1-1000，B节点分配1001-2000）  

但是这三种实际上生成的唯一的，近似增加的序列号。但是实际上序列号和因果一致不是完全因果一致的。  
为什么不是因果一致呢?  
对于情况1，每个节点处理的速率是由不一致的，只要两个节点处于处理速率不一致的情况下，分配的ID必然和实际的顺序关系无法对应  
对于情况2， 墙上时钟发生偏移的情况  
对于情况3， 如果sharding的路由发生了变化之后，那么之前1-1000的因果关系就不存在了。  

那么我们是否没有办法可以在非强主的情况可以获取一个序列号与因果一致可以对应上的吗？  

我们还有一个方法！！！此处大神来了Leslie Lamport的Lamport TimeStamp可以解决这个问题  

#### <span class="fourth">LamportTimeStamp</span>
<!-- ![LamportTimeStamp](/assets/img/posts/LamportTimestamp.png)   -->
{% asset_img LamportTimestamp.png LamportTimeStamp %}
每个节点一开始的时候都会有一个唯一的标识符（每次初始为0）， 然后每个节点都有一个计数器来记录各自处理的请求总数。到此与之前的时间戳的并无差别，但是Lamport这里处理的亮点出来了，每个节点和每个客户端都会跟踪迄今为止最大的计数器值，并且在每个请求中附带该最大计数器值。当节点收到某个请求的时候，如果发现请求内的最大计数器值大于自身的计数器值，会更新自己的计数器值（Raft中是通过投票和心跳的RPC来同步这个计数器的值，在Raft的概念里面叫做Term，后面讲解Raft的时候会进一步解析，此处只是一个插叙）。

LamportTimeStamp好像是解决了分布式系统顺序号和因果一致的关系，但是是已经足够解决分布式系统中常见的问题了吗？  
问题如下：  
    一个系统的用户名只能由唯一的用户持有，两个用户并发地向系统同时进行注册，我们必须要保证一个成功，一个失败    
虽然看上去我们是可以把两个请求通过编号把两个并发的请求变成了一个有顺序的问题，但是实际上要保证上面的是唯一的有两个前提：  
    1. 节点收到用户请求的时候需要马上判断请求时成功还是失败  
    2. 必须要收集系统的所有创建用户的请求，比较序号  
但是这个显然是不可能的，只要网络出现问题了，我们就无法做到上面的两个问题。

那么我们如果要知道全局关系是否确定，就需要提到后面的一个概念，全序关系广播

### <a id="2.2.2"><span class="thirdtitle">全局关系的广播</span></a>
继续先从一个主从复制的系统开始说起，主节点接受写请求并且变成顺序的操作。  
但是在分布式系统领域，如何扩展系统的吞吐量使之突破单一主节点限制以及处理主节点失效时的故障切换。这类的问题被称为全序关系广播或者原子广播。  
全序关系广播： 通常指节点之间交换信息的某种协议。有两个特性  
    1. 可靠发送（没有消息丢失，如果消息发送到某一个节点，它一定要发送到其他的节点）  
    2. 严格有序（消息总是以相同的顺序发送到每个节点）  

#### <span class="fourth">使用全序关系广播</span>
Zookeeper和Etcd的共识服务就实现了全序关系广播。（那么全序关系广播和共识之间的关系？）  

全序关系广播就是数据库所需要，每条消息表达数据库的写请求，而且每个副本段相同的顺序处理这些写请求，那么所有副本可以保持一致。这个也被称为状态机复制。  

全局关系广播的另一个要点，顺序在发送消息时已经是确定的，如果消息发送成功，节点不允许追溯将某条消息插到先前的位置上。只能进行追加，这样全序关系广播比时间戳的排序要求更强  
应用场景：  
    1.我们可以使用全序关系广播来实现可串行化的事务。（每个消息表示为一个确定性质的事务，并且作为存储过程来执行）    
    2.提供Fencing令牌锁的服务（把取锁的请求变成一个消息追加到日志中，序列号直接可以变成令牌返回）  

#### 全序关系广播来实现线性化存储
全序关系广播是基于异步模型，保证消息以固定顺序的可靠发送，但是不保证消息何时发送成功，但是可线性化更多地强调读取时能看到最新的写入值。  
我们可以通过追加的日志的方式使得使用全序关系广播在写入的方式上与可线性化做到一致  
    1. 在接受请求的本地节点中追加一条消息，指明写入的信息  
    2. 读取日志，广播到其他节点，等待回应  
    3. 如果回复中有冲突，则失败，返回错误给客户端；否则返回成功给客户端  
通过日志追加可以把并发有效的转换为多条的日志来保证因果顺序关系。但是读取却还没做到这个语义  
为了读取可以与可线性化一致，有以下方法可以解决这个问题：
    1. 把读的请求也变成日志的方式追加到排序广播，通过变成日志的情况下可以确定了顺序的问题。（ETCD采用的是这种方式）  
    2. 如果可以以可线性化的方式获取最新日志的消息位置，则查询位置，直到该位置之前的所有条目读发送给你，再进行读取（ZooKeeper使用的方式）  
    3. 可以从同步更新的副本中进行读取，保证每次读的是最新的值。  

所以最终可线性化的原子地对寄存器做CAS的操作与全序关系广播其实等价于共识的问题。

# <a id="Capability"><span class="toptitle">分布式共识的能力边界</span></a>
共识： 使得分布式系统中就某件事情达成一致  
共识的使用场景：  
    1. 主节点选举（防止脑裂问题）  
    2. 原子事务的提交（跨节点或分区的数据库，事务在部分节点成功，部分节点失败，需要进行回滚）  
我们先从事务的原子性开始，讨论2PC（2阶段提交），之后再会谈及其他更好的共识算法的实现（ZooKeeper， Raft）  

## <a id="3.1"><span class="secondtitle">分布式事务 </span></a>
事务原子性的目的：  
一个多笔写操作的事务在执行过程中出现意外情况，为上层应用提供一个简单的语义，全部成功或者全部失败  
单机原子提交：  
数据库把事务写入持久化部分，然后把提交记录追加写入到磁盘的日志文件中。所以在单节点上面，事务提交非常依赖与数据持久写入磁盘的顺序关系  
    1. 先写入数据，再提交记录  
    2. 事务提交或者中止的关键点在于磁盘完成日志的时候，在完成写之前崩溃的情况下，事务需要中止；如果日志在完成写入后发生崩溃，事务被安全提交  

但是对于分布式多节点，处理方法有所差别，原因可能是以下这几个问题引起的  
    1. 某些节点发现不满足要求中止了事务，但是部分节点通过并且提交    
    2. 部分请求可能在网络不稳定的情况下丢失，超时而中止；但是其他请求可能成功提交  
    3. 节点可能在写入日志前崩溃，而且在恢复后回滚（原来数据没有成功，回滚可能会出现问题）  
如果一部分节点提交了事务，但是部分节点放弃了事务，会导致集群中节点信息的不一致，而且事务一旦被提交，即使事后发现其他节点中止，也无法撤销本节点的事务  

事务提交后不能撤销的原因：   
一旦数据提交，就会被其他事务可见，然后客户端（应用层）就会做出相应的反应，这个是构成读-提交隔离的基础。但是如果允许撤销的话，那么所有之后的读-提交级别的任务，然后会产生多级的级联式追溯和撤销。导致系统压力巨大甚至崩溃。并且不能保证数据是否能够准确落盘。  

因为不允许撤销事务，因为如果一个错误的事务被提交了，必然需要一个新的事务来抵消它的影响，但是这个需要应用层来进行处理，就不符合我们需要给应用层提供的原子性。  

### 2PC 
两阶段提交（2PC）是一种在多个节点上实现事务原子提交的算法。要么全部提交，要么全部不提交。

##### 2PC的流程（为什么可以解决上面单阶段提交的问题）
1. 应用启动分布式事务的时候，先去像协调者获取事务ID（全局唯一）
2. 应用程序在每个参与的节点启动单节点事务，并且把事务ID附加到到参与者的事务上。如果这个阶段发生异常，可以协调者和其他参与者可以安全中止
3. 应用程序准备提交事务时，协调者回向参与者发送携带事务ID准备请求，只要有一个发现失败的情况下，通知全部放弃事务
4. 参与者收到请求后，确保自己时候可以提交事务（包括硬件故障和软件故障的确认），然后检查是否有冲突或者违规。一旦返回是，节点会承诺提交事务。（保证了参与者不会撤销事务）
5. 当协调者收到所有的参与者返回后，要决定是否提交事务，并且把此决定写入到硬盘的事务日志（WAL）中，防止掉电后可能出现的异常
6. 协调者向参与者发送提交/放弃 事物的请求，只要是提交，每个参与者会重试到成功为止(包括了进程崩溃的重试、节点重启等情况)

所以他是在保证了上面的一个重要前提，只要提交了就不能撤回。
在参与者在返回给协调者的时候保证了单向性
并且协调者确定提交了之后也是保证了不可逆，因此保证了2pc可以不会出现那些异常的情况。

但是2PC的单点在于协调者的问题（虽然可以采用共识层来写协调者保证高可用）但是整个过程中的热点也会出现在协调者上面。

### 实际生产上面的分布式事务
### 异构分布式事务
虽然分布式事务因为这样可能会有阻塞而导致性能和吞吐量的下降，但是是否没有折中的方法来使用类似的语义呢？
目前更多的是采用了异构的方法（如数据库+MQ）的执行异构的分布式事务

对于异构形式的分布式事务，当且仅当数据库中处理消息的事务成功提交，消息队列才会标记该消息处理完毕。
但只要其中一个出现失败的情况，两个部分都必须进行中止的操作。

<b>保证消息可以有效处理有且仅有一次</b>

目前有XA的异构的分布式事务的标准。

## <a id="3.2"><span class="secondtitle">支持容错的共识</span></a>
共识问题的形式化描述： 一个或者多个节点可以提议某些值，由共识算法来决定最终的值。

基于上面的描述共识算法必须满足以下的性质：
    1. 协商一致性（所有节点都接受相同的决议）
    2. 诚实性（所有节点不能反悔，对某项提议不能有两次决定）
    3. 合法性（决定了值V, 那么V肯定是由某个节点提议的）
    4. 可终止性（如果节点不崩溃最终一定可以达成协议）

协商一致性和诚实的属性定义了共识算法的核心思想：决定一致的结果，并且一旦决定就不能改变。  
合法性是为了排除一些无意义的方案，  
可终止性是容错的体现，避免了整个系统的空转  
根据之前提及的： 可终止性是活性， 其他三个是安全方面的属性。

因为共识算法的目的是解决在大多数节点正常的情况下正确运行才能确保终止性。

### 共识算法和全序广播的关系

共识算法(如Raft、Paxos、Zab)等都不是直接采用上述的形式化模型。而是决定了一系列的值，然后采用全序关系广播算法来进行实现。  

全序关系广播要点： 消息按相同的顺序发送到所有的节点，有且只有一次。  
这样的方式相当于多轮的共识过程。在每一轮，节点会提出之后需要发出的信息，然后决定下一个顺序。

对于上面的提到的四个性质：
1. 由于协商的一致性，所有节点决定以相同的顺序发送相同的消息。
2. 由于诚实性，消息不会重复
3. 由于合法性， 消息不会被破坏，也不是凭空捏造
4. 由于可终止性，消息不会丢失

### Epoch 和 Quorum

对于共识算法来说，都采用了一个弱化的保证，定义了一个世代编号（Epoch）来确保每个世代里面，主节点是唯一的。
如果发现当前主节点失效的情况，节点开始选举新一轮的主节点，Epoch号递增。在主节点做出任何决定的时候，都需要检查是否由比它更高的Epoch号码。
主节点必须从Quorum中收集投票，并且把提议传送到各个节点中，等待节点的返回，当只要没有更高的Epoch主节点
时，才会对当前的提议进行投票。

此处其实是由两个操作组成：1. 选举主节点； 2.对主节点的提议进行投票

注意的一点： 参与两轮的Quorum必须要由重合，这样才能保证主节点没有更高的Epoch，保证正确性。

与2PC的差别， 2PC是需要协调者向每个参与者做出“是”的返回才能进行，而共识算法可以通过直接以集群的多数来确定是否通过决议

### 共识算法的局限性

共识算法虽然为分布式系统带来了好处，为一切不确定的系统带来了安全属性和容错性。并且可以通过全序关系广播以容错的方式实现线性化的原子操作。

但是共识也是有代价的：
1. 节点投票是一个同步复制过程，性能可能需要妥协（但是可以通过减少选举的频率来减少整个开销）
2. 共识体系需要严格的多数节点才能执行，换句话来说，最少需要3个节点才能运行
3. 多数的共识算法假定了一组固定的参与投票的节点集（主要是为了理解的方便）
4. 共识算法需要依靠超时来对节点失败进行检测，但是可能会出现因为网络原因的误判导致经常切主，数据搬移多，对外的性能降低。
5. 网络的影响较大，如果出现频繁切主的情况，可能会有长时间出现无法正常对外进行服务

### 基于共识算法的成员与协调服务

对于像ZooKeeper、Etcd这些分布式键值对的服务，暴露的API与数据库是十分类似的，但是为什么我们需要在共识层上面去构建这些服务呢？

#### 作用
这些分布式键值对数据库主要是针对保存少量，可完全载入内存的数据。采用容错的全序广播算法在所有节点上复制数据使得可以实现高可用的目标。
1. 线性化的原子操作
多个节点同时去获取锁，只有一个会成功，共识协议会保证操作满足原子性和线性化，即使节点出现部分失败的情况
2. 操作全序
之前分布式系统问题的文章有提及过Fencing令牌的问题，
3. 故障检测
客户端会与这些服务保持一个长连接，如果长时间重连失败的情况下，会把该客户端拥有的资源全部释放
4. 更改通知
客户端可以通过读取服务来发现其他的客户端的行为

#### 对外的功能
1. 节点任务分配
计算作业调度系统(Yarn)，分区资源调度（如Multi-Raft中的ShardMaster)
2. 服务发现
解决云环境中服务启停而注册到的服务变更(consul提供的服务)
3. 成员服务
节点是否可用并且获取主节点

### 如何验证一个线性化系统
（此部分会结合Mit6.824和PingCap tikv 的线性化验证的代码阅读和文章来描述）


# <a id="ShareNote">ShareNote</a>
1. [Dynamo](https://pdos.csail.mit.edu/6.824/papers/dynamo.pdf)
2. [Riak](https://github.com/basho/riak_core)
3. [Cassandra](http://cassandra.apache.org/)
4. [数据密集型应用系统设计](https://book.douban.com/subject/30329536/)