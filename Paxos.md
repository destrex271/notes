# Paxos

## The Consensus Problem

**Consensus**: When multiple computers agree on something

### Why do we need Consensus in Distributed Systems?

 - To decide on resource allocation(mutual exclusion)
 - To agree on leaders(elections)
 - To agree on common ordering of events 


### In context of replication
 - Replication is one of most common use of consensus. 
 - Replication provides fault tolerance
 - Replication provides scale
 - Design choices:
    - Funnel writes to a single coordinator, which are then eventuallly distributed to other replicas(eventual consistency). Read from any replica. Ensures in-order delivery
    - Send updates to any system, but then that system becomes responsible for in order delivery

 - For design choice I -> Consensus required to elect coordinator(consensus for coord election)
 - For design choide II -> Consensus algo needs to be run for each update to ensure everyone agrees on order(consensus for data ordering)

### Formal Proerties of Async Consensys
 - *Validity:* Only proposed values can be decided. If p decides on v, then some other process must have proposed v.
 - *Uniform Agreement:* Two uncrashing processes cannot decide on different values
 - *Integrity:* Each process can decide a value at most once
 - *Termination:* Eventually reach conclusive result; no infinite loop stuff


## Paxos

 - Algo to achieve consensus in a distributed set of computers that communicate async 
 - Simply selects a single value from one or more proposed values
 - Informs every node of the selected value
 - Detour: [Replicated Log](https://martinfowler.com/articles/patterns-of-distributed-systems/replicated-log.html) => keep state of multiple nodes sync by replicating WAL
 - For situations like replicated log: run Paxos multiple times: called multi-paxos
 - Provides abortable consensus i.e. if a node does not agree on the elected value -> it simply terminates instead of being blocked forever otherwise it needs to propose the value again to Paxos algo. If the nodes have decided they need to accept the proposed value

### Assumptions for Paxos
 - Concurrent proposals: Values will be proposed by one or more systems concurrently
 - Validity: Chosen value must be one of the proposed values -> no random number selection
 - Majority Rule: If majority of Paxos servers agree on a value -> we have consensus.
    - Implies majority servers also need to work for the algo to run. To survive m failures we need 2m+1 systems
    - Simply means -> more than half of the servers must be active to avoid failures
 - Async Network: Messages may get lost or are delayed -> network may also get partitioned(Division of a network into subnets)
 - Fail-stop faults: Systems may restart but need to remember their previous state.
 - Unicast: Communication is point to point. No mechanisms to multicast a message `atomically` to the set of Paxos servers
 - Announcement: Results are made available to each node once consensus is reached
