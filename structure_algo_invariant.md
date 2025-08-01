# 1. The main Data structure of QMDB:

## 1.1 Entry Structure

Think of an **Entry** as a single record in a filing cabinet, but with some special properties:

### What's Inside Each Entry:
1. **Key** - The label or identifier
2. **Value** - The actual data stored
3. **Next Key Hash** - A pointer to the next entry in alphabetical order by their keys
4. **Version** - A unique identifier for the transaction that created or modified the entry
5. **Serial Number** - A unique ID number for this entry within its shard
6. **dsn_list** - A list of Deactivated serial numbers

### Special Properties:
- **Immutable**: Once written, an entry can never be changed. Instead, a new version is created.
- **Sorted**: All entries are automatically kept in alphabetical order by their keys

### Visual Example:
```
Entry A: Key="apple", Value="red fruit", Next="0x1d3f"
Entry B: Key="banana", Value="yellow fruit", Next="0x1e36"  
Entry C: Key="cherry", Value="red berry", Next="0x12fe"
```

## 1.2 Twig Structure

A **Twig** is like a mini filing cabinet that can hold exactly 2,048 entries. Think of it as a fixed-size container with special organization:

### What's Inside Each Twig:
1. **Left Side (Entry Tree)**: A tree structure containing all the entry hashes
2. **Right Side (Active Bits Tree)**: A tree structure tracking which entries are still valid
3. **Root**: A combined hash that represents the entire twig

### Visual Structure:

```plaintext
                             TwigRoot
                                |
                ┌───────────────┴───────────────┐
                │                               │
           Left Root                       Right Root
         (Entry Tree)                  (Active Bits Tree)
                │                               │
         [11-level binary tree]         [3-level binary tree]
                │                               │
      2048 Entry Hashes (leaves)      8 leaves × 256 ActiveBits
```


### Key Concepts:

#### **Fixed Size**
- Every twig holds exactly 2,048 entries (no more, no less)
- When a twig fills up, a new twig is created
- This makes the system predictable and efficient

#### **Two Trees in One**
- **Left Tree**: Stores the actual entry data in a tree structure
- **Right Tree**: Keeps track of which entries are still "active" (not deleted)

#### **Active Bits**
Think of active bits as a checklist:
- Each bit represents one entry
- `1` = entry is still valid
- `0` = entry has been deleted or replaced
- This allows the system to "delete" entries without actually removing them

# 2. The Hierarchical Merkle Tree Architecture

QMDB's Merkle tree is built like a **nested set of Russian dolls** - each level contains and organizes the level below it. Here's how entries and twigs fit into this hierarchy:

## 2.1. **The Complete Tree Structure**

```
Global Root (Level 64)
├── Shard 0 Root
│   ├── Twig 0 Root (Level 12)
│   ├── Twig 1 Root (Level 12)
│   └── ... (more twigs)
├── Shard 1 Root
│   ├── Twig 0 Root (Level 12)
│   └── ... (more twigs)
└── ... (14 more shards)
```

## 2.2. **How Entries Become Tree Leaves**

### **Entry → Leaf Node**
- Each **entry** becomes a **leaf node** in the Merkle tree
- The leaf contains the **hash of the entry's data**
- These leaves are at **Level 0** of the twig's internal tree

```
Entry: Key="apple", Value="red fruit", Next="0xa9823", Version=100, SN=42
    ↓ (hash the entry)
Leaf Node: Hash(entry_data) = 0x1234...abcd
```

### **Visual Example of Entry to Leaf:**
```
Entry A → Leaf 0: Hash("apple|red fruit|banana|100|42")
Entry B → Leaf 1: Hash("banana|yellow fruit|cherry|101|43")  
Entry C → Leaf 2: Hash("cherry|red berry|date|102|44")
...
Entry 2047 → Leaf 2047: Hash("zebra|striped animal|end|1247|2289")
```

## 2.3. **How Twigs Organize Entries**

### **Twig as a Mini Merkle Tree**

The concrete visual structure of Twig as a Mini Merkle Tree can be found in the previous section on the Data Structure of QMDB. 

### **The Two-Tree Structure Within Each Twig**

#### **Left Tree (Entry Tree)**
- Contains the actual entry data
- Each leaf is the hash of an entry
- Internal nodes are hashes of their children

