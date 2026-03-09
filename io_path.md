# The I/O Path: From read()/write() to Spinning Platters and Back

> Source base: `/home/inineapa/Lab/linux-6.19`

---

## Before You Begin

You know `read()` and `write()`. You know data ends up on disk. But between your syscall and the magnetic platter (or flash cell), the data passes through five distinct layers, each with its own abstraction of what "data" and "location" mean. A file offset becomes a logical block, then a physical block, then a sector, then a DMA address on a hardware ring buffer. Each layer speaks a different language.

This document traces both directions — **write (RAM → storage)** and **read (storage → RAM)** — through every layer, showing exactly how the abstraction transforms at each boundary. For each layer, we show what the data looks like, what the target looks like, and what the layer is essentially *saying* to the next layer down.

For the details of each individual layer, see: `vfs.md`, `fs.md`, `block.md`, `drivers.md`.

---

## 1. The Layers at a Glance

```
┌──────────────────────────────────────────────────────────────┐
│                        USER SPACE                            │
│  write(fd, "Hello", 5)                                       │
│  Data: a char buffer        Target: a file descriptor + offset│
└──────────────┬───────────────────────────────────────────────┘
               │  syscall boundary
┌──────────────▼───────────────────────────────────────────────┐
│  1. VFS LAYER              (fs/read_write.c, mm/filemap.c)   │
│  Data: user buffer → page   Target: struct file + loff_t pos │
│  "Copy this user buffer into the page cache at this position"│
├──────────────┬───────────────────────────────────────────────┤
│  2. FILESYSTEM LAYER       (fs/ext4/inode.c, extents.c)      │
│  Data: dirty page/folio     Target: inode + logical block #   │
│  "Map logical block N of this inode to a physical disk block" │
├──────────────┬───────────────────────────────────────────────┤
│  3. BLOCK LAYER            (block/blk-mq.c, bio.c)           │
│  Data: bio (page + offset)  Target: block_device + sector #   │
│  "Write these pages to sectors S through S+N on this device"  │
├──────────────┬───────────────────────────────────────────────┤
│  4. DEVICE DRIVER          (drivers/block/*, drivers/nvme/*)  │
│  Data: DMA scatter-gather   Target: HW queue + tag            │
│  "DMA this scatter list to LBA range X, slot it in queue Y"  │
├──────────────┬───────────────────────────────────────────────┤
│  5. HARDWARE               (NVMe SSD, SATA HDD, virtio-blk)  │
│  Data: bytes on the wire    Target: physical LBA              │
│  "Write these bytes to this address on the storage medium"    │
└──────────────┴───────────────────────────────────────────────┘
```

---

## 2. How the Abstraction Changes at Each Layer

### What the "data" looks like:

| Layer | Data representation | Example |
|-------|-------------------|---------|
| User space | `char *buf` — a flat byte array | `"Hello World\n"` |
| VFS / Page cache | `struct folio` — a page-aligned memory page (4 KB) | Folio at index 0 of the file's address_space |
| Filesystem | Dirty buffer heads on a folio, with logical→physical block mapping | Logical block 42 → physical block 98304 |
| Block layer | `struct bio` — a scatter-gather list of (page, offset, len) segments | bio: sector 786432, 8 sectors, 2 page segments |
| Device driver | DMA descriptors / command structures (NVMe SQE, SCSI CDB) | NVMe Write command: LBA=786432, count=8, PRP list |
| Hardware | Electrical signals / NAND flash program operations | Bytes written to flash page at physical address |

### What the "target" looks like:

| Layer | Target representation | Example |
|-------|----------------------|---------|
| User space | File descriptor + implicit offset | fd=3, current position=0 |
| VFS | `struct file` + `loff_t pos` (byte offset in file) | File "/mnt/data.txt", offset 0 |
| Filesystem | `struct inode` + logical block number | Inode 12, logical block 0 |
| Block layer | `struct block_device` + sector number | /dev/sda1, sector 786432 |
| Device driver | Hardware queue + request tag + LBA | HW queue 0, tag 17, LBA 786432 |
| Hardware | Physical location on medium | NAND die 2, plane 0, block 1234, page 56 |

---

## 3. The Write Path — RAM to Storage

### 3.1 Layer 1: User Space → VFS

> *"Here is my buffer. Put it at this position in this file."*

```
write(fd, buf, 4096)
  → ksys_write(fd, buf, 4096)              [fs/read_write.c:727]
    → Resolve fd to struct file *
    → Extract current position (f_pos)
    → vfs_write(file, buf, 4096, &pos)     [read_write.c:666]
      → file_start_write(file)             [freeze protection]
      → new_sync_write(file, buf, 4096, &pos) [read_write.c:583]
        → Wrap buf in struct iov_iter
        → Construct struct kiocb from file + pos
        → file->f_op->write_iter(kiocb, iter) [→ ext4_file_write_iter]
```

**What changes**: The file descriptor (an integer) becomes a `struct file` pointer. The raw byte buffer becomes an `iov_iter` (an iterator over potentially scattered buffers). The implicit offset becomes an explicit `loff_t`.

### 3.2 Layer 2: VFS → Page Cache (Buffered Write)

> *"Copy this data into the page cache. Mark the page dirty. I'll deal with actually writing it to disk later."*

```
ext4_file_write_iter()
  → generic_perform_write(kiocb, iter)     [mm/filemap.c:4286]
    │
    │  For each page-sized chunk of data:
    │
    ├─ a_ops->write_begin(kiocb, mapping, pos, len, &folio, &fsdata)
    │    → ext4_write_begin()               [fs/ext4/inode.c:1280]
    │    → Finds or allocates a folio in the page cache
    │    → Starts a jbd2 journal transaction
    │    → Returns a locked folio
    │
    ├─ copy_folio_from_iter_atomic(folio, offset, len, iter)
    │    → Copies user data into the page cache folio     [line 4332]
    │
    ├─ a_ops->write_end(kiocb, mapping, pos, len, copied, folio, fsdata)
    │    → ext4_write_end()                 [inode.c:1434]
    │    → Marks buffer heads dirty
    │    → Updates inode size if needed
    │    → Stops the jbd2 transaction
    │    → Unlocks the folio
    │
    └─ balance_dirty_pages_ratelimited(mapping)           [line 4307]
         → Throttles the writer if too many dirty pages
```

