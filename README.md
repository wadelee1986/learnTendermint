# learnTendermint

Tendermint Core is Byzantine Fault Tolerant(BFT) middleware that takes a state transition machine and  securely replicates it on many machines.

tendermint的核心是拜占庭容错中间件，通过状态机实现；副本存在多台机器上。


文档地址：https://tendermint.readthedocs.io/en/master/


TendermintCore is a high-performance blockchain consensus engine that enables you to run Byzantine fault tolerant applications, written in any programming language, on many machines spread across the globe, with strong security guarantees

speed: Tendermint blocks can commit to finality in the order of 1 second. TendermintCore can handle transaction volume at the rate of 10,000 transactions per second for 250byte transactions. The bottleneck is in the application.
security: Tendermint consensus is not just fault tolerant, it's optimally Byzantine fault-tolerant, with accountability. When the blockchain forks, there is a way to determine liability.
scalability: Unlike PoW, running parallel blockchains does not diminish the speed or security of each chain, so sharding is trivial. The algorithm can scale to hundreds or thousands of validators (depending on desired block times), and will only get better over time with advances in bandwidth and cpu capacity.



算法：

1、The network is composed of optionally connected nodes. Nodes directly connected to a particular node are called peers.
   网络是由一部分连接的节点组成；
   被连接的节点称为peers

2、The consensus process in deciding the next block (at some height H) is composed of one or many rounds.
  一致性的过程就是决定下一个块是由一轮或多轮组成

3、NewHeight, Propose, Prevote, Precommit, and Commit represent state machine states of a round. (aka RoundStep or just "step").
  NewHeight Propose prevote  precommit  and commit 是一轮的状态，也称为一轮的步骤或步骤；
4、A node is said to be at a given height, round, and step, or at (H,R,S), or at (H,R) in short to omit the step.
  一个节点可以说是一个指定的 高度，轮，和步骤； HRS or HR

5、To prevote or precommit something means to broadcast a prevote vote or precommit vote for something.
  prevote 或precommit 的意思是 广播一个  prevote 票  或 precommit vote 针对一些内容；

6、A vote at (H,R) is a vote signed with the bytes for H and R included in its sign-bytes.
  一票是有签名的针对节点的H和R

7、+2/3 is short for "more than 2/3"
8、1/3+ is short for "1/3 or more"
9、A set of +2/3 of prevotes for a particular block or <nil> at (H,R) is called a proof-of-lock-change or PoLC for short.

一组大于2/3的 prevote  vote  广播消息针对具体的块或nil 在(H,R)上 被称为 lockchange 证明；简写Polc



STATE MACHINE OVERVIEW


At each height of the blockchain a round-based protocol is run to determine the next block. Each round is composed of three steps (Propose, Prevote, and Precommit), along with two special steps Commit and NewHeight.

在每一个区块链的高度上，运行一个基于轮回的协议决定下一个块；每一个轮回都由三个步骤: 1\propose  2\prevote  3\precommit; 并伴随着两个重要的步骤： 4\commit  5\NewHeight

In the optimal scenario, the order of steps is:
完美的情景下， 步骤：

NewHeight -> (Propose -> Prevote -> Precommit)+ -> Commit -> NewHeight ->...
The sequence (Propose -> Prevote -> Precommit) is called a round. There may be more than one round required to commit a block at a given height.
propose -> prevote -> precommit 过程叫做轮回；在一个指定的区块链高度上提交一个块有可能需要多于一次的轮回。

Examples for why more rounds may be required include:
多次轮回提交的例子包括：
The designated proposer was not online.
指定的proposer不在线了
The block proposed by the designated proposer was not valid.
指定的proposer的块的预案 不能用了
The block proposed by the designated proposer did not propagate in time.
指定的proposer的块的预案 过了一段时间不再propagate了
The block proposed was valid,
but +2/3 of prevotes for the proposed block were not received in time for enough validator nodes by the time they reached the Precommit step.
Even though +2/3 of prevotes are necessary to progress to the next step, at least one validator may have voted <nil> or maliciously（坏人） voted for something else.



The block proposed was valid, and +2/3 of prevotes were received for enough nodes,
 but +2/3 of precommits for the proposed block were not received for enough validator nodes.



Some of these problems are resolved by moving onto the next round & proposer.
Others are resolved by increasing certain round timeout parameters over each successive round.


                          State Machine Diagram

                            +-------------------------------------+
                            v                                     |(Wait til `CommmitTime+timeoutCommit`)
                      +-----------+                         +-----+-----+
         +----------> |  Propose  +--------------+          | NewHeight |
         |            +-----------+              |          +-----------+
         |                                       |                ^
         |(Else, after timeoutPrecommit)         v                |
   +-----+-----+                           +-----------+          |
   | Precommit |  <------------------------+  Prevote  |          |
   +-----+-----+                           +-----------+          |
         |(When +2/3 Precommits for block found)                  |
         v                                                        |
   +--------------------------------------------------------------------+
   |  Commit                                                            |
   |                                                                    |
   |  * Set CommitTime = now;                                           |
   |  * Wait for block, then stage/save/commit block;                   |
   +--------------------------------------------------------------------+


BACKGROUND GOSSIP
A node may not have a corresponding validator private key,
but it nevertheless plays an active role in the consensus process by relaying relevant meta-data, proposals, blocks, and votes to its peers.

A node that has the private keys of an active validator and is engaged in signing votes is called a validator-node.
All nodes (not just validator-nodes) have an associated state (the current height, round, and step) and work to make progress.

Between two nodes there exists a Connection,
and multiplexed on top of this connection are fairly throttled Channels of information.
An epidemic gossip protocol is implemented among some of these channels to bring peers up to speed on the most recent state of consensus. For example,

Nodes gossip PartSet parts of the current round's proposer's proposed block.
  A LibSwift inspired algorithm is used to quickly broadcast blocks across the gossip network.
Nodes gossip prevote/precommit votes.
   A node NODE_A that is ahead of NODE_B can send NODE_B prevotes or precommits for NODE_B's current (or future) round to enable it to progress forward.
Nodes gossip prevotes for the proposed PoLC (proof-of-lock-change) round if one is proposed.
Nodes gossip to nodes lagging in blockchain height with block commits for older blocks.
Nodes opportunistically gossip HasVote messages to hint peers what votes it already has.
Nodes broadcast their current state to all neighboring peers. (but is not gossiped further)
There's more, but let's not get ahead of ourselves here.




**********8需要弄明白的事情：
1、proposer
A proposal is signed and published by the designated proposer at each round.
The proposer is chosen by a deterministic and non-choking round robin selection algorithm that selects proposers in proportion to their voting power. (see implementation)

A proposal at (H,R) is composed of a block and an optional latest PoLC-Round < R which is included iff the proposer knows of one.
This hints the network to allow nodes to unlock (when safe) to ensure the liveness property.

2、validater
A node that has the private keys of an active validator and is engaged in signing votes is called a validator-node.
All nodes (not just validator-nodes) have an associated state (the current height, round, and step) and work to make progress.

好文档：http://blog.csdn.net/simple_the_best/article/details/77198837
