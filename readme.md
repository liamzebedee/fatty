fatty
=====

Fatty is a distributed synchronisation service, based on Google's Chubby, integrating Cairo CPU's and recursive composition of proofs.

## Background.

Chubby is a distributed locking service by Google. A lock service is a form of consensus service that converts the problem of reaching consensus to handing out locks. Chubby itself is implemented using the Paxos consensus algorithm.

Lock services are useful, because they can synchronise services in a loosely-coupled manner. For a distributed database, such as Google's [Bigtable](https://en.wikipedia.org/wiki/Bigtable), workload of reads/writes are usually distributed across a fleet of worker nodes, and managed by a single master. To detect failures in the worker nodes, the master must synchronise with every node to determine its liveness - whether it is running or it has failed. 

Notably, locks are also useful for sharing small pieces of persistent state. Google cites developers using the lock service to lookup the locations of servers, as a simplistic DNS service.

Bigtable uses locks to detect crash failures / the liveness of nodes. When a tablet node starts, it acquires a lock on a file in the `tablet-servers` directory to signal its liveliness. e.g. `lock("/bigtable/tablet-servers/12312")`. Locks are maintained using **leases**, wherein the client renews the lease periodically using a keepalive. If the client's lease expires, the lock becomes invalid, and the master can detect the node has died. 

Unfortunately, in a [Byzantine environment](https://en.wikipedia.org/wiki/Byzantine_fault), we cannot assume that a node's liveness means it is functioning correctly. Simply - a database worker node may report it is online, but not be doing any work. 

**Problem:** how can we synchronise services in a Byzantine environment?

## Concept.

Cairo introduces a computer architecture where every program's execution can be proven, via STARK proofs. A developer can write a program in the Cairo language, which when executed, generates a trace. That trace can be fed into a Cairo prover (such as [Giza](https://github.com/maxgillett/giza)), generating a STARK proof. The magic of STARK proofs is that they are cheaper to verify than running the full computation. A Cairo proof costs `O(log^2 t)` in time, vs. `O(t)` for a naive verification of state transition.

What if every node ran its program on top of a Cairo CPU? This would mean we could verify Byzantine faults easily - each node generates a proof of its execution, and if the proof is valid, we know the node is functioning correctly. 

How can we synchronise services using this? Well, before we were using Chubby's locks to detect liveliness. We could extend the protocol to detect byzantine faults, by requiring clients to provide execution proofs of their Cairo machine. 

An interesting way to look at Cairo CPU's is not of a state machine, but of a clock. A state machine's transition is stated informally as a `S_t+1 = Y(S_t, T)`, where `S` is the current state, `t` is the timestep, `Y` is the transition function (the program), and `T` is the transaction. Sending a transcation to a machine will advance the state machine one timestep, `t+1`. We could imagine that as the state machine's clock. 

By transferring a state machine's proof, we are actually performing clock sync. And clock sync can be used to determine liveliness. For example - the state machine is at time `t=10`, we send a transaction to it, and if we don't receive a proof back of the clock being advanced to `t=11` within a defined timeout, then we can verify the machine isn't live! 

## Design.

We implement a distributed synchronisation service for a byzantine environment. The data model is similar to Google's Chubby - there exists a filesystem-like interface, wherein clients can acquire locks on files, and store small pieces of state.

The purpose of this design is to:

 (1) distribute work in a BFT manner to a fleet of Cairo machines
 (2) provide a lightweight mechanism for machine discovery and communication

A Bigtable master node acquires a uniquely named lock. The lock is specified by the Cairo program hash and starting state of the node. The master will store its IP address, so that other nodes can communicate with it and read state. Then it will begin by declaring tablet nodes.

Any node may come online and become the master. It simply needs to run the Cairo program, and then generate a proof that it has loaded the program onto its machine. 

Nodes take leases from the master state service in this system. The nodes regularly roll-up their current state transitions (for example, from T=10 to T=20) into a recursive Cairo proof, and submit it to the master. This is equivalent to the lease renewal in Google's Chubby and happens on an interval of every 12s, mirroring Chubby's design.

