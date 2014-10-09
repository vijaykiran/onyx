## Internal Design

This chapter outlines how Onyx works on the inside to meet the required properties of a distributed data processing system. This is not a formal proof nor an iron-clad specification for other implementations of Onyx. I will do my best to be transparent about how everything is working under the hood - good and bad.

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [High Level Components](#high-level-components)
  - [Coordinator](#coordinator)
  - [Peer](#peer)
  - [Virtual Peer](#virtual-peer)
  - [ZooKeeper](#zookeeper)
  - [HornetQ](#hornetq)
- [Cross Entity Communication](#cross-entity-communication)
- [Virtual Peer States](#virtual-peer-states)
  - [Description](#description)
  - [State Transitions](#state-transitions)
- [Coordinator Event Handling](#coordinator-event-handling)
  - [Serial Execution](#serial-execution)
  - [Fault Tolerant Logging](#fault-tolerant-logging)
  - [Timeouts](#timeouts)
- [Coordinator/Virtual Peer Interaction](#coordinatorvirtual-peer-interaction)
  - [Notation](#notation)
  - [1 Peer - happy path](#1-peer---happy-path)
  - [1 Peer - times out while acking](#1-peer---times-out-while-acking)
  - [1 Peer - dies while acking](#1-peer---dies-while-acking)
  - [1 Peer - dies while acking](#1-peer---dies-while-acking-1)
  - [1 Peer - dies while sealing](#1-peer---dies-while-sealing)
  - [1 Peer - dies while completing](#1-peer---dies-while-completing)
  - [2 Peers - 1 waits](#2-peers---1-waits)
  - [2 Peers - 1 joins while sealing](#2-peers---1-joins-while-sealing)
  - [2 Peers - 1 joins late and dies](#2-peers---1-joins-late-and-dies)
  - [2 Peers - 1 dies while sealing](#2-peers---1-dies-while-sealing)
- [Segment Transportation](#segment-transportation)
  - [HornetQ Single Server](#hornetq-single-server)
  - [HornetQ Cluster](#hornetq-cluster)
- [Virtual Peer Task Execution](#virtual-peer-task-execution)
  - [Phases of Execution](#phases-of-execution)
  - [Pipelining](#pipelining)
  - [Local State](#local-state)
- [Sentinel Values in a Distributed Setting](#sentinel-values-in-a-distributed-setting)
  - [Sentinel-in-the-Middle](#sentinel-in-the-middle)
  - [Sentinel Reduction](#sentinel-reduction)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

### High Level Components

#### Coordinator

The Coordinator is single node in the cluster responsible for doing distributed coordination. It receives jobs from Onyx clients, assigns work to peers, and handles peer failure.

#### Peer

A Peer is a node in the cluster responsible for processing data. It is similar to Storm's Worker node. A peer generally refers to a physical machine, though in the documentation, "peer" and "virtual peer" are often used interchangeably.

#### Virtual Peer

A Virtual Peer refers to a single peer process running on a single physical machine. Each virtual peer spawns about 15 threads to support itself since it is a pipelined process. The Coordinator sees all virtual peers as equal, whether they are on the same physical peer or not. Virtual peers *never* communicate with each other - they only communicate with the Coordinator.

#### ZooKeeper

Apache ZooKeeper is used as storage and communication layer. ZooKeeper takes care of things like CAS, consensus, leader election, and sequential file creation. ZooKeeper watches are at the heart of how Onyx virtual peers communicate with the Coordinator.

#### HornetQ

HornetQ is employed for shuttling segments between virtual peers for processing. HornetQ is a queueing platform from the JBoss stack. HornetQ queues can cluster for scalability.

### Cross Entity Communication

ZooKeeper is used to facilitate communication between the Coordinator and each virtual peer. Onyx expects to use the `/onyx` path in ZooKeeper without interference. The structure of this directory looks like the following tree (descriptions in-line):

- `/onyx`
  - `/<deployment UUID>` # `:onyx/id` in Coordinator and Peer
    - `/peer`
      - `/<UUID>` # composite node, pointers to other znodes
    - `/status`
      - `/<UUID>` # presence signifies that peer should continue to commit to HornetQ
    - `/task`
      - `/<UUID>`
        - `/task-<sequential-id>` # composite node, pointers to znodes about this task
        - `/task-<sequential-id>.complete` # marker node, signifies that task is complete
    - `/ack`
      - `/<UUID>` # peer touches this znode to acknowledge task acceptance
    - `/catalog`
      - `/<UUID>` # data node for the catalog
    - `/job-log`
      - `/offer-<sequential id>` # durable log of job IDs to do round-robin job dispersal
    - `/job`
      - `/<UUID>` # composite node, pointers to other znodes about this job
    - `/payload`
      - `/<UUID>` # composite node, pointers to other znodes about this payload for a task
    - `/pulse`
      - `/<UUID>` # ephemeral node for a peer to signify that it is still online
    - `/completion`
      - `/<UUID>` # peer touches this znode to signal that its finished with its task
    - `/cooldown`
      - `/<UUID>` # peer listens on this node for response from Coordinator about completion
    - `/workflow`
      - `/<UUID>` # data node for the workflow
    - `/election`
      - `/proposal-<sequential id>` # leader election nodes for stand-by Coordinators
    - `/plan`
      - `/<UUID>` # Durable log of jobs to submit to the Coordinator
    - `/seal`
      - `/<UUID>` # Peer touches this node to signal that it can seal the next queue
    - `/exhaust`
      - `/<UUID>` # Peer listens to this node for response from Coordinator about sealing
    - `/coordinator` # Durable log entries of all the prior actions for fault tolerance
      - `/revoke-log`
        - `/log-entry-<sequential id>`
      - `/born-log`
        - `/log-entry-<sequential id>`
      - `/seal-log`
        - `/log-entry-<sequential id>`
      - `/ack-log`
        - `/log-entry-<sequential id>`
      - `/death-log`
        - `/log-entry-<sequential id>`
      - `/evict-log`
        - `/log-entry-<sequential id>`
      - `/offer-log`
        - `/log-entry-<sequential id>`
      - `/shutdown-log`
        - `/log-entry-<sequential id>`
      - `/exhaust-log`
        - `/log-entry-<sequential id>`
      - `/planning-log`
        - `/log-entry-<sequential id>`
      - `/complete-log`
        - `/log-entry-<sequential id>`
    - `/peer-state`
      - `/<UUID>`
        - `/state-<sequential id>` # Data node for state of the peer
    - `/shutdown`
      - `/<UUID>` # Peer listens to this node, shuts down on trigger

### Virtual Peer States

#### Description

Each virtual peer can be in exactly one state at any given time. The Coordinator tracks the state of each virtual peer. In fact, the virtual peer does not know its only state. The state for each peer is maintained so that the Coordinator can make intelligent decisions about how to allocate work across the cluster.

The peer states are:

- `:idle` - Peer is not allocated to a task
- `:acking` - Peer is deciding whether it wants to accept a task given to it by the Coordinator
- `:active` - Peer is executing a task
- `:waiting` - Peer has finished executing a task, but other peers are still finishing the same task. Cannot yet be reallocated to `:idle`, otherwise peer would continuously be allocated and removed from the same task.
- `:sealing` - Peer is propagating the sentinel onto the task's egress queues.
- `:revoked` - Peer has had its task taken away from it. Peer will shortly be killed off by the Coordinator.
- `:dead` - Peer has crashed, and may not receive any more tasks.

#### State Transitions

Peer states transition to new states. The transitions are specified below.

- An `:idle` peer may transition to:
  - `:acking`: Peer is given a task by the Coordinator
  - `:dead`: Peer crashes
  - `:idle`: Stand-by Coordinator wakes up and replays log entries

- An `:acking` peer may transition to:
  - `:active`: Peer acknowledges a task from the Coordinator
  - `:revoked`: Peer fails to acknowledge a task before the Coordinator times out
  - `:dead`: Peer crashes
  - `:acking`: Stand-by Coordinator wakes up and replays log entries

- An `:active` peer may transition to:
  - `:sealing`: Peer reads the sentinel from an ingress queue, and the ingress queue is depleted
  - `:waiting`: Peer reades the sentinel value from an ingress queue, and the ingress queue is depleted, but other peers are `:acking` or `:active`.
  - `:dead`: Peer crashes
  - `:active`: Stand-by Coordinator wakes up and replays log entries

- A `:waiting` peer may transition to:
  - `:idle`: Other peers completing the same task finish or crash
  - `:dead`: Peer crashes
  - `:waiting`: Stand-by Coordinator wakes up and replays log entries

- A `:sealing` peer may transition to:
  - `:idle`: Peer tells Coordinate that the task is complete, and no other peers are `:acking` or `:active` on the same task
  - `:dead`: Peer crashes
  - `:sealing`: Stand-by Coordinator wakes up and replays log entries

- A `:revoked` peer may transition to:
  - `:dead`: Peer crashes or Coordinator manually kills off peer
  - `:revoked`: Stand-by Coordinator wakes up and replays log entries

- A `:dead` peer may transition to:
  - `:dead`: Stand-by Coordinator wakes up and replays log entries

### Coordinator Event Handling

The Coordinators holds watches on znodes in ZooKeeper and reacts to events. The events are placed onto core.async channels and serialized.

#### Serial Execution

All events coming out of ZooKeeper are serialized onto a single core.async channel. This techniques allows ZooKeeper to act "transactional" for the particular workload that Onyx places on it. It's designed this way so that multiple writers can read incomplete or inconsistent values from ZooKeeper just before commiting to storage.

#### Fault Tolerant Logging

Just before each event's action is executed, the event is written to a durable log in ZooKeeper. When the Coordinator boots up, it runs through the log from the last known checkpoint and replays entries. All actions in the Coordinator are idempontent, so they are can be safely replayed if they crashed mid-action.

#### Timeouts

As of present, the Coordinator only has one situation where it employs timeouts. When a Coordinator offers a task to a peer, the peer must acknowledge that it wants to execute that task. If the peer doesn't make a decision about whether or not it wants the task within the timeout interval, the Coordinator revokes the task from the peer and marks the peer as dead. The peer can no longer receive new tasks.

### Coordinator/Virtual Peer Interaction

This section captures the interactions between the Coordinator and a peer under a variety of interesting circumstances. In particular, I try to show how Onyx is able to recover in the face of crashed peers at very specific points in time by indicating what the states of the peer are on the left of each diagram.

#### Notation

- "/": Right-way black-slash indicates blocking. No forward progress happens until the next message is received.
- "C": Coordinator
- "P", "P1", "P2": Peers
- "Green box": Peer state as known by the Coordinator

Time flows from top to bottom.

#### 1 Peer - happy path

![1 Peer 1 Task](img/1-peer-1-task.png)

#### 1 Peer - times out while acking

![1 Peer times out](img/1-peer-times-out.png)

#### 1 Peer - dies while acking

![1 Peer dies acking](img/1-peer-dies-acking.png)

#### 1 Peer - dies while acking

![1 Peer dies active](img/1-peer-dies-active.png)

#### 1 Peer - dies while sealing

![1 Peer dies sealing](img/1-peer-dies-sealing.png)

#### 1 Peer - dies while completing

![1 Peer dies completing](img/1-peer-dies-completing.png)

#### 2 Peers - 1 waits

![1 Peer waits](img/2-peers-1-waits.png)

#### 2 Peers - 1 joins while sealing

![1 Peer joins sealing](img/2-peers-1-joins-while-sealing.png)

#### 2 Peers - 1 joins late and dies

![1 Peer joins and dies](img/2-peers-1-joins-late.png)

#### 2 Peers - 1 dies while sealing

![1 Peer dies sealing](img/2-peers-1-dies-sealing.png)

### Segment Transportation

#### HornetQ Single Server

#### HornetQ Cluster

<Symetric clustering, session per thread, message offloading, load balancing>

### Virtual Peer Task Execution

#### Phases of Execution

#### Pipelining

#### Local State

### Sentinel Values in a Distributed Setting

#### Sentinel-in-the-Middle

#### Sentinel Reduction

