# Ideas for HHS v3
Scratch pad for ideas / discussion for v3 of HHS

## Changes wrt v2

In general, we aim for a more modular approach, where it'd be simpler to either re-use some of this components, or swap them out for other implementations.

### Data representation

- Use JSON directly, instead of using an object serialization format that outputs JSON (this is a _huge_ simplification).
- Do not attempt to marshall in-memory _live_ data into the data representation automatically. Use an at-rest format with an explicit load/store mechanism (more simplification).
- As in v2, use a single Merkle-ized data structure to represent information.
- Incorporate the **snapshots into the Merkle-ized data**. Rough outline:
  - Changes are still modelled as operations, and form a Merkle-DAG that follows the partial order defined causally.
  - Whenever the DAG reaches a (configurable) height N, the current state is written out as a snapshot that's referenced from op(s) at height N.
  - The exact format of the state snapshot depends on the data type, but it is composed by a few Merkle-ized AVL trees.
  - _Note:_ The AVL trees are deterministic only if keys are inserted in the same order. To get determinism, ops in the history fragment starting from the previous snapshot are applied following a pre-defined order (e.g. BFS) when creating the next one.
  - _Example:_ if the type is a set of records, similar to a table in a RDBMs, one AVL tree may contain the data itself, while be additional AVL trees may work as indexes that would allow access to the data by matching or defining ranges over some of the columns. 
  - The type still must define how an op changes state (associatively and commutatively as in op-based CRDTs), but now must also provide a merge function for snapshots.
  - _Note:_ Snapshots must retain any metadata that's necessary to perform merges, like in state-based CRDTs.
  - The Merkle-AVL trees respect the O(n) hard limit on changed nodes, and thus are robust vs. a malicious participant trying to de-generate the tree by crafting the inserted keys.
  - The snapshots may contain indexes (in the form of additional AVL trees) that aid at verifyibly obtaining subsets of the data contained in the snapshot.
- In v2, a form of **pseudo-transactionality** was allowed by having operations in one instance depend upon the validity of operations in another, and using a **cascade undo-redo** mechanism whenever there were data races. This guaranteed that operations could _always_ be merged, at the cost of exceptionally having to undo some stuff in the process (and also sofware complexity). Replace all that by just declaring explictly which properties of the other instance are being claimed (e.g. _user A is an administrator_), do it independently of any causal history, and just have the merge fail if one of the instances claims a property that's no longer valid. It is then up to whenever outputted those changes to either abandon them or re-package them starting from the last valid snapshot.

### Replication

- Use a well established protocol for authentication of parties in a connection (Noise or MLS are candidates ATM).
- Leverage the simplified data representation (now plain JSON) to create a simple spec of the replication protocol.
- The v2 sync implementation carries quite a bit of complexity stemming from trying to guess which objects are needed on the other side. Simplify by just exchanging headers and then using explicit requests. History is going to be _much_ shorter now anyway!
- When replicating snapshots:
  - Now it's possibe to do _partial_ data replication! Using indexes, the client can request only a subset of the keys, that will correspond to a branch in one of the AVL trees.
  - Snaphots that are _close_ in causal history should be also very similar in terms of the number of hashed nodes that need to be exchanged.
- It would be nice if, after replicating a snapshot partially by using a subset of the keys in an index, it'd be possible to add/edit/delete items in that subset (but if there are additional indexes that need to be updated, this may be impractical).
- To prevent costly merges, only follow changes that are at most N*k operations behing the current state (similar to the change window defined by José Orlicki for the Pulsar chain we implemented on top of v2). If someone stays offline for a long time, it's up to his client to re-package the changes on top of the current state (unless the state-based sync is really efficient and this doesn't hurt, who knows).

### Networking & Gossip

- Make gossip dead-simple and independent of any signalling that browsers may need.
- Support WebRTC for browser-to-browser sync, but on-par with any other transports (following the _better modularization_ objective stated above).

### Usage modes

- Support both explicit synchronization by using the sync library (like in v2), but also provide adapters to perform sync by doing **automatic diffing** (could work for files and for databases). In this mode, for example, an application could use a local database, and just configure a sync process to do sync. Some caveats may apply: IDs in the db should be random (e.g. UUIDs), a column could be added to the syncrhonized tables to indicate the replication state of each column, to be written by the sync process and read by the app, etc.