**What changes**: The user's byte buffer is now a dirty folio in the page cache, indexed by the file's `address_space` and the page index (file offset / PAGE_SIZE). The target is no longer "file + offset" — it is "address_space + page index."

**The return to user space happens here.** `write()` returns successfully. The data is in RAM, not on disk. This is why `write()` is fast — you are writing to memory, not to a disk.

### 3.3 The Gap: Dirty Pages Waiting

The dirty pages sit in the page cache. Three things can trigger the next phase:

1. **Timer**: The writeback thread wakes up every 5 seconds (configurable via `dirty_writeback_interval`)
2. **Memory pressure**: Too many dirty pages (exceeds `dirty_background_ratio`) → background writeback starts
3. **Explicit sync**: `fsync(fd)`, `sync()`, or `msync()` — the application demands the data be on disk

### 3.4 Layer 3: Page Cache → Filesystem (Writeback)

> *"These pages are dirty. Figure out which disk blocks they belong to, and send the I/O."*

```
wb_workfn()                                [fs/fs-writeback.c:2386]
  → wb_do_writeback()
    → writeback_inodes_wb()                [fs-writeback.c:2122]
      → __writeback_inodes_wb()
        → writeback_sb_inodes()
          → __writeback_single_inode()
            → do_writepages(mapping, wbc)  [mm/page-writeback.c:2587]
              → mapping->a_ops->writepages(mapping, wbc)
                → ext4_writepages()        [fs/ext4/inode.c:3009]
```

Inside `ext4_writepages()`:

```
For each dirty page range:
  1. ext4_map_blocks(handle, inode, &map, flags)   [inode.c:693]
     → ext4_ext_map_blocks()                       [extents.c:4191]
     → Walk the extent tree: logical block → physical block
     → If delayed allocation: allocate physical blocks NOW
     → Returns: "logical block 42 → physical block 98304, length 8"

  2. Build a bio:
     → bio_alloc(bdev, nr_pages, REQ_OP_WRITE, GFP_NOIO)  [block/bio.c:512]
     → bio->bi_iter.bi_sector = physical_block * (block_size / 512)
     → bio_add_folio(bio, folio, len, offset)              [bio.c:1076]

  3. Submit:
     → submit_bio(bio)                                     [block/blk-core.c:911]
```

**What changes**: The target transforms from "inode + logical block" to "block_device + sector number." The filesystem has done the critical translation — consulting the extent tree to find where on the physical disk this file's data lives. The data is still the same page in memory, but now it is described as a bio vector (page + offset + length) pointing at a sector range.

### 3.5 Layer 4: Block Layer → Device Driver

> *"Write these pages to sectors 786432–786439 on /dev/sda. I've merged adjacent I/Os and scheduled them for fairness."*

```
submit_bio(bio)                            [block/blk-core.c:911]
  → submit_bio_noacct(bio)                 [blk-core.c:782]
    → __submit_bio(bio)                    [blk-core.c:626]
      → blk_mq_submit_bio(bio)            [block/blk-mq.c:3131]
        │
        ├─ Try to merge with existing request:
        │    blk_mq_attempt_bio_merge()    [blk-mq.c:3024]
        │    → Check plug list, then scheduler queue
        │    → If adjacent sectors: merge into existing request
        │
        ├─ If no merge, create new request:
        │    blk_mq_bio_to_request(rq, bio) [blk-mq.c:2675]
        │    → rq->__sector = bio->bi_iter.bi_sector
        │    → rq->__data_len = bio->bi_iter.bi_size
        │    → rq->bio = bio
        │
        └─ Dispatch to driver:
             blk_mq_try_issue_directly(hctx, rq) [blk-mq.c:2758]
               → q->mq_ops->queue_rq(hctx, &bd)
                 → The driver's queue_rq callback
```

**What changes**: The bio becomes a `struct request` — which may contain multiple merged bios for adjacent sectors. The target changes from "block_device + sector" to "hardware queue + request tag." The block layer has:
- Merged adjacent I/Os (turning 8 small writes into 1 large write)
- Applied scheduling policy (mq-deadline, bfq, or none)
- Selected a hardware queue (for multi-queue devices like NVMe)

### 3.6 Layer 5: Device Driver → Hardware

> *"Program NVMe submission queue entry: opcode=Write, LBA=786432, count=8, PRP=<DMA addresses of the pages>."*

The driver's `queue_rq()` callback:

1. Maps the request's pages for DMA (`dma_map_sg()`)
2. Builds a hardware-specific command (NVMe SQE, SCSI CDB, virtio descriptor)
3. Places the command on the hardware submission queue
4. Rings the hardware doorbell (writes to an MMIO register)
5. Returns `BLK_STS_OK`

The hardware reads the command from the submission queue via DMA, reads the data pages from memory via DMA, and writes the data to the storage medium.

### 3.7 Completion — Hardware → User Space

The completion path runs in reverse:

```
1. Hardware finishes writing, posts a completion entry, raises an interrupt

2. Driver interrupt handler:
   → Reads completion queue
   → Finds the request tag → looks up struct request
   → blk_mq_complete_request(rq)               [block/blk-mq.c]
     → blk_mq_end_request(rq, BLK_STS_OK)

3. Block layer:
   → bio_endio(bio)                             [block/bio.c:1632]
     → bio->bi_end_io(bio)                      [filesystem's callback]

4. Filesystem callback:
   → end_buffer_async_write(bh)                 [fs/buffer.c:387]
     → Clears buffer_dirty
     → If all buffers in folio are done:
       → folio_end_writeback(folio)             [mm/filemap.c:1676]
         → Clears PG_writeback flag
         → Wakes any waiters (fsync blocked here)

5. If fsync() was waiting:
   → fsync() sees writeback complete → returns to user space
```

