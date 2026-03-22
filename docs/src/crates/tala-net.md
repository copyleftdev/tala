# tala-net

Distributed networking layer for TALA. Provides core distributed types (node identity, partitioning, membership), a TLV message codec, and an in-process channel-based transport for testing without a real network. Real QUIC transport will be added in a future phase (see spec-04).

## Key Types

| Type | Description |
|------|-------------|
| `NodeId` | Unique identifier for a node in the cluster |
| `PeerId` | Type alias for `NodeId` in transport contexts |
| `PartitionId` | Identifies a partition of the intent graph |
| `Message` | Framed messages exchanged between nodes |
| `PartitionAssignment` | Maps a partition to its owner and replicas |
| `PartitionTable` | Cluster-wide partition routing table |
| `MemberState` | Liveness state of a cluster member (SWIM model) |
| `MembershipList` | Versioned cluster membership list |
| `InProcessNetwork` | Simulated network of channel-connected transports |
| `InProcessTransport` | A single node's view of the in-process network |

## Key Functions

| Function | Description |
|----------|-------------|
| `encode(msg)` | Serialize a `Message` into a TLV byte buffer |
| `decode(data)` | Deserialize a `Message` from a TLV byte buffer |

---

## NodeId and PartitionId

```rust
/// Unique identifier for a node in the cluster.
#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash)]
pub struct NodeId(pub u64);

/// Alias for NodeId used in transport contexts.
pub type PeerId = NodeId;

/// Identifies a partition of the intent graph.
#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash)]
pub struct PartitionId(pub u32);
```

---

## Message

The set of messages exchanged between nodes. All variants are serializable via the TLV codec.

```rust
#[derive(Clone, Debug, PartialEq)]
pub enum Message {
    /// Health check request.
    Ping { from: NodeId, seq: u64 },
    /// Health check response.
    Pong { from: NodeId, seq: u64 },
    /// Forward an intent to the partition owner.
    IntentForward { partition: PartitionId, payload: Vec<u8> },
    /// Replicate a segment to a replica node.
    SegmentSync { partition: PartitionId, segment_data: Vec<u8> },
    /// Broadcast membership changes.
    MembershipUpdate { members: Vec<NodeId>, version: u64 },
    /// Broadcast partition table changes.
    PartitionTableUpdate { assignments: Vec<PartitionAssignment>, version: u64 },
}
```

---

## PartitionTable

Cluster-wide routing table mapping partitions to their owner and replica nodes.

```rust
#[derive(Clone, Debug)]
pub struct PartitionTable {
    pub version: u64,
    pub assignments: Vec<PartitionAssignment>,
}

#[derive(Clone, Debug, PartialEq)]
pub struct PartitionAssignment {
    pub partition_id: PartitionId,
    pub owner: NodeId,
    pub replicas: Vec<NodeId>,
}
```

### Methods

```rust
impl PartitionTable {
    /// Return the owner of a given partition, if assigned.
    pub fn owner_of(&self, partition: PartitionId) -> Option<NodeId>;

    /// Return all partitions owned by or replicated on a given node.
    pub fn partitions_for(&self, node: NodeId) -> Vec<PartitionId>;

    /// Consistent-hash an intent ID (16 raw UUID bytes) to a partition.
    /// Uses FNV-1a over the id bytes, then reduces modulo `num_partitions`.
    /// Returns `PartitionId(0)` if `num_partitions` is zero.
    pub fn partition_for_intent(id_bytes: &[u8; 16], num_partitions: u32) -> PartitionId;
}
```

### Example

```rust
use tala_net::{NodeId, PartitionId, PartitionTable, PartitionAssignment};

let table = PartitionTable {
    version: 1,
    assignments: vec![
        PartitionAssignment {
            partition_id: PartitionId(0),
            owner: NodeId(10),
            replicas: vec![NodeId(11), NodeId(12)],
        },
        PartitionAssignment {
            partition_id: PartitionId(1),
            owner: NodeId(11),
            replicas: vec![NodeId(10)],
        },
    ],
};

assert_eq!(table.owner_of(PartitionId(0)), Some(NodeId(10)));
assert_eq!(table.owner_of(PartitionId(1)), Some(NodeId(11)));
assert_eq!(table.owner_of(PartitionId(99)), None);

// Node 10 owns partition 0 and is a replica on partition 1
let p10 = table.partitions_for(NodeId(10));
assert_eq!(p10.len(), 2);

// Consistent hashing
let id = [1u8; 16];
let p = PartitionTable::partition_for_intent(&id, 64);
assert!(p.0 < 64);
```

---

## MembershipList

