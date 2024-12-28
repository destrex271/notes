<a href="https://static.googleusercontent.com/media/research.google.com/en//archive/spanner-osdi2012.pdf">Paper Link</a>
# Spanner Paper

**What is Spanner?**
 - Scalable, globally-distributed database
 - database that shards data acrossmany sets of Paxos Machine


## Compared to Others

Problems with bigtable include:
 - Difficult to use for some kind of applications having complex evolving schemas
 - Problems with applications that want strong consistency in presence of wide-area replication

Megastore: Semi-relational data model and support for synchronous replication but has poor write throughput

In Spanner:
 - Data stored in schematized semi-relational tables
 - Data is versioned -> Each version automatically timestamped with its commit time
 - old versions, subject to configurable garbage-collection policies
 - provides SQL based query language
 - externally consistent reads and writes
 - globally consistent reads at a timestamp

**TrueTime API**
 - Exposes clock uncertainity
 - Gaurantees on Spanner's timestamps depends on the bounds that implementation provides
 - Large uncertainity -> spanner slows down instead of wait out that uncertainity



## Spanner Universe

Spanner Deployment -> Universe

Each zone has: 
 - ZoneMaster: One out of thousand spanservers. Assigns data to spanservers and latter serves data to clients
 - Spanservers
 - perzone location proxies: used by clients to locate spanservers assigned to serve their data
 - placement driver: handles automated movement of data across zones on the timescale of minutes. periodically communicates with spanservers to find data that needs to be moved, wither to meet updated replication constraints or to balance load.

### SpanServer Software Stack: Implementation

Tablet: Similar to bigtable's tablet abstraction -> implements a bag of following mappings


```(key:string, timestamp:int64) -> string```

Spanner assigns timestamps to data

Tablet's state is stored in set of B-tree like files and a WAL, all on DFS called Colossus

<ul>Replication Support</ul>:  Each Spanserver implements a single Paxos state machine on top of each tablet

Paxos state machines are used to implement a consistently replicated bag of mappings. KV mapping state of each replica is stored in its corresponding tablet.
Writes must initiate the Paxos protocol at the leader -> reads access state directly from the underlying tablet at any replica that is sufficiently up to date. Set of replicas -> Paxos Group


At every replica there is:
 - There is a leader
 - implements a lock table for concurrency control
 - lock table contains -> state of 2-phase locking: maps range of keys to lock states
 - Longlived Paxos Leader is required fr managing lock table
 - Designed for long lived transactions(Same for bigtable)

Per Replica we have:
 - Transaction manager: To support distributed transactions
 - Participation Leader: Implemented by transaction manager
 - Participant Slaves: Other replicas in the group

*Note*: One Paxos Group => bypass transaction manager since lock table & Paxos together provide transactionality

Coordinator Leader: One Participant group chosen as the leader 

State of transaction manager is stored in underlying Paxos Group

Coordinator Slaves: Slaves of the group 


## Directories & Placements

**Directory:** Bucketing abstraction. Set of contigous keys that share a common prefix
Directories help in controlling hte locality of data 

Directory is also the unit of data placement.
All data in a directory has the same replication configuration. Data movement is done directory by directory.
Directories can be moved while client operations are going on 


## Data Model

Not strictly relational

Exposes the following data features to applications:
 - Data model based on schemtized semi-relational tables
 - A Query Language
 - General Purpose Transactions


 *Claim:* Two Phase commit is too expensive to support
 *Reason:* Performance & availability problems


Running two phase commits over Paxos mitigates problem of bottleneck created by multiple transactions performed via code.

Table Interleaving: allows clients to describe localilty of relationships that exist between multiple tables -> necessary for good performance in a sharded distributed database


# True Time API
<a href="https://sookocheff.com/post/time/truetime/">Time Master at Google data center</a>

*TTinterval:* Represents time; An interval with bounded time uncertainity

*TTstamp:* Enpoints of TTinterval

*TT.now():* Returns a TTinterval gaurenteed to contain the absolute time (how?)

Instantaneous Error Bound: epsilon

