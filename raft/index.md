# Raft学习总结 #

Raft是Paxos之后的另一种**分布式一致性协议**。当今世界上所谓的其他的分布式一致性协议都是这两种协议的变种。Raft相对Paxos来说，更加易于理解和实现。

本人才疏学浅，难免有理解不正确的地方，请各位路过的大牛指正~

## Raft概念 ##

![](https://github.com/ridge-chen/ridge-chen.github.io/raw/master/raft/images/state-machine.jpg)

1. Raft把成员分为**Leader**，**Fellower**，和**candidate**。由Leader统一接收用户请求和发送响应。
2. 其中candidate只有在选举的过程中出现。在非选举的过程中，只有Leader和Fellower。
3. Leader接受用户请求后产生日志项，并发送给其他成员。在大多数成员接受日志后，确认日志是**committed**，应用于本地状态机并发送用户响应。在后续的消息中通知其他成员这条日志是committed。
4. 所有成员都本地维护一份日志，并由Leader来通知哪些日志是**committed**，并应用于本地状态机。


## Raft核心 ##

**保证分布式日志的一致性**

有一些细节需要说明：

- 日志在各个成员上并不是完全一致，但是**committed的日志链**是完全一致的。committed的日志链最终都会被传递到每个成员，因此committed的日志可以安全地应用于本地状态机。
- 一条日志是否是committed，由Leader来发现（我这里用了发现，而不是决定）。只要满足下面条件的日志就是committed，即使这时重新做了Leader选举并更换了Leader。
  - **这条日志由当前Leader在当前任期内生成，并在当前任期内成功分发到大多数成员**
- 另外，committed的日志前的所有日志也是committed，在Leader切换过程中，有可能出现这种情况。通常，新任Leader都会提交一个空的日志项来加快处理前任Leader产生的，但是还没有committted的日志。
- Leader将日志的committed信息发送给其他成员


为保证上面提到的分布式日志的一致性，Raft在一些方面作出了限制，以下分别阐述。

## 日志分发 ##

![](https://github.com/ridge-chen/ridge-chen.github.io/raw/master/raft/images/log-commit.jpg)

每个成员都保存有一份有序本地日志链，同时也保存了commitIndex。这个变量表示到这个日志为止的所有日志都是committted，可以安全提交到本地状态机。

Leader在处理用户请求过程中生成日志项，并发送给其他成员。当大部分成员都接受该项日志后，日志项成为committed。Leader会应用该项日志到本地状态机，并把结果发送给用户。其他成员会在未来的通信中收到committed的通知并应用于本地状态机。

Leader并不需要立即通知其他成员，因为即使之后出现Leader切换，或者小部分成员下线，新的Leader也会包含该项日志并最终发送给所有成员。这部分会在Leader的选举中说明。

## Leader选举 ##
由于在Leader选举后，新的Leader保存的日志将成为权威日志并被延续下去，所有Fellower保存的不一致的日志都将会被抛弃。为了保证“日志一致性”这个原则，Raft在Leader的选举中对新的Leader提出了要求，以确保新的Leader包含所有committed的日志。
具体的原则是：

**- 当一个节点收到Candidae的RequestVote请求时，会对比自己和Candidate的日志新旧程度，只有当Candidate的日志不比自己旧时，才会投票。**

由于任何已提交的日志项都是已经被大部分节点接受的，收到大部分节点投票的节点Canidicate必然拥有最新的日志项。

## 日志compact ##

![](https://github.com/ridge-chen/ridge-chen.github.io/raw/master/raft/images/snapshot.jpg)

随着系统的运行，日志量会越来越大，Raft采用最简单的一种方案：**快照**。在快照中，Raft保存了状态机的状态和对应的日志项的位置信息。这样，快照之前的日志项可以被清除了。


## 参考 ##

[Raft论文](https://raft.github.io/raft.pdf)

论文中给出了基于RPC的详细实现方式，以及Raft协议的完整证明。另外还有相对Paxos的更易于理解的评估，Raft Leader选举的性能测试等。

