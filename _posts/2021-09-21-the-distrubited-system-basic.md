---
layout:     post
title:      "分布式系统基本原理"
subtitle:   "The distrubited system basic theory"
date:       2021-09-21 12:00:00
author:     "Reverie"
catalog: false
header-style: text
tags:
  - 分布式
---



# 互斥算法

对于同一共享资源，一个程序正在使用的时候也不希望被其他程序打扰。这，就要求同一时刻只能有一个程序能够访问这种资源。

在分布式系统里，这种排他性的资源访问方式，叫作**分布式互斥（Distributed Mutual Exclusion），而这种被互斥访问的共享资源就叫作临界资源（Critical Resource）。**

## 集中式算法

引入一个协调者程序，得到一个分布式互斥算法。每个程序在需要访问临界资源时，先给协调者发送一个请求。如果当前没有程序使用这个资源，协调者直接授权请求程序访问；否则，按照先来后到的顺序为请求程序“排一个号”。如果有程序使用完资源，则通知协调者，协调者从“排号”的队列里取出排在最前面的请求，并给它发送授权消息。拿到授权消息的程序，可以直接去访问临界资源。

![img](https://cdn.nlark.com/yuque/0/2021/jpeg/12934290/1628218566578-4bd99999-aed8-4e55-8928-6407752b5fca.jpeg)

如图所示，程序 1、2、3、4 为普通运行程序，另一个程序为协调者。当程序 2 和程序 4 需要使用临界资源时，它们会向协调者发起申请，请求协调者授权。

但是临界资源正在被程序 3 使用。这时，协调者根据程序 2 和 4 的申请时间顺序，依次将它们放入等待队列。程序 3 使用完临界资源后，通知协调者释放授权。此时，协调者从等待队列中取出程序 4，并给它发放授权。这时，程序 4 就可以使用临界资源了。

**每个程序完成一次临界资源访问，需要进行 3 次消息交互：**

1. 向协调者发送请求授权信息，1 次消息交互；
2. 协调者向程序发放授权信息，1 次消息交互；

1. 程序使用完临界资源后，向协调者发送释放授权，1 次消息交互。

这个算法的**缺点**：

- **协调者会成为系统的性能瓶颈。**如果有 100 个程序要访问临界资源，那么协调者要处理 100*3=300 条消息。协调者处理的消息数量会随着需要访问临界资源的程序数量线性增加。
- **容易引发单点故障问题。**协调者故障，会导致所有的程序均无法访问临界资源，导致整个系统不可用。



**小结一下：**集中式算法具有简单、易于实现的特点，但可用性、性能易受协调者影响。在可靠性和性能有一定保障的情况下，比如中央服务器计算能力强、性能高、故障率低，或者中央服务器进行了主备备份，主故障后备可以立马升为主，且数据可恢复的情况下，集中式算法可以适用于比较广泛的应用场景。

## 民主协商：分布式算法

当一个程序要访问临界资源时，先向系统中的其他程序发送一条请求消息，在接收到所有程序返回的同意消息后，才可以访问临界资源。其中，请求消息需要包含所请求的资源、请求者的 ID，以及发起请求的时间。

**在分布式领域中，我们称之为分布式算法，或者使用组播和逻辑时钟的算法。**



一个程序完成一次临界资源的访问，需要进行如下的信息交互：

1. 向其他 n-1 个程序发送访问临界资源的请求，总共需要 n-1 次消息交互；
2. 需要接收到其他 n-1 个程序回复的同意消息，方可访问资源，总共需要 n-1 次消息交互。

可以看出，一个程序要成功访问临界资源，至少需要 2*(n-1) 次消息交互。假设，现在系统中的 n 个程序都要访问临界资源，则会同时产生 2n(n-1) 条消息。总结来说，**在大型系统中使用分布式算法，消息数量会随着需要访问临界资源的程序数量呈指数级增加，容易导致高昂的“沟通成本”。**

这个算法可用性很低，主要包括两个方面的原因：

- 当系统内需要访问临界资源的程序增多时，容易产生“信令风暴”，也就是程序收到的请求完全超过了自己的处理能力，而导致自己正常的业务无法开展。
- 一旦某一程序发生故障，无法发送同意消息，那么其他程序均处在等待回复的状态中，使得整个系统处于停滞状态，导致整个系统不可用。所以，相对于集中式算法的协调者故障，分布式算法的可用性更低。

因此，**分布式算法适合节点数目少且变动不频繁的系统，且由于每个程序均需通信交互，因此适合 P2P 结构的系统**。比如，运行在局域网中的分布式文件系统，具有 P2P 结构的系统等。



Hadoop 的分布式文件系统 HDFS 的文件修改就是一个典型的应用分布式算法的场景。

如下图所示，处于同一个局域网内的计算机 1、2、3 中都有同一份文件的备份信息，且它们可以相互通信。这个共享文件，就是临界资源。当计算机 1 想要修改共享的文件时，需要进行如下操作：

1. 计算机 1 向计算机 2、3 发送文件修改请求；
2. 计算机 2、3 发现自己不需要使用资源，因此同意计算机 1 的请求；

1. 计算机 1 收到其他所有计算机的同意消息后，开始修改该文件；
2. 计算机 1 修改完成后，向计算机 2、3 发送文件修改完成的消息，并发送修改后的文件数据；

1. 计算机 2 和 3 收到计算机 1 的新文件数据后，更新本地的备份文件。

![img](https://cdn.nlark.com/yuque/0/2021/jpeg/12934290/1628220724217-e694df89-091e-4779-a24b-a6aa3971aa44.jpeg)

**归纳一下：**分布式算法是一个“先到先得”和“投票全票通过”的公平访问机制，但通信成本较高，可用性也比集中式算法低，**适用于临界资源使用频度较低，且系统规模较小**的场景。

## 轮值 CEO：令牌环算法

所有程序构成一个环结构，令牌按照顺时针（或逆时针）方向在程序之间传递，收到令牌的程序有权访问临界资源，访问完成后将令牌传送到下一个程序；若该程序不需要访问临界资源，则直接把令牌传送给下一个程序。

![img](https://cdn.nlark.com/yuque/0/2021/jpeg/12934290/1628221788127-a00ce8d2-4e9a-44f5-9b1f-02d7af51096d.jpeg)

同时，在一个周期内，每个程序都能访问到临界资源，因此令牌环算法的公平性很好。

但是，不管环中的程序是否想要访问资源，都需要接收并传递令牌，所以也会带来一些无效通信。假设系统中有 100 个程序，那么程序 1 访问完资源后，即使其它 99 个程序不需要访问，也必须要等令牌在其他 99 个程序传递完后，才能重新访问资源，这就降低了系统的实时性。

综上，**令牌环算法非常适合通信模式为令牌环方式的分布式系统**，例如移动自组织网络系统。一个典型的应用场景就是无人机通信。



# 分布式选举

主节点，在分布式集群中负责对其他节点的协调和管理。主节点可以保证其他节点的有序运行，保证数据在每个节点上的一致性。所以主节点如果出现故障，集群就会乱套。

选举的作用就是选出一个主节点，由它来协调和管理其他节点，以保证集群有序运行和节点间数据的一致性。

## 长者为大：Bully算法

Bully 算法是一种霸道的集群选主算法，它的选举原则在所有活着的节点中，选取 ID 最大的节点作为主节点。

初始化时，所有节点都是平等的，都是普通节点，并且都有成为主的权利。但是，当选主成功后，有且仅有一个节点成为主节点，其他所有节点都是普通节点。当且仅当主节点故障或与其他节点失去联系后，才会重新选主。

Bully 算法在选举过程中，需要用到以下 3 种消息：

- Election 消息，用于发起选举；
- Alive 消息，对 Election 消息的应答；

- Victory 消息，竞选成功的主节点向其他节点发送的宣誓主权的消息。

它的假设条件是，每个节点都知道其他节点的ID

1. 每个节点都会判断自己是不是ID最大的，如果是则发起Victory消息宣布主权。
2. 如果不是最大的ID，则向其他ID比自己大的节点发送Election消息，等回复

1. 在给定时间范围内，如果没有收到其他节点回复的ALive消息（也就是比自己ID大的节点都没消息），则认为自己成为主节点，并向其他节点发送 Victory 消息，宣誓自己成为主节点；若接收到来自比自己 ID 大的节点的 Alive 消息，则等待其他节点发送 Victory 消息；
2. 若本节点收到比自己 ID 小的节点发送的 Election 消息，则回复一个 Alive 消息，告诉它我还活着并且我比你大，你就不用多想了。

**小结一下**。Bully 算法的选择特别霸道和简单，谁活着且谁的 ID 最大谁就是主节点，其他节点必须无条件服从。

**优点**是，选举速度快、算法复杂度低、简单易实现。

**缺点**在于，需要每个节点有全局的节点信息，因此额外信息存储较多；其次，任意一个比当前主节点 ID 大的新节点或节点故障后恢复加入集群的时候，都可能会触发重新选举，成为新的主节点，**如果该节点频繁退出、加入集群，就会导致频繁切主**。

## 民主投票：Raft算法

Raft 算法是典型的多数派投票选举算法，其选举机制与我们日常生活中的民主投票机制类似，核心思想是“少数服从多数”。也就是说，Raft 算法中，获得投票最多的节点成为主。

采用 Raft 算法选举，集群节点的角色（状态）有 3 种：

- **Leader**，即主节点，同一时刻只有一个 Leader，负责协调和管理其他节点；
- **Candidate**，即候选者，每一个节点都可以成为 Candidate，节点在该角色下才可以被选为新的 Leader；

- **Follower**，Leader 的跟随者，不可以发起选举。

Raft 选举的流程，可以分为以下几步：

1. 初始化时，所有节点均为 Follower 状态。
2. 开始选主时，所有节点的状态由 Follower 转化为 Candidate，并向其他节点发送选举请求。

1. 其他节点根据接收到的选举请求的先后顺序，回复是否同意成为主。这里需要注意的是，在每一轮选举中，**一个节点只能投出一张票**。
2. 若发起选举请求的节点获得超过一半的投票，则成为主节点，其状态转化为 Leader，其他节点的状态则由 Candidate 降为 Follower。Leader 节点与 Follower 节点之间会定期发送心跳包，以检测主节点是否活着。

1. 当 Leader 节点的任期到了，即发现其他服务器开始下一轮选主周期时，Leader 节点的状态由 Leader 降级为 Follower，进入新一轮选主。



Raft 算法中，选主是周期进行的，包括选主和任值两个时间段，选主阶段对应投票阶段，任值阶段对应节点成为主之后的任期。但也有例外的时候，如果主节点故障，会立马发起选举，重新选出一个主节点。

**小结一下。**

优点：Raft 算法选举速度快、算法复杂度低、易于实现；

缺点：它要求系统内每个节点都可以相互通信，且需要获得过半的投票数才能选主成功，因此通信量大。

该算法选举稳定性比 Bully 算法好，这是因为当有新节点加入或节点故障恢复后，会触发选主，但不一定会真正切主，除非新节点或故障后恢复的节点获得投票数过半，才会导致切主。

## 优先级的民主投票：ZAB 算法

ZAB（ZooKeeper Atomic Broadcast）选举算法是为 ZooKeeper 实现分布式协调功能而设计的。相较于 Raft 算法的投票机制，ZAB 算法增加了通过节点 ID 和数据 ID 作为参考进行选主，节点 ID 和数据 ID 越大，表示数据越新，优先成为主。相比较于 Raft 算法，ZAB 算法尽可能保证数据的最新性。所以，ZAB 算法可以说是对 Raft 算法的改进。

使用 ZAB 算法选举时，集群中每个节点拥有 3 种角色：

- **Leader**，主节点；
- **Follower**，跟随者节点；

- **Observer**，观察者，无投票权。

选举过程中，集群中的节点拥有 4 个状态：

- **Looking 状态**，即选举状态。当节点处于该状态时，它会认为当前集群中没有 Leader，因此自己进入选举状态。
- **Leading 状态**，即领导者状态，表示已经选出主，且当前节点为 Leader。

- **Following 状态**，即跟随者状态，集群中已经选出主后，其他非主节点状态更新为 Following，表示对 Leader 的追随。
- **Observing 状态**，即观察者状态，表示当前节点为 Observer，持观望态度，没有投票权和选举权。

投票过程中，每个节点都有一个唯一的三元组 (server_id, server_zxID, epoch)，其中 server_id 表示本节点的唯一 ID；server_zxID 表示本节点存放的数据 ID，数据 ID 越大表示数据越新，选举权重越大；epoch 表示当前选取轮数，一般用逻辑时钟表示。

ZAB 选举算法的核心是“少数服从多数，ID 大的节点优先成为主”，因此选举过程中通过 (vote_id, vote_zxID) 来表明投票给哪个节点，其中 vote_id 表示被投票节点的 ID，vote_zxID 表示被投票节点的服务器 zxID。**ZAB 算法选主的原则是：server_zxID 最大者成为 Leader；若 server_zxID 相同，则 server_id 最大者成为 Leader。**

**小结一下**。ZAB 算法性能高，对系统无特殊要求，采用广播方式发送信息，若节点中有 n 个节点，每个节点同时广播，则集群中信息量为 n*(n-1) 个消息，容易出现广播风暴；且除了投票，还增加了对比节点 ID 和数据 ID，这就意味着还需要知道所有节点的 ID 和数据 ID，所以选举时间相对较长。但该算法选举稳定性比较好，当有新节点加入或节点故障恢复后，会触发选主，但不一定会真正切主，除非新节点或故障后恢复的节点数据 ID 和节点 ID 最大，且获得投票数过半，才会导致切主。

# 分布式共识

**分布式共识的本质就是“存异求同”，选举的过程就是就是求同的过程。**

**分布式选举问题，其实就是传统的分布式共识方法，主要是基于多数投票策略实现的。**

## 什么是分布式共识？

**分布式共识就是在多个节点均可独自操作或记录的情况下，使得所有节点针对某个状态达成一致的过程。**通过共识机制，我们可以使得分布式系统中的多个节点的数据达成一致。



## 分布式共识方法

Raft 是一种实现分布式共识的协议。

# 分布式事务

事务，其实是包含一系列操作的、一个有边界的工作序列，有明确的开始和结束标志，且要么被完全执行，要么完全失败，即 all or nothing。通常情况下，我们所说的事务指的都是本地事务，也就是在单机上的事务。

而**分布式事务，就是在分布式系统中运行的事务，由多个本地事务组合而成。**在分布式场景下，对事务的处理操作可能来自不同的机器，甚至是来自不同的操作系统。文章开头提到的电商处理订单问题，就是典型的分布式事务。

事务的特征 ACID，也是分布式事务的基本特征，其中 ACID 具体含义如下：

- **原子性（Atomicity**），即事务最终的状态只有两种，全部执行成功和全部不执行。若处理事务的任何一项操作不成功，就会导致整个事务失败。一旦操作失败，所有操作都会被取消（即回滚），使得事务仿佛没有被执行过一样。
- **一致性（Consistency）**，是指事务操作前和操作后，数据的完整性保持一致或满足完整性约束。比如，用户 A 和用户 B 在银行分别有 800 元和 600 元，总共 1400 元，用户 A 给用户 B 转账 200 元，分为两个步骤，从 A 的账户扣除 200 元和对 B 的账户增加 200 元 ; 一致性就是要求上述步骤操作后，最后的结果是用户 A 还有 600 元，用户 B 有 800 元，总共 1400 元，而不会出现用户 A 扣除了 200 元，但用户 B 未增加的情况 (该情况，用户 A 和 B 均为 600 元，总共 1200 元)。

- **隔离性（Isolation）**，是指当系统内有多个事务并发执行时，多个事务不会相互干扰，即一个事务内部的操作及使用的数据，对其他并发事务是隔离的。
- **持久性（Durability）**，也被称为永久性，是指一个事务完成了，那么它对数据库所做的更新就被永久保存下来了。即使发生系统崩溃或宕机等故障，只要数据库能够重新被访问，那么一定能够将其恢复到事务完成时的状态。



分布式事务基本能够满足 ACID，其中的 C 是强一致性，也就是所有操作均执行成功，才提交最终结果，以保证数据一致性或完整性。但随着分布式系统规模不断扩大，复杂度急剧上升，达成强一致性所需时间周期较长，限定了复杂业务的处理。

为了适应复杂业务，出现了 **BASE 理论，该理论的一个关键点就是采用最终一致性代替强一致性。**

## 如何实现分布式事务？

3 种基本方法：

- 基于 XA 协议的二阶段提交协议方法；
- 三阶段提交协议方法；

- 基于消息的最终一致性方法。

基于 XA 协议的二阶段提交协议方法和三阶段提交协议方法，采用了强一致性，遵从 ACID，基于消息的最终一致性方法，采用了最终一致性，遵从 BASE 理论。

### 基于 XA 协议的二阶段提交方法

XA 是一个分布式事务协议，规定了事务管理器和资源管理器接口。因此，XA 协议可以分为两部分，即事务管理器和本地资源管理器。事务管理器作为协调者，负责各个本地资源的提交和回滚；而资源管理器就是分布式事务的参与者，通常由数据库实现，比如 Oracle、DB2 等商业数据库都实现了 XA 接口。



二阶段提交协议（The two-phase commit protocol，2PC）

两阶段提交协议的执行过程，分为投票（voting）和提交（commit）两个阶段。



**投票为第一阶段**，**协调者**（Coordinator，即**事务管理器**）会向**事务的参与者**（Cohort，即**本地资源管理器**）发起执行操作的 CanCommit 请求，并等待参与者的响应。参与者接收到请求后，会执行请求中的事务操作，记录日志信息但不提交，待参与者执行成功，则向协调者发送“Yes”消息，表示同意操作；若不成功，则发送“No”消息，表示终止操作。



当所有的参与者都返回了操作结果（Yes 或 No 消息）后，**系统进入了提交阶段**。在提交阶段，协调者会根据所有参与者返回的信息向参与者发送 DoCommit 或 DoAbort 指令：

- 若协调者收到的都是“Yes”消息，则向参与者发送“DoCommit”消息，参与者会完成剩余的操作并释放资源，然后向协调者返回“HaveCommitted”消息；
- 如果协调者收到的消息中包含“No”消息，则向所有参与者发送“DoAbort”消息，此时发送“Yes”的参与者则会根据之前执行操作时的回滚日志对操作进行回滚，然后所有参与者会向协调者发送“HaveCommitted”消息；

- 协调者接收到“HaveCommitted”消息，就意味着整个事务结束了。



举个例子

![img](https://cdn.nlark.com/yuque/0/2021/png/12934290/1628824770208-a982ca33-c809-45ca-b4ff-041a71efe8ec.png)![img](https://cdn.nlark.com/yuque/0/2021/png/12934290/1628825409723-834190e4-ef66-4748-aaab-353ec9d46f41.png)

商城系统：第一阶段，订单系统将订单数据库锁住，增加一条购买信息，然后回复yes给协调者。但是库存系统库存不足，所以回复no给协调者。第二阶段：协调者收到包含no消息，则向参与者都发送DoAbort消息。于是订单系统回滚，释放资源。订单系统和库存系统完成操作后，向协调者发送HaveCommited消息，表示完成事务的撤销操作。



**二阶段提交的算法思路可以概括为**：协调者下发请求事务操作，参与者将操作结果通知协调者，协调者根据所有参与者的反馈结果决定各参与者是要提交操作还是撤销操作。

**缺点**：

- **同步阻塞问题**：二阶段提交算法在执行过程中，所有参与节点都是事务阻塞型的。也就是说，当本地资源管理器（参与者）占有临界资源时，其他资源管理器如果要访问同一临界资源，会处于阻塞状态。
- **单点故障问题：**基于 XA 的二阶段提交算法类似于集中式算法，一旦事务管理器发生故障，整个系统都处于停滞状态。尤其是在提交阶段，一旦事务管理器发生故障，资源管理器会由于等待管理器的消息，而一直锁定事务资源，导致整个系统被阻塞。

- **数据不一致问题：**在提交阶段，当协调者向参与者发送 DoCommit 请求之后，如果发生了局部网络异常，或者在发送提交请求的过程中协调者发生了故障，就会导致只有一部分参与者接收到了提交请求并执行提交操作，但其他未接到提交请求的那部分参与者则无法执行事务提交。于是整个分布式系统便出现了数据不一致的问题。

### 三阶段提交方法

对二阶段提交方法的缺点进行改进。主要是引入了**超时机制**和**准备阶段**。

- 同时在协调者和参与者中引入超时机制。如果协调者或参与者在规定的时间内没有接收到来自其他节点的响应，就会根据当前的状态选择提交或者终止整个事务。
- 在第一阶段和第二阶段中间引入了一个准备阶段，也就是在提交阶段之前，加入了一个预提交阶段。在预提交阶段排除一些不一致的情况，保证在最后提交之前各参与节点的状态是一致的。

#### 第一，CanCommit 阶段。

CanCommit 阶段与 2PC 的投票阶段类似：协调者向参与者发送请求操作（CanCommit 请求），询问参与者是否可以执行事务提交操作，然后等待参与者的响应；参与者收到 CanCommit 请求之后，回复 Yes，表示可以顺利执行事务；否则回复 No。

CanCommit 阶段不同节点之间的事务请求成功和失败的流程，如下所示。

![img](https://cdn.nlark.com/yuque/0/2021/png/12934290/1628826694446-1a5709ca-224c-4b44-ba5a-31b838511d17.png)

#### 第二，PreCommit 阶段。

协调者根据参与者的回复情况，来决定是否可以进行 PreCommit 操作。

如果所有参与者回复的都是“Yes”，那么协调者就会执行事务的预执行：

- **发送预提交请求。**协调者向参与者发送 **PreCommit** 请求，进入预提交阶段。
- **事务预提交**。参与者接收到 PreCommit 请求后执行事务操作，并将 Undo 和 Redo 信息记录到事务日志中。

- **响应反馈**。如果参与者成功执行了事务操作，则返回 **ACK** 响应，同时开始等待最终指令。

如果有参与者向协调者发送了“No”消息，或者等待超时之后，协调者都没有收到参与者的响应，就执行中断事务的操作：

- **发送中断请求**。协调者向所有参与者发送“**Abort**”消息。
- **终断事务**。参与者收到“Abort”消息之后，或超时后仍未收到协调者的消息，执行事务的终断操作。

![img](https://cdn.nlark.com/yuque/0/2021/png/12934290/1628838792452-6a053a99-9fd0-481f-b21f-d39de77d397e.png)

#### 第三，DoCommit 阶段。

DoCmmit 阶段进行真正的事务提交，根据 PreCommit 阶段协调者发送的消息，进入执行提交阶段或事务中断阶段。

- **执行提交阶段：**

- - 发送提交请求。协调者接收到所有参与者发送的 Ack 响应，从预提交状态进入到提交状态，并向所有参与者发送 DoCommit 消息。
  - 事务提交。参与者接收到 DoCommit 消息之后，正式提交事务。完成事务提交之后，释放所有锁住的资源。

- - 响应反馈。参与者提交完事务之后，向协调者发送 Ack 响应。
  - 完成事务。协调者接收到所有参与者的 Ack 响应之后，完成事务。

- **事务中断阶段：**

- - 发送中断请求。协调者向所有参与者发送 Abort 请求。
  - 事务回滚。参与者接收到 Abort 消息之后，利用其在 PreCommit 阶段记录的 Undo 信息执行事务的回滚操作，并释放所有锁住的资源。

- - 反馈结果。参与者完成事务回滚之后，向协调者发送 Ack 消息。
  - 终断事务。协调者接收到参与者反馈的 Ack 消息之后，执行事务的终断，并结束事务。

![img](https://cdn.nlark.com/yuque/0/2021/png/12934290/1628951988178-494eaa78-86be-4b57-8742-5c75ddf1f96a.png)

在 DoCommit 阶段，当参与者向协调者发送 Ack 消息后，如果长时间没有得到协调者的响应，在默认情况下，参与者会自动将超时的事务进行提交，不会像两阶段提交那样被阻塞住。

### 基于分布式消息的最终一致性方案

解决一致性问题的核心思想就是：将需要**分布式处理的事务通过消息或者日志的方式异步执行**，消息或日志可以存到本地文件、数据库或消息队列中，再通过业务规则进行失败重试。这个案例，就是使用**基于分布式消息的最终一致性方案**解决了分布式事务的问题。

基于分布式消息的最终一致性方案的事务处理，引入了一个消息中间件（Message Queue，MQ），用于在多个应用之间进行消息传递。基于消息中间件协商多个节点分布式事务执行操作的示意图，如下所示。

![img](https://cdn.nlark.com/yuque/0/2021/png/12934290/1628952438525-34977ba7-5c20-42ae-bf03-b6e1934e5c90.png)

以网上购物为例。根据基于分布式消息的最终一致性方案，用户 A 通过终端手机首先在订单系统上操作，然后整个购物的流程如下所示。

![img](https://cdn.nlark.com/yuque/0/2021/png/12934290/1628953140726-98aa0bf2-4acd-48a8-8eb7-6ff3f5af5b60.png)

**分布式事务的一致性是实现分布式事务的关键问题，目前来看还没有一种很简单、完美的方案可以应对所有场景。**



**三种对比如下：**

![img](https://cdn.nlark.com/yuque/0/2021/jpeg/12934290/1628953173578-01d61a52-3fe5-48fc-a1f3-a980daa7cce4.jpeg)

# CAP理论

CAP分别是指**一致性（Consistency）**、**可用性（Availablity）**、**分区容忍性（Partition Tolerance）**，一般分布式系统只能满足其三项中的两项。

### 一致性（Consistency）

一致性是指“**所有节点同时看到相同的数据**”也就是说在更新操作成功并返回到客户端后，所有节点在同一时间的数据完全一致，所有节点所拥有的数据都是最新版本。

### 可用性（Availability）

可用性指的是“**任何时候，读写都是成功的**”，即服务一直可用，而且响应时间在正常范围内。

### 分区容错性（Partition Tolerance）

分区容错性是指“**当部分结点出现消息丢失，或分区故障时，分布式系统仍然能够继续运行**”，即系统容忍网络出现分区，且在遇到某个结点或网络分区之间出现不可达的情况下，仍然能够对外提供满足一致性和可用性的服务。



在分布式系统中，**P的满足是基本要素**，一般是在CP或AP中进行选择，实现更好的C或者提升A的性能。





# BASE 理论

包括基本可用（Basically Available）、柔性状态（Soft State）和最终一致性（Eventual Consistency）。

- 基本可用：分布式系统出现故障的时候，允许损失一部分功能的可用性。比如，某些电商 618 大促的时候，会对一些非核心链路的功能进行降级处理。
- 柔性状态：在柔性事务中，允许系统存在中间状态，且这个中间状态不会影响系统整体可用性。比如，数据库读写分离，写库同步到读库（主库同步到从库）会有一个延时，其实就是一种柔性状态。

- 最终一致性：事务在操作过程中可能会由于同步延迟等问题导致不一致，但最终状态下，数据都是一致的。

可见，BASE 理论为了支持大型分布式系统，通过牺牲强一致性，保证最终一致性，来获得高可用性，是对 ACID 原则的弱化。具体到今天的三种分布式事务实现方式，二阶段提交、三阶段提交方法，遵循的是 ACID 原则，而消息最终一致性方案遵循的就是 BASE 理论。

二阶段和三阶段方法是维护强一致性的算法，它们针对刚性事务，实现的是事务的 ACID 特性。而基于分布式消息的最终一致性方案更适用于大规模分布式系统，它维护的是事务的最终一致性，遵循的是 BASE 理论，因此适用于柔性事务。



# Quorum算法