---
title: Raft
comments: true
tags:
  - GAP
  - Go
  - MIT
  - 6.824
  - raft
categories: Sys
abbrlink: c51c
date: 2020-03-16 13:59:44
---

结合 MIT-6.824-2018 的 lab2 来简单写下学习raft的笔记。<!--more-->

Raft是一种共识算法，它在容错和性能上与Paxos等效，但更容易理解学习。可以通过 http://thesecretlivesofdata.com/raft/ 简单认知下Raft共识算法。

共识是容错分布式系统中的一个基本问题，共识涉及多个服务器就价值达成一致。共识通常出现在复制状态机的背景下，每个服务器都有一个状态机和一个日志，每个状态机都从其日志中获取输入命令。以哈希表为例，日志将包含诸如 set x=3 之类的命令，而共识算法要确保所有状态机都 x=3 作为第n个命令应用。即每个状态机处理相同系列的命令，从而产生相同系列的结果并到达相同系列的状态。

# Raft Paper

Raft 选举出一个leader来管理复制日志：leader从客户端接收日志条目，将其复制到其他服务器上，并在保证安全的时候应用日志条目到他们的状态机中。由此，Raft 将一致性问题分解成三个子问题：**Leader Election**, **Log Replication**, **Safety**（所有状态机都应执行相同的命令序列）

## Raft Properties

Raft 在任何时候都保证以下的各个特性。

* **Election Safety**：一个给定term最多能选出一位leader
* **Leader Append-Only**：leader不能删除或修改自身log中entries，只能追加新entries
* **Log Matching**：如果两个log在某一index处的entries具有相同term，则这两个log在该index前的所有entries相同。 
* **Leader Completeness**：一个committed的entry会始终存在于之后terms的leaders的log中。
* **State Machine Safety**：若一个给定index的entry已经被某台服务器apply，则不会有服务器会在其log的index位置apply别的entry的。

## Raft Basics

