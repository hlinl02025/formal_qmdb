#### The main Data structure of QMDB:

## Entry Structure

Think of an **Entry** as a single record in a filing cabinet, but with some special properties:

### What's Inside Each Entry:
1. **Key** - The label or identifier (like a file name)
2. **Value** - The actual data stored (like the file contents)
3. **Next Key Hash** - A pointer to the next entry in alphabetical order
4. **Version** - A timestamp showing when this entry was created
5. **Serial Number** - A unique ID number for this entry within its section

### Special Properties:
- **Immutable**: Once written, an entry can never be changed. Instead, a new version is created.
- **Linked**: Each entry knows which entry comes next in alphabetical order
- **Sorted**: All entries are automatically kept in alphabetical order by their keys

### Visual Example:
```
Entry A: Key="apple", Value="red fruit", Next="banana"
Entry B: Key="banana", Value="yellow fruit", Next="cherry"  
Entry C: Key="cherry", Value="red berry", Next="date"
```

## Twig Structure

A **Twig** is like a mini filing cabinet that can hold exactly 2,048 entries. Think of it as a fixed-size container with special organization:

### What's Inside Each Twig:
1. **Left Side (Entry Tree)**: A tree structure containing all the entry hashes
2. **Right Side (Active Bits Tree)**: A tree structure tracking which entries are still valid
3. **Root**: A combined hash that represents the entire twig

### Visual Structure:
```
Twig Root
├── Left Root (Entry Tree Root)
│   ├── Level 0: 2048 entry hashes (the actual data)
│   ├── Level 1: 1024 internal nodes
│   ├── Level 2: 512 internal nodes
│   └── ... (up to Level 11)
└── Right Root (Active Bits Tree Root)
    ├── Level 8: 256 active bits (tracking valid entries)
    ├── Level 9: 128 internal nodes
    ├── Level 10: 64 internal nodes
    └── Level 11: 32 internal nodes
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

## The Hierarchical Merkle Tree Architecture

QMDB's Merkle tree is built like a **nested set of Russian dolls** - each level contains and organizes the level below it. Here's how entries and twigs fit into this hierarchy:

## 1. **The Complete Tree Structure**

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

## 2. **How Entries Become Tree Leaves**

### **Entry → Leaf Node**
- Each **entry** becomes a **leaf node** in the Merkle tree
- The leaf contains the **hash of the entry's data**
- These leaves are at **Level 0** of the twig's internal tree

```
Entry: Key="apple", Value="red fruit", Next="banana", Version=100, SN=42
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

## 3. **How Twigs Organize Entries**

### **Twig as a Mini Merkle Tree**
Each twig is a **complete binary Merkle tree** with exactly 2,048 leaves:

```
Twig Root (Level 11)
├── Left Root (Level 10)
│   ├── Level 9: 512 nodes
│   ├── Level 8: 256 nodes
│   ├── Level 7: 128 nodes
│   ├── Level 6: 64 nodes
│   ├── Level 5: 32 nodes
│   ├── Level 4: 16 nodes
│   ├── Level 3: 8 nodes
│   ├── Level 2: 4 nodes
│   ├── Level 1: 2 nodes
│   └── Level 0: 2048 leaves (the entries!)
└── Right Root (Active Bits Tree)
    ├── Level 8: 256 active bits
    ├── Level 9: 128 nodes
    ├── Level 10: 64 nodes
    └── Level 11: 32 nodes
```

### **The Two-Tree Structure Within Each Twig**

#### **Left Tree (Entry Tree)**
- Contains the actual entry data
- Each leaf is the hash of an entry
- Internal nodes are hashes of their children

#### **Right Tree (Active Bits Tree)**
- Tracks which entries are still valid
- Each leaf represents 8 entries (1 bit per entry)
- Allows "deletion" without actually removing data

## 4. **How Twigs Connect to Higher Levels**

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

## 5. **The Complete Data Flow**

### **From Entry to Global Root:**
```
1. Entry Data → Hash → Leaf Node (Level 0)
2. 2048 Leaf Nodes → Hash → Internal Nodes (Levels 1-10)
3. Left Tree Root + Right Tree Root → Hash → Twig Root (Level 11)
4. Twig Root → Leaf → Shard Tree (Level 12+)
5. Shard Roots → Hash → Global Root (Level 64)
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

## 6. **Key Relationships and Properties**

### **Entry Properties in the Tree:**
- **Position**: Each entry has a fixed position in its twig (0-2047)
- **Serial Number**: Unique identifier within the shard
- **Hash**: The entry's data is hashed to create the leaf node
- **Ordering**: Entries are sorted by key hash within each twig

### **Twig Properties in the Tree:**
- **Fixed Size**: Always exactly 2,048 entries
- **Independent**: Each twig can be processed separately
- **Verifiable**: The twig root proves the integrity of all its entries
- **Updatable**: Only the current twig (youngest) can be modified

### **Tree Level Relationships:**
```
Level 0: Entry hashes (2,048 per twig)
Level 1-10: Internal nodes of entry tree
Level 11: Twig roots
Level 12+: Shard tree nodes
Level 64: Global root
```
#### The core algorithms of QMDB:
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
8.  inspector.on_read_entry(prev_entry)
9.  
10. // Create new entry inheriting next pointer
11. new_entry ← Entry {
12.     key: key,
13.     value: value,
14.     next_key_hash: prev_entry.next_key_hash(),
15.     version: version,
16.     serial_number: serial_number
17. }
18. dsn_list ← ∅
19. create_pos ← storage.append_entry(new_entry, dsn_list)
20. inspector.on_append_entry(new_entry)
21. 
22. // Update previous entry to point to new entry
23. inspector.on_deactivate_entry(prev_entry)
24. prev_changed ← Entry {
25.     key: prev_entry.key(),
26.     value: prev_entry.value(),
27.     next_key_hash: key_hash,
28.     version: version,
29.     serial_number: serial_number + 1
30. }
31. dsn_list ← [prev_entry.serial_number()]
32. new_pos ← storage.append_entry(prev_changed, dsn_list)
33. inspector.on_append_entry(prev_changed)
34. 
35. // Update indices
36. indexer.add_kv(key_hash, create_pos, serial_number)
37. indexer.change_kv(prev_entry.key_hash[0:10], old_pos, new_pos, 
38.     prev_entry.serial_number(), serial_number + 1)
39. 
40. storage.try_twice_compact()
41. 
42. RETURN EntryMutationResult {
43.     num_active: 2,
44.     num_deactive: 1
45. }
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
8.  inspector.on_read_entry(old_entry)
9.  
10. // Deactivate old entry and create updated version
11. inspector.on_deactivate_entry(old_entry)
12. new_entry ← Entry {
13.     key: key,
14.     value: value,
15.     next_key_hash: old_entry.next_key_hash(),
16.     version: version,
17.     serial_number: serial_number
18. }
19. dsn_list ← [old_entry.serial_number()]
20. new_pos ← storage.append_entry(new_entry, dsn_list)
21. inspector.on_append_entry(new_entry)
22. 
23. // Update index to reflect new position
24. indexer.change_kv(key_hash, old_pos, new_pos,
25.     old_entry.serial_number(), serial_number)
26. 
27. storage.try_twice_compact()
28. 
29. RETURN EntryMutationResult {
30.     num_active: 1,
31.     num_deactive: 1
32. }
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
7.  inspector.on_read_entry(del_entry)
8.  del_entry_sn ← del_entry.serial_number()
9.  old_next_key_hash ← del_entry.next_key_hash()
10. 
11. // Deactivate entry to delete
12. inspector.on_deactivate_entry(del_entry)
13. 
14. // Find previous entry in linked list
15. read_entry_buf.clear()
16. prev_pos ← storage.read_previous_entry(height, key_hash, read_entry_buf,
17.     predicate: entry.next_key_hash == key_hash)
18. prev_entry ← read_entry_buf.as_entry_bz()
19. inspector.on_read_entry(prev_entry)
20. inspector.on_deactivate_entry(prev_entry)
21. 
22. // Update previous entry to skip deleted entry
23. prev_changed ← Entry {
24.     key: prev_entry.key(),
25.     value: prev_entry.value(),
26.     next_key_hash: old_next_key_hash,
27.     version: version,
28.     serial_number: serial_number
29. }
30. dsn_list ← [del_entry_sn, prev_entry.serial_number()]
31. new_pos ← storage.append_entry(prev_changed, dsn_list)
32. inspector.on_append_entry(prev_changed)
33. 
34. // Update indices
35. indexer.erase_kv(key_hash, del_entry_pos, del_entry_sn)
36. k80 ← prev_entry.key_hash[0:10]
37. indexer.change_kv(k80, prev_pos, new_pos,
38.     prev_entry.serial_number(), serial_number)
39. 
40. storage.try_twice_compact()
41. 
42. RETURN EntryMutationResult {
43.     num_active: 1,
44.     num_deactive: 2
45. }
```

# How QMDB can guarantee the Linked List Invariant:

QMDB maintains linked list invariant, which is defined as follows:

QMDB maintains a sorted linked list where each entry contains a `next_key_hash` field that points to the next entry in lexicographic order. The invariant states:

```
∀ entry_i, entry_j: entry_i.key_hash < entry_j.key_hash ⟹ 
entry_i.next_key_hash = entry_j.key_hash
```


This invariant is maintained through:
1. **Predicate-based insertion** ensuring correct positioning
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
  
2. **Atomic linked list updates** preserving connectivity

For any three consecutive entries `A`, `B`, `C` in the linked list:

**Before any operation:**
```
A.key_hash < B.key_hash < C.key_hash
A.next_key_hash = B.key_hash
B.next_key_hash = C.key_hash
```

### **Create Operation Proof**

When inserting entry `X` between `A` and `B`:

1. **Predicate ensures:** `A.key_hash < X.key_hash < B.key_hash`
2. **New entry creation:** `X.next_key_hash = A.next_key_hash = B.key_hash`
3. **Previous entry update:** `A.next_key_hash = X.key_hash`

**Result:**
```
A.key_hash < X.key_hash < B.key_hash < C.key_hash
A.next_key_hash = X.key_hash
X.next_key_hash = B.key_hash
B.next_key_hash = C.key_hash
```

### **Delete Operation Proof**

When deleting entry `B` between `A` and `C`:

1. **Save reference:** `old_next = B.next_key_hash = C.key_hash`
2. **Update previous:** `A.next_key_hash = old_next = C.key_hash`

**Result:**
```
A.key_hash < C.key_hash
A.next_key_hash = C.key_hash
C.next_key_hash = D.key_hash
```

3. **Append-only operations** preventing structural corruption
### **A. Append-Only Design**

The append-only nature ensures:
- **No in-place modifications** that could break the linked list
- **Immutable history** preserves the linked list structure
- **Deactivation tracking** maintains referential integrity

### **B. Atomic Operations**

Each mutation operation is atomic:
- **Create**: Two entries created (new + updated previous)
- **Delete**: One entry created (updated previous)
- **Update**: One entry created (updated version)

This ensures the linked list is always in a consistent state.

4. **Index consistency** 

The index maintains consistency with the linked list:
```
self.indexer.add_kv(&key_hash[..], create_pos, serial_number);
self.indexer.change_kv(
    &prev_entry.key_hash()[..10],
    old_pos,
    new_pos,
    prev_entry.serial_number(),
    serial_number + 1,
);
```
The indices are updated to reflect the new positions in time. 