---

## 4. The Read Path — Storage to RAM

### 4.1 Layer 1: User Space → VFS

> *"Give me 4096 bytes from this file at the current position."*

```
read(fd, buf, 4096)
  → ksys_read(fd, buf, 4096)               [fs/read_write.c:704]
    → Resolve fd to struct file *
    → vfs_read(file, buf, 4096, &pos)       [read_write.c:552]
      → new_sync_read(file, buf, 4096, &pos) [read_write.c:481]
        → file->f_op->read_iter(kiocb, iter)
          → ext4_file_read_iter()
            → generic_file_read_iter()      [mm/filemap.c:2951]
```

### 4.2 Layer 2: VFS → Page Cache (Cache Hit — Fast Path)

> *"Check if this page is already in memory. If so, just copy it out."*

```
generic_file_read_iter()
  → filemap_read(kiocb, iter, 0)            [mm/filemap.c:2763]
    │
    ├─ filemap_get_pages(kiocb, count, &fbatch) [filemap.c:2662]
    │   → filemap_get_read_batch()
    │     → Look up page cache: address_space + page_index
    │     → FOUND — folio is present and uptodate
    │
    └─ copy_folio_to_iter(folio, offset, len, iter)
       → copy_to_user(buf, page_data + offset, len)
       → Return 4096 (bytes read)
```

**Cache hit latency**: ~1–2 microseconds. No disk I/O at all.

### 4.3 Layer 2b: Page Cache Miss → Readahead → Filesystem

> *"The page is not in memory. Read it from disk — and while you're at it, prefetch the next few pages too."*

```
filemap_get_pages()
  → filemap_get_read_batch()  → MISS (folio not found)
  │
  ├─ page_cache_sync_ra(ractl, req_count)   [mm/readahead.c:554]
  │   → do_page_cache_ra()                  [readahead.c:314]
  │     → page_cache_ra_unbounded()         [readahead.c:210]
  │       → Allocate folios for the readahead window (e.g., 32 pages)
  │       → read_pages()
  │         → mapping->a_ops->readahead(ractl)
  │           → ext4_readahead()            [fs/ext4/inode.c:3399]
  │             → mpage_readahead()         [fs/mpage.c:359]
  │
  └─ OR: filemap_create_folio()             [filemap.c:2595]
       → Allocate one folio
       → filemap_read_folio()               [filemap.c:2486]
         → mapping->a_ops->read_folio()
           → ext4_read_folio()              [inode.c:3383]
```

### 4.4 Layer 3: Filesystem → Block Layer (Read I/O)

> *"I need logical blocks 0–7 of inode 12. They live at physical blocks 98304–98311. Read them."*

Inside `mpage_readahead()` or `ext4_read_folio()`:

```
1. For each folio in the readahead batch:
   → ext4_map_blocks(NULL, inode, &map, 0)      [inode.c:693]
     → Walk extent tree → logical block 0 maps to physical block 98304
     → Returns physical block + length

2. Build bio:
   → bio_alloc(bdev, nr_pages, REQ_OP_READ, GFP_KERNEL)
   → bio->bi_iter.bi_sector = 98304 * 8    (for 4K blocks, 8 sectors per block)
   → bio_add_folio(bio, folio, 4096, 0)    (for each folio)
   → bio->bi_end_io = mpage_end_io         (completion callback)

3. Submit:
   → submit_bio(bio)                        [block/blk-core.c:911]
```

### 4.5 Layers 4–5: Block Layer → Driver → Hardware (Same as Write)

The block layer processes the read bio the same way as a write bio — merge, schedule, dispatch via `queue_rq()`. The hardware reads data from the storage medium and DMAs it into the page cache pages.

### 4.6 Completion — Data Arrives

```
1. Hardware completes read, posts completion, raises interrupt

2. Driver → blk_mq_complete_request(rq)

3. bio_endio(bio)                           [block/bio.c:1632]
   → mpage_end_io() or end_buffer_async_read()
     → Checks I/O status
     → If successful: mark folio as uptodate
     → folio_end_read(folio, true)          [mm/filemap.c:1524]
       → Sets PG_uptodate
       → Clears PG_locked
       → Wakes waiters

4. filemap_read() was waiting for the folio lock:
   → Folio is now uptodate
   → copy_folio_to_iter() → copy_to_user()
   → read() returns 4096 to user space
```

---

## 5. What Each Layer Says to the Next

Here is the entire write path as a conversation between the layers:

```
User space → VFS:
  "Write these 4096 bytes from my buffer to file descriptor 3 at offset 0."

VFS → Page Cache:
  "Find or create the page at index 0 of inode 12's address_space.
   Copy the user's bytes into it. Mark it dirty."

Page Cache → Filesystem (later, during writeback):
  "Page 0 of inode 12 is dirty. It needs to go to disk."

Filesystem → Block Layer:
  "I looked up inode 12's extent tree. Logical block 0 lives at
   physical block 98304 on /dev/sda. Here's a bio: read/write
   these pages at sector 786432."

Block Layer → Device Driver:
  "Here's a request: WRITE, starting sector 786432, 8 sectors,
   scatter-gather list of pages. It's tagged as request #17
   on hardware queue 0."

Device Driver → Hardware:
  "NVMe Write command: namespace 1, LBA 786432, block count 8.
   Data is at these physical DMA addresses. Doorbell ring."

Hardware → Storage Medium:
  "Program flash cells at physical page address 0x1234_5678
   with these 4096 bytes."
```

And the read path:

