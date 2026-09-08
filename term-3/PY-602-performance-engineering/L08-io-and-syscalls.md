# PY-602 · Lesson 08 — I/O, Syscalls, and Zero-Copy

**Estimated study time:** 3.5 hours
**Prerequisites:** L01, L02, L06; PY-601 L05

---

## 1. Orientation

```python
with open("big.txt") as f:
    for line in f:            # ~0.9 s for a 1 GB file
        process(line)

with open("big.txt", "rb") as f:
    while chunk := f.read(1 << 20):   # ~0.25 s
        process_chunk(chunk)
```

Same data, same disk. The difference is the number of times you crossed a boundary — into
the interpreter's buffering layer, into the kernel, and back — and how much you got each
time.

I/O performance is almost entirely about **how much work you do per crossing**. This lesson
is about where the crossings are and what each costs.

## 2. Theory

### 2.1 The cost of a syscall

A system call switches to kernel mode: save registers, switch stack, validate arguments, do
the work, switch back. Roughly **1–3 µs** on modern hardware — worse since the Spectre and
Meltdown mitigations, which added page-table switching on many systems.

Consequences:

- **10⁶ one-byte writes = several seconds of pure syscall overhead.** One 1 MB write is
  ~3 µs.
- A syscall is ~5,000× a function call and ~10× a DRAM access. It sits between "cache miss"
  and "SSD read" in L06's hierarchy.

So: **batch.** Every layer of the I/O stack exists to batch on your behalf, and knowing which
layer is doing what tells you where your time goes.

Count them: `strace -c -f python app.py` (Linux) or `dtruss` (macOS) gives a syscall
histogram. A surprising number and you have found the problem without reading any code.

### 2.2 The layers

Reading a line from a text file involves:

```
your code
  → TextIOWrapper       decodes bytes → str, splits lines
  → BufferedReader      8 KB buffer (io.DEFAULT_BUFFER_SIZE)
  → FileIO              read() syscall
  → kernel page cache   may already hold the data
  → block device
```

Each layer has a cost and a batching effect:

- **`TextIOWrapper`** — decoding is real work (UTF-8 validation and, for non-ASCII, widening
  per PEP 393). Opening in binary mode and decoding once per large chunk is meaningfully
  faster when you do not need per-line strings.
- **`BufferedReader`** — turns many small reads into few large ones. Its default 8 KB buffer
  is conservative; `open(path, buffering=1<<20)` reduces syscalls 128-fold for a sequential
  scan.
- **The page cache** — a repeated read of a hot file never touches the disk. This is why
  benchmarks are 10× faster on the second run, and why you must decide whether you are
  measuring cold or warm and say which.

**Line iteration** (`for line in f`) is convenient and does per-line Python-level work:
find the newline, slice, decode, allocate a `str`. For 10⁷ lines that is 10⁷ allocations. If
you only need to count, search, or split on a delimiter, working in `bytes` on large chunks
avoids all of it.

### 2.3 Reading files fast

Ordered by speed for a sequential scan of a large file:

```python
# 1. mmap — no copy into user space at all; the pages are mapped
import mmap
with open(path, "rb") as f, mmap.mmap(f.fileno(), 0, access=mmap.ACCESS_READ) as mm:
    count = mm.count(b"\n")

# 2. big binary chunks
with open(path, "rb", buffering=0) as f:
    while chunk := f.readinto(buf):   # readinto: no allocation per chunk
        ...

# 3. buffered binary reads
with open(path, "rb") as f:
    while chunk := f.read(1 << 20):
        ...

# 4. text line iteration
with open(path) as f:
    for line in f: ...
```

**`mmap`** maps the file into your address space; reads become page faults served from the
page cache with no copy into a user buffer. Excellent for random access, for repeatedly
scanning the same file, and for sharing across processes. Its costs: it is not faster for a
single sequential pass on a cold file (you still fault in every page); page faults are not
free; and a file larger than address space needs windowing. Also `mmap` on a file being
written concurrently is a correctness question, not just a performance one.

**`readinto`** reuses a buffer, eliminating one allocation per chunk (L03 §2.6).