Versioned cluster membership list following the SWIM protocol model. Each member has a liveness state: `Alive`, `Suspect`, or `Dead`. Version is bumped on every state transition.

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum MemberState {
    Alive,
    Suspect,
    Dead,
}

#[derive(Clone, Debug)]
pub struct MembershipList {
    pub members: Vec<(NodeId, MemberState)>,
    pub version: u64,
}
```

### Methods

```rust
impl MembershipList {
    /// Create an empty membership list (version 0).
    pub fn new() -> Self;

    /// Add a member as Alive. If the node already exists, its state is
    /// set back to Alive. Bumps the version.
    pub fn add_member(&mut self, node: NodeId);

    /// Transition a member to Suspect. Bumps the version.
    /// No-op if the node is not present.
    pub fn mark_suspect(&mut self, node: NodeId);

    /// Transition a member to Dead. Bumps the version.
    /// No-op if the node is not present.
    pub fn mark_dead(&mut self, node: NodeId);

    /// Return all members whose state is Alive.
    pub fn alive_members(&self) -> Vec<NodeId>;
}
```

### Example

```rust
use tala_net::{MembershipList, MemberState, NodeId};

let mut ml = MembershipList::new();
ml.add_member(NodeId(1));
ml.add_member(NodeId(2));
ml.add_member(NodeId(3));
assert_eq!(ml.alive_members().len(), 3);
assert_eq!(ml.version, 3);

ml.mark_suspect(NodeId(2));
assert_eq!(ml.alive_members().len(), 2);

ml.mark_dead(NodeId(2));
assert_eq!(ml.alive_members().len(), 2);

// Re-add a dead node -> back to Alive
ml.add_member(NodeId(2));
assert_eq!(ml.alive_members().len(), 3);
```

---

## TLV Codec

Serializes and deserializes `Message` values using a Type-Length-Value wire format.

**Wire layout:** `[tag:1 byte][length:4 bytes LE][payload:length bytes]`

| Tag | Message |
|-----|---------|
| `0x01` | Ping |
| `0x02` | Pong |
| `0x03` | IntentForward |
| `0x04` | SegmentSync |
| `0x05` | MembershipUpdate |
| `0x06` | PartitionTableUpdate |

```rust
/// Encode a Message into a TLV byte buffer.
pub fn encode(msg: &Message) -> Vec<u8>;

/// Decode a Message from a TLV byte buffer.
/// Returns TalaError::SegmentCorrupted on invalid or truncated input.
pub fn decode(data: &[u8]) -> Result<Message, TalaError>;
```

### Example

```rust
use tala_net::{encode, decode, Message, NodeId};

let msg = Message::Ping { from: NodeId(42), seq: 1 };
let bytes = encode(&msg);
let decoded = decode(&bytes).unwrap();
assert_eq!(decoded, msg);
```

---

## InProcessNetwork and InProcessTransport

A simulated network for testing distributed protocols without real sockets. Nodes are connected via `mpsc` channels. Messages are delivered in FIFO order per sender-receiver pair.

### InProcessNetwork

```rust
pub struct InProcessNetwork { /* private */ }

impl InProcessNetwork {
    /// Create a new empty in-process network.
    pub fn new() -> Self;

    /// Register a node and return its transport handle.
    pub fn add_node(&self, id: NodeId) -> InProcessTransport;
}
```

### InProcessTransport

```rust
pub struct InProcessTransport { /* private */ }

impl InProcessTransport {
    /// Send a message to a specific peer. Silently drops if the peer is not
    /// registered (mirrors UDP-like fire-and-forget semantics).
    pub fn send(&self, to: NodeId, msg: Message);

    /// Try to receive the next message (non-blocking).
    /// Returns (sender_id, message) or None if no message is available.
    pub fn recv(&self) -> Option<(NodeId, Message)>;

    /// Broadcast a message to all registered peers except self.
    pub fn broadcast(&self, msg: Message);
}
```

### Example

```rust
use tala_net::{InProcessNetwork, NodeId, Message};

let net = InProcessNetwork::new();
let t1 = net.add_node(NodeId(1));
let t2 = net.add_node(NodeId(2));
let t3 = net.add_node(NodeId(3));

// Point-to-point
t1.send(NodeId(2), Message::Ping { from: NodeId(1), seq: 1 });
let (from, msg) = t2.recv().unwrap();
assert_eq!(from, NodeId(1));

// Broadcast
t1.broadcast(Message::Pong { from: NodeId(1), seq: 42 });
assert!(t2.recv().is_some()); // t2 receives
assert!(t3.recv().is_some()); // t3 receives
assert!(t1.recv().is_none()); // t1 does NOT receive its own broadcast
```