*TT.after(t):* true if t has definetly passed

*TT.before(t):* true if t has definetly not passed

*t_abs(e):* Denote absolute time fo an event e


*Time references used:* GPS & Atomic clocks

**Why two references?**

Becuase they have different failure modes. GPS reference source vulnerabilities include antena and receiver failures, local radio interference and system outages
Atomic clocks can fail in ways not related to GPS and each other and over long periods of time can drift significantly due to frequency error

Implemented by a set of Time master machines per datacenter and a  timeslave daemon per machine.


#### Master Configuration

 - Majority Masters Uses GPS receivers
 - Seperated Physically to reduce the effects of antenna failures,radio interference and spoofing
 - Armageddon Masters: Remaining Masters use Atomic Clocks
 - Cost of Armageddon Master is same as that of a GPS Master
 - All Master's Time references are compared against each other
 - Armageddon Masters advertise uncertainity derived from conservatively applied worst clock drift
 - GPS master advertises uncertainity that is typically close to zero


Daemons Regularly Poll a variety of masters to reduce vulnerability to errors from any one master

Apply a variant of **Marzullo's Algorithm** to detect and reject liars & synchronize local machine clocks to non liars

**Marzullo's Algorithm:** Expanded Algorithm used to select sources for estimating accurate time from a number of noisy time sources

_**Machine Eviction:**_ Machines that exhibit frequency excursions larger than the worst casse bound derived from component specifications and operating environment. **Protects against broken clock**

Between synchronizations -> one daemon advertises slowly increasing tim uncertainity, derived from applied worst case local clock drift
Also depends on Time Master uncertainity and communication delay to time masters. It is typically a sawtooth function of time varying about 1 to 7 ms over each poll interval

# Concurrency Control

Describes how TrueTime is used to gaurantee the correctness properties are used to implement features such as:
 - Externally consistent transactions 
 - Lock Free read-only transactions
 - Non-blocking Reads in the past


### Timestamp Management

Spanner supports read-write, read-only transactions & snapshot reads

Standalone writes -> implemented as Read-Write transactions

Non-Snapshot standalone reads -> Read Only transactions

### Paxos Leader Leases
 - Uses timed leases to make leadership long-lived(10s default)
 - Potential leader sends requests for times lease votes; Once votes counted -> leader knows it has lease
 - Replica extends lease vote implicitly on a successful write 
 - Leader requests leasse-vote extensions if near expiration
 - *Lease Starting*: Leader has quorum votes
 - *Lease Endings*: Leader no longer hsa quorum votes
 - Depends on disjoint invariant: for each Paxos Grp -> leader's lease is disjoint from every other leader's lease
   uses true time to ensure disjointness

Simply each leader takes turns, only one leader should be active at a time to avoid conflicts 


### Assigning Timestamps to RW transactions

Transactional Reads & Writes use 2-phase locking

Can be assigned timestamps at any time when all locks have been acquired, but before any locks have been released

For any given transaction same timestamp is assigned which is assigned to the Paxos write

**Spanner depends on Monotonicity Invariant:** Assigns timestamps to Paxos writes in monotonically increasing order, even across leaders

Whenever a timestamp 's' is assigned, 's_max' is advanced to s to preserve disjointness

*Note:* Invariant -> Same under given conditions; Homogenous -> Same always

**External Inconsistency Invariant:** If Start of T2 occurs after COMMIT of T1, T2_TS > T1_TS

Two rules for executing transactions and assigning timestamps:

 - **Start:** Leader for write T_t assigns a commit timestamp s+i no less than the value of TT.now().latest
 - **Commit Wait:** Coordinator Leader ensures clients cannot see any data commited by T_i unitl TT.after(s_i) is true


#### Serving Reads at a Timestamp

Determine if Replica state is sufficiently upto date to satisfy a read -> Monotonicity Invariant

Safe Time: Maximum timestamp at which a replica is up to date. tracked by each replica

Satisfaction criteria for a read => t <= t_safe

t_safe = min(t_paxos_safe, t_TM_safe)

TM -> Transaction Manager