`os.preadv`, `os.sendfile`, and `os.copy_file_range` are the low-level tools when you need
them.

### 2.4 Writing

```python
f.write(data)        # into the buffer
f.flush()            # buffer → kernel (a write syscall)
os.fsync(f.fileno()) # kernel → durable storage
```

Three distinct levels of "written", and confusing them is the source of both performance
myths and data-loss bugs:

- **Buffered** — in your process. Lost on crash.
- **Flushed** — in the kernel's page cache. Survives a process crash; lost on power loss.
- **fsync'd** — on stable storage. Survives power loss. **Costs 0.1–10 ms** depending on the
  device, and is the reason databases are slower than you expect.

`fsync` per record is why a naive append-only log does 100 writes/second. Batching many
records into one `fsync` is the entire reason for group commit in databases (DI-721 L03).

Durability also requires `fsync` on the *directory* after creating a file, or the entry may
not survive. Almost nobody does this and it is a real source of lost files after power loss.

For a file you are about to write in full, `posix_fallocate` avoids fragmentation, and
`O_DIRECT` bypasses the page cache (rarely what you want unless you are writing a database).

### 2.5 Network I/O

```python
sock.send(data)      # copies user buffer → kernel socket buffer
sock.recv(4096)      # copies kernel buffer → a NEW bytes object
sock.recv_into(buf)  # copies into YOUR buffer — no allocation
```

Cost structure per message: a syscall, one or two copies, and — the dominant term for small
messages — the **round trip**. A same-datacenter RTT is ~0.5 ms; that is ~150 syscalls' worth
of time. So for network work, **the number of round trips dominates everything else**.

This is SE-521 L09's chatty-API argument in hardware terms, and PY-601 L09's batching
argument again. One request returning 1,000 items beats 1,000 requests, by roughly three
orders of magnitude, and no amount of local optimization recovers it.

**Nagle's algorithm** buffers small writes to avoid sending tiny packets; combined with
delayed ACKs it can add ~40 ms to a request/response exchange. `TCP_NODELAY` disables it and
is correct for request/response protocols:

```python
sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)
```

Most HTTP libraries set it; if you write a protocol yourself, you must.

**Zero-copy** paths that avoid user-space copies entirely:

```python
os.sendfile(out_fd, in_fd, offset, count)   # file → socket, entirely in the kernel
socket.socket.sendfile(file)                 # the same, via the socket API
```

Useful for static file serving; irrelevant if you transform the data (you have to see it to
transform it).

### 2.6 Latency versus throughput for I/O

Different problems, different tools:

| | Throughput | Latency |
|---|---|---|
| Goal | bytes/second | time for one operation |
| Helps | large reads, batching, parallelism, compression | avoiding round trips, `TCP_NODELAY`, prefetch, caching |
| Hurts | small operations, per-item syscalls | batching, queueing, Nagle |

**Batching improves throughput and worsens latency.** That is the fundamental tension, and it
is a product decision (L01 §2.2).

**Parallelism helps I/O throughput** because devices have queue depth: an NVMe SSD can serve
dozens of requests concurrently and needs a deep queue to reach its rated throughput. One
thread issuing sequential reads leaves most of the device idle. This is why `asyncio` or a
thread pool helps disk I/O as well as network I/O — a fact people often get wrong because
"disk is sequential".

### 2.7 Serialization

Frequently the dominant cost in a service, and almost always underestimated.

Rough relative costs for the same structured payload (measure your own — these move):

| Format | Encode | Decode | Size | Note |
|---|---|---|---|---|
| `json` (stdlib) | 1.0 | 1.0 | 1.0 | baseline |
| `orjson` | ~0.2 | ~0.3 | 1.0 | Rust; the easy win |
| `msgspec` | ~0.1 | ~0.15 | ~0.8 | with schema validation included |
| `pickle` (proto 5) | ~0.3 | ~0.4 | ~0.9 | Python-only; unsafe on untrusted input |
| MessagePack | ~0.3 | ~0.3 | ~0.7 | cross-language |
| Protobuf | ~0.3 | ~0.25 | ~0.5 | schema, versioning, cross-language |
| Arrow IPC | ~0.02 | ~0.01 | ~0.6 | columnar; effectively zero-copy for arrays |

Two structural points beyond the constants:

- **Arrow's advantage is not a faster parser; it is not parsing at all.** The wire format
  *is* the memory format, so "decoding" is pointing at a buffer. For large tabular data this
  is a different category of thing, not a constant-factor win.
- **Schema formats (Protobuf, Arrow, msgspec) beat schemaless ones** on both size and speed,
  because they do not encode field names per record and do not need to discover types. The
  cost is a schema to maintain and evolve (DI-721 L09).

The first thing to try in a JSON-heavy service is `orjson` or `msgspec`: a one-line change
for a 3–10× serialization improvement, and serialization is often 20–40% of a service's CPU.

### 2.8 Compression

A CPU-versus-bytes trade, and the right answer depends entirely on the link:

| Codec | Compression | Speed | Use |
|---|---|---|---|
| none | 1× | ∞ | fast local links |
| lz4 | ~2× | ~500 MB/s | hot paths, in-memory caches |
| zstd (level 1–3) | ~3× | ~200–400 MB/s | **the general default** |
| zstd (level 19) | ~4× | ~5 MB/s | write-once, read-many archives |
| gzip | ~3× | ~50 MB/s | compatibility |

The rule: compress when `bytes_saved / link_bandwidth > compression_time`. On a 100 Mb/s
link, compression nearly always wins. On a 100 Gb/s intra-datacenter link, it often loses.
Measure the link before assuming.

zstd's dictionary mode is a large and underused win for many small similar messages — train
a dictionary on a sample and get 2–5× better ratios on payloads too small for normal
compression to help.

## 3. Construction: an I/O optimization study

Take a real I/O-bound job — log processing, a bulk import, a file conversion, an API client.

**Step 1 — establish the regime.** `perf_counter` versus `process_time` (L02 §2.5). If CPU
time is 10% of wall clock, you are waiting; find out for what.

**Step 2 — count the syscalls.** `strace -c -f` (or `dtruss`). Report the histogram. Look for:
a huge count of small `read`/`write`, unexpected `stat`/`open` (a config file read per
request? a `.py` stat per import?), and `fsync` frequency.

**Step 3 — the read path.** Implement all four approaches of §2.3 and measure, **cold and
warm** (`echo 3 > /proc/sys/vm/drop_caches` between runs on Linux, with root; on macOS,
`purge`). Report both, because the ratio between them tells you whether you are measuring the
disk or the page cache.

**Step 4 — decode once.** If the job reads text, compare per-line text iteration with
chunked binary reads plus a single decode. Report the ratio and the allocation count
difference (`tracemalloc`).

**Step 5 — the write path.** Measure: unbuffered writes, buffered, buffered with a large
buffer, and each with and without `fsync` per record versus per batch. Report the table.
State explicitly what durability guarantee each row provides — that column is the assessed
part, because the fast rows are fast precisely because they promise less.

**Step 6 — parallelism and queue depth.** For a disk-bound read workload, measure throughput
at 1, 2, 4, 8, 16 concurrent readers. Find the point where the device saturates. This
surprises people who believe disk I/O is inherently sequential.

**Step 7 — serialization.** If the job serializes, swap `json` for `orjson` and `msgspec` and
measure. Then, if the data is tabular, try Arrow IPC and report the difference in kind.

**Step 8 — compression.** Measure end-to-end time with none, lz4, and zstd-3, over the actual
link (or a simulated one with `tc netem`). Report the crossover bandwidth.

**Step 9 — the report.** Every change with its measurement, the durability implications of
the write path, and the final end-to-end number against the L01 baseline.

## 4. Failure modes

- **Per-line text iteration over a huge file** when bytes would do.
- **Default 8 KB buffering** for a sequential scan of a large file.
- **A syscall per item.** Batch.
- **`fsync` per record** — and, conversely, *not* `fsync`ing when durability was promised.
- **Not `fsync`ing the directory** after creating a file.
- **Confusing flushed with durable.**
- **Measuring warm and reporting it as cold**, or vice versa, without saying which.
- **`recv()` allocating a new `bytes` per call** in a hot loop; use `recv_into`.
- **Nagle plus delayed ACK** adding 40 ms to a request/response protocol.
- **Chatty protocols.** N round trips where one would do.
- **One thread against an NVMe device**, leaving 80% of its throughput unused.
- **`json` in a service** where `orjson` is a one-line change.
- **Compressing on a fast link**, or not compressing on a slow one — both without measuring.
- **Ignoring serialization** when profiling a "network-bound" service; it is often the
  actual cost.

## 5. Exercises

### Warm-up (25 min)

**W1.** Count the lines of a 1 GB file five ways (text iteration, binary chunks, `readinto`,
`mmap.count`, `wc -l` as a reference). Report cold and warm timings.

**W2.** `strace -c` a simple script. Explain the top three syscalls by count.

**W3.** Measure the cost of `fsync` on your machine: writes/second with `fsync` per record
and with `fsync` per 1,000 records.

### Core (2.5 h)

**C1 — The I/O study.** Complete §3, all nine steps. Deliverable: the syscall histogram, the
cold/warm read table, the write table **with the durability column**, the queue-depth curve,
the serialization comparison, the compression crossover, and the end-to-end result.

**C2 — Zero-copy.** Build a small file server three ways: read-then-send, `sendfile`, and
`mmap`-then-send. Measure throughput and CPU usage for each on a 1 GB file. Explain the CPU
difference in terms of copies.

**C3 — Serialization swap.** Take a service that serializes JSON. Measure the serialization
fraction of total CPU (L02). Swap to `orjson`, then to `msgspec` with schemas. Report the
CPU reduction, the end-to-end latency change, and every behavioural difference you had to
handle (datetime formats, key ordering, non-string keys, NaN, subclasses).

**C4 — Round trips.** Take a client that makes N requests in a loop. Measure. Then implement:
pipelining, batching into one request, and concurrent requests with a bounded semaphore.
Report all four at N = 10, 100, 1,000. Explain which wins where in terms of RTT versus
bandwidth versus server-side cost.

### Challenge

**X1.** Compare `asyncio`, threads, and `io_uring` (via a binding such as `liburing`-based
libraries, on Linux 5.10+) for a high-concurrency file-read workload. Report throughput,
CPU per operation, and syscall counts. Then write 800 words on the structural difference
between readiness-based and completion-based I/O (PY-601 L05 §2.1) and what it means for
Python's future async story.

**X2.** Instrument a real service to attribute wall clock to: DNS, TCP connect, TLS
handshake, request write, server processing, response read, and deserialization. Produce the
breakdown for p50 and p99 separately. The p99 breakdown almost always differs qualitatively
from p50 — report the difference and what it implies about where to optimize.

## 6. Self-check

1. What does a syscall cost, and where does it sit in the latency hierarchy?
2. Name the layers between `for line in f` and the block device, and the batching each does.
3. Give four ways to read a large file, ordered, and say when `mmap` does *not* help.
4. Distinguish buffered, flushed, and fsync'd, and give the cost of the last.
5. Why does parallelism help disk throughput?
6. What is Nagle's algorithm plus delayed ACK, and what does it cost?
7. Why is Arrow's speed a difference in kind rather than degree?
8. Give the rule for when compression pays.

## 7. Primary sources

- Kerrisk, *The Linux Programming Interface*, chs. 4–5 (file I/O), 13 (buffering), 49
  (`mmap`), 61 (sockets advanced).
- Gregg, *Systems Performance*, chs. 8 (file systems) and 9 (disks).
- `io` module documentation — the layered-architecture section.
- Apache Arrow columnar and IPC format specifications.
- Facebook/Meta's zstd documentation, particularly the dictionary section.

---

**Previous:** [L07](L07-extending-python.md) · **Next:**
[L09 — Capacity, Queueing, and Performance in Production](L09-capacity-and-production.md)
