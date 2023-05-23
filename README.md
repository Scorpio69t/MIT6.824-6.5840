# 1 Lab2A of MIT6.5840-Distributed-System
MIT6.5840(also early called 6.824) Distributed System in Spring 2023  

#  Lab2A - Leader Election
本Lab2A经过了总共1000次的测试操作，0bug，所以大家可以放心参考

本分支的部分代码，尤其是StartElection方法，参考了MIT公开课Lecture5的内容，
[B站MIT空开课6.824](https://www.bilibili.com/video/BV1qk4y197bB?p=5&vd_source=30a5e5e3e51925b04ff7de1bcc435e3f
)
从第47分钟开始看，讲解的比较好，配合我的csdn笔记食用更佳（可以直接定位到第8节）
[lecture 5 notes for MIT6.824](https://editor.csdn.net/md/?articleId=130708325)

如果大家对go语言还不熟，也可以把前47分钟的内容认真看一遍，重点也在我的那篇csdn笔记中

# 实现流程
## 创建三个raft实例并且分别初始化
1. 初始化的信息包括自己的实例编号，对等节点数组，实例编号就是其索引，初始为0的任期，为-1的投票对象，设置为Follower的状态，
2. 启动时还应该设置心跳周期和一个选举定时器，心跳周期一般是110ms，保证每秒内不超过10次，选举定时器的随机超时时间设置范围在300ms-600ms,一般随机生成，这样能保证在结点数足够的情况下一次选举周期内大概率能选出一个Leader
3. 为每一个raft实例开启一个ticker协程，这个协程用于超时选举以及发送心跳，这里的ticker协程会持续监控所有存活的实例，每过50毫秒会被唤醒一次，主要用来判断节点的状态，并且做出相应的反馈，
如果当前节点为Follower节点或者Candidate且该节点的选举定时器过期，则重置并且试图发起一次选举操作，如果该节点是Leader节点，则会进行一次判断该leader节点的心跳定时器是否过期，如果是就续期操作，
这个时候会发送一个空的AppendEntries给所属的从节点。
4. 选举流程详解
   - 对于发送端：4.1 针对Follower或者Candidate节点发起的选举操作，先加发送端节点的锁，发出RequestVote RPC之前，它会首先将自己置为Candidate状态、重置自己的选举定时器、任期号自增以及给自己投一票，然后开始拉取选票，
   这个会遍历自己的对等节点数组，并且会开启对等节点数量的协程，向对应的节点发送RequestVote RPC
   并且收集选票，如果票数大于总结点数的一半，那么就可以认为自己是leader并且迅速发送一个空数据的
   AppendEntries给所有的从节点，其他的从节点收到了任期号大于自己的心跳就认为这个Leader合法，
   同时执行一些更新操作，包括将自己转变为Follower状态，更新自己的任期号
   - 对于接收端：4.2 先加接收端节点的锁，如果发送端的任期小于自己的则投拒绝票,同时将自己的投票结果和任期返回给Candidate节点，
   如果相反，则投赞成票，接收端重置自己的选举计时器，更新自己的任期.
5. 对于心跳流程
   - 对于发送端：5.1 心跳只有Leader节点能发送，在一个节点首次由Candidate成为Leader
   后会立即发送一个心跳给对等节点，除此之外，raft实例的监控协程也会定期监控，如果发现自己监控的
   raft实例变成了Leader节点，那么它就会在任期内每一个心跳周期内发送一个心跳给所有对等节点；
   这里也是采用开启多个协程并行发送的rpc的。（如果心跳发送过程中丢失了怎么办？）
   - 对于接收端：5.2 接收到Leader节点的心跳后，会比较两个任期，如果Leader的任期小于自己的则直接丢弃并且
   响应false，如果大于等于则更新自己的任期，重置自己的状态为Follower,同时重置选举定时器，将自己的
   的投票状态设置为-1，表示手里有一票没用，因为已经选出了leader

# Q&A
## 1 论文中Figure2的RequestVote RPC及代码段解释：

![img.png](img.png)

重点是尚亮部分：
> If votedFor is null or candidateId, and candidate’s log is at
least as up-to-date as receiver’s log, grant vote

代码体现：
![img_1.png](img_1.png)

Q1: 针对RequestVote rpc的接收方，为什么需要将发送方的candidateId和自己的votedFor进行比较并且结果可以作为是否投票的依据？

答：首先论文中尚亮的部分涉及到日志复制问题，update字段就是表示是否更新自己的日志，因为log replication是Lab2B的工作，所以update这里默认为true，要解决这个问题，
就要看什么时候会出现“rf.votedFor == args.CandidateId”这种情况，一般发送方是leader节点且leader正在发送log AppendEntries RPC时的时候，会出现rf.votedFor == args.CandidateId,
而且这个时候Leader的任期等于Follower的任期，还要看自己投的Leader是不是这个发送方，如果是，则直接更新自己的日志并且刷新自己的选举定时器。

Q2：为什么更新日志的时候从节点也要刷新自己的选举定时器?  

答：因为更新日志的RPC相当于自己的Leader还存活（不然怎么发送RequestAppendEntries），则选举定时器也必须更新一下


## 为什么心跳周期一般是110ms，保证每秒内不超过10次，选举定时器的随机超时时间设置范围在300ms-600ms, 这样能保证在结点数足够的情况下一次选举周期内一定能选出一个Leader？
  

答：在等待选票期间，候选人可能会收到来自另一台声称自己是领导者的服务器的AppendEntries RPC（追加日志条目远程过程调用）。如果领导者的任期（包含在其RPC中）至少与候选人当前的任期一样大，那么候选人将承认领导者的合法地位，并返回到追随者状态。如果RPC中的任期小于候选人当前的任期，则候选人会拒绝该RPC，并继续保持候选人状态。 
第三种可能的结果是候选人既没有赢得选举也没有失败：如果许多追随者同时成为候选人，选票可能会被分割，以至于没有候选人获得过半数选票。当出现这种情况时，每个候选人将超时并通过增加自己的任期并启动另一轮的RequestVote RPC来开始新的选举。然而，如果没有额外的措施，分割选票可能会无限重复。 
Raft算法使用随机化的选举超时来确保分割选票的情况很少发生，并且能够迅速解决。为了首先防止分割选票，选举超时时间会从一个固定的区间（例如150-300毫秒）中随机选择。这样可以使服务器的选举超时时间分散开来，以至于在大多数情况下，只有一台服务器会超时；它会在其他服务器超时之前赢得选举并发送心跳信号。相同的机制也适用于处理分割选票的情况。每个候选人在选举开始时会重新启动其随机选举超时时间，并在该超时时间结束之前等待，然后开始下一轮选举；这样可以降低新一轮选举中再次出现分割选票的可能性。


## raft的代码实现如何利用AppendEntries实现心跳？
答：因为论文中规定AppendEntries为空就表示Leader只是发送心跳，没有包含日志数据，ticker中的定时器体现了这个理论, 因为ticker每间隔50ms就会探测一次节点的状态，如果节点是Leader状态并且探测到该节点的
定时器已经过期，则这个时候必须发送心跳（为什么不采用在发送日志数据时稍带心跳呢？可能是为了减少I/O从而使得心跳更快的达到对方，毕竟心跳定时器过期了，现在要给它快点续期。）

![img_2.png](img_2.png)


## 本Lab的一个大坑是选举定时器和心跳定时器的设置