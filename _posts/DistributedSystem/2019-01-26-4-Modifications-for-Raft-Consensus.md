## 全局唯一的数据库 ID

### 作用

防止其他集群的成员加入到当前集群中

&nbsp;

## Pre-Vote

### 作用

防止“term inflation”（即一个存在网络问题的机器发起选举，但是没有收到回应，导致 term 一直增长，直到很长一段时间之后才成功当选 leader）。



### 行为

引入 pre-vote 算法，在成员转变为 `candidate` 之前会先发起一次 pre-vote rpc，当这个 rpc 收到多数成员的同意之后才会发起真正的选举。

```
Pre-Vote RPC

arguments:
nextTerm: caller's term + 1
candidateId: caller
lastLogIndex
lastLogTerm

results:
term: currentTerm, for caller to update itself
voteGranted: true means caller would receive vote if it was a candidate

receiver:
1. reply false if last AppendEntries call was received less than election timeout ago(leader stickness)
2. reply false if nextTerm < currentTerm
3. if caller's log is at least as up-to-date as receiver's log, return true
```

另外，对于 pre-vote 来说，不存在同一任期一个成员只能投一次票这样的限制，因为到达真正的投票阶段时，只会有一个 leader。



## Leader stickness

### 作用

防止 leader 的频繁切换。对于 raft 来说这是正确的行为，但是对于应用来说，如果需要频繁的切换访问 leader 会带来很大的问题。



### 行为

收到投票时，在一个 election timeout 之内当前成员曾收到过 AppendEntry RPC，那么它会拒绝这次投票。