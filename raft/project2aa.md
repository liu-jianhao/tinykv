## project2 RaftKV——Leader election
这次实现的是Raft算法，该算法分为三部分实现：
+ Leader election
+ Log replication
+ Raw node interface

### 预备知识
+ 什么是Raft算法？
可以看看下面这篇文章了解一下
https://blog.csdn.net/westbrookliu/article/details/99713334
还可以在Raft官网玩玩：https://raft.github.io/

### 第一步：看代码
基本上只要看`raft`目录下的几个文件：
+ `raft.go`：主要要补充的文件，实现了Raft算法
+ `log.go`：Raft日志的实现
+ `raft_paper_test.go`：测试用例
+ `proto/pkg/eraftpb/eraft.pb.go`：一些协议值

### 第二步：跑代码
```
make project2aa
```
不用看，肯定过不了=_=

### 第三步：写代码
当然，首先要理解Raft算法才能动手写。然后理解一下`raft.go`的几个函数：
####  `tick()`：
看名字就和时间有关，调用一次这个函数说明过了一个时间单位，这个函数主要就是判断一些超时时间
```go
func (r *Raft) tick() {
	// Your Code Here (2A).
	switch r.State {
	case StateFollower, StateCandidate:
		r.electionElapsed++
		if r.electionElapsed >= r.randomElectionTimeout {
			r.electionElapsed = 0
			_ = r.Step(pb.Message{
				MsgType: pb.MessageType_MsgHup,
				From:    r.id,
				Term:    r.Term,
			})
		}
	case StateLeader:
		r.heartbeatElapsed++
		if r.heartbeatElapsed >= r.heartbeatTimeout {
			r.heartbeatElapsed = 0
			_ = r.Step(pb.Message{
				MsgType: pb.MessageType_MsgBeat,
				To:      r.id,
				Term:    r.Term,
			})
		}
	}
}
```
####  `becomeXXX`：当角色转换时需要做的一些状态值的变更
```go
func (r *Raft) resetTime() {
	r.electionElapsed = 0
	r.heartbeatElapsed = 0
	r.randomElectionTimeout = r.electionTimeout + rand.Intn(r.electionTimeout)
	r.votes = make(map[uint64]bool, 0)
	r.Lead = None
}

// becomeFollower transform this peer's state to Follower
func (r *Raft) becomeFollower(term uint64, lead uint64) {
	// Your Code Here (2A).
	r.resetTime()
	if term != r.Term {
		r.Term = term
		r.Vote = None
	}
	r.State = StateFollower
}

// becomeCandidate transform this peer's state to candidate
func (r *Raft) becomeCandidate() {
	// Your Code Here (2A).
	r.resetTime()
	r.State = StateCandidate
	r.Term++
	r.Vote = r.id
	r.votes[r.id] = true
	if len(r.Prs) <= 1 {
		r.becomeLeader()
	}
}

// becomeLeader transform this peer's state to leader
func (r *Raft) becomeLeader() {
	// Your Code Here (2A).
	// NOTE: Leader should propose a noop entry on its term
	if r.State != StateLeader {
		r.resetTime()
		r.State = StateLeader
		r.Lead = r.id
	}
}
```
#### `Step()`：收到一条消息时需要对根据角色及消息类型做处理
```go
func (r *Raft) Step(m pb.Message) error {
	// Your Code Here (2A).
	if m.Term > r.Term {
		r.becomeFollower(m.Term, m.From)
	}

	switch r.State {
	case StateFollower:
		switch m.MsgType {
		case pb.MessageType_MsgHup:
			r.becomeCandidate()
			for p := range r.Prs {
				if p != r.id {
					r.sendRequestVote(p)
				}
			}
		case pb.MessageType_MsgHeartbeat:
			r.becomeFollower(m.Term, m.From)
		case pb.MessageType_MsgAppend:
			r.becomeFollower(m.Term, m.From)
		}
	case StateCandidate:
		switch m.MsgType {
		case pb.MessageType_MsgHup:
			r.becomeCandidate()
			for p := range r.Prs {
				if p != r.id {
					r.sendRequestVote(p)
				}
			}
		case pb.MessageType_MsgHeartbeat:
			r.becomeFollower(m.Term, m.From)
		case pb.MessageType_MsgRequestVoteResponse:
			r.votes[m.From] = !m.Reject
			agree, reject := 0, 0
			for _, v := range r.votes {
				if v {
					agree++
				} else {
					reject++
				}
			}
			if len(r.Prs) == 1 || agree > len(r.Prs)/2 {
				r.becomeLeader()
				for p := range r.Prs {
					if p != r.id {
						r.sendHeartbeat(p)
					}
				}
			}
			if reject > len(r.Prs)/2 {
				r.becomeFollower(r.Term, None)
			}
		case pb.MessageType_MsgAppend:
			r.becomeFollower(m.Term, m.From)
		}
	case StateLeader:
		switch m.MsgType {
		case pb.MessageType_MsgBeat:
			for p := range r.Prs {
				if p != r.id {
					r.sendHeartbeat(p)
				}
			}
		case pb.MessageType_MsgPropose:
			for p := range r.Prs {
				if p != r.id {
					r.sendAppend(p)
				}
			}
		}
	}

	// 三种角色通用
	if m.MsgType == pb.MessageType_MsgRequestVote {
		reject := true
		if ((r.Vote == None && r.Lead == None) || r.Vote == m.From) &&
			(m.LogTerm > r.RaftLog.LastTerm() ||
				(m.LogTerm == r.RaftLog.LastTerm() && m.Index >= r.RaftLog.LastIndex())) {
			reject = false
			r.Vote = m.From
		}
		msg := pb.Message{
			MsgType: pb.MessageType_MsgRequestVoteResponse,
			To:      m.From,
			From:    r.id,
			Term:    r.Term,
			Reject:  reject,
		}
		r.msgs = append(r.msgs, msg)
	}
	return nil
}
```
其余的详见代码