t_TM_safe -> Inf at replica if there are 0 prep (but not commited transactions) => transaction in between the two phases of two-phase commit

Commit Protocol -> ensures thatevery participant knows a lower bound on a prepared transactions timestamp

### Assigning Timestamps to RO Transactions

Phases of RO Transaction execution:
 - Assign timestamp s_read
 - Execute reads as snapshot reads at s_read

 *Snapshot reads can execute at any replicas which are sufficiently upto_date(t <= t_safe)*

Spanner should assign the oldest timestamp that preserves external consistency


### Read-Write Transactions

writes buffered at client until commited

Reads in RW Transactions -> use wound-wait to avoid deadlocks

*wound_wait:* Pre-emptive, T_n is allowed to wait for data held by T_k only if it has a timestamp larger than that of T_k, otherwise kill T_k(wounded by T_n)
If transaction occured before -> pre-empt and terminate else wait
Younger Transaction is only killed and restarted


### Read-Only Transactions

Requires negotiation between all of the Paxos Groups involved in reads

Spanner requires scope expression for every RO transaction -> summarizes keys read by the entire transaction

Automatically infers scope for standalone queries

**Single-Site Read**
 - Scope values served by single Paxos machine
 - Issues read only transaction to group's leader
 - Leader assigns s<sub>read</sub> and execcutes read
 - Spanner does better than TT.now().latest => Uses internal mechanisms instead of calling TT.now().latest
 - Internal Mech: Define LastTS() to be the timestamp of last commited write
                  if no prepared transaction => s<sub>read</sub> = LastTS()

**Multiple Paxos Sites involved in Read**
 - Scope value served by multiple Paxos groups
 - Complex Method(not implemented): Do a round of communication with all of the group's leaders to negotiate s<sub>read</sub> based on LastTS()
 - Simple Implementation(current): Avoids negotiation rounds, just has its reads execute at s<sub>read</sub> = TT.now() [[may wait for sometime to advance]]  
 - All reads in the transactions can be sent to sufficiently upto-date replicas

### Schema-Change Transactions

 - Enables spanner to support atomic schema changes
 - Infeasible to use standard transaction -> since large number of participants(no of groups in db)
 - *NOTE:* Bigtable also supports atomic schema changes in one data center => but is blocking in nature
 - Spanner: Non-Blocking Schema Changes
 - Explicitly assigned a timestamp in the future, registered in prepare phase
 - Reads & Writes synchronize with any registered schema-change timestamp at time t
 - Possible because of TrueTime

### Refinements

Weakness of t_TM_safe => a single prepared transaction prevents t_safe from advancing => No reads can occur at later timestamps, even if reads do not conflict

Solved using Lock Table => stores information related to mapping key ranges to lock metadata

Read Arrives -> needs to be checked against fine-grained safe time for key ranges with which the read conflicts

<hr/>

Weakness of LastTS -> if transaction just commited, a non conflicting RO transaction must still be assigned s<sub>read</sub> as to follow that transaction

Execution of Read delayed.

Same fix as t_TM_safe

<hr/>

T_paxos_safe weakness -> 
 - Cannot advance in absence of Paxos writes
 - A snapshot read at t cannot execute at Paxos groups whose last write happended before t
 - Solved via leader-lease invariance(each leader has different lease interval; no two leaders at the same time)
 - Maintains a mapping MinNextTS(n) from Paxos seq number n to min timestamp that maybe assigned to PAxos seq number n+1
 - Therefore replica can advance T_Paxos_safe = MinNextTS(n) - 1 when applied thru n
 - T_paxos_safe = (t < t_Paxos_safe)?MinNextTS() - 1 : t
 - Single Leader can enforece MinNextTS() easily: promised timestamps lie within leader's lease => disjoint invariance
 - To move MinNextTS() beyond lease time, leader needs to extend their lease by taking qourum vote
 - smax always advanced to the highest val in MinNextTS() to preserve disjointness
 - Leader by default advances MinNextTS() every 8 seconds
 - Leader also advances MinNextTS() on demand from salves