```
User space → VFS:
  "Give me 4096 bytes from file descriptor 3 at offset 0."

VFS → Page Cache:
  "Check if page index 0 of inode 12 is cached."
  → "It's not. We need to fetch it."

Page Cache → Filesystem:
  "Please fill this empty page — it's for logical offset 0 of inode 12.
   Also, prefetch the next 31 pages while you're at it."

Filesystem → Block Layer:
  "Inode 12, logical block 0 = physical block 98304. Here's a bio:
   READ sectors 786432–786495 from /dev/sda into these 32 pages."

Block Layer → Device Driver:
  "Request #42 on hardware queue 0: READ, sector 786432, 256 sectors.
   DMA the data into these page addresses."

Device Driver → Hardware:
  "NVMe Read command: namespace 1, LBA 786432, block count 32.
   Put the data at these DMA addresses. Doorbell ring."

Hardware → Driver (completion):
  "Done. Data is in memory. Here's the completion entry."

Driver → Block Layer:
  "Request #42 completed successfully."

Block Layer → Filesystem:
  "Bio complete. Status: success."

Filesystem → Page Cache:
  "Pages are filled and uptodate."

Page Cache → VFS:
  "Here's the folio. It's valid."

VFS → User Space:
  "Copied 4096 bytes to your buffer. read() returns 4096."
```

---

## 6. The Numbers at Each Layer

To make the layer transitions concrete, let us trace a specific example:

**Scenario**: Writing 4096 bytes to offset 0 of a file on an ext4 filesystem with 4 KB blocks, on a partition starting at sector 2048 of an NVMe SSD.

| Layer | Data | Target | Size Unit |
|-------|------|--------|-----------|
| User space | `buf` at `0x7fff1234` | fd=3, offset=0 | bytes |
| VFS | Folio in page cache | address_space page_index=0 | bytes |
| Filesystem | Folio + buffer head | inode 12, logical block 0 → physical block 98304 | blocks (4096 B) |
| Block layer | bio, 1 segment | /dev/nvme0n1p1, sector 786432 (= 98304 × 8) | sectors (512 B) |
| Block (absolute) | request | /dev/nvme0n1, sector 788480 (= 786432 + 2048 partition offset) | sectors |
| NVMe driver | SQE (submission queue entry) | Namespace 1, LBA 788480, count 8 | LBAs (512 B or 4096 B) |
| Hardware | NAND program | Die/plane/block/page | cells |

The sector number changes at the partition boundary — the block layer adds the partition's `start_sect` to translate from partition-relative to device-absolute addressing. This is invisible to the filesystem.

---

## 7. The Writeback Timeline

Most `write()` calls never touch the disk synchronously. Here is the timeline:

```
Time 0ms:      write(fd, buf, 4096)
               → Data copied into page cache
               → Page marked dirty
               → write() returns to user space
               ✓ User's perspective: "done"

Time 0-5000ms: Dirty page sits in page cache
               Process continues executing
               More writes may accumulate

Time ~5000ms:  Writeback timer fires (dirty_writeback_interval)
               wb_workfn() wakes up
               Sees dirty pages above background threshold

Time ~5001ms:  ext4_writepages() called
               Walks dirty pages, maps blocks, builds bios
               submit_bio() sends I/O to block layer

Time ~5002ms:  Block layer merges and schedules
               queue_rq() sends to NVMe driver
               NVMe command posted to submission queue

Time ~5003ms:  NVMe SSD completes write
               Interrupt → completion → folio_end_writeback()
               Page is now clean

Time ~5003ms:  ✓ Data is on persistent storage
```

With `fsync()`:

```
Time 0ms:      write(fd, buf, 4096) → returns immediately
Time 1ms:      fsync(fd)
               → Triggers immediate writeback of all dirty pages for this file
               → Waits for all I/O completion
               → Issues disk cache flush
               → Returns when data is guaranteed persistent
Time ~4ms:     fsync() returns
               ✓ Data guaranteed on persistent storage
```

---

## 8. Direct I/O — Bypassing the Page Cache

When you open a file with `O_DIRECT`, the page cache is bypassed entirely:

```
User space → VFS:
  "Write these 4096 bytes directly to disk. Don't cache them."

VFS → Filesystem:
  generic_file_write_iter() detects IOCB_DIRECT
  → ext4_dio_write_iter()
    → iomap_dio_rw()                [fs/iomap/direct-io.c:841]

Filesystem → Block Layer:
  → iomap iterates extents
  → Builds bios pointing to the USER'S pages (not page cache pages)
  → submit_bio()

Block Layer → Driver → Hardware:
  (same as buffered path, but the pages are the user's buffer,
   not copies in the page cache)

Completion:
  → bio_endio() → iomap_dio_bio_end_io()
  → When all bios complete, iomap_dio_rw() returns
  → write() returns to user space only AFTER data is on disk
```

**Key difference**: No copy. The user's buffer is DMA'd directly to/from the disk. This is why databases (PostgreSQL, MySQL with InnoDB) use `O_DIRECT` — they manage their own caching and do not want the kernel duplicating data in the page cache.

**Requirements**: The user buffer must be aligned to the filesystem block size (typically 4 KB), and the I/O size must be a multiple of the block size.

---

## 9. Memory-Mapped I/O — Page Faults as Reads

When you `mmap()` a file and then read from the mapped memory, the read happens via a page fault:

```
User space:
  ptr = mmap(NULL, 4096, PROT_READ, MAP_SHARED, fd, 0);
  char c = ptr[0];    ← page fault! page is not yet in memory

CPU:
  → Page fault exception (vector 14)
  → do_page_fault() → handle_mm_fault()

VFS:
  → filemap_fault()                         [mm/filemap.c:3507]
    → Look up page cache: MISS
    → do_sync_mmap_readahead()              → readahead
    → filemap_read_folio()
      → ext4_read_folio()                   → submit_bio() → disk read
    → Wait for I/O completion
    → Install PTE mapping page cache folio
    → Return to user — CPU retries the faulting instruction

User space:
  char c = ptr[0];    ← succeeds, data is in page cache
```

For mmap'd writes, the data goes to the page cache (same folio), the folio is marked dirty, and the writeback path is identical to buffered `write()`. The difference is just how the data gets into the page cache — `copy_from_user()` vs. direct CPU store via a PTE.

---

## 10. The fsync() / sync() Path

### 10.1 fsync(fd) — Sync One File

```
fsync(fd)
  → do_fsync(fd, 0)                        [fs/sync.c:206]
    → vfs_fsync(file, 0)                   [sync.c:200]
      → vfs_fsync_range(file, 0, LLONG_MAX, 0) [sync.c:180]
        → file->f_op->fsync(file, 0, LLONG_MAX, 0)
          → ext4_sync_file()               [fs/ext4/fsync.c]
            │
            ├─ filemap_write_and_wait_range()
            │   → Write all dirty pages for this file
            │   → Wait for all writeback to complete
            │
            ├─ jbd2_complete_transaction()
            │   → Commit the current journal transaction
            │   → Wait for journal I/O to complete
            │
            └─ blkdev_issue_flush()
                → Send a cache flush command to the disk
                → Wait for the disk to confirm data is persistent
```

### 10.2 sync() — Sync Everything

```
sync()
  → ksys_sync()                            [fs/sync.c]
    → For each mounted filesystem:
      → sync_filesystem(sb)
        → Write all dirty inodes
        → Write all dirty pages
        → Commit journal
        → Flush disk cache
```

---

## 11. I/O Scheduling — What Happens in the Block Layer

Between `submit_bio()` and `queue_rq()`, the block layer can reorder, merge, and throttle I/O:

### 11.1 Bio Merging

Adjacent bios are merged into a single request:

```
Bio A: sector 1000, 8 sectors (4 KB)
Bio B: sector 1008, 8 sectors (4 KB)
                    ↓ merge
Request: sector 1000, 16 sectors (8 KB), 2 bio segments
```

This reduces the number of commands sent to the hardware.

### 11.2 I/O Schedulers

| Scheduler | Strategy | Best for |
|-----------|----------|----------|
| **none** | FIFO, no reordering | NVMe SSDs (already parallel internally) |
| **mq-deadline** | Deadline-based: reads expire in 500ms, writes in 5s | HDDs, general purpose |
| **bfq** | Weighted fair queuing with I/O priority classes | Desktop, latency-sensitive |
| **kyber** | Latency-feedback controller | Low-latency SSDs |

```bash
# Check current scheduler:
cat /sys/block/sda/queue/scheduler
# [mq-deadline] kyber bfq none

# Change scheduler:
echo bfq > /sys/block/sda/queue/scheduler
```

### 11.3 Plugging

The block layer uses **plugging** to batch I/O submissions:

```c
blk_start_plug(&plug);       // start batching
// ... submit multiple bios ...
blk_finish_plug(&plug);      // flush the batch to the scheduler
```

Without plugging, each `submit_bio()` might immediately dispatch to the driver. With plugging, bios accumulate and are dispatched together — enabling better merging and scheduling.

---

## 12. Summary: The Full Round Trip

### Write (4096 bytes to a file):

| Step | Function | What happens |
|------|----------|-------------|
| 1 | `write()` | Syscall, fd → struct file |
| 2 | `generic_perform_write()` | User buffer → page cache folio (dirty) |
| 3 | `write()` returns | **User resumes. Data is NOT on disk.** |
| 4 | `wb_workfn()` | Writeback thread wakes (timer or pressure) |
| 5 | `ext4_writepages()` | Walks extent tree: logical → physical block |
| 6 | `submit_bio()` | Bio created: page + sector + device |
| 7 | `blk_mq_submit_bio()` | Merge, schedule, dispatch |
| 8 | `queue_rq()` | Driver programs hardware (DMA + command) |
| 9 | Hardware interrupt | I/O complete |
| 10 | `folio_end_writeback()` | Page marked clean, waiters woken |

### Read (4096 bytes from a file, cache miss):

| Step | Function | What happens |
|------|----------|-------------|
| 1 | `read()` | Syscall, fd → struct file |
| 2 | `filemap_read()` | Check page cache → MISS |
| 3 | `page_cache_sync_ra()` | Trigger readahead (prefetch next N pages) |
| 4 | `ext4_readahead()` | Walk extent tree: logical → physical block |
| 5 | `submit_bio()` | Bio created: page + sector + device (READ) |
| 6 | `blk_mq_submit_bio()` | Merge, schedule, dispatch |
| 7 | `queue_rq()` | Driver programs hardware (DMA + command) |
| 8 | Hardware interrupt | Data DMA'd into page cache pages |
| 9 | `folio_end_read()` | Page marked uptodate, waiters woken |
| 10 | `copy_to_user()` | Data copied to user buffer |
| 11 | `read()` returns | **User gets data.** |

---

## 13. Function Quick Reference

| Function | File:Line | Layer | Role |
|----------|-----------|-------|------|
| `ksys_read()` | `fs/read_write.c:704` | VFS | Resolve fd, call vfs_read |
| `ksys_write()` | `read_write.c:727` | VFS | Resolve fd, call vfs_write |
| `vfs_read()` | `read_write.c:552` | VFS | Dispatch to file_operations |
| `vfs_write()` | `read_write.c:666` | VFS | Dispatch to file_operations |
| `generic_file_read_iter()` | `mm/filemap.c:2951` | Page cache | Buffered/direct read router |
| `filemap_read()` | `filemap.c:2763` | Page cache | Buffered read loop |
| `filemap_get_pages()` | `filemap.c:2662` | Page cache | Find or create folios |
| `filemap_read_folio()` | `filemap.c:2486` | Page cache | Trigger single-page read |
| `page_cache_sync_ra()` | `mm/readahead.c:554` | Readahead | Synchronous readahead |
| `generic_perform_write()` | `mm/filemap.c:4286` | Page cache | Buffered write loop |
| `do_writepages()` | `mm/page-writeback.c:2587` | Writeback | Call filesystem writepages |
| `wb_workfn()` | `fs/fs-writeback.c:2386` | Writeback | Writeback worker thread |
| `filemap_fault()` | `mm/filemap.c:3507` | mmap | Page fault handler for files |
| `bio_alloc()` | `block/bio.c:512` | Block | Allocate a bio |
| `bio_add_folio()` | `bio.c:1076` | Block | Add page to bio |
| `submit_bio()` | `block/blk-core.c:911` | Block | Submit bio for I/O |
| `blk_mq_submit_bio()` | `block/blk-mq.c:3131` | Block | Multi-queue submission |
| `blk_mq_dispatch_rq_list()` | `blk-mq.c:2106` | Block | Dispatch to driver |
| `bio_endio()` | `block/bio.c:1632` | Block | I/O completion |
| `folio_end_read()` | `mm/filemap.c:1524` | Page cache | Read completion |
| `folio_end_writeback()` | `filemap.c:1676` | Page cache | Write completion |
| `vfs_fsync_range()` | `fs/sync.c:180` | VFS | fsync implementation |
| `iomap_dio_rw()` | `fs/iomap/direct-io.c:841` | Filesystem | Direct I/O |

