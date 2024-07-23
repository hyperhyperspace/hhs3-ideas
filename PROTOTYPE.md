# Prototype idea for HHS v3

V3 maintains the same concept as before: we provide building blocks for creating distributed data types, with operational semantics, that can later be synchronized automatically using the Hyper Hyper Space protocol & library under BFT conditions. We also will provide a mechanism for cross-referencing state across object instances, enabling composition of types while still being able to synchronize automatically (including cutting-edge use-cases like for example the capability system we prototyped in V2).

We have 3 important design changes:

**1. Materialized state in the Merkle-ized representation**

In V2, state was materialized by feeding the operations that were previously synchronized and validated to an instance (a javascript object) of the data type being synchronized. This instance would process all the operations, and the resulting state would be the state of that javacript object, that was used directly (to display in a UI, perform further state changes, or by other synchronized objects when composing state for entire applications). To speed up initialization, the store provided by the library allowed types to save snapshots of their internal state.

That has two main drawbacks: as state grows, loading the entire state of the object instance to memory can become problematic. Furthermore, the only validation available to peers forces them to fetch and verify the entire operational history. Second, sometimes validation needs access to state as it was at a given point in time, and that's not easily accesilbe (in the Pulsar prototype, this was attained by applying operations in reverse, but it added significant complexity).

In V3, all operations will include a hash of the resulting state after operation application. To do this, state will be represented using key-value maps, Merkle-ized using Merkle Search Trees [3] (after evaluating the alternatives discussed before, we considered the commutativity of MST construction outweights the fact that the balance of the tree is only probabilistic - furthermore, a "smoothing" transformation could be applied to make attempts to imbalance the tree even more computationally expensive, without altering its balancing property).

Merkle Search Trees are "stable" in the sense that modifying a tree only creates O(log(n)) new entries, thus allowing the existing Merkle-ization to be mostly reused as state is changed operationally. This enables a mixed strategy for syncrhonization/validation, where state can be synchronized initially and then kept up-to-date by streaming new operations. Furthermore, the maps in the state can include secondary indexes, to facilitate (verifiable) remote querying and partial sync (including directed sync, where the most urgently needed data can be verifiably transmitted first).

Therefore, state will now be materialized in the internal Hyper Hyper Space store, and be available for verifiable querying at different points in operational history. This also removes the memory bottleneck for state computation.

**2. Operational semantics are now transparent**

Type definition in V2 used direct manipulation of the resulting javascript object instance to materialize state. In V3, types will need to feed back to the engine the key-value updates a given operation needs to apply to the different maps that comprise state. As we'll see, this enables nicer design choices for the rest of the system.

Assuming a valid initial state, the validity of an operation can now be proved out-of-context, by providing Merkle proofs for the bits of state the operation depends upon, plus the neighboring nodes needed to re-construct the Merkle Search Trees of the resulting state. This simplifies the construction of the proposed state transition logbook from the 2021 proposal enormously.

**3. Data type composition is now based on state, not operational history**

In V2, when the state of a type depended on the state of another (for example, the set of messages in a chat room dependend on a capability system based on owner/admin/user roles), this was expressed using operational causal dependencies. An operation inserting a message would depend on an operation, authored by an admin, granting its author access to the room, that would in turn depend on another operation where the admin was granted such status. The validity of operations would be first revoked locally in each type (e.g. admin rights being revoked, access to a room being removed, etc.). If operation invalidation was concurrent (in DAG-history) with any new operational dependencies, these would be undone. Their own consequences would then have to be undone, and this could trigger undo/redo chains that were handled by the sync engine.

While the method used was sound, it was complex to program (to the point the main author had to code all the dependency logic himself), and was overly restrictive: in the example above, sending the message did not depend on the fact that permission was granted, but on one specific op doing so. This did not present problems for this use case, but could in the future. For example, it was impossible to assert that a given position in our RGA implementation held a given value, since that cannot be inferred from a fixed set of ops (the exact positions in the array depend on the entire operational history up to that point!)

In V3, these dependencies are resolved differently: as operations advance the state of the dependent type, they move forward a reference to the states of the types being depenent upon. When an operation needs to assert something about the state of another type, the system can check that such assertions are valid in the referenced state position. When applying concurrent operations that reference different points in dependent types, the same pessimistic rule can be applied as before: the depended-upon states have first to be merged, and then the assertions have to be valid in this merged state for the operation be deemed valid.

While the causal links that drive validity may still underneath be the same undo/redo chains as in V2, this is now expressed naturally as state advancement, and should be easier to reason about. Furthermore, properties that were not expressible before (e.g. asserting the value of a given position in a replicated array) can now be expressed simply as propositions on the materialized state.