### 跑通测试
```
$ make project2aa
GO111MODULE=on go test -v --count=1 --parallel=1 -p=1 ./raft -run 2AA
=== RUN   TestFollowerUpdateTermFromMessage2AA
--- PASS: TestFollowerUpdateTermFromMessage2AA (0.00s)
=== RUN   TestCandidateUpdateTermFromMessage2AA
--- PASS: TestCandidateUpdateTermFromMessage2AA (0.00s)
=== RUN   TestLeaderUpdateTermFromMessage2AA
--- PASS: TestLeaderUpdateTermFromMessage2AA (0.00s)
=== RUN   TestStartAsFollower2AA
--- PASS: TestStartAsFollower2AA (0.00s)
=== RUN   TestLeaderBcastBeat2AA
--- PASS: TestLeaderBcastBeat2AA (0.00s)
=== RUN   TestFollowerStartElection2AA
--- PASS: TestFollowerStartElection2AA (0.00s)
=== RUN   TestCandidateStartNewElection2AA
--- PASS: TestCandidateStartNewElection2AA (0.00s)
=== RUN   TestLeaderElectionInOneRoundRPC2AA
--- PASS: TestLeaderElectionInOneRoundRPC2AA (0.00s)
=== RUN   TestFollowerVote2AA
--- PASS: TestFollowerVote2AA (0.00s)
=== RUN   TestCandidateFallback2AA
--- PASS: TestCandidateFallback2AA (0.00s)
=== RUN   TestFollowerElectionTimeoutRandomized2AA
--- PASS: TestFollowerElectionTimeoutRandomized2AA (0.00s)
=== RUN   TestCandidateElectionTimeoutRandomized2AA
--- PASS: TestCandidateElectionTimeoutRandomized2AA (0.00s)
=== RUN   TestFollowersElectionTimeoutNonconflict2AA
--- PASS: TestFollowersElectionTimeoutNonconflict2AA (0.00s)
=== RUN   TestCandidatesElectionTimeoutNonconflict2AA
--- PASS: TestCandidatesElectionTimeoutNonconflict2AA (0.00s)
=== RUN   TestVoter2AA
--- PASS: TestVoter2AA (0.00s)
=== RUN   TestLeaderElection2AA
--- PASS: TestLeaderElection2AA (0.00s)
=== RUN   TestLeaderCycle2AA
--- PASS: TestLeaderCycle2AA (0.00s)
=== RUN   TestVoteFromAnyState2AA
--- PASS: TestVoteFromAnyState2AA (0.00s)
=== RUN   TestSingleNodeCandidate2AA
--- PASS: TestSingleNodeCandidate2AA (0.00s)
=== RUN   TestRecvMessageType_MsgRequestVote2AA
--- PASS: TestRecvMessageType_MsgRequestVote2AA (0.00s)
=== RUN   TestCandidateResetTermMessageType_MsgHeartbeat2AA
--- PASS: TestCandidateResetTermMessageType_MsgHeartbeat2AA (0.00s)
=== RUN   TestCandidateResetTermMessageType_MsgAppend2AA
--- PASS: TestCandidateResetTermMessageType_MsgAppend2AA (0.00s)
=== RUN   TestDisruptiveFollower2AA
--- PASS: TestDisruptiveFollower2AA (0.00s)
=== RUN   TestRecvMessageType_MsgBeat2AA
--- PASS: TestRecvMessageType_MsgBeat2AA (0.00s)
=== RUN   TestCampaignWhileLeader2AA
--- PASS: TestCampaignWhileLeader2AA (0.00s)
=== RUN   TestSplitVote2AA
--- PASS: TestSplitVote2AA (0.00s)
PASS
ok      github.com/pingcap-incubator/tinykv/raft        0.418s
```