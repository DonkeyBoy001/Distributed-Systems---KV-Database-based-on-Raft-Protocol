# Distributed-Systems---KV-Database-based-on-Raft-Protocol
A distributed Key-Value (KV) database system was developed using the Raft consensus protocol.



# MIT6.824-lab2

This is the lab2 experimental record of the MIT6.824 distributed system. Under the framework given by the course group, the experiment initially implements the consensus algorithm raft, including leader election, log addition, log persistence, log snapshot and other functions, and provides several interfaces for upper applications to use the raft layer to achieve simple distributed applications (for example, lab3 is a raft algorithm based on lab2 to realize a key-value pair storage service)

> 这是MIT6.824分布式系统lab2实验记录，该实验在课程组所给出的框架下，初步实现了共识算法raft，包括领导人选举、日志追加、日志持久化、日志快照等功能，并提供了几个接口供上层应用可以使用该raft层实现简单的分布式应用（比如lab3就是基于lab2的raft算法 实现了一个键值对存储的服务）



##  Overall structure

> 整体架构

![raft架构](/Users/zhouzhenzhou/Desktop/Distributed-Systems---KV-Database-based-on-Raft-Protocol/MIT6.824-lab2/raft架构.jpg)

##  Implementation details

实现细节

**Node state**: Each node in the raft algorithm has three states, namely leader, follower and candidate. The mutual conversion rules are as follows.

> **节点状态**：raft算法中每个节点都有三种状态，分别是leader，follower，candidate，其互相转换规则