#### **Right Tree (Active Bits Tree)**
- Tracks which entries are still valid
- Each leaf represents 8 entries (256 bit per entry)
- Allows "deletion" without actually removing data

## 2.4. **How Twigs Connect to Higher Levels**

### **Twig Roots Become Leaves**
Each twig's root becomes a **leaf node** in the next level up:

```
Twig 0 Root → Leaf in Shard Tree
Twig 1 Root → Leaf in Shard Tree
Twig 2 Root → Leaf in Shard Tree
...
```

### **Shard Level Organization**
```
Shard Root (Level 13+)
├── Twig 0 Root (from Level 12)
├── Twig 1 Root (from Level 12)
├── Twig 2 Root (from Level 12)
└── ... (more twig roots)
```

## 2.5. **The Complete Data Flow**

### **From Entry to Global Root:**
```
1. Entry Data → Hash → Leaf Node (Level 0)
2. 2048 Leaf Nodes → Hash → Internal Nodes (Levels 1-10)
3. Left Tree Root + Right Tree Root → Hash → Twig Root (Level 11)
4. Twig Root → Leaf → Shard Tree (Level 12+)
5. Shard Roots → Global Root (Level 64)
```

### **Visual Example:**
```
Entry "apple" → Hash → Leaf 0
    ↓ (with 2047 other entries)
Twig 0 Left Root
    ↓ (combined with Active Bits Root)
Twig 0 Root
    ↓ (with other twig roots)
Shard 0 Root
    ↓ (with other shard roots)
Global Root
```

## 2.6. **Key Relationships and Properties**

### **Entry Properties in the Tree:**
- **Position**: Each entry has a fixed position in its twig (0-2047)
- **Serial Number**: Unique identifier within the shard
- **Hash**: The entry's data is hashed to create the leaf node

### **Twig Properties in the Tree:**
- **Fixed Size**: Always exactly 2,048 entries
- **Independent**: Each twig can be processed separately
- **Verifiable**: The twig root proves the integrity of all its entries
- **Updatable**: Only the current twig (youngest) and old twigs' right trees can be modified

### **Tree Level Relationships:**
```
Level 0: Entry hashes (2,048 per twig)
Level 1-10: Internal nodes of entry tree
Level 11: Twig roots
Level 12+: Shard tree nodes
Level 64: Global root
```
# 3. The core algorithms of QMDB:

There are mainly three algorithms: CREATE-KV, UPDATE-KV, DELETE-KV and COMPACT. The following are the pseudocodes of these algorithms. 


## Algorithm 1: CREATE-KV

**Input:** `read_entry_buf`, `key_hash`, `key`, `value`, `shard_id`, `version`, `serial_number`, `r`

**Output:** `EntryMutationResult`

```
1.  ASSERT value ≠ ∅
2.  height ← split_task_id(version).height
3.  
4.  // Find previous entry in sorted key space
5.  old_pos ← storage.read_previous_entry(height, key_hash, read_entry_buf, 
6.      predicate: entry.key_hash < key_hash ∧ key_hash < entry.next_key_hash)
7.  prev_entry ← read_entry_buf.as_entry_bz()
8.  
9.  // Create new entry inheriting next pointer
10. new_entry ← Entry {
11.     key: key,
12.     value: value,
13.     next_key_hash: prev_entry.next_key_hash(),
14.     version: version,
15.     serial_number: serial_number
16. }
17. dsn_list ← ∅
18. create_pos ← storage.append_entry(new_entry, dsn_list)
19. 
20. // Update previous entry to point to new entry
21. prev_changed ← Entry {
22.     key: prev_entry.key(),
23.     value: prev_entry.value(),
24.     next_key_hash: key_hash,
25.     version: version,
26.     serial_number: serial_number + 1
27. }
28. dsn_list ← [prev_entry.serial_number()]
29. new_pos ← storage.append_entry(prev_changed, dsn_list)
30. 
31. // Update entry position mappings
32. add_entry_position(key_hash, create_pos, serial_number)
33. update_entry_position(prev_entry.key_hash[0:10], old_pos, new_pos, 
34.     prev_entry.serial_number(), serial_number + 1)
35. 
36. storage.try_twice_compact()
37. 
38. RETURN EntryMutationResult {
39.     num_active: 2,
40.     num_deactive: 1
41. }
```

## Algorithm 2: UPDATE-KV

**Input:** `read_entry_buf`, `key_hash`, `key`, `value`, `shard_id`, `version`, `serial_number`, `r`

**Output:** `EntryMutationResult`

```
1.  ASSERT value ≠ ∅
2.  height ← split_task_id(version).height
3.  
4.  // Find existing entry to update
5.  old_pos ← storage.read_prior_entry(height, key_hash, read_entry_buf,
6.      predicate: entry.key == key)
7.  old_entry ← read_entry_buf.as_entry_bz()
8.  
9.  // Create updated version
10. new_entry ← Entry {
11.     key: key,
12.     value: value,
13.     next_key_hash: old_entry.next_key_hash(),
14.     version: version,
15.     serial_number: serial_number
16. }
17. dsn_list ← [old_entry.serial_number()]
18. new_pos ← storage.append_entry(new_entry, dsn_list)
19. 
20. // Update entry position mapping
21. update_entry_position(key_hash, old_pos, new_pos,
22.     old_entry.serial_number(), serial_number)
23. 
24. storage.try_twice_compact()
25. 
26. RETURN EntryMutationResult {
27.     num_active: 1,
28.     num_deactive: 1
29. }
```

## Algorithm 3: DELETE-KV

**Input:** `read_entry_buf`, `key_hash`, `key`, `shard_id`, `version`, `serial_number`, `r`

**Output:** `EntryMutationResult`

```
1.  height ← split_task_id(version).height
2.  
3.  // Find entry to delete
4.  del_entry_pos ← storage.read_prior_entry(height, key_hash, read_entry_buf,
5.      predicate: entry.key == key)
6.  del_entry ← read_entry_buf.as_entry_bz()
7.  del_entry_sn ← del_entry.serial_number()
8.  old_next_key_hash ← del_entry.next_key_hash()
9. 
10. // Find previous entry in linked list
11. read_entry_buf.clear()
12. prev_pos ← storage.read_previous_entry(height, key_hash, read_entry_buf,
13.     predicate: entry.next_key_hash == key_hash)
14. prev_entry ← read_entry_buf.as_entry_bz()
15. 
16. // Update previous entry to skip deleted entry
17. prev_changed ← Entry {
18.     key: prev_entry.key(),
19.     value: prev_entry.value(),
20.     next_key_hash: old_next_key_hash,
21.     version: version,
22.     serial_number: serial_number
23. }
24. dsn_list ← [del_entry_sn, prev_entry.serial_number()]
25. new_pos ← storage.append_entry(prev_changed, dsn_list)
26. 
27. // Update entry position mappings
28. remove_entry_position(key_hash, del_entry_pos, del_entry_sn)
29. k80 ← prev_entry.key_hash[0:10]
30. update_entry_position(k80, prev_pos, new_pos,
31.     prev_entry.serial_number(), serial_number)
32. 
33. storage.try_twice_compact()
34. 
35. RETURN EntryMutationResult {
36.     num_active: 1,
37.     num_deactive: 2
38. }
```

## Algorithm 4: COMPACT

**Input:** `ebw` (Entry Buffer Writer Guard), `r` (Optional Operation Record), `comp_idx` (Compaction Index)

**Output:** None (void function)

```
1.  // Find next active entry to compact
2.  (job, kh) ← LOOP
3.      job ← compact_consumer.consume()
4.      entry_bz ← EntryBz { bz: job.entry_bz }
5.      kh ← entry_bz.key_hash()
6.      
7.      // Check if entry is still active
8.      IF is_active_entry(kh, job.old_pos, entry_bz.serial_number()) THEN
9.          BREAK (job, kh)
10.     END IF
11.     
12.     // Return inactive entry to queue
13.     compact_consumer.send_returned(job)
14. END LOOP
15. 
16. // Create new entry with updated serial number
17. new_entry ← Entry {
18.     key: entry_bz.key(),
19.     value: entry_bz.value(),
20.     next_key_hash: entry_bz.next_key_hash(),
21.     version: entry_bz.version(),
22.     serial_number: sn_end
23. }
24. 
25. // Prepare deactivation list
26. dsn_list ← [entry_bz.serial_number()]
27. 
28. // Append new entry to storage
29. new_pos ← ebw.append(new_entry, dsn_list)
30. 
31. // Update entry position mapping
32. update_entry_position(kh, job.old_pos, new_pos, dsn_list[0], sn_end)
33. 
34. // Update metadata
35. sn_end ← sn_end + 1
36. sn_start ← entry_bz.serial_number() + 1
37. compact_done_pos ← job.old_pos + entry_bz.len()
38. 
39. // Return job to queue for reuse
40. compact_consumer.send_returned(job)
```

