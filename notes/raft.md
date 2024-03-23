# RAFT 学习笔记
## 节点状态
### 通用持久状态
- currentTerm: 节点最新任期
- voteFor: 本任期内票选的Candidate索引，-1代表还没投票，leader应当投给自己
- log[]: 日志条目

### 通用易失状态
- commitIndex: 本节点已提交的最新日志索引
- lastApplied: 本节点已被应用到状态机的最新日志索引

### Leader特有易失状态
- nextIndex[]: leader要发送给follower的下一条日志的索引
- matchIndex[]: leader已知的已经复制到follower的最新日志索引

## RPC
### RequestVote (Candidate -> Other)
#### 请求参数
- term: 候选人所处任期号
- candidateId: 候选人id
- lastLogIndex: 候选人最新日志索引
- lastLogTerm: 候选人最新日志任期号

#### 回复参数
- term: 接收方最新任期号
- voteGranted: 接收方是否投票

#### 发送方(Candidate)逻辑
1. currentTerm++，任期号自增1
2. voteFor = me，给自己投票
3. 持久化状态
4. 发送请求投票RPC
------
5. 收到回复后，判断reply.term是否大于自己的currentTerm
1. 若大于则进入新任期，更新currentTerm、voteFor=-1、变为follower，持久化，结束
2. 若不大于（就应该等于），判断是否给自己投票，若不投票则结束
3. 若给自己投票，统计选票计数+1，若选票大于节点数量的一半，成为leader，否则结束
4. 成为leader后，对所有节点发送空日志

#### 接收方(Other)逻辑
1. 若请求方任期号小于自身任期号，拒绝投票，并返回自身较大的任期号，结束
2. 若请求方任期号较大或相等，进入是否投票的判断逻辑：
   1. 若voteFor不为-1且不与请求的candidateId相等，不投票，结束
   2. 若voterFor为-1或与请求的candidateId相等，需要进一步判断双方的日志谁更新：
      1. 若请求的lastLogTerm更大，说明请求方日志更新，投票
      2. 若请求的lastLogTerm与本节点最新日志term相等，判断请求中lastLogIndex与本节点最新日志索引
      3. 若请求方lastLogIndex大于等于本节点最新日志索引，说明请求方日志超前或同步，投票
      4. 否则，不投票，结束
3. 投票：自身voteFor=args.candidateId，持久化
4. 向请求方回复投票成功
5. 更新触发选举超时的时间点

### AppendEntry (Leader -> Follower)
#### 请求参数
- term: 领导者当前任期号
- leaderId: 领导者id
- prevLogIndex: 领导者紧邻添加日志之前的日志索引(nextIndex[i]前一个日志)
- prevLogTerm: 紧邻添加日志之前的日志任期号
- entries[]: 添加的日志索引
- leaderCommit: 领导者已提交的最新日志索引

#### 回复参数
- term: follower 当前任期
- success: prevLogIndex和prevLogTerm是否能与follower匹配上
 
#### 发送方(Leader)逻辑
一般leader会周期性地为每个非自己的节点发送该RPC，若有可发送的日志（lastIndex >= nextIndex[i]）则发送可发送的日志，若没有则发送空日志数组，充当心跳。  
对每个非自己的节点：  
1. 若 lastLogIndex >= nextIndex[i]，则提取日志数组作为参数，否则使用空数组
2. 异步发送RPC，异步处理回复，这样可以同时向所有节点发送而不被阻塞
----------
3. 若回复的term > currentTerm，说明自己落后了，应当更新任期并成为follower，结束
4. 若回复成功，说明发送的prevLogIndex和prevLogTerm与follower匹配上了，这时候就可以更新matchIndex[i]和nextIndex[i]了 matchIndex[i]=prevLogIndex+len(entries) nextIndex[i]=matchIndex[i]+1
5. 若回复失败，说明没有匹配上，这时候就要更新leader的nextIndex[i]了，nextIndex[i]--，等待下一轮重新发送
6. Leader 应当在适当的时候更新 commitIndex，比如收到成功的回复后或定期检查。
   1. 更新CommitIndex：找到一个最新的日志索引，满足超过一半的matchIndex[i]都大于等于该日志索引，这就是新的commitIndex

#### 接收方(Follower)逻辑
1. 如果发送的任期号小于自己的任期号，则不接受，并回复自身较大的任期号以更新旧leader，结束
2. 如果发送的任期号大于自己的任期号，则更新自己进入新的任期
3. 重置超时选举的时间点（因为受到心跳或添加日志代表能与leader联系上）
4. 若请求中的prevLogIndex在本节点不存在，说明在新添加的日志之前，本节点还有一部分日志没有与leader同步成功，因此回复false
5. 若请求中的prevLogIndex在本节点存在，则对比prevLogTerm与本节点对应日志的term
6. 若prevLogTerm与本节点对应日志的term不同，说明日志冲突，回复false让leader的nextIndex[i]回退
7. 若prevLogTerm与本节点对应日志term相同，说明与leader日志同步，可以正常添加新日志
8. 如果要添加的日志数组为空，说明是心跳，不需要添加日志，更新一下commitIndex就好了，如果不为空则需要添加发送的日志
   1. 添加日志：如果本节点最新索引并不是prevLogIndex，说明prevLogIndex之后的日志是与leader不同的，需要删除。删除后再追加entries中的内容即可。
   2. 更新commitIndex：如果leaderCommit > commitIndex，commitIndex = min(leaderCommit, lastIndex)

### InstallSnapshot (Leader -> Follower)
#### 请求参数
#### 回复参数
#### 发送方逻辑
#### 接收方逻辑
