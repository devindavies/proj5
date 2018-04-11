Raft Whitepaper Notes
---

*this is general and probably formatted horribly*

*https://raft.github.io/raft.pdf*

---

##### Intro:

* Raft is consensus algo meant to be understandable
* equiv to Paxos in terms of performance and fault tolerance
* Raft consensus ideally understandable by a wider audience

##### Consensus:

* fundamental problem in fault-tolerant (FT) distributed systems
* basically replicated state across multiple machines

---

### WhitePaper

##### Abstract

* Raft is a consensus algorithm for managing a replicated
log.

* . In order to enhance understandability,
Raft separates the key elements of consensus, such as
leader election, log replication, and safety, and it enforces
a stronger degree of coherency to reduce the number of
states that must be considered

**Strong Leader**

* log entries only flow from the leader to other servers

**Leader Election**

*  uses randomized timers to elect leaders

**Membership Changes**

* majorities of two different configurations overlap during transitions

*... skipping the parts just explaining why Paxos is bad ...*

**The Raft Consensus Algorithm**

* Raft is an algorithm for managing a replicated log

Raft **first** *elects a distinguished leader*
	
	* This leader then assumes entirety of control over log

	*  The leader accepts log entries from clients, replicates them on other servers, and tells servers when it is safe to apply log entries to their state machines

	* [error handling] A leader can fail or become disconnected from the other servers, in which case a new leader is elected.

so... 

** Leader Requirements **

	* Leader election: a new leader must be chosen when an existing leader fails (Section 5.2).

	* Log replication: the leader must accept log entries

However these things are still important to know...

**Election Safety:** at most one leader can be elected in a given term.

**Leader Append-Only:** a leader never overwrites or deletes entries in its log; it only appends new entries.

**Log Matching:** if two logs contain an entry with the same index and term, then the logs are identical in all entries up through the given index.

**Leader Completeness:** if a log entry is committed in a given term, then that entry will be present in the logs of the leaders for all higher-numbered terms.

**State Machine Safety:** if a server has applied a log entry at a given index to its state machine, no other server will ever apply a different log entry for the same index.

**IMPORTANT SAFETY**

If a log entry is made in a server's state machine that entry is **IMUTABLE** and cannot be changed.

##### Raft Basics

5 servers is common for fault tolerance of 2 servers (2 can fail and consensus still maintained)


**At any given time each server is in one of three states:**

**Leader** - Usually just one of these (all others followers)

**Follower** - passive they issue no requests on their own but simply respond to requests from leaders and candidates

**Candidate** - candidate is used to elect a new leader 


**TIME is defined in sequences called TERMS**

*  arbitrary length
*  numbered with consecutive integers
*  Each term begins with an election, in which one or more candidates attempt to become leader
*  If elections fuck up or a number needed for split doesn't work -> results in a session with no leader -> next term tries again

Terms act as a **logical clock** in Raft, and they allow servers
to detect obsolete information such as stale leaders.

Each server **stores a current term number**, which increases monotonically over time

Current terms are exchanged whenever servers communicate (just follow the logic below...)

	* if one server’s current
	term is smaller than the other’s, then it updates its current
	term to the larger value. If a candidate or leader discovers
	that its term is out of date, it immediately reverts to follower
	state. If a server receives a request with a stale term
	number, it rejects the request.

Raft servers communicate using remote procedure calls (RPCs) - cool, just like ETH

	* RequestVote RPCs are initiated by
	candidates during elections (Section 5.2), and AppendEntries
	RPCs are initiated by leaders to replicate log entries
	and to provide a form of heartbeat (Section 5.3). Section
	7 adds a third RPC for transferring snapshots between
	servers. Servers retry RPCs if they do not receive a response
	in a timely manner, and they issue RPCs in parallel
	for best performance.

##### Leader Election

Raft uses a heartbeat mechanism to trigger leader election.
When servers start up, they **begin as followers**. 

Leaders send **periodic heartbeats**
	
	* If a follower receives no communication over a period of time
		called the election timeout, then it assumes there is no viable
		leader and begins an election to choose a new leader

**How to Start an Election**

1.  follower increments its current
term and transitions to candidate state

2. It then votes for
itself and issues RequestVote RPCs in parallel to each of
the other servers in the cluster

3. A candidate continues in
this state until one of three things happens: 

	(a) it wins the election

	(b) another server establishes itself as leader

	(c) a period of time goes by with no winner

A candidate wins an election if it receives votes from
a majority of the servers

. Each server will vote for at most one candidate in a
given term, on a first-come-first-served basis 

 Once a candidate wins an election, it
becomes leader. It then sends heartbeat messages to all of
the other servers to establish its authority and prevent new
elections

**weird edge case:**

While waiting for votes, a candidate may receive an
AppendEntries RPC from another server claiming to be
leader. If the leader’s term (included in its RPC) is at least
as large as the candidate’s current term, then the candidate
recognizes the leader as legitimate and returns to follower
state. If the term in the RPC is smaller than the candidate’s
current term, then the candidate rejects the RPC and continues
in candidate state.

**final case**

a candidate neither
wins nor loses the election: if many followers become
candidates at the same time, votes could be split so that
no candidate obtains a majority. When this happens, each
candidate will time out and start a new election by incrementing
its term and initiating another round of RequestVote
RPCs. However, without extra measures split votes
could repeat indefinitely.

randomized election timeouts to ensure that
split votes are rare and that they are resolved quickly

	* election timeouts are
	chosen randomly from a fixed interval

	* Each candidate
	restarts its randomized election timeout at the start of an
	election, and it waits for that timeout to elapse before
	starting the next election; this reduces the likelihood of
	another split vote in the new election.

##### Log Replication

Once a leader has been elected, it begins servicing
client requests. Each client request contains a command to
be executed by the replicated state machines.



The leader
appends the command to its log as a new entry

 If followers crash or run slowly,
or if network packets are lost, the leader retries AppendEntries
RPCs indefinitely

The leader decides when it is safe to apply a log entry
to the state machines; such an entry is called committed.