# 4. How QMDB guarantees the Linked List Invariant:

QMDB maintains the linked list invariant, which is defined as follows:

QMDB maintains a sorted linked list where each entry contains a `next_key_hash` field that points to the next entry in lexicographic order. The invariant states:

```
∀ entry_i, entry_j: entry_i.key_hash < entry_j.key_hash ⟹ 
entry_i.next_key_hash = entry_j.key_hash
```


This invariant is maintained through:
## 4.1. Predicate-based insertion ensuring correct positioning
The key mechanism is in the `read_previous_entry` function with its predicate:

```rust
// In create_kv function
let old_pos = self.storage.read_previous_entry(
    height, 
    key_hash, 
    read_entry_buf, 
    |entry| {
        entry.key_hash() < *key_hash && &key_hash[..] < entry.next_key_hash()
    }
);
```

This predicate ensures:
- `entry.key_hash() < *key_hash`: The found entry comes before the new key
- `&key_hash[..] < entry.next_key_hash()`: The new key comes before what the entry currently points to
  
## 4.2. Atomic linked list updates preserving connectivity

For any three consecutive entries `A`, `B`, `C` in the linked list:

### Before any operation:
```
A.key_hash < B.key_hash < C.key_hash
A.next_key_hash = B.key_hash
B.next_key_hash = C.key_hash
```

### Create Operation 

When inserting entry `X` between `A` and `B`:

1. **Predicate ensures:** `A.key_hash < X.key_hash < B.key_hash`
2. **New entry creation:** `X.next_key_hash = A.next_key_hash = B.key_hash`
3. **Previous entry update:** `A.next_key_hash = X.key_hash`

### Result:
```
A.key_hash < X.key_hash < B.key_hash < C.key_hash
A.next_key_hash = X.key_hash
X.next_key_hash = B.key_hash
B.next_key_hash = C.key_hash
```

### Delete Operation 

When deleting entry `B` between `A` and `C`:

1. **Save reference:** `old_next = B.next_key_hash = C.key_hash`
2. **Update previous:** `A.next_key_hash = old_next = C.key_hash`

### Result:
```
A.key_hash < C.key_hash
A.next_key_hash = C.key_hash
C.next_key_hash = D.key_hash
```

## 4.3. Append-only operations preventing structural corruption
### A. Append-Only Design

The append-only nature ensures:
- **No in-place modifications** that could break the linked list
- **Immutable history** preserves the linked list structure
- **Deactivation tracking** maintains referential integrity

### B. Atomic Operations

Each mutation operation is atomic:
- **Create**: Two entries created (new + updated previous)
- **Delete**: One entry created (updated previous)
- **Update**: One entry created (updated version)

This ensures the linked list is always in a consistent state.

## 4.4. Index consistency

The index maintains consistency with the linked list, the indices are updated to reflect the new positions in time. 

# 5. How QMDB guarantees Temporal Verifiability Invariant:

QMDB maintains the Temporal Verifiability Invariant, which is defined as follows:


∀ entry ∈ σ, ∀ time t: ∃ verification_procedure(entry, t) → {active, inactive}



The invariant is maintained because:

1. **Existence**: Every entry has a unique serial number and version
2. **Deactivation**: Deactivated entries are explicitly tracked in DSN lists
3. **Indexing**: Active entries are tracked in real-time index
4. **Temporal**: Version numbers provide temporal ordering
5. **Cryptographic**: Merkle tree ensures integrity of all state changes

One can run the following verification algorithm to check the active state of any entry. 

**Verification Algorithm**
```
verify_active(entry, time) {
    // Check if entry existed at time
    if (entry.version > time) return inactive
    
    // Check if entry was deactivated before time
    if (entry.serial_number $\in$ deactivated_before_time(time)) return inactive
    
    return active
}
```

Note that the verification algorithm is based on the Unique Key Assumption that QMDB guarantees:

$$\forall \text{entry}_i, \text{entry}_j \in \sigma: \text{entry}_i.\text{key} = \text{entry}_j.\text{key} \implies \text{entry}_i = \text{entry}_j$$