> 如下![图 4 ](https://github.com/maemual/raft-zh_cn/raw/master/images/raft-%E5%9B%BE4.png)

* Under normal circumstances, there is only one leader, the rest are follower, the leader will continue to run until the leader fails to crush or network fluctuations, resulting in the follower become a candidate to run for the new leader

  > 正常情况下，只有一个leader，其余都是follower，leader会持续运行，直到leader出现故障crush或网络出现波动，导致follower变成candidate竞选新leader

* The follower never initiates an RPC request, but only responds to the leader or the RPC of the candidate. If it does not receive any RPC for a period of time (election timeout), it becomes a candidate and initiates an election (if the leader works normally, he will continue to send heartbeat messages in his spare time to ensure its authority)

  > follower从不主动发起RPC请求，而只是响应leader或candidate的RPC，如果它在一段时间内（election timeout）没有收到任何RPC，它就成为candidate，并发起election（leader如果正常运转，会在空闲时间不断发送heartbeat消息保证它的权威）

**Term**: raft uses term as a logical clock, so that each node can distinguish expired information and expired leader.

> **任期Term**：raft将term作为逻辑时钟使用，让每个节点可以分辨过期的信息以及过期的leader

**Log log**: each log in craft stores three pieces of information, a command passed down from the upper application, a term value when the command is received, and a log index index value that grows sequentially, defined as follows, (Index0 stores the log index value at the very beginning of the entire log)

> **日志log**：raft中每个日志都存储了三个信息，一个是上层应用传下来的command，一个是接收到命令时的term值，一个是顺序增长的日志索引index值，定义如下，(Index0存储了整个日志最开始的日志索引值)

```go
type Log struct {
	Entries []Entry
	Index0  int
}

type Entry struct {
	Command interface{}
	Term    int
	Index   int
}
```

As the implementation of the log snapshot, index0 will continue to change, the log cut, the length of the log, take the last log and other operations will become more complex, so in log.go encapsulated a series of functions for the log, such as cutend () function: cut the end of the log

> 由于在实现日志快照后，index0将会不断变化，日志的裁剪、求日志的长度、取最后一个日志等等操作都会变得较为复杂，故在log.go中封装了一系列针对日志的函数，比如cutend()函数：裁剪尾部日志

```go
func (l *Log) cutend(idx int) {
	l.Entries = l.Entries[0 : idx-l.Index0]
}
```

**Leader Election**

> **领导人选举**

* Follower adds his current term and switches to candidate

* It votes for itself and calls Request Vote RPC to let other servers vote (voters should record the term of the campaign to their current term)

  * Judgment rules: Use the term and idx of the last entry in the log to determine which update, the larger term update, and the longer update of the same term.
  * Voting rules: If the voter's own log is newer than candidate, it will refuse to vote.

* If a candidate is the first to win the vote of most servers of the same term, it becomes a leader, and once it becomes a leader, immediately send a heartbeat message to all servers, declare authority and stop other campaigns.

* If a candidate receives an Append Entries RPC claiming to be a leader during the campaign

  * :: The "leader" has a lower term than the candidate: the candidate rejects the request and continues the campaign
  * The "leader's" term is not less than Candidate's: Candidate abandons the campaign and moves to Follower.

* If multiple candidates split the vote, resulting in no leader being selected, each candidate times out and increases the current term to restart another round of RequestVote RPC

  * raft uses a random election timeout to ensure that this rarely happens: each server's election timeout event is randomly selected from an interval, using a decentralized approach to deal with this vote-splitting scenario

    ```go
    //重置选举超时时间：350~700ms
    func (rf *Raft) resetElectionTimer() {
    	t := time.Now()
    	t = t.Add(350 * time.Millisecond)
    	ms := rand.Int63()%350
    	t = t.Add(time.Duration(ms) * time.Millisecond)
    	rf.electionTime = t
    }
    ```

**Log append**

> **日志追加**

* The leader receives the client's request and puts these commands in his own log, and then calls Append Entries RPC in parallel to send these log entries to all followers.

* After receiving the confirmation of most followers, the leader executes (committed) the entry and apply. At the same time, in each Append Entries RPC, it will pass its commitindex to others. Followers so that they can also update their commitindex

  ```go
  // Update follower's commitindex and try to wake up the applier thread
  if args.LeaderCommit > rf.commitIndex {
  	rf.commitIndex = min(args.LeaderCommit, rf.lastLogIndex())
  	rf.applyCond.Broadcast()
  }
  ```

* The leader will maintain a nextidx for each follower, which is the index of the next log entries that the leader will send to this follower.

  * When the leader is just selected, the nextidx of each follower is the index after the leader's last log.

  * The log entries sent by the leader to the follower contain prelogindex and prelogterm, pointing to the previous log that the leader wants to send the log. If the follower finds the previous log and lea If der is different, then AppendEntries RPC fails.

  * The leader then modifies the nextidx of the follower to reduce it, and then repeats step two until the previous log of the follower matches the previous log that the leader wants to send.

    * Optimization: when the follower returns a failed call to AppendEntries, it is accompanied by Xterm, Xindex, and Xlen to help the leader locate the first log under the term value where the conflict occurred

      ```go
      	for xIndex := args.PrevLogIndex; xIndex > rf.log.Index0 ; xIndex-- {
      	if rf.log.at(xIndex-1).Term != xTerm {
      		reply.XIndex = xIndex
      		break
      	}
      }
      ```

  * The follower then deletes all the logs after the successful match, and stores the logs he sent to him.

Log persistence

> 日志持久化

This part is relatively simple. When any of the three states that need to be persisted changes, you can immediately call `persist()` to save the raftstate.

Among them, currentTerm saves the current term value of the leader, votedFor saves the vote selection of the follower, and log saves the log of the node, such as commitindex, lastapplied, nextindex[ ], matchindex[] and other states can be gradually adjusted and restored in the sending and return of Append Entries RPC through the saved log, so there is no need to save

**Log snapshot**

> **日志快照**

* In order to prevent the log from growing longer and longer, snapshot is used to snapshot some early logs.

* In addition to application data information, the snapshot content also includes some metadata information: such as the term and index contained in the last log in the snapshot.

* Once the system persists the snapshot to the disk, it can delete the log and snapshot before the last log of the snapshot.

* Sometimes, the progress of the follower is too slow, and the leader has deleted the log that should be sent to it. At this time, the leader must send a snapshot to the follower through installsnapshot RPC.

  * If this snapshot contains log information that the follower does not have (that is, the log progress in the snapshot is faster than all the log progress of the follower), the follower will receive the snapshot and update it to its own snapshot information.

  > For this implementation, I chose follower to not directly delete the logs contained in the snapshot when receiving the installsnapshot RPC, but to actually trim the logs only after the upper tier application calls `CondInstallSnapshot` to determine the successful application of the snapshot
  >
  > Because in the implementation logic, the log information contained in the snapshot on the raft node is only invalidated when the upper-level application calls `CondInstallSnapshot` to determine that the snapshot can be installed.

This part is more cumbersome, involves a lot of judgment and operation on the log, because in this part of the log Index0 will also change, so also need to reconstruct the previous log append function, including the follower log for the empty processing, including the leader log for the empty processing and so on.

I chose to create the node at the very beginning, in the log at subscript 0 to save an empty log (to facilitate the upper application has not yet received commands, you can successfully implement the heartbeat packet logic sent by the leader), and in the log for the snapshot and cut, the log at subscript 0 is a valid log, there are two scenarios

* The log is not empty; this condition does not affect the processing of heartbeat packet logic
* Log is empty, in this case, I choose to persist lastsnapshotIndex, lastsnapshotTerm, use these two variables on behalf of the heartbeat packet logic processing of the function of prevLogIndex, prevLogTerm



# MIT6.824-lab3

This is a transcript of the MIT 6.824 Distributed Systems lab3 experiment, which implements a basic key-value pair storage service based on the craft of lab2 within the framework given by the course team

> 这是MIT6.824分布式系统lab3实验记录，该实验在课程组所给出的框架下，基于lab2的raft实现了一个基础的键值对存储服务

## Overall structure

> 整体架构

![kvraft架构](/Users/zhouzhenzhou/Desktop/Distributed-Systems---KV-Database-based-on-Raft-Protocol/MIT6.824-lab3/kvraft架构.jpg)

## Realization details

**client端**

* Implement the task timeout mechanism on the client side to simplify the design of the server side
* Every time a request is sent on the client side, it is sent in parallel to improve the efficiency of the request task.

* Use channels for the transmission of tasks between threads, and each Task structure contains a resultCh channel to return the results, which is convenient to return the corresponding correct results to Task.

**server端**

* On the server side, clientLastTaskIndex is used to save the subscript of the last task completed by each client, which is used to determine whether the task sent by a client has been completed, and if it has been completed, the historical results are returned.
* use Get, PutAppend function to receive the task, and use the cond condition variable to sleep and wait for the task to be sent to the craft layer to complete the replication of apply to the server, in the applier to execute the craft layer to commit and apply the command, wake up the previously sleeping thread and return results