---

## 14. Error Propagation — When I/O Fails

Errors travel back up the same path, carried in status fields at each layer boundary.

### 14.1 Hardware → Driver

The hardware reports errors through its completion queue. For NVMe, the completion entry contains a status field (e.g., `NVME_SC_DATA_XFER_ERROR`, `NVME_SC_INTERNAL`). The driver translates this into a `blk_status_t`:

```c
// drivers/nvme/host/core.c
blk_status_t nvme_error_status(u16 status)
{
    switch (status & NVME_SCT_SC_MASK) {
    case NVME_SC_SUCCESS:        return BLK_STS_OK;
    case NVME_SC_CAP_EXCEEDED:   return BLK_STS_NOSPC;
    case NVME_SC_ONCS_NOT_SUPPORTED: return BLK_STS_NOTSUPP;
    case NVME_SC_WRITE_FAULT:
    case NVME_SC_READ_ERROR:
    case NVME_SC_UNWRITTEN_BLOCK: return BLK_STS_MEDIUM;
    ...
    }
}
```

### 14.2 Driver → Block Layer

The driver completes the request with a status:

```
blk_mq_end_request(rq, status)     [block/blk-mq.c]
  → blk_update_request(rq, status, bytes)
    → For each bio in the request:
      → bio->bi_status = status      ← error stored in bio
      → bio_endio(bio)
```

### 14.3 Block Layer → Filesystem

`bio_endio()` calls the bio's completion callback with the error in `bio->bi_status`:

```c
// block/bio.c:1632
void bio_endio(struct bio *bio)
{
    // ... chain handling ...
    if (bio->bi_end_io)
        bio->bi_end_io(bio);    // filesystem's callback
}
```

The filesystem callback checks `bio->bi_status`:
- For reads: `end_bio_extent_readpage()` or `mpage_end_io()` — sets page error flags
- For writes: `end_buffer_async_write()` — maps errors to `AS_EIO` in the address_space

### 14.4 Filesystem → Page Cache → VFS → User Space

**Read errors**: If a folio fails to read, `filemap_read()` returns `-EIO`:

```
filemap_read()
  → filemap_get_pages()
    → filemap_read_folio()          → submits bio
    → folio_wait_locked()           → waits for completion
    → folio_test_uptodate(folio)    → FALSE on error
  → Returns -EIO
→ read() returns -1, errno = EIO
```

**Write errors**: Since `write()` returns before data reaches disk, errors appear later:
- If the writeback thread finds an error, it sets `AS_EIO` or `AS_ENOSPC` on the address_space
- `fsync()` checks and reports these deferred errors:

```
fsync(fd)
  → ext4_sync_file()
    → file_check_and_advance_wb_err()       [fs/sync.c]
      → errseq_check_and_advance(&mapping->wb_err, &file->f_wb_err)
      → If a writeback error occurred since last fsync → returns -EIO
  → fsync() returns -1, errno = EIO
```

This is a subtle and important point: **`write()` can succeed but the data can fail to reach disk.** The only way to detect writeback errors is to call `fsync()` and check its return value. This is why database engines always check `fsync()` returns.

### 14.5 Error Sequence Numbers (errseq_t)

The kernel uses `errseq_t` (`include/linux/errseq.h`) to track writeback errors without losing them across multiple file descriptors:

```
mapping->wb_err:  an errseq_t (counter + error code)

When a writeback error occurs:
  → errseq_set(&mapping->wb_err, -EIO)
  → Increments counter + stores error

When fsync() checks:
  → errseq_check_and_advance(&mapping->wb_err, &file->f_wb_err)
  → Compares file's last-seen counter with mapping's current counter
  → If mapping has a newer error → report it, advance file's counter
```

This ensures that every file descriptor sees each error at least once, even if multiple processes have the same file open.

---

## 15. io_uring — Asynchronous I/O Without Context Switches

`io_uring` provides a different entry point into the I/O path, but converges with the same lower layers:

### 15.1 Submission Path

```
User space:
  → Writes SQE (submission queue entry) into shared ring buffer
  → io_uring_enter() syscall (or polled mode — no syscall needed)

Kernel (io_uring/rw.c):
  → io_read() or io_write()
    → Sets up struct kiocb from the SQE
    → kiocb->ki_flags |= IOCB_NOWAIT (for non-blocking)
    → file->f_op->read_iter(kiocb, iter) or write_iter()
      → Same ext4/VFS path as regular read()/write()
```

From the filesystem's perspective, an io_uring read/write looks identical to a regular one — it goes through the same `file_operations` callbacks. The difference is in how the kernel manages the request lifecycle.

### 15.2 Completion Path

```
When I/O completes (bio_endio → ... → kiocb_done):
  → io_req_complete_post()
    → Writes a CQE (completion queue entry) into the shared ring buffer
    → If polled: user space picks it up without a syscall
    → If IRQ-driven: signals via eventfd or io_uring_enter()
```

### 15.3 Where io_uring Diverges