在任何时刻，每一个服务器节点都处于这三个状态之一：leader、followers、candidates。
通常系统中只有一个leader且其他节点全都是followers，在一个任期内，leader一直保持，直到自己宕机。Followers都是被动的：他们不会发送任何请求，只是简单的响应来自leader / candidates的请求。
Leader处理所有的客户端请求（如果一个客户端和followers联系，那followers会把请求重定向给leader）
![Go与分布式_Raft_states.png](https://i.loli.net/2020/02/21/ANVsJ8aCyM5KifD.jpg)

Raft 把时间分割成任意长度的**任期terms**。任期用连续的整数标记。每一段任期从一次**选举**开始。

每一个节点存储一个当前任期号current term，这一编号在整个时期内单调的增长。当服务器之间通信的时候会交换current term；如果一个服务器的current term比其他人小，那么他会更新到较大的编号值。Raft 算法中服务器节点之间通信使用远程过程调用（RPCs），并且基本的一致性算法只需要两种类型的 RPCs。请求投票（RequestVote） RPCs 由候选人在选举期间发起，然后附加条目（AppendEntries）RPCs 由领导人发起，用来复制日志和提供一种心跳机制。

### Leader election

Raft 使用一种heartbeat机制来触发领导人选举。当服务器程序启动时，各自都是followers。Leader会周期性的向所有followers发送心跳包 (AppendEntries RPCs that carry no log entries) 来维持自己的权威。Followers只要收到了来自 leader / candidates 的有效RPCs就会一直保持followers。但如果一个follower在一段时间（random，一般在150ms-300ms）里没有接收到任何消息，即**选举超时**，那他会认为系统中无可用leader，并发起选举以选出新的leader。

要开始一次选举过程，followers先要current term++且转换到candidates，然后并行的向集群中的其他服务器节点发送RequestVote的 RPCs 来给自己投票。Candidates会继续保持当前状态直到以下三件事情之一发生：

* *(a) **it wins the election**：*candidate从整个集群的大多数服务器节点获得了针对同一term的选票。每个服务器按先来先服务原则在每个term最多投一张选票。一旦candidate赢得选举，则立即成为leader并向其他的服务器发送heartbeat来建立自己的权威并且阻止产生别的leader。

* *(b) **another server establishes itself as leader**：*等待投票时candidate收到别的服务器leader声明的AppendEntries RPC，如果该RPC中leader的term >= candidate的term，则candidate承认该leader并回到follower；否则candidate会拒绝该RPC并保持candidate 。

* *(c) **a period of time goes by with no winner**：*多个followers同时成为candidates来瓜分选票，导致无人能赢得大多数人的支持。为了避免瓜分次数过多，每个candidate在发RequestVote RPCs时都会设置一个**随机的**等待时间，若超时则重新发送RequestVote RPCs 。这样每次瓜分后，所有candidates都会超时，current term++开启新一轮选举。

### Log replication

客户端只和leader交互，leader把客户端指令作为一条log entry附加到自己log中，然后并行的发起 AppendEntries RPCs 给 followers，让他们复制这条entry。当这其被**安全的**复制，leader会将这条entry内容应用在自身状态机上并将其commit，同时return result给客户端。接下来leader会通知followers该entry已经committed，followers将其应用到各自状态机，完成各自log上该entry的commit。leader会不断的重复尝试 AppendEntries RPCs （尽管已经回复了客户端）直到所有的followers都最终存储了所有的日志条目。

![Go与分布式_Raft_log.png](https://i.loli.net/2020/02/28/n4uPxsZJC8eYlgA.jpg)

> 日志由有序序号标记的条目组成。每个条目都包含创建时的任期号（图中框中的数字）和一个状态机需要执行的指令。一个条目当可以安全的被应用到状态机中去的时候，就认为是可以提交了。

在leader将创建的日志条目复制到大多数的服务器上的时候，此时可以认为该 log entry 应用到状态机中是安全的，即该 log entry is **committed**（如图中条目 7）。同时，leader的日志中之前的所有日志条目也都会被提交，包括之前所有terms里的条目。Leader跟踪 committed entries 的 highest index（例如图中为7），并将其写进未来所有的 AppendEntries RPCs 好让 followers 清楚。Followers会将 committed entries 按 log 顺序应用到本地的状态机中。

Raft 的日志机制维护了不同服务器的日志之间的高层次的一致性，同时也组成了上文中的日志匹配特性：

- 不同log中的两个entry拥有相同index&term -> 他们存了相同的指令且之前的所有entries也全部相同。

*理由：leader会把前一个entry的index和term写进当前AppendEntries RPC中，如果followers发现在自己的log中找不到符合的entry，就会拒收这个新entry以保证其log和leader的log顺序一致。*

正常操作中，logs肯定会保持一致性。但存在这种情况，leader在自己log中写入了一堆entries但在这些entries还没全部committed时leader就先自己崩溃了，下一term它作为follower，就会出现和当前leader的log不一致情况。Raft的解决办法是直接把followers冲突的那部分entries覆盖成leader的对应entries。

Leader针对每一个follower维护了一个 **nextIndex**（即将要发给followers的entry的index）当一个leader刚获得权力的时候，初始化所有的 nextIndex  = leader log highest index + 1。如果出现AppendEntries RPC被followers拒绝的情况，leader会让 nextIndex-- 并重试。最终 nextIndex 会停在两者log一致的点，此时的AppendEntries RPC将不再被拒绝，即可followers冲突的entries全部删除并逐个追加leader的entries。通过这种机制，leader在获得权力的时候就不需要任何特殊的操作来恢复一致性。

### Safety

我们需要保证每个状态机都会按照相同的顺序执行相同的指令。但如果出现，一些entries被大部分followers接受并commit，但存在一个follower处于unavailable没有接受这些entries，但偏偏该follower在下一term成为了leader，这就会导致那些committed entries明明执行了却被强制删除，即不同状态机执行不同的指令序列。

Raft通过在leader election中增加一些限制来保证：任何leader对于给定的term，都拥有之前terms里所有的committed entries (即上文提到的 the Leader Completeness Property)

在raft的投票环节，每个candidates发出的RequestVote RPC都会包含其last log entry的term和index，followers会根据这些信息拒绝掉没有自己新（term越大越新，term相同index越大约新）的投票请求。这样选出来的leader至少和大部分followers一样新，而entries是在大部分followers都复制后才会commit，即committed entries存在于大部分followers上，如果能和大部分followers一样新，就能保证存储了所有的committed entries。

![Go与分布式_Raft_crash.png](https://i.loli.net/2020/02/29/7HLPm9AKQG54wvR.jpg)

Leader不能断定之前term中保存到大部分服务器的entry是否已经commit，如图，S1作leader，开始复制entry_2，复制了小部分followers后换leader了。S5作leader，没有entry_2且刚接受到客户端的entry_3，继续换leader。S1作leader，继续复制那条entry_2并成功复制到大部分followers上，但在entry_2即将commit前，又被换leader。若S5作leader（因为entry_3的term>entry_2的term，所以能收到S2,S3,S4的选票），此时entry_3会覆盖所有的entry_2；若S1继续作leader，则S1的entries都会被提交。

> leader复制之前term的entries时，这些entries的term不变（图c复制entry_2时，其term依旧为2 ）

由此，对于之前term中的entries，leader不会通过计算其副本数目的方式去commit，事实上leader只会对当前term的entries用计算副本数目的方式来commit。而当前term的entries被commit，由于日志匹配特性，之前的entries都会被间接commit。

## Condenced Summary

![Go与分布式_Raft_summary.png](https://i.loli.net/2020/02/20/NzSX6Eq8yjUhPem.png)

### AppendEntries RPC

由leader负责调用来复制日志指令；也会用作heartbeat，receiver实现：

1. 如果 `term < currentTerm` 就返回 false 
2. 如果日志在 prevLogIndex 位置处的 entry 和 prevLogTerm 不匹配，则返回 false 
3. 如果已经存在的日志条目和新的产生冲突（index相同但term不同），删除这一条和之后所有的 entries
4. 附加日志中尚未存在的任何新条目
5. 如果 `leaderCommit > commitIndex`，令 commitIndex = MIN(leaderCommit, 新日志条目索引值)

### RequestVote RPC

由candidates负责调用来征集选票，receiver实现：

1. 如果`term < currentTerm`返回 false 
2. 如果 votedFor 为 null / candidateId，且candidate的日志至少和receiver一样新，那就投票给他

### Rules for Servers

#### All Servers

- 如果`commitIndex > lastApplied`，那就 lastApplied ++，并把`log[lastApplied]`应用到状态机中
- 如果接收到的RPC请求或响应中，term`T > currentTerm`，则令`currentTerm = T`，并切换状态为followers

#### Followers

- 响应来自leader和candidates的RPCs
- 如果在选举超时前一直没有收到leader或candidates的RPCs，就自己变成candidate

#### Candidates

- 在转变成candidates后就立即开始选举过程
  - 自增当前的任期号（currentTerm）
  - 给自己投票
  - 重置选举计时器
  - 发送请求投票的 RPC 给其他所有服务器
- 如果接收到大多数服务器的选票，那么就变成领导人
- 如果接收到来自新的领导人的 AppendEntries RPC，转变成跟随者
- 如果选举过程超时，再次发起一轮选举

#### Leader

- 一旦成为领导人：发送空的AppendEntries RPC (heartbeat) 给其他所有的服务器；在一定的空余时间之后不停的重复发送，以阻止followers超时
- 如果接收到来自客户端的请求：附加条目到本地日志中，在条目被应用到状态机后响应客户端
- 如果对于一个跟随者，最后日志条目的index >= nextIndex，则发送从 nextIndex 开始的所有日志条目：
  - 如果成功：更新相应 followers 的 nextIndex 和 matchIndex
  - 如果因为日志不一致而失败，减少 nextIndex 重试
- 如果存在一个N满足`N > commitIndex`，且大多数的`matchIndex[i] ≥ N`成立，并且`log[N].term == currentTerm`成立，则令`commitIndex = N`

## Cluster membership changes

实际操作中偶尔会改变集群的配置，但服务器直接从旧配置转换到新配置很容易出现分歧。为避免存在新旧配置在同一时间下出现各自leader，即同一时刻新旧配置同时生效的情况，配置更改必须使用两阶段方法。在 Raft 中，集群先切换到一个过渡的配置，称之为共同一致；一旦共同一致被commit，那系统就切换到新配置上。共同一致是老配置和新配置的结合：

- 日志条目会被复制给集群中新、老配置的所有服务器。
- 新、旧配置的服务器都可以成为leader。
- 达成一致（针对选举和提交）需要分别在两种配置上获得大多数的支持。

当leader接收到一个改变配置从 C-old 到 C-new 的请求，他会创建configuration entry: C-old,new来存储配置，并复制给followers。各个服务器总是用latest configuration entry作为自己的配置，无论他是否committed。

一旦 C-old,new 被提交（C-old，C-new双方大部分都复制成功），那么无论是 C-old 还是 C-new，在没有经过对方批准的情况下都不可能做出决定，并且领导人完全特性保证了只有拥有 C-old,new 日志条目的服务器才有可能被选举为领导人。这个时候，领导人创建一条关于 C-new 配置的日志条目并复制给集群就是安全的了。

在关于重新配置还有三个问题需要提出：

* 新的服务器可能初始化没有存储任何的日志条目。此时他们需要时间来更新追赶，而没法提交新的条目。Raft 在配置更新之前使用了一种额外的阶段：该阶段里，新的服务器加入，接收leader的entries但无投票权，直到追赶上了集群中的其他机器。

* 当前leader不在C-new中，即当前leader节点即将下线。在C-new被commit之后Leader实际已经从集群中脱离，会退位到follower状态，此时可以对Leader节点进行下线操作，而新集群则会在C-new的配置下重新选举出一个Leader。

*  移除不在 C-new 中的服务器可能会扰乱集群。这些服务器因为不在C-new里，所以leader根本不会给他们发心跳，之后这些节点就会选举超时，发送term++的请求投票 RPCs，这样会导致当前leader回退成follower。为了避免这种情况，除非leader超时，不然各节点拒收投票请求RPCs。同时在每次选举前等待一个选举超时时间，这样每次旧节点在发起选举前需要等待一段时间，那这段时间新Leader可以发送心跳，减少影响。 

## Log compaction

Raft 需要一定的机制去清除日志里积累的陈旧的信息，快照Snapshotting是最简单的压缩方法。在快照系统中，整个系统的状态都以快照的形式写入到稳定的持久化存储中，然后到那个时间点之前的日志全部丢弃。
![Go与分布式_Raft_snapShot.png](https://i.loli.net/2020/03/01/xXltDZv8yfmi1n6.jpg)
每个服务器独立的创建快照，快照只包括committed entries。Raft 包含状态机状态和少量元数据（last index&term）到快照中，保留元数据是为了支持快照后第一个条目的附加日志请求时的一致性检查，因为这个条目需要前一日志条目的索引值和任期号。为了支持集群成员更新，快照中也将最后的一次配置作为最后一个条目存下来。一旦服务器完成一次快照，他就可以删除最后索引位置之前的所有日志和快照了。

但当followers运行缓慢或作为C-new刚加入集群时，需要leader发送快照让他们更新到最新状态。这种情况下leader使用 InstallSnapshot RPCs 来发送快照。当followers通过这种 RPC 接收到快照时，会决定如何处理log：通常快照会含当前log没有的信息，此时会丢弃整个log，用快照代替；也可能接收到的快照是自己日志的前面部分（网络重传或者错误），那么被快照包含的entries会被全部删除，但快照后面的entries仍然有效，必须保留。
![Go与分布式_Raft_snapShotRPC.png](https://i.loli.net/2020/03/01/gbPEGHk894xwdID.png)

## Client interaction

Raft 要求客户端只和leader交互，所以当客户端请求到follower时，如果集群里存在leader，follower会将leader地址发给客户端；如果此时集群里无leader，客户端会等待、重试。

Raft 的目标是要实现线性化语义(each operation appears to execute instantaneously, exactly once, at some point between its invocation and its response)。但如果，leader在commit该entry之后，回复客户端之前crash，那么客户端会和新leader重试这条指令，导致这条命令就被再次执行。解决方案就是客户端对于每一条指令都赋予一个唯一的序列号。然后，状态机跟踪每条指令最新的序列号和相应的响应。如果接收到一条指令，它的序列号已经被执行了，那么就立即返回结果，而不重新执行指令。

只读操作可以直接处理而不用写进日志。但在不增加任何限制的情况下，这么做可能会返回脏数据，因为leader可能在response客户端时被退回成follower，此时原先leader的信息已经变脏。线性化的读操作必须不能返回脏数据，Raft 需要使用两个额外的措施在不使用日志的情况下保证这一点。

* Leader必须有关于被提交日志的最新信息。在leader任期开始的时候，他可能不知道哪些是已经被提交的。为了知道这些信息，他需要在他的任期里提交一条日志条目。Raft 中通过leader在任期开始时commit一个空白entry到日志中去来实现。
* Leader在处理只读的请求前必须检查自己是否已经被废黜了。Raft 中通过leader在响应只读请求之前，先和集群中的大多数节点交换一次心跳信息来处理这个问题。