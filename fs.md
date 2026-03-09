# Filesystems in Linux 6.19 — ext4 and the VFS Interface

> Source base: `/home/inineapa/Lab/linux-6.19`

---

## Before You Begin

If you have used `mkfs.ext4`, `mount`, and then read/written files, you already know what a filesystem does from the outside: it turns a raw block device into a tree of named files and directories. What you have not seen is how ext4 organizes data on disk — block groups, extent trees, and the journal — and how the VFS layer sits between your `read()`/`write()` calls and ext4's on-disk layout.

This document covers the ext4 filesystem as a concrete example, starting with the on-disk format and working up to how ext4 plugs into the VFS through operation tables. For the VFS abstractions themselves (superblock, inode, dentry, file), see `vfs.md`. For how I/O flows through all the layers end-to-end, see `io_path.md`.

---

## 1. The On-Disk Layout — How ext4 Organizes a Partition

### 1.1 Block Groups

An ext4 filesystem divides the partition into **block groups**, each containing a fixed number of blocks (typically 32,768 blocks = 128 MB with 4 KB blocks). This grouping keeps related data close together on disk, reducing seek times.

```
Partition:
┌──────────┬──────────────┬──────────────┬──────────────┬─────┐
│ Boot     │ Block Group 0│ Block Group 1│ Block Group 2│ ... │
│ (1024 B) │              │              │              │     │
└──────────┴──────────────┴──────────────┴──────────────┴─────┘

Each Block Group:
┌────────┬────────┬──────┬──────┬────────────┬────────────────┐
│Super   │Group   │Block │Inode │ Inode      │ Data           │
│Block   │Desc    │Bitmap│Bitmap│ Table      │ Blocks         │
│(copy)  │Table   │      │      │            │                │
│        │(copy)  │(1blk)│(1blk)│(N blocks)  │(rest of group) │
└────────┴────────┴──────┴──────┴────────────┴────────────────┘
```

- **Superblock**: The filesystem metadata — total blocks, total inodes, block size, features, UUID. Stored at byte offset 1024 in block group 0, with backup copies in certain other groups.
- **Group Descriptor Table**: One entry per block group, describing where the bitmaps and inode table are for that group.
- **Block Bitmap**: One bit per block in the group — 1 = allocated, 0 = free.
- **Inode Bitmap**: One bit per inode slot in the group — 1 = allocated, 0 = free.
- **Inode Table**: Array of on-disk inode structures for this group.
- **Data Blocks**: The actual file content and directory data.

### 1.2 The Superblock

`struct ext4_super_block` (`fs/ext4/ext4.h:1345–1470`) is the on-disk superblock:

| Field | Line | Purpose |
|-------|------|---------|
| `s_inodes_count` | 1346 | Total number of inodes in the filesystem |
| `s_blocks_count_lo` | 1347 | Total block count (low 32 bits) |
| `s_free_blocks_count_lo` | 1349 | Free blocks count |
| `s_free_inodes_count` | 1350 | Free inodes count |
| `s_first_data_block` | 1351 | First data block (0 for 4K blocks, 1 for 1K blocks) |
| `s_log_block_size` | 1352 | Block size = 2^(10 + this value). For 4K: value = 2 |
| `s_blocks_per_group` | 1354 | Blocks per block group (typically 32768) |
| `s_inodes_per_group` | 1356 | Inodes per block group |
| `s_magic` | 1361 | Magic number: `0xEF53` |
| `s_state` | 1362 | Filesystem state (clean, has errors, etc.) |
| `s_feature_compat` | 1387 | Compatible feature flags |
| `s_feature_incompat` | 1388 | Incompatible feature flags |
| `s_journal_inum` | 1405 | Inode number of the journal file |
| `s_uuid` | 1390 | 128-bit filesystem UUID |

When you run `mkfs.ext4 /dev/sda1`, it writes this superblock at byte 1024, creates the block group structures, and initializes the bitmaps and inode tables.

### 1.3 The Block Group Descriptor

`struct ext4_group_desc` (`ext4.h:403–428`):

| Field | Line | Purpose |
|-------|------|---------|
| `bg_block_bitmap_lo` | 405 | Block number of the block bitmap |
| `bg_inode_bitmap_lo` | 406 | Block number of the inode bitmap |
| `bg_inode_table_lo` | 407 | Block number of the first inode table block |
| `bg_free_blocks_count_lo` | 408 | Free blocks in this group |
| `bg_free_inodes_count_lo` | 409 | Free inodes in this group |
| `bg_used_dirs_count_lo` | 410 | Number of directories in this group |
| `bg_flags` | 411 | INODE_UNINIT, BLOCK_UNINIT, INODE_ZEROED |
| `bg_checksum` | 416 | Group descriptor checksum |

### 1.4 The On-Disk Inode

`struct ext4_inode` (`ext4.h:804–863`) stores all file metadata on disk:

```c
struct ext4_inode {
    __le16 i_mode;            // 805 — file type + permissions (S_IFREG, 0644, etc.)
    __le16 i_uid;             // 806 — owner UID (low 16 bits)
    __le32 i_size_lo;         // 807 — file size (low 32 bits)
    __le32 i_atime;           // 808 — last access time
    __le32 i_ctime;           // 809 — inode change time
    __le32 i_mtime;           // 810 — last modification time
    __le32 i_dtime;           // 811 — deletion time (0 if not deleted)
    __le16 i_links_count;     // 813 — hard link count
    __le32 i_blocks_lo;       // 814 — block count (in 512-byte units)
    __le32 i_flags;           // 815 — inode flags (EXT4_EXTENTS_FL, etc.)
    ...
    __le32 i_block[EXT4_N_BLOCKS]; // 827 — 15 block pointers (60 bytes)
    ...
};
```

The `i_block[15]` array is the heart of data mapping. In modern ext4, it does not hold block pointers — it holds an **extent tree** root (when `EXT4_EXTENTS_FL` is set in `i_flags`).

---

## 2. The Extent Tree — Mapping File Offsets to Disk Blocks

### 2.1 Why Extents?

The old ext2/ext3 scheme used indirect block pointers: 12 direct pointers, then single/double/triple indirect blocks. For a 1 GB file, this created thousands of individual block pointers scattered across the disk.

Extents are smarter: a single extent says "logical blocks 0–999 map to physical blocks 50000–50999." One 12-byte structure replaces 1000 individual pointers.

### 2.2 Extent Structures

`struct ext4_extent` (`fs/ext4/ext4_extents.h:56–61`):

```c
struct ext4_extent {
    __le32 ee_block;      // 57 — first logical block this extent covers
    __le16 ee_len;        // 58 — number of blocks (MSB: unwritten flag)
    __le16 ee_start_hi;   // 59 — physical block (high 16 bits)
    __le32 ee_start_lo;   // 60 — physical block (low 32 bits)
};
```

An extent says: "logical blocks `ee_block` through `ee_block + ee_len - 1` are stored at physical blocks `ee_start` through `ee_start + ee_len - 1`."

`struct ext4_extent_header` (`ext4_extents.h:78–84`):

```c
struct ext4_extent_header {
    __le16 eh_magic;      // 79 — magic number 0xF30A
    __le16 eh_entries;    // 80 — number of valid entries
    __le16 eh_max;        // 81 — max entries that fit
    __le16 eh_depth;      // 82 — depth (0 = leaf with extents)
    __le32 eh_generation; // 83 — generation for COW
};
```

`struct ext4_extent_idx` (`ext4_extents.h:67–73`) — internal (non-leaf) nodes:

```c
struct ext4_extent_idx {
    __le32 ei_block;      // 68 — covers logical blocks from this value
    __le32 ei_leaf_lo;    // 69 — pointer to child block (low 32 bits)
    __le16 ei_leaf_hi;    // 71 — pointer to child block (high 16 bits)
    __u16  ei_unused;
};
```

### 2.3 The Extent Tree Structure

Small files fit their entire extent tree inside the inode's `i_block[15]` array (60 bytes):

```
i_block[]:
┌─────────────────┬────────┬────────┬────────┬────────┐
│ ext4_extent_hdr  │extent 0│extent 1│extent 2│extent 3│
│ (12 bytes)       │(12 B)  │(12 B)  │(12 B)  │(12 B)  │
│ eh_depth=0       │        │        │        │        │
│ eh_entries=4     │        │        │        │        │
└─────────────────┴────────┴────────┴────────┴────────┘
```

For larger files, the tree grows deeper. The root still lives in `i_block[]`, but it contains index entries (`ext4_extent_idx`) pointing to blocks that hold the next level:

```
Inode (i_block):          Depth 1:              Depth 0 (leaf):
┌──────────┐            ┌──────────┐          ┌──────────┐
│ header   │            │ header   │          │ header   │
│ depth=1  │            │ depth=0  │          │ depth=0  │
├──────────┤            ├──────────┤          ├──────────┤
│ idx[0] ──┼──────────► │ extent 0 │          │ extent 5 │
│ idx[1] ──┼─────┐      │ extent 1 │          │ extent 6 │
│ idx[2] ──┼──┐  │      │ extent 2 │          │ ...      │
└──────────┘  │  │      └──────────┘          └──────────┘
              │  └────► (another leaf block)
              └──────► (another leaf block)
```

Maximum depth is 5 levels (`EXT4_MAX_EXTENT_DEPTH`), which supports astronomically large files.

### 2.4 Block Mapping

`ext4_ext_map_blocks()` (`fs/ext4/extents.c:4191`) is the core function that translates a logical block number to a physical block number:

1. Walk the extent tree from the root (`i_block[]`) using binary search at each level
2. If a matching extent is found → return the physical block
3. If no extent covers the requested block → allocate new blocks, insert a new extent
4. Update the extent tree, possibly splitting or merging entries

The higher-level `ext4_map_blocks()` (`fs/ext4/inode.c:693`) wraps this with a cache layer (the extent status tree) so repeated lookups for the same logical block are fast.

---

## 3. Directories — Storing the File Tree

### 3.1 Directory Entries

A directory is just a file whose content is a list of directory entries. Each entry (`struct ext4_dir_entry_2`, `ext4.h:2416–2422`) contains:

```c
struct ext4_dir_entry_2 {
    __le32 inode;         // 2417 — inode number (0 = deleted entry)
    __le16 rec_len;       // 2418 — total entry size (includes padding)
    __u8   name_len;      // 2419 — actual name length
    __u8   file_type;     // 2420 — EXT4_FT_REG_FILE, EXT4_FT_DIR, etc.
    char   name[];        // 2421 — variable-length name (NOT null-terminated)
};
```

Entries are packed back-to-back in directory blocks, with `rec_len` used for alignment and to skip over deleted entries (deleted = the previous entry's `rec_len` absorbs the deleted space).

### 3.2 Linear vs HTree Directories

For small directories, ext4 uses **linear search** — it scans every entry to find a name. For large directories, it uses **HTree** (hash tree) indexing:

- The first block becomes a hash index root
- Names are hashed (half-MD4 or TEA algorithm) into buckets
- Lookups go: hash the name → find the right leaf block → linear search within that block

This makes directory lookup O(1) instead of O(n) for directories with thousands of entries.

### 3.3 Directory Operations

`ext4_dir_inode_operations` (`fs/ext4/namei.c:4212–4231`):

| Operation | Function | Purpose |
|-----------|----------|---------|
| `.lookup` | `ext4_lookup()` (line 1760) | Find a name in the directory |
| `.create` | `ext4_create()` (line 2806) | Create a new file |
| `.mkdir` | `ext4_mkdir()` (line 2988) | Create a new directory |
| `.unlink` | `ext4_unlink()` (line 3292) | Remove a file |
| `.rmdir` | `ext4_rmdir()` | Remove a directory |
| `.symlink` | `ext4_symlink()` | Create a symbolic link |
| `.link` | `ext4_link()` | Create a hard link |
| `.rename` | `ext4_rename2()` | Rename / move a file |

---

## 4. Journaling — Crash Consistency with jbd2

### 4.1 The Problem: Metadata Corruption

Creating a file requires multiple disk writes: update the inode bitmap, write the inode, update the directory entry, update the block bitmap, update the group descriptor. If power fails midway, the filesystem is inconsistent — the bitmap says a block is allocated, but the inode does not reference it (or vice versa).

Without journaling, the fix was `fsck` — a full filesystem scan that could take hours on large disks. Journaling eliminates that.

### 4.2 How the Journal Works

ext4 uses the **jbd2** (Journaling Block Device 2) layer (`fs/jbd2/`, `include/linux/jbd2.h`). The core idea:

1. **Before** modifying any metadata on disk, write the changes to a **journal** (a reserved area on the same filesystem, typically inode 8)
2. **After** the journal write completes, write the actual metadata to its final location
3. If the system crashes during step 2, on next mount the journal is **replayed** — the metadata changes are re-applied from the journal

This guarantees that metadata is always consistent. Either the old state or the new state is on disk — never a half-finished state.

### 4.3 Transaction API

Every metadata modification is wrapped in a **transaction**:

```c
handle_t *handle;

// Start a transaction (reserve journal blocks)
handle = ext4_journal_start(inode, EXT4_HT_INODE, needed_blocks);

// Get write access to a buffer (copies original to journal)
ext4_journal_get_write_access(handle, sb, bh);

// Modify the buffer...
modify_metadata(bh);

// Mark the buffer as dirty in the journal
ext4_journal_dirty_metadata(handle, bh);

// Commit the transaction
ext4_journal_stop(handle);
```

Key types:
- **`handle_t`** — represents one atomic operation within a transaction
- **`transaction_t`** — groups multiple handles into one commit
- **`journal_t`** (`jbd2.h:742+`) — the journal itself

### 4.4 The Three Journaling Modes

| Mode | What is journaled | Performance | Safety |
|------|-------------------|-------------|--------|
| **journal** | Metadata AND data | Slowest — all data written twice | Safest — no data loss |
| **ordered** (default) | Metadata only, but data is written before metadata commit | Medium | Safe — data on disk before metadata says it exists |
| **writeback** | Metadata only, data can be written in any order | Fastest | Weakest — stale data may appear in files after crash |

Mount option: `mount -t ext4 -o data=ordered /dev/sda1 /mnt`

In **ordered** mode (the default), when you `write()` data to a file, ext4:
1. Writes the data blocks to their final location first
2. Then commits the metadata (inode size, extent tree) to the journal
3. If power fails after step 1 but before step 2, the data is on disk but unreferenced — no stale data exposure

---

## 5. How ext4 Plugs Into the VFS

The VFS is a layer of abstraction that makes all filesystems look the same to user space. Every filesystem must implement a set of **operation tables** — function pointer structs that the VFS calls when user space does file operations. See `vfs.md` for the full VFS architecture.

### 5.1 The Operation Tables

ext4 provides four main operation tables:

```
User space:   open()   read()   write()   mkdir()   stat()
                │        │        │          │         │
VFS:         ───┼────────┼────────┼──────────┼─────────┤
                │        │        │          │         │
ext4:        ┌──┴────────┴────────┴──┐  ┌────┴─────────┴──┐
             │ ext4_file_operations  │  │ ext4_dir_inode_  │
             │ (file.c:963)         │  │ operations       │
             │ .open  → ext4_file_  │  │ (namei.c:4212)   │
             │          open        │  │ .lookup → ext4_  │
             │ .read_iter → ext4_  │  │           lookup  │
             │             file_    │  │ .create → ext4_  │
             │             read_iter│  │           create  │
             │ .write_iter → ...    │  │ .mkdir → ext4_   │
             └──────────────────────┘  │          mkdir   │
                                       └──────────────────┘
             ┌──────────────────────┐  ┌──────────────────┐
             │ ext4_aops            │  │ ext4_sops         │
             │ (inode.c:3976)       │  │ (super.c:1626)    │
             │ .read_folio          │  │ .alloc_inode      │
             │ .writepages          │  │ .write_inode      │
             │ .write_begin         │  │ .evict_inode      │
             │ .write_end           │  │ .sync_fs          │
             └──────────────────────┘  └──────────────────┘
```

### 5.2 struct super_operations — Filesystem Lifecycle

`ext4_sops` (`fs/ext4/super.c:1626–1646`):

| Callback | ext4 Function | When VFS calls it |
|----------|--------------|-------------------|
| `.alloc_inode` | `ext4_alloc_inode` | VFS needs a new in-memory inode (allocates `ext4_inode_info`) |
| `.free_inode` | `ext4_free_in_core_inode` | In-memory inode is being freed |
| `.write_inode` | `ext4_write_inode` (line 5687) | VFS wants the inode flushed to disk |
| `.dirty_inode` | `ext4_dirty_inode` | Inode metadata changed (timestamps, etc.) |
| `.evict_inode` | `ext4_evict_inode` (line 165) | Inode is being deleted or dropped from cache |
| `.put_super` | `ext4_put_super` | Filesystem is being unmounted |
| `.sync_fs` | `ext4_sync_fs` | `sync()` or `syncfs()` called |
| `.statfs` | `ext4_statfs` | `statfs()` / `df` — report free space |

### 5.3 struct file_operations — File I/O

`ext4_file_operations` (`fs/ext4/file.c:963–983`):

| Callback | ext4 Function | When VFS calls it |
|----------|--------------|-------------------|
| `.read_iter` | `ext4_file_read_iter` | `read()` or `pread()` on a file |
| `.write_iter` | `ext4_file_write_iter` | `write()` or `pwrite()` on a file |
| `.open` | `ext4_file_open` | `open()` on a file |
| `.release` | `ext4_release_file` | Last `close()` on a file |
| `.fsync` | `ext4_sync_file` | `fsync()` or `fdatasync()` |
| `.mmap_prepare` | `ext4_file_mmap_prepare` | `mmap()` on a file |
| `.llseek` | `ext4_llseek` | `lseek()` with `SEEK_DATA`/`SEEK_HOLE` support |
| `.fallocate` | `ext4_fallocate` | `fallocate()` — preallocate space |
| `.splice_read` | `ext4_file_splice_read` | `splice()` for zero-copy I/O |

### 5.4 struct address_space_operations — Page Cache Interface

This is the most important table for understanding how data actually moves. `ext4_aops` (`fs/ext4/inode.c:3976–3990`):

| Callback | ext4 Function | When VFS/page cache calls it |
|----------|--------------|------------------------------|
| `.read_folio` | `ext4_read_folio` (line 3383) | Page cache miss — need to read one page from disk |
| `.readahead` | `ext4_readahead` (line 3399) | Readahead — prefetch multiple pages |
| `.writepages` | `ext4_writepages` (line 3009) | Writeback — flush dirty pages to disk |
| `.write_begin` | `ext4_write_begin` (line 1280) | Prepare a page for writing (allocate blocks, lock page) |
| `.write_end` | `ext4_write_end` (line 1434) | Complete writing to a page (mark dirty, update size) |
| `.dirty_folio` | `ext4_dirty_folio` | Mark a page/folio as dirty |
| `.bmap` | `ext4_bmap` | Legacy logical-to-physical block mapping |

---

## 6. The Read Path — Reading a File Through ext4

When you call `read(fd, buf, 4096)`, here is what ext4 does (see `io_path.md` for the full end-to-end view):

### 6.1 VFS Calls ext4

```
read(fd, buf, 4096)
  → vfs_read()                             [fs/read_write.c]
    → file->f_op->read_iter()
      → ext4_file_read_iter()              [fs/ext4/file.c]
        → generic_file_read_iter()         [mm/filemap.c]
          → filemap_read()
```

For buffered I/O, ext4 delegates to the generic page cache code in `filemap_read()`.

### 6.2 Page Cache Lookup

`filemap_read()` checks if the requested data is already in the page cache:

- **Cache hit**: The page is in memory → copy data to user buffer, done
- **Cache miss**: The page is not in memory → must read from disk

### 6.3 Cache Miss — ext4_read_folio()

On a cache miss, the page cache calls `ext4_read_folio()` (`inode.c:3383`):

1. Allocates buffer heads for the page
2. Calls `ext4_map_blocks()` to translate the file offset (logical block) to a disk block (physical block)
3. `ext4_map_blocks()` walks the extent tree → finds the physical block number
4. Creates a `bio` (block I/O request) pointing to the physical block
5. Submits the bio via `submit_bio()` → enters the block layer

### 6.4 Readahead

The kernel does not read just one page — it predicts you will read sequentially and triggers **readahead** via `ext4_readahead()` (`inode.c:3399`). This reads multiple consecutive pages in a single large I/O, amortizing the disk seek cost.

---

## 7. The Write Path — Writing a File Through ext4

### 7.1 Buffered Write — The write_begin/write_end Pattern

```
write(fd, buf, 4096)
  → vfs_write()                            [fs/read_write.c]
    → file->f_op->write_iter()
      → ext4_file_write_iter()             [fs/ext4/file.c]
        → generic_perform_write()          [mm/filemap.c]
```

`generic_perform_write()` calls ext4 in a two-phase pattern for each page:

**Phase 1 — write_begin** (`ext4_write_begin()`, line 1280):
1. Start a jbd2 transaction (`ext4_journal_start()`)
2. Find or allocate the page in the page cache
3. If the page is not up-to-date and we are writing less than a full page, read the existing content from disk first
4. Lock the page and set up buffer heads

**Phase 2 — Data copy** (done by generic VFS code):
- `copy_from_user(page_address(page) + offset, user_buf, len)`
- The user's data is now in the page cache

**Phase 3 — write_end** (`ext4_write_end()`, line 1434):
1. Mark the buffer heads (and the page) as dirty
2. Update the inode's size if the write extended the file
3. Stop the jbd2 transaction (`ext4_journal_stop()`)
4. Unlock the page

At this point, `write()` returns to user space. **The data is NOT on disk yet** — it is in the page cache, marked dirty.

### 7.2 Delayed Allocation

In the default ordered journaling mode, ext4 uses **delayed allocation** — blocks are not allocated when you call `write()`. Instead:

1. `write()` → data goes into the page cache, marked dirty
2. The extent status tree records the logical blocks as "delayed" (not yet mapped to physical blocks)
3. Later, when the writeback thread flushes dirty pages, `ext4_writepages()` allocates the physical blocks
4. This allows ext4 to see the full extent of the write and allocate contiguous blocks — much better than allocating one block at a time

This is why ext4 can achieve near-sequential I/O even for random `write()` patterns — it defers allocation until it knows the full picture.

### 7.3 Writeback — Flushing Dirty Pages to Disk

Dirty pages are flushed to disk by the kernel's writeback threads (see `io_path.md` Section 7 for the full writeback pipeline). When the writeback code calls `ext4_writepages()` (`inode.c:3009`):

1. For each dirty page range, allocate physical blocks via `ext4_map_blocks()`
2. Create bio requests mapping the page cache pages to their physical disk blocks
3. Submit the bios via `submit_bio()` → block layer → device driver → disk
4. Start and commit jbd2 transactions for the metadata changes (extent tree updates, inode size)

### 7.4 fsync — Forcing Data to Disk

When you call `fsync(fd)`:

```
fsync(fd)
  → vfs_fsync()                            [fs/sync.c]
    → file->f_op->fsync()
      → ext4_sync_file()                   [fs/ext4/fsync.c]
```

`ext4_sync_file()`:
1. Flushes all dirty pages for this file to disk
2. Commits the current jbd2 transaction (ensures all metadata is in the journal)
3. Issues a disk cache flush (ensures the data is on persistent storage, not just in the disk's volatile write cache)

After `fsync()` returns, the data is guaranteed to survive a power failure.

---

## 8. Block Allocation — The Multi-Block Allocator (mballoc)

### 8.1 The Goal

When ext4 needs to allocate disk blocks for a file, it wants to:
- Allocate **contiguous** blocks (better sequential read performance)
- Keep blocks **close to the inode** (same block group if possible)
- Avoid **fragmentation** (don't scatter small allocations everywhere)

### 8.2 The Allocation Strategy

The multi-block allocator (`fs/ext4/mballoc.c`) uses a multi-stage strategy with increasing desperation:

| Stage | Name | Strategy |
|-------|------|----------|
| 0 | `CR_POWER2_ALIGNED` | Find a power-of-2 aligned free extent (best for large files) |
| 1 | `CR_GOAL_LEN_FAST` | In-memory search for the goal length |
| 2 | `CR_BEST_AVAIL_LEN` | Reduce the goal and search again |
| 3 | `CR_GOAL_LEN_SLOW` | Read block bitmaps from disk |
| 4 | `CR_ANY_FREE` | First-fit — take any free block |

### 8.3 Preallocation

ext4 preallocates extra blocks beyond what was requested, betting that the file will grow. Two types:

- **Inode preallocation**: Extra blocks reserved for a specific file. If the file does grow, the preallocated blocks are already contiguous.
- **Group preallocation**: Blocks reserved for a locality group (per-CPU). When multiple small files are created, they get blocks from the same area, reducing fragmentation.

Preallocation state is tracked in `struct ext4_prealloc_space` (`mballoc.h:118`):

```c
struct ext4_prealloc_space {
    unsigned long pa_pstart;     // physical block start
    unsigned long pa_lstart;     // logical block start
    unsigned short pa_len;       // total length
    unsigned short pa_free;      // free blocks remaining
    unsigned short pa_type;      // MB_INODE_PA or MB_GROUP_PA
    ...
};
```

### 8.4 The Buddy Allocator

Each block group has a buddy bitmap (`struct ext4_group_info`, `ext4.h:3502`), similar to the buddy system in the memory allocator (see `mm.md`). It tracks free extents by order (power of 2), enabling fast lookup of contiguous regions.

---

## 9. The In-Memory Representation

### 9.1 ext4_sb_info — The In-Memory Superblock

When ext4 is mounted, the kernel creates `struct ext4_sb_info` (`ext4.h:1538–1687`), a much larger structure than the on-disk superblock:

| Field | Line | Purpose |
|-------|------|---------|
| `s_es` | 1554 | Pointer to the on-disk superblock (in a buffer) |
| `s_sbh` | 1553 | Buffer head containing the superblock |
| `s_group_desc` | 1556 | Array of buffer heads for group descriptors |
| `s_journal` | 1590 | Pointer to the jbd2 journal |
| `s_group_info` | 1619 | Buddy allocator state per block group |
| `s_mount_opt` | 1557 | Mount options bitmask |
| `s_desc_size` | 1539 | Group descriptor size |
| `s_inodes_per_block` | 1540 | Inodes per block |

### 9.2 ext4_inode_info — The In-Memory Inode

`struct ext4_inode_info` (`ext4.h:1040–1212`) wraps the VFS `struct inode` with ext4-specific data:

```c
struct ext4_inode_info {
    __le32 i_data[15];               // 1041 — copy of on-disk i_block[]
    __u32  i_block_group;            // 1052 — which block group this inode is in
    loff_t i_disksize;               // 1118 — on-disk file size
    struct rw_semaphore i_data_sem;  // 1130 — protects extent tree
    struct inode vfs_inode;          // 1131 — the embedded VFS inode
    struct jbd2_inode *jinode;       // 1132 — journal inode

    /* Extent status tree — cached mappings */
    struct rb_root i_es_tree;        // 1150 — extent status tree root
    rwlock_t i_es_lock;              // 1151 — extent status tree lock

    /* Delayed allocation */
    unsigned int i_reserved_data_blocks; // 1145 — reserved blocks
    struct rb_node i_prealloc_node;  // 1146 — preallocation tree
    ...
};
```

The VFS inode (`vfs_inode`) is embedded — not a pointer. The VFS allocates `ext4_inode_info` via `alloc_inode()`, and `container_of()` recovers the `ext4_inode_info` from the embedded VFS inode:

```c
static inline struct ext4_inode_info *EXT4_I(struct inode *inode)
{
    return container_of(inode, struct ext4_inode_info, vfs_inode);
}
```

---

## 10. Mounting — From mount() to ext4_fill_super()

### 10.1 The Mount Path

```
mount -t ext4 /dev/sda1 /mnt
  → mount(2) syscall
    → path_mount()                         [fs/namespace.c]
      → do_new_mount()
        → vfs_get_tree()
          → ext4_get_tree()                [fs/ext4/super.c:5807]
            → get_tree_bdev()              [fs/super.c]
              → ext4_fill_super()          [super.c:5757]
```

### 10.2 ext4_fill_super() — Bringing the Filesystem to Life

`ext4_fill_super()` (line 5757) calls `__ext4_fill_super()` (line 5292), which:

1. **Read the superblock** from byte 1024 of the block device via `ext4_load_super()` (line 5312)
2. **Validate the magic number** (must be `0xEF53`)
3. **Check feature flags** — if the on-disk features include an `incompat` flag the kernel does not understand, refuse to mount
4. **Allocate `ext4_sb_info`** and initialize in-memory state
5. **Read group descriptors** — load all block group descriptors into memory
6. **Initialize the journal** — load the jbd2 journal, replay if needed (crash recovery)
7. **Get the root inode** — `ext4_iget(sb, EXT4_ROOT_INO)` reads inode 2 (the root directory) from disk
8. **Create the root dentry** — `d_make_root(root_inode)` creates the dentry for `/mnt`
9. **Set operation tables** — `sb->s_op = &ext4_sops` (line 5419)

After this, the VFS can navigate the filesystem: looking up paths, opening files, reading data — all through the operation tables ext4 registered.

### 10.3 The Filesystem Type Registration

At module load time (or boot for built-in), ext4 registers itself:

```c
// fs/ext4/super.c:7450
static struct file_system_type ext4_fs_type = {
    .owner          = THIS_MODULE,
    .name           = "ext4",
    .init_fs_context = ext4_init_fs_context,
    .parameters     = ext4_param_specs,
    .kill_sb        = ext4_kill_sb,
    .fs_flags       = FS_REQUIRES_DEV | FS_ALLOW_IDMAP | FS_MGTIME | FS_LBS,
};

// Registration (during module init):
register_filesystem(&ext4_fs_type);
```

This tells the VFS: "I exist, my name is 'ext4', and here is how to create my filesystem context when someone mounts me."

---

## 11. Extended Attributes and ACLs

### 11.1 Extended Attributes (xattrs)

ext4 supports extended attributes — arbitrary key-value pairs attached to inodes. Used for SELinux labels, POSIX ACLs, capabilities, and user-defined metadata.

Storage locations:
- **Inline**: If the xattr fits in the inode's extra space (after the standard inode fields)
- **External block**: A separate block pointed to by `i_file_acl` in the on-disk inode

Common namespaces:
- `user.*` — user-defined attributes
- `security.*` — SELinux labels, capabilities
- `system.posix_acl_access` — POSIX ACLs
- `trusted.*` — only root can access

```bash
# Set an attribute:
setfattr -n user.comment -v "important file" /mnt/data.txt

# Get an attribute:
getfattr -n user.comment /mnt/data.txt

# List all attributes:
getfattr -d /mnt/data.txt
```

---

## 12. Filesystem Features and Compatibility

### 12.1 Feature Flags

ext4 uses three categories of feature flags in the superblock:

| Category | Field | Meaning |
|----------|-------|---------|
| **Compatible** | `s_feature_compat` | Kernel can mount even if it does not understand these |
| **Read-only compatible** | `s_feature_ro_compat` | Kernel can mount read-only if it does not understand |
| **Incompatible** | `s_feature_incompat` | Kernel must refuse to mount if it does not understand |

### 12.2 Key ext4 Features

| Feature | Flag | Purpose |
|---------|------|---------|
| Extents | `INCOMPAT_EXTENTS` | Extent tree instead of indirect blocks |
| 64-bit | `INCOMPAT_64BIT` | Filesystem > 16 TB |
| Flex block groups | `INCOMPAT_FLEX_BG` | Group bitmaps/tables can be in other groups |
| Dir htree | `COMPAT_DIR_INDEX` | Hash tree directory indexing |
| Huge file | `RO_COMPAT_HUGE_FILE` | Files > 2 TB |
| Metadata checksums | `RO_COMPAT_METADATA_CSUM` | crc32c checksums on metadata |
| Inline data | `INCOMPAT_INLINE_DATA` | Small files stored inside the inode |

You can see the features of a filesystem with:

```bash
tune2fs -l /dev/sda1 | grep features
# Output: has_journal extent huge_file flex_bg metadata_csum ...
```

---

## 13. Function Quick Reference

| Function | File:Line | Role |
|----------|-----------|------|
| `ext4_fill_super()` | `fs/ext4/super.c:5757` | Mount: initialize filesystem |
| `__ext4_fill_super()` | `super.c:5292` | Core mount logic |
| `ext4_file_read_iter()` | `fs/ext4/file.c` | Read from file |
| `ext4_file_write_iter()` | `file.c` | Write to file |
| `ext4_read_folio()` | `fs/ext4/inode.c:3383` | Read one page from disk |
| `ext4_readahead()` | `inode.c:3399` | Readahead multiple pages |
| `ext4_write_begin()` | `inode.c:1280` | Prepare page for writing |
| `ext4_write_end()` | `inode.c:1434` | Complete write to page |
| `ext4_writepages()` | `inode.c:3009` | Flush dirty pages to disk |
| `ext4_map_blocks()` | `inode.c:693` | Map logical block → physical block |
| `ext4_ext_map_blocks()` | `fs/ext4/extents.c:4191` | Extent tree block mapping |
| `ext4_lookup()` | `fs/ext4/namei.c:1760` | Directory lookup |
| `ext4_create()` | `namei.c:2806` | Create a new file |
| `ext4_mkdir()` | `namei.c:2988` | Create a directory |
| `ext4_unlink()` | `namei.c:3292` | Remove a file |
| `ext4_write_inode()` | `super.c:5687` | Write inode to disk |
| `ext4_evict_inode()` | `super.c:165` | Delete inode from disk |
| `ext4_sync_file()` | `fs/ext4/fsync.c` | fsync implementation |
| `ext4_alloc_inode()` | `super.c` | Allocate in-memory inode |

---

## 14. Debugging and Inspecting ext4

### 14.1 Filesystem Information

```bash
# Show superblock and features:
tune2fs -l /dev/sda1

# Show filesystem stats:
dumpe2fs /dev/sda1 | head -50

# Check filesystem consistency:
fsck.ext4 -n /dev/sda1    # -n = no changes, just report
```

### 14.2 Inode and Block Inspection

```bash
# Show inode details for a file:
stat /mnt/myfile.txt
debugfs -R "stat <inode_number>" /dev/sda1

# Show extent tree:
debugfs -R "extents <inode_number>" /dev/sda1

# Show directory contents with inode numbers:
ls -li /mnt/
```

### 14.3 Tracing ext4 Operations

```bash
# Trace ext4 read/write events:
sudo trace-cmd record -e ext4:ext4_da_write_begin \
                      -e ext4:ext4_da_write_end \
                      -e ext4:ext4_writepages \
                      sleep 5
trace-cmd report

# Trace block allocation:
sudo trace-cmd record -e ext4:ext4_allocate_blocks \
                      -e ext4:ext4_mballoc_alloc \
                      sleep 5
trace-cmd report

# Trace journal commits:
sudo trace-cmd record -e jbd2:jbd2_start_commit \
                      -e jbd2:jbd2_end_commit \
                      sleep 10
trace-cmd report
```

### 14.4 Common Issues

| Symptom | Likely Cause | Investigation |
|---------|-------------|---------------|
| "Structure needs cleaning" | Metadata corruption | `fsck.ext4 /dev/sda1` |
| Very slow writes | Journal full, waiting for commit | Check `jbd2` tracepoints |
| Running out of space with free blocks | Reserved blocks (5% default) | `tune2fs -m 1 /dev/sda1` |
| Cannot create files with free inodes | Inode limit per group exhausted | `df -i` to check inode usage |
| Fragmentation | Many small writes over time | `e4defrag /mnt/` |

---

## 15. Inline Data — Small Files Inside the Inode

For very small files, ext4 can store the data directly inside the inode itself, avoiding the overhead of allocating a separate data block.

### 15.1 How It Works

An ext4 inode is 256 bytes on disk (default). The standard inode fields consume about 160 bytes. The remaining ~100 bytes are available for extended attributes (`i_extra_isize` controls the split). With `INCOMPAT_INLINE_DATA`, ext4 can store file data in this space instead of using extent tree → data block.

```
Standard inode (file with data block):
┌──────────────────────────┐    ┌──────────────┐
│  Inode fields (160 B)    │    │              │
│  Extent tree root        │───→│ Data Block   │
│  (points to data block)  │    │ (4096 bytes) │
│  Extra space / xattrs    │    └──────────────┘
└──────────────────────────┘

Inline data inode (small file):
┌──────────────────────────┐
│  Inode fields (160 B)    │
│  inline data (≤~60 B)    │   ← file content is RIGHT HERE
│  (in i_block[] area +    │      no separate data block needed
│   xattr space)           │
└──────────────────────────┘
```

The data is stored in two places within the inode:
1. **`i_block[]`** area (60 bytes): normally holds extent tree root, repurposed for inline data
2. **Extended attribute area**: additional bytes in the `system.data` xattr

### 15.2 When Inline Data Is Used

- File must be very small (typically ≤ 160 bytes depending on inode size and xattr usage)
- Feature must be enabled: `tune2fs -O inline_data /dev/sda1`
- When a file grows beyond the inline limit, ext4 converts it to a regular extent-based file (`ext4_convert_inline_data()`)

### 15.3 Implementation

```c
// fs/ext4/inline.c

int ext4_readpage_inline(struct inode *inode, struct folio *folio)
{
    // Read data directly from inode → copy into the folio
    // No bio needed, no block layer involvement
}

int ext4_write_inline_data(struct inode *inode, struct ext4_iloc *iloc,
                           void *buffer, loff_t pos, unsigned int len)
{
    // Write data directly into the inode's on-disk representation
    // Uses the journal for crash consistency
}

int ext4_try_to_write_inline_data(struct address_space *mapping,
                                  struct inode *inode, loff_t pos,
                                  unsigned len, struct folio **foliop)
{
    // Attempt to write inline; if data too large, fall back to
    // ext4_convert_inline_data() → normal extent-based write
}
```

**Performance**: Inline data is significantly faster for tiny files — reading avoids the block layer entirely (no bio, no disk seek). This matters for directories with many small configuration files.

---

## 16. Preallocation — fallocate()

### 16.1 The Problem

If you write a file sequentially, ext4's delayed allocation does a good job of allocating contiguous blocks. But if you know the final file size in advance, you can do even better by preallocating:

```c
int fd = open("/mnt/data.bin", O_CREAT | O_WRONLY, 0644);
fallocate(fd, 0, 0, 1024 * 1024 * 1024);  // preallocate 1 GB
// Now write — all blocks already allocated, guaranteed contiguous
```

### 16.2 How ext4 Implements fallocate

```
fallocate(fd, mode, offset, len)
  → vfs_fallocate(file, mode, offset, len)     [fs/open.c]
    → file->f_op->fallocate()
      → ext4_fallocate()                       [fs/ext4/extents.c:4556]
```

`ext4_fallocate()` works by:

1. Calculates the range of logical blocks to preallocate
2. For each chunk of blocks:
   - Starts a jbd2 transaction
   - Calls `ext4_map_blocks()` with `EXT4_GET_BLOCKS_CREATE_UNWRIT_EXT`
   - The multi-block allocator (mballoc) allocates contiguous physical blocks
   - Creates **unwritten extents** — blocks are allocated on disk but marked as "unwritten" in the extent tree
3. Updates the inode's block count (but NOT the file size)

### 16.3 Unwritten Extents

Preallocated blocks are marked with the `EXTENT_UNWRITTEN` flag in the extent tree:

```c
// fs/ext4/ext4_extents.h
#define EXT4_EXT_UNWRITTEN  0x8000  /* bit 15 in ee_len */
```

An unwritten extent means:
- The physical blocks are allocated — no other file can use them
- Reading returns zeros (not garbage from previous file data)
- When the application writes data, the extent is converted to "written"

This is a security feature: without it, `fallocate()` would expose stale data from previously deleted files.

### 16.4 Punch Hole and Collapse Range

`fallocate()` also supports the inverse — punching holes and collapsing ranges:

```c
// Punch a hole (deallocate blocks in the middle of a file):
fallocate(fd, FALLOC_FL_PUNCH_HOLE | FALLOC_FL_KEEP_SIZE, offset, len);
// → ext4_punch_hole() → removes extents → blocks returned to free space

// Collapse a range (remove a section and shift data down):
fallocate(fd, FALLOC_FL_COLLAPSE_RANGE, offset, len);
// → ext4_collapse_range() → removes extents + shifts subsequent extents

// Insert a range (shift data up to create a gap):
fallocate(fd, FALLOC_FL_INSERT_RANGE, offset, len);
// → ext4_insert_range() → shifts extents up + creates unwritten gap
```

---

## 17. The Dentry Cache and Path Lookup

While ext4 stores directories on disk, the VFS dentry cache (`dcache`) keeps recently accessed directory entries in memory. This is critical for path lookup performance.

### 17.1 Path Lookup → Dentry Cache → ext4

```
open("/mnt/data/file.txt", O_RDONLY)
  → path_openat()                              [fs/namei.c]
    │
    ├─ For each component ("/mnt", "data", "file.txt"):
    │   │
    │   ├─ __d_lookup(parent, &name)           [fs/dcache.c]
    │   │   → Hash table lookup in dcache
    │   │   → If FOUND: return cached dentry (no disk I/O!)
    │   │
    │   └─ If NOT in dcache:
    │       → inode->i_op->lookup(parent_inode, dentry, flags)
    │         → ext4_lookup()                  [fs/ext4/namei.c]
    │           → ext4_find_entry(dir, &name, &de)
    │             → Reads directory blocks from disk
    │             → Searches linear list or htree
    │           → ext4_iget(sb, ino)
    │             → Reads inode from disk
    │           → d_splice_alias(inode, dentry) → inserts into dcache
    │
    └─ Result: struct file * pointing to the resolved inode
```

### 17.2 Negative Dentries

The dcache also caches lookup *failures*:

```
open("/mnt/nonexistent", O_RDONLY)
  → path lookup
    → dcache miss for "nonexistent"
    → ext4_lookup() → searches directory → NOT FOUND
    → d_add(dentry, NULL)   ← negative dentry: "this name does not exist"
  → Returns -ENOENT

open("/mnt/nonexistent", O_RDONLY)   [second attempt]
  → path lookup
    → dcache hit for "nonexistent" → negative dentry
    → Returns -ENOENT immediately (no disk I/O)
```

Negative dentries are important: without them, repeated lookups of nonexistent files would hit the disk every time. This matters for applications that probe for optional config files, or for repeated `stat()` calls on build systems.

### 17.3 Dentry Invalidation

Dentries must be invalidated when the directory changes:

- `unlink()` → `d_delete()` converts dentry to negative
- `rename()` → moves dentries in the cache
- `mkdir()` / `rmdir()` → adds/removes dentries
- NFS: remote changes require explicit `d_invalidate()` because another client may have modified the directory

### 17.4 Inode Cache

Closely related to the dcache is the inode cache (`inode_hashtable`). When ext4 reads an inode from disk via `ext4_iget()`, the in-memory `struct inode` (with its `ext4_inode_info` wrapper) is cached in the inode hash table. Subsequent lookups of the same inode number return the cached object without disk I/O.

```c
// fs/inode.c
struct inode *iget_locked(struct super_block *sb, unsigned long ino)
{
    struct inode *inode = find_inode_fast(sb, ino);  // hash lookup
    if (inode) return inode;                          // cached!
    // ... allocate new inode, mark I_NEW, caller fills from disk ...
}
```

---

## 18. Error Handling and Filesystem Consistency

### 18.1 What Happens When ext4 Detects Corruption

ext4 can detect metadata corruption via checksums (when `metadata_csum` is enabled):

```c
// fs/ext4/ext4.h
void ext4_error(struct super_block *sb, const char *fmt, ...);
void ext4_error_inode(struct inode *inode, const char *func,
                      unsigned int line, ext4_fsblk_t block,
                      const char *fmt, ...);
```

When `ext4_error()` is called, the behavior depends on the mount option `errors=`:

| Mount Option | Behavior |
|-------------|----------|
| `errors=continue` | Log the error, continue running (data may be inconsistent) |
| `errors=remount-ro` | Log the error, remount filesystem read-only (default) |
| `errors=panic` | Kernel panic — used for critical data servers where silent corruption is worse than downtime |

### 18.2 The ext4 Error Path

```
ext4 detects bad checksum on a group descriptor:
  → ext4_error(sb, "group descriptor checksum invalid")
    → Logs to kernel ring buffer (dmesg)
    → Sets EXT4_FLAGS_SHUTDOWN on the superblock info
    → If errors=remount-ro:
      → sb->s_flags |= SB_RDONLY
      → All subsequent write operations fail with -EROFS
    → Records the error in the superblock on disk:
      → ext4_commit_super(sb)
      → Sets s_state to EXT4_ERROR_FS
      → Records error timestamp, error function, error line
```

### 18.3 Journal Recovery

On mount, if the superblock shows the filesystem was not cleanly unmounted:

```
ext4_fill_super()
  → ext4_load_journal()
    → jbd2_journal_load()
      → jbd2_journal_recover()
        → Scans the journal log from the last checkpoint
        → For each committed transaction:
          → Replays the metadata blocks from journal to their real locations
        → After replay: filesystem metadata is consistent
        → Clears the journal
```

This happens transparently during `mount()`. The user may see messages like:
```
EXT4-fs (sda1): recovery complete
EXT4-fs (sda1): mounted filesystem with ordered data mode
```

### 18.4 ENOMEM and ENOSPC Handling

Two common error paths in ext4:

**Out of space (ENOSPC)**:
```
write() → ext4_write_begin() → ext4_journal_start()
  → If delayed allocation writeback needs blocks:
    → ext4_map_blocks() → ext4_mb_new_blocks()
      → mballoc cannot find free blocks
      → Returns -ENOSPC
  → ext4 attempts to reserve blocks early:
    → ext4_claim_free_clusters(sbi, 1)
    → If insufficient free clusters → ENOSPC propagated to write()
```

**Out of memory (ENOMEM)**:
```
read() → filemap_read_folio() → folio allocation fails
  → Returns -ENOMEM
  → VFS propagates to read() → user sees -1, errno = ENOMEM
  → The folio allocator may trigger reclaim:
    → Evict clean page cache pages → retry the allocation
    → If still failing → OOM killer may be invoked
```

---

## 19. ext4 in Comparison — How Other Filesystems Differ

Since the VFS provides a uniform interface, all filesystems implement the same operation tables. But their internal strategies differ significantly:

### 19.1 Comparison Table

| Aspect | ext4 | XFS | Btrfs | F2FS |
|--------|------|-----|-------|------|
| **On-disk format** | Block groups, inode table, extent tree | Allocation groups, B+tree for everything | Copy-on-write B-tree, chunk-based | Log-structured, zones + segments |
| **Max file size** | 16 TB (4 KB blocks) | 8 EB | 16 EB | 3.94 TB |
| **Max volume** | 1 EB | 8 EB | 16 EB | 16 TB |
| **Journaling** | jbd2 (metadata + optional data) | Metadata journal (log) | No journal — CoW is inherently consistent | Multi-head log, checkpoint |
| **Allocation** | mballoc (buddy allocator) | B+tree free space, delayed alloc | Chunk allocator + B-tree free space | Section-based, LFS allocation |
| **Checksums** | Optional (`metadata_csum`) | Metadata CRC32c | All metadata + optional data CRC32c | Metadata + optional data |
| **Snapshots** | No | No (but reflink copies) | Yes (CoW subvolumes) | No |
| **RAID** | External (md/dm) | External | Built-in (RAID 0/1/5/6/10) | External |
| **Best for** | General purpose, stable | Large files, high throughput | Snapshots, checksums, flexibility | Flash/SSD, Android |

### 19.2 How Copy-on-Write (Btrfs) Changes the Write Path

In ext4, a write modifies data in place (the page cache page maps to a fixed physical block):

```
ext4 write to existing block:
  1. Write new data to page cache at the SAME physical block location
  2. Writeback: overwrite the old data on disk
  3. Journal protects metadata (extent tree, inode)
```

In Btrfs, writes NEVER overwrite existing data:

```
Btrfs write to existing block:
  1. Write new data to page cache
  2. Writeback: allocate a NEW physical block, write data there
  3. Update parent node in B-tree to point to new block
  4. Update parent's parent... up to the root
  5. Atomically swap the root pointer
  → The old data is still on disk (can be used for snapshots)
```

This is why Btrfs can do snapshots (just keep the old root pointer) but has higher write amplification (each write cascades up the tree).

### 19.3 How XFS Differs from ext4

Both are traditional "update in place" filesystems, but XFS uses different data structures:

- **B+trees everywhere**: Free space, inodes, directory entries — all stored in B+trees (vs ext4's bitmaps + linear/htree)
- **Allocation groups**: Similar to ext4's block groups but with independent B+tree allocation indexes
- **Delayed allocation**: Like ext4, but XFS had it first
- **Extent-based**: Like ext4's extent tree, but XFS uses B+trees for extents too (handles extreme fragmentation better)
- **No inode table**: XFS allocates inodes dynamically from B+tree; ext4 pre-allocates the inode table at mkfs time

### 19.4 What They All Share (Via VFS)

Despite these differences, all filesystems present the same interface to user space:

```c
// Every filesystem implements these:
struct file_operations {      // read, write, mmap, ioctl
struct inode_operations {     // lookup, create, unlink, rename
struct super_operations {     // alloc_inode, write_inode, sync_fs
struct address_space_operations { // read_folio, writepages, write_begin/end
```

An application calling `read()`, `write()`, `open()`, `stat()`, or `mmap()` works identically regardless of whether ext4, XFS, Btrfs, or F2FS is underneath. The VFS is the abstraction that makes this possible.

---

*This document covers the ext4 filesystem in Linux 6.19. For other filesystems (btrfs, XFS, F2FS), the VFS interface is the same — they implement the same operation tables — but their on-disk formats, allocation strategies, and consistency mechanisms differ significantly. The `Documentation/filesystems/ext4/` directory in the kernel source has additional ext4-specific documentation.*