| Aspect | read()/write() | io_uring |
|--------|---------------|----------|
| Submission | 1 syscall per I/O | Batched: many SQEs per syscall (or zero syscalls) |
| Completion | Synchronous: `read()` blocks until done | Asynchronous: CQE appears in ring, no blocking |
| Context switches | 2 per I/O (in + out of kernel) | Amortized: 0–1 per batch |
| Buffered write | Same path, both mark folios dirty | Same path, both mark folios dirty |
| Direct I/O | `iomap_dio_rw()` blocks or uses AIO | `iomap_dio_rw()` with async completion via CQE |
| Data path (filesystem→block→driver) | Identical | Identical |

The key insight: io_uring accelerates the **entry and exit** of the I/O path (fewer syscalls, no blocking), but the core path through VFS → filesystem → block → driver is the same.

### 15.4 io_uring for Polling (IORING_SETUP_IOPOLL)

For the lowest latency on NVMe:

```
User space:
  → Submits SQE to shared ring (no syscall)

Kernel:
  → Instead of waiting for a hardware interrupt:
    → io_uring polls the NVMe completion queue in a tight loop
    → blk_mq_poll(q, cookie)           [block/blk-mq.c]
      → q->mq_ops->poll(hctx)          [driver's poll callback]
      → Checks hardware CQ for completions
  → When complete, writes CQE to ring

User space:
  → Polls the CQ ring (no syscall)
  → Sees completion immediately
```

This eliminates both syscalls AND interrupt overhead. Sub-microsecond latency is achievable.

---

## 16. Tracing the I/O Path — Debugging Tools

### 16.1 ftrace / trace-cmd

The entire I/O path is instrumented with tracepoints:

```bash
# Trace the full write path (VFS → filesystem → block → completion):
sudo trace-cmd record \
    -e syscalls:sys_enter_write \
    -e ext4:ext4_write_begin \
    -e ext4:ext4_write_end \
    -e ext4:ext4_writepages \
    -e ext4:ext4_da_write_begin \
    -e ext4:ext4_es_insert_delayed_extent \
    -e block:block_bio_queue \
    -e block:block_rq_issue \
    -e block:block_rq_complete \
    -p $(pidof myapp) \
    sleep 5

trace-cmd report
```

Output shows the exact sequence:
```
myapp-1234  write(fd=3, buf=0x..., count=4096)     ← user syscall
myapp-1234  ext4_write_begin: dev=sda1 ino=12      ← filesystem
myapp-1234  ext4_write_end: dev=sda1 ino=12        ← filesystem done
kworker-56  ext4_writepages: dev=sda1 ino=12       ← writeback (later)
kworker-56  block_bio_queue: 8,1 W sector=786432   ← enters block layer
kworker-56  block_rq_issue: 8,1 W sector=786432    ← dispatched to driver
<irq>       block_rq_complete: 8,1 W sector=786432 ← hardware done
```

### 16.2 BPF / bpftrace

```bash
# Latency histogram for block I/O:
sudo bpftrace -e '
tracepoint:block:block_rq_issue { @start[args->sector] = nsecs; }
tracepoint:block:block_rq_complete /@start[args->sector]/ {
    @usecs = hist((nsecs - @start[args->sector]) / 1000);
    delete(@start[args->sector]);
}'

# Trace read() → disk read latency per file:
sudo bpftrace -e '
tracepoint:syscalls:sys_enter_read /comm == "myapp"/ {
    @read_start[tid] = nsecs;
}
tracepoint:syscalls:sys_exit_read /@read_start[tid]/ {
    @read_latency_us = hist((nsecs - @read_start[tid]) / 1000);
    delete(@read_start[tid]);
}'
```

### 16.3 blktrace / blkparse

```bash
# Trace block layer events on /dev/sda:
sudo blktrace -d /dev/sda -o - | blkparse -i -

# Output columns:
#  8,0  1  42  0.000000000 1234  Q  W 786432 + 8 [myapp]    ← Queued
#  8,0  1  43  0.000001234 1234  G  W 786432 + 8 [myapp]    ← Got request
#  8,0  1  44  0.000002345 1234  I  W 786432 + 8 [myapp]    ← Inserted to scheduler
#  8,0  1  45  0.000003456 1234  D  W 786432 + 8 [myapp]    ← Dispatched to driver
#  8,0  0  12  0.000123456    0  C  W 786432 + 8  [0]       ← Completed
```

Event codes: **Q**ueue → **G**et-request → **I**nsert → **D**ispatch → **C**omplete.

### 16.4 /proc and /sys Introspection

```bash
# See pending I/O per process:
cat /proc/<pid>/io
# rchar, wchar: bytes read/written (includes page cache)
# read_bytes, write_bytes: actual block device I/O

# Block device stats (sectors read/written, time in queue):
cat /sys/block/sda/stat

# Per-queue depth and dispatched counts:
ls /sys/block/nvme0n1/mq/*/

# Current I/O scheduler:
cat /sys/block/sda/queue/scheduler

# Queue depth (max outstanding requests):
cat /sys/block/nvme0n1/queue/nr_requests
```

### 16.5 Mapping the Path in Practice

When debugging a slow `read()` or `write()`:

```
Step 1: Is it the syscall itself or the writeback?
  → strace -T -e read,write ./myapp
  → If write() is fast (~microseconds): the bottleneck is in writeback, not the syscall

Step 2: Is data reaching the block layer?
  → trace-cmd: look for block:block_bio_queue events
  → If missing: data is only in the page cache, not yet submitted

Step 3: Is the block layer dispatching quickly?
  → blktrace: measure Q→D time (queue to dispatch)
  → If slow: I/O scheduler contention or plug delay

Step 4: Is the hardware fast?
  → blktrace: measure D→C time (dispatch to complete)
  → If slow: storage device is the bottleneck

Step 5: Is completion propagating back?
  → ftrace: check for delays between block_rq_complete and folio_end_writeback
```

---

## 17. The Page Cache as the Central Crossroads

The page cache is where all buffered I/O paths converge, and understanding its role ties the entire I/O path together:

### 17.1 Every Path Passes Through

```
                          Page Cache
                     (struct address_space)
                    ┌─────────────────────┐
  read(fd, buf, N) → │                     │ ← writeback thread
                    │   Indexed by:       │
  write(fd, buf, N)→ │   (inode, page_idx) │ → ext4_writepages()
                    │                     │
  mmap + load     → │   Each folio:       │ ← readahead engine
                    │   - PG_uptodate     │
  io_uring read   → │   - PG_dirty        │ → submit_bio()
                    │   - PG_writeback    │
  splice/sendfile → │   - PG_locked       │ ← bio_endio()
                    └─────────────────────┘
```

### 17.2 Folio State Machine

A folio in the page cache transitions through states that reflect the I/O path:

```
                           allocate folio
                               │
                               ▼
                    ┌──────────────────────┐
                    │      NEW             │
                    │  (not uptodate,      │
                    │   not dirty)         │
                    └──────────┬───────────┘
                               │ read from disk
                               │ (filemap_read_folio)
                               ▼
                    ┌──────────────────────┐
         read() ←──│    UPTODATE          │──→ copy_to_user()
                    │  (clean, in cache)   │
                    └──────────┬───────────┘
                               │ write() / store via mmap
                               │ (mark_page_dirty)
                               ▼
                    ┌──────────────────────┐
                    │     DIRTY            │
                    │  (modified in RAM,   │
                    │   not yet on disk)   │
                    └──────────┬───────────┘
                               │ writeback starts
                               │ (ext4_writepages → submit_bio)
                               ▼
                    ┌──────────────────────┐
                    │   WRITEBACK          │
                    │  (being DMA'd to     │──→ fsync() blocks here
                    │   disk right now)    │
                    └──────────┬───────────┘
                               │ bio_endio → folio_end_writeback
                               │
                               ▼
                    ┌──────────────────────┐
                    │    UPTODATE (clean)  │──→ can be reclaimed
                    │  (data is on disk    │    by memory pressure
                    │   AND in cache)      │
                    └──────────────────────┘
```

### 17.3 Memory Pressure and Reclaim

When the system runs low on memory, the page reclaim path (`mm/vmscan.c`) evicts clean folios from the page cache. This directly affects I/O:

- **Clean folios**: Free to reclaim immediately (data is on disk)
- **Dirty folios**: Must be written back first → triggers I/O → then reclaimed
- **Writeback folios**: Already being written → wait for completion → then reclaim
- **Mapped folios** (via mmap): Must unmap PTEs first → TLB shootdown → then reclaim

This is why under memory pressure, I/O performance degrades — the kernel must flush dirty pages to free memory, creating I/O traffic that competes with the application's own I/O.

### 17.4 The readahead Window

The readahead engine (`mm/readahead.c`) is a page cache optimization that pre-fills folios before `read()` asks for them:

```
First read at offset 0:
  → Synchronous readahead: read pages 0–3 (16 KB)
  → The app gets page 0, pages 1–3 are in the cache for later

Second read at offset 4096:
  → Cache hit! Page 1 is already there (no disk I/O)

When the app reaches page 2:
  → Asynchronous readahead triggered
  → Background: read pages 4–11 (32 KB, window grows)
  → The app never blocks — data arrives before it is needed

Window growth:
  → initial_ra_pages (128 KB default) → doubles → caps at max_ra_pages
```

This is why sequential `read()` in a loop achieves near-disk bandwidth — the readahead engine keeps the page cache one step ahead of the application.

---

## 18. Putting It All Together — End-to-End Latency Budget

For a single 4 KB buffered write that eventually reaches an NVMe SSD:

| Phase | Time | Where |
|-------|------|-------|
| `write()` syscall → page cache copy | ~1–5 μs | CPU: syscall + copy_from_user |
| Dirty page sits in cache | 0–30 seconds | RAM: waiting for writeback timer or pressure |
| Writeback: extent tree lookup | ~1–10 μs | CPU: ext4_map_blocks |
| Writeback: bio creation + submission | ~1–5 μs | CPU: bio_alloc + submit_bio |
| Block layer: merge + schedule + dispatch | ~1–10 μs | CPU: blk-mq path |
| Driver: build NVMe command + doorbell | ~0.5–2 μs | CPU → MMIO |
| NVMe internal write | ~10–100 μs | Hardware: flash program |
| Completion interrupt + softirq | ~1–5 μs | CPU: irq handler → bio_endio |
| **Total `write()` latency (user sees)** | **~1–5 μs** | **Just the page cache copy** |
| **Total data-on-disk latency** | **~15–150 μs** | **After writeback triggers** |
| **Total including writeback delay** | **~0.015–30 s** | **Timer or pressure driven** |

For a single 4 KB buffered read (cache miss) from NVMe:

| Phase | Time | Where |
|-------|------|-------|
| `read()` syscall → page cache miss | ~1–3 μs | CPU: filemap_read → no folio found |
| Readahead + bio creation + submission | ~2–10 μs | CPU: readahead + submit_bio |
| Block layer dispatch | ~1–5 μs | CPU: blk-mq path |
| NVMe internal read | ~10–80 μs | Hardware: flash read |
| Completion + copy_to_user | ~1–5 μs | CPU: bio_endio → copy |
| **Total `read()` latency (user sees)** | **~15–100 μs** | **Blocked until data arrives** |

The asymmetry is fundamental: `write()` is fast because the page cache absorbs it; `read()` on a cache miss must wait for hardware.

---

*This document provides a birds-eye view of the Linux 6.19 I/O path. For the internals of each layer, see: `vfs.md` (VFS abstractions), `fs.md` (ext4 filesystem), `block.md` (block layer and I/O schedulers), `drivers.md` (device driver model). Understanding this end-to-end flow is essential for diagnosing I/O performance issues, debugging filesystem corruption, and writing storage software.*
