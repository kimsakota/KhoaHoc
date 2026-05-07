# Hướng Dẫn Viết Báo Cáo Khoa Học - Dự Án VstHelper

> **Mục tiêu**: Cung cấp quy trình toàn diện để viết bài báo khoa học về đề tài VstHelper - Embedded NoSQL Database Engine với các kỹ thuật tối ưu hóa bộ nhớ và hiệu năng cao.

---

## Mục Lục

1. [Tổng Quan Đề Tài](#1-tổng-quan-đề-tài)
2. [Các Hướng Nghiên Cứu](#2-các-hướng-nghiên-cứu)
3. [Quy Trình Viết Bài](#3-quy-trình-viết-bài)
4. [Benchmark & Đo Lường](#4-benchmark--đo-lường)
5. [Cấu Trúc Bài Báo](#5-cấu-trúc-bài-báo)
6. [Tài Liệu Tham Khảo](#6-tài-liệu-tham-khảo)
7. [Công Cụ Hỗ Trợ](#7-công-cụ-hỗ-trợ)
8. [Timeline](#8-timeline)
9. [Checklist Hoàn Thành](#9-checklist-hoàn-thành)

---

## 1. Tổng Quan Đề Tài

### 1.1 Giới thiệu VstHelper

VstHelper là một embedded NoSQL/document database engine được viết bằng C# với unsafe code, tập trung vào hiệu năng cực cao cho các ứng dụng nhúng và real-time.

```
┌─────────────────────────────────────────────────────────────────────┐
│                         VstHelper Architecture                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   ┌─────────────┐     ┌─────────────┐     ┌─────────────┐          │
│   │  User App   │────▶│  JSON API   │────▶│  KeyIndexer │          │
│   └─────────────┘     └─────────────┘     └─────────────┘          │
│                              │                    │                │
│                              ▼                    ▼                │
│                       ┌─────────────┐     ┌─────────────┐          │
│                       │  DataPage   │◀───▶│  KeyEntry   │          │
│                       └─────────────┘     └─────────────┘          │
│                              │                                      │
│                              ▼                                      │
│                       ┌─────────────┐     ┌─────────────┐          │
│                       │ MemoryPool  │     │  Win32File   │          │
│                       │ (Slab Alloc)│     │  (MemoryMap) │          │
│                       └─────────────┘     └─────────────┘          │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 Các điểm nghiên cứu chính

| Thành phần | Công nghệ | Giá trị nghiên cứu |
|---|---|---|
| Memory Management | Slab Allocator + Bitmask CAS | Zero fragmentation, O(1) allocation |
| Threading | RWLock + Lock-free operations | Multi-core scalability |
| Sorting | Parallel QuickSort (Median-of-three) | High-throughput indexing |
| Hashing | Inline XXHash64/MD5 | Zero GC pressure |
| I/O | Memory-mapped files + Win32 API | Zero-copy operations |
| Data Structures | Custom RawList, PointerList | Bypass BCL overhead |

### 1.3 Đối tượng hưởng lợi

- **Nhà phát triển IoT/Embedded systems**: Giảm memory fragmentation
- **Kỹ sư Viễn thông**: Low-latency cho real-time processing
- **Nhà nghiên cứu Database**: Mô hình lock-free allocator
- **Game developers**: Consistent frame rate không GC pause

---

## 2. Các Hướng Nghiên Cứu

### 2.1 Hướng 1: Lock-free Memory Allocator (Trọng tâm chính)

#### 2.1.1 Mô tả

Nghiên cứu sâu về kiến trúc Slab Allocator với lock-free CAS operations trong môi trường .NET.

#### 2.1.2 Các điểm cần phân tích

```
┌─────────────────────────────────────────────────────────────────────┐
│  Slab Allocator Design                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  MemoryPool (16 bins)                                               │
│  ├── Bin[0]: 16 bytes ────▶ MemoryPageGroup[0] ──▶ 64 slots     │
│  ├── Bin[1]: 32 bytes ────▶ MemoryPageGroup[1] ──▶ 64 slots     │
│  ├── Bin[2]: 64 bytes ────▶ MemoryPageGroup[2] ──▶ 64 slots     │
│  ├── ...                                                            │
│  └── Bin[15]: 8KB ──────▶ MemoryPageGroup[15] ──▶ 64 slots       │
│                                                                      │
│  Key Features:                                                       │
│  ✓ Fixed-size slots (no fragmentation)                              │
│  ✓ Bitmask tracking (FreeSlots &= ~mask)                           │
│  ✓ CAS atomic operations                                            │
│  ✓ O(1) allocation/deallocation                                     │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

#### 2.1.3 Các paper liên quan cần đọc

| Paper | Nội dung | Citation |
|---|---|---|
| "Hoard: A Lock-Free Allocator" | Lock-free memory allocator design | Berger et al., 2000 |
| "jemalloc: A General Purpose Allocator" | Scalable memory allocator | Evans, 2006 |
| "tcmalloc: Thread-Caching Malloc" | Per-thread caches | Google, ongoing |
| "The Memory Management in the .NET Runtime" | .NET GC internals | Maeda, 2012 |

#### 2.1.4 Câu hỏi nghiên cứu

```
1. Làm thế nào để thiết kế allocator không gây GC pressure trong .NET?
2. Bitmask-based allocation hiệu quả như thế nào so với free list?
3. Lock-free CAS có thực sự nhanh hơn mutex trong multi-threaded context?
```

#### 2.1.5 Metrics cần đo

- Allocation latency (nanoseconds)
- Deallocation latency
- Fragmentation rate after N operations
- Cache miss rate
- False sharing incidents

---

### 2.2 Hướng 2: Parallel QuickSort Optimization

#### 2.2.1 Mô tả

Phân tích thuật toán QuickSort song song với các tối ưu hóa: median-of-three pivot, insertion sort hybrid, và parallel threshold.

#### 2.2.2 Kiến trúc thuật toán

```
┌─────────────────────────────────────────────────────────────────────┐
│  Parallel QuickSort Flow                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  public void Sort(KeyEntry* entries, int count)                     │
│  │                                                                  │
│  ├── Stack-based iterative QuickSort (không recursion overflow)    │
│  │                                                                  │
│  ├── Pivot Selection: Median-of-three                                │
│  │   ├── left[0], middle[count/2], right[count-1]                   │
│  │   └── Swap để đưa pivot về position count-1                      │
│  │                                                                  │
│  ├── Partition: Lomuto scheme                                       │
│  │   └── Tất cả < pivot bên trái, > pivot bên phải                  │
│  │                                                                  │
│  ├── Hybrid Optimization:                                           │
│  │   ├── if (range < 128) → Insertion Sort (cache-friendly)        │
│  │   └── else if (range > threshold) → Parallel.Invoke()           │
│  │                                                                  │
│  └── Repeat cho đến khi stack empty                                │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

#### 2.2.3 Các paper liên quan

| Paper | Nội dung |
|---|---|
| "Analysis of Quicksort Partitions" | Lomuto vs Hoare partition schemes |
| "Parallel Sorting by Regular Sampling" | Sample sort for load balancing |
| "The Euclidean Algorithm and Quicksort" | Median-of-three effectiveness |

#### 2.2.4 Câu hỏi nghiên cứu

```
1. Optimal threshold cho insertion sort hybrid là bao nhiêu?
2. Parallel.Invoke overhead vs speedup từ multi-core?
3. Median-of-three pivot selection có giảm worst-case probability không?
```

---

### 2.3 Hướng 3: Embedded Database Performance

#### 2.3.1 Mô tả

Đánh giá hiệu năng tổng thể của VstHelper như một embedded database engine và so sánh với các đối thủ.

#### 2.3.2 So sánh với đối thủ

```
┌─────────────────────────────────────────────────────────────────────┐
│  Comparison Matrix                                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Feature              │ VstHelper │ SQLite │ LMDB │ RocksDB        │
│  ─────────────────────┼───────────┼────────┼──────┼─────────       │
│  Language             │ C#        │ C      │ C    │ C++            │
│  GC-Free Hot Path     │ ✅        │ N/A    │ N/A  │ N/A            │
│  Lock-free Allocator  │ ✅        │ ❌     │ ✅   │ ✅             │
│  Memory-mapped I/O    │ ✅        │ ✅     │ ✅   │ ✅             │
│  Parallel Index Build │ ✅        │ ❌     │ ❌   │ ✅             │
│  JSON Support         │ ✅        │ ✅     │ ❌   │ ✅             │
│  ACID Transactions    │ 🔨        │ ✅     │ ✅   │ ✅             │
│  License              │ MIT       │ Public │ Open │ Apache         │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

#### 2.3.3 Benchmark workloads

| Workload | Mô tả | Tỷ lệ |
|---|---|---|
| **Write-heavy** | Sequential insert | 100% write |
| **Read-heavy** | Point queries | 95% read / 5% write |
| **Mixed** | Realistic workload | 70% read / 30% write |
| **Bulk load** | Initial data load | 100% write |
| **Update** | In-place modifications | 50% read / 50% update |

---

### 2.4 Hướng 4: Thread Synchronization in Database Systems

#### 2.4.1 Mô tả

Phân tích chiến lược RWLock và lock-free data structures trong context của embedded database.

#### 2.4.2 RWLock Implementation Analysis

```
┌─────────────────────────────────────────────────────────────────────┐
│  LockEngine State Machine                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  State Encoding:                                                     │
│  ├── >= 0: Số lượng reader đang giữ lock (shared count)             │
│  └── -1:   Writer đang giữ lock (exclusive)                        │
│                                                                      │
│  EnterRead():                                                        │
│  ├── CAS(_state, s, s+1) cho đến khi thành công                    │
│  └── Nếu fail → SpinWait → WaitForSingleObject(1ms)                 │
│                                                                      │
│  EnterWrite():                                                       │
│  ├── CAS(_state, 0, -1) cho đến khi thành công                     │
│  └── Nếu fail → SpinWait → WaitForSingleObject(1ms)                 │
│                                                                      │
│  ExitRead():                                                         │
│  └── Interlocked.Decrement → nếu = 0 → SetEvent(đánh thức writer)  │
│                                                                      │
│  ExitWrite():                                                        │
│  └── Interlocked.Exchange(0) → SetEvent(đánh thức tất cả)         │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Quy Trình Viết Bài

### 3.1 Phase 1: Nghiên cứu (2 tuần)

#### Bước 1.1: Systematic Literature Review

```markdown
## Tìm kiếm paper

### Keywords cho search:
- "lock-free allocator" AND ".NET" AND "performance"
- "slab allocator" AND "benchmark"
- "parallel quicksort" AND "multi-core"
- "embedded database" AND "memory-mapped"
- "RWLock" AND "scalability"

### Databases để tìm:
1. Google Scholar (scholar.google.com)
2. ACM Digital Library
3. IEEE Xplore
4. arXiv (arxiv.org)
5. Semantic Scholar

### Inclusion criteria:
- Paper từ 2015 trở lại
- Có benchmark/evaluation
- Source code available (bonus)
- Liên quan trực tiếp đến 1 trong 4 hướng nghiên cứu

### Exclusion criteria:
- Non-peer-reviewed
- Không có đóng góp mới (survey-only)
- Không có experimental results
```

#### Bước 1.2: Đọc và note-taking

```
┌─────────────────────────────────────────────────────────────────────┐
│  Reading Log Template                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Paper: [Title]                                                      │
│  Authors: [Names]                                                    │
│  Year: [Year]                                                        │
│  Venue: [Conference/Journal]                                         │
│                                                                      │
│  Summary (2-3 sentences):                                            │
│  ────────────────────────────────────────────────────────────────── │
│  [Viết tóm tắt ngắn]                                                │
│                                                                      │
│  Contribution:                                                       │
│  ────────────────────────────────────────────────────────────────── │
│  1. [Contribution 1]                                                │
│  2. [Contribution 2]                                                │
│                                                                      │
│  Method:                                                             │
│  ────────────────────────────────────────────────────────────────── │
│  [Experiment setup, benchmarks used]                                │
│                                                                      │
│  Results:                                                            │
│  ────────────────────────────────────────────────────────────────── │
│  [Key numbers, comparisons]                                         │
│                                                                      │
│  Relevance to VstHelper:                                             │
│  ────────────────────────────────────────────────────────────────── │
│  [Liên quan như thế nào đến đề tài của bạn]                        │
│                                                                      │
│  Quotes to cite:                                                     │
│  ────────────────────────────────────────────────────────────────── │
│  "[Direct quote nếu có]"                                             │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

#### Bước 1.3: Tổ chức tài liệu tham khảo

```
references/
├── allocator/
│   ├── berger-hoard-2000.pdf
│   ├── evans-jemalloc-2006.pdf
│   └── reference-manager.json
├── sorting/
│   ├── musyx-parallel-sorting.pdf
│   └── sorting-survey-2020.pdf
├── database/
│   ├── lmdb-architecture.pdf
│   └── rocksdb-paper.pdf
└── tools/
    ├── benchmarkdotnet-guide.pdf
    └── latex-template.pdf
```

---

### 3.2 Phase 2: Thiết kế Experiment (1 tuần)

#### Bước 2.1: Xác định Research Questions

```
┌─────────────────────────────────────────────────────────────────────┐
│  Research Questions Template                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  RQ1: [Performance của Slab Allocator]                              │
│  ──────────────────────────────────────────────────────────────────  │
│  "Lock-free slab allocator có đạt được hiệu năng cao hơn so với     │
│   system allocator trong multi-threaded workload không?"             │
│                                                                      │
│  Hypothesis H1:                                                      │
│  "VstHelper allocator sẽ có latency thấp hơn 30% và ít              │
│   fragmentation hơn 50% so với system allocator"                   │
│                                                                      │
│  Metric:                                                            │
│  • Primary: Allocation latency (ns/op)                              │
│  • Secondary: Memory fragmentation (%)                              │
│  • Baseline: System.Allocator, jemalloc                             │
│                                                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  RQ2: [Parallel Sort Scalability]                                    │
│  ──────────────────────────────────────────────────────────────────  │
│  "Parallel QuickSort với median-of-three và insertion sort hybrid    │
│   scale hiệu quả trên nhiều cores không?"                          │
│                                                                      │
│  Hypothesis H2:                                                      │
│  "Speedup gần như linear với số cores cho dataset >1M elements"    │
│                                                                      │
│  Metric:                                                            │
│  • Primary: Speedup ratio (time_sequential / time_parallel)         │
│  • Secondary: CPU utilization (%)                                   │
│  • Baseline: Sequential QuickSort, std::sort                        │
│                                                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  RQ3: [End-to-end Database Performance]                              │
│  ──────────────────────────────────────────────────────────────────  │
│  "VstHelper có đạt được throughput cao hơn SQLite và LMDB           │
│   trong embedded workload không?"                                    │
│                                                                      │
│  Hypothesis H3:                                                      │
│  "VstHelper đạt 2x throughput trong write-heavy workload"          │
│                                                                      │
│  Metric:                                                            │
│  • Primary: Operations/second                                        │
│  • Secondary: Latency distribution (p50, p95, p99)                  │
│  • Baseline: SQLite, LMDB                                           │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

#### Bước 2.2: Thiết kế Experiment Matrix

```
┌─────────────────────────────────────────────────────────────────────┐
│  Experiment Matrix                                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Exp# │ RQ │ Metric            │ Baseline         │ Variables       │
│  ─────┼────┼───────────────────┼─────────────────┼───────────────  │
│  E1   │ RQ1│ Alloc latency     │ malloc/free     │ Thread count    │
│  E2   │ RQ1│ Fragmentation %   │ jemalloc        │ Operation count │
│  E3   │ RQ1│ Cache miss rate   │ tcmalloc        │ Allocation size │
│  E4   │ RQ2│ Sort time         │ Sequential QS   │ Dataset size    │
│  E5   │ RQ2│ Speedup ratio     │ .NET Array.Sort │ Thread count    │
│  E6   │ RQ2│ CPU utilization   │ TPL DataFlow    │ Partition size  │
│  E7   │ RQ3│ Throughput (ops/s)│ SQLite          │ Workload type   │
│  E8   │ RQ3│ Latency (μs)      │ LMDB            │ Concurrency     │
│  E9   │ RQ3│ Memory usage      │ RocksDB         │ Data size       │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

### 3.3 Phase 3: Implementation & Benchmarking (2 tuần)

#### Bước 3.1: BenchmarkDotNet Setup

```csharp
// BenchmarkDotNet configuration cho VstHelper

[SimpleJob(RuntimeMoniker.Net80)]
[SimpleJob(RuntimeMoniker.Net60)]
[SimpleJob(RuntimeMoniker.Net48)]
public class MemoryAllocatorBenchmark
{
    private MemoryPool _pool;
    
    [GlobalSetup]
    public void Setup()
    {
        _pool = new MemoryPool(16);
    }
    
    [Benchmark(Baseline = true)]
    public long SystemAlloc_Free()
    {
        var ptr = Marshal.AllocHGlobal(256);
        Marshal.FreeHGlobal(ptr);
        return ptr.ToInt64();
    }
    
    [Benchmark]
    public long SlabAlloc_Rent_Return()
    {
        var slot = MemoryPool.Rent(256);
        MemoryPool.Return(slot);
        return slot.Handle.ToInt64();
    }
}
```

#### Bước 3.2: Statistical Rigor

```
┌─────────────────────────────────────────────────────────────────────┐
│  Statistical Requirements                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Sample Size Calculation:                                            │
│  ─────────────────────────────────────────────────────────────────  │
│  • Minimum 30 runs cho mỗi benchmark                                │
│  • Use Student's t-test cho significance                            │
│  • Report: mean, median, std dev, CI (95%)                         │
│                                                                      │
│  Warmup & Stability:                                                 │
│  ─────────────────────────────────────────────────────────────────  │
│  • Warmup: 10-30 iterations                                         │
│  • Cool down: 5 iterations                                          │
│  • Verify: Coefficient of Variation < 5%                           │
│                                                                      │
│  Outlier Handling:                                                   │
│  ─────────────────────────────────────────────────────────────────  │
│  • Remove outliers > 2 std dev                                      │
│  • Document any GC pauses (dotnet-counters)                         │
│                                                                      │
│  Reproducibility:                                                   │
│  ─────────────────────────────────────────────────────────────────  │
│  • Seed for random number generators (if any)                       │
│  • Pin process to specific cores (if needed)                        │
│  • Document system configuration                                    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

#### Bước 3.3: Profiling Checklist

```markdown
## Profiling Tools & Metrics

### CPU Profiling
- [ ] Intel VTune Profiler (or AMD μProf)
- [ ] dotnet-trace for .NET-specific analysis
- [ ] Metrics: CPI, instruction retired, branch mispredictions

### Memory Profiling
- [ ] dotMemory (JetBrains) or Memory Profiler
- [ ] GC pauses monitoring (dotnet-counters)
- [ ] Metrics: Allocation rate, GC count, heap size

### Cache Profiling
- [ ] perf stat -e cache-misses,cache-references
- [ ] VTune: Cache bandwidth analysis
- [ ] Metrics: L1/L2/L3 miss rate

### I/O Profiling
- [ ] strace / dtrace for system calls
- [ ] Metrics: Read/write latency, throughput
```

---

### 3.4 Phase 4: Writing (3-4 tuần)

#### Bước 4.1: Writing Order

```
Khuyến nghị thứ tự viết:

1. Viết Methods trước (dễ nhất, đã có results)
   ↓
2. Viết Evaluation/Results (có figures từ benchmark)
   ↓
3. Viết Abstract & Introduction (tổng hợp từ các phần khác)
   ↓
4. Viết Background & Related Work (context cho contribution)
   ↓
5. Viết Design/Implementation (giải thích WHY)
   ↓
6. Viết Conclusion & Future Work
   ↓
7. Polish Abstract cuối cùng
```

#### Bước 4.2: Section-by-Section Guide

---

### 3.4.1 Abstract (150-250 words)

```
STRUCTURE:
┌─────────────────────────────────────────────────────────────────────┐
│  Context (1 sentence)                                               │
│  "Embedded systems require low-latency storage without GC pauses"   │
│                                                                      │
│  Problem (1 sentence)                                                │
│  "Existing .NET allocators introduce unpredictable latency..."     │
│                                                                      │
│  Solution (1-2 sentences)                                          │
│  "We present VstHelper, a lock-free slab allocator with..."        │
│                                                                      │
│  Evaluation (1-2 sentences)                                        │
│  "Experiments show 2.3x faster allocation and 5x less             │
│   fragmentation compared to jemalloc..."                           │
│                                                                      │
│  Impact (1 sentence)                                               │
│  "This work enables real-time applications on .NET..."            │
└─────────────────────────────────────────────────────────────────────┘
```

---

### 3.4.2 Introduction (1 page)

```
STRUCTURE:
┌─────────────────────────────────────────────────────────────────────┐
│  Paragraph 1: Context & Motivation (5-6 sentences)                 │
│  ─────────────────────────────────────────────────────────────────  │
│  • Start broad: embedded systems, IoT, real-time requirements       │
│  • Narrow down: memory management challenges in .NET                 │
│  • End with: "However, existing solutions suffer from..."            │
│                                                                      │
│  Paragraph 2: Problem Statement (3-4 sentences)                     │
│  ─────────────────────────────────────────────────────────────────  │
│  • State the specific problem clearly                               │
│  • Quantify the problem if possible                                 │
│  • "This leads to GC pauses of 10-50ms..."                         │
│                                                                      │
│  Paragraph 3: Our Approach (4-5 sentences)                         │
│  ─────────────────────────────────────────────────────────────────  │
│  • Introduce VstHelper                                              │
│  • Highlight key innovations (3-4 bullet points)                   │
│  • Explain how it addresses the problem                             │
│                                                                      │
│  Paragraph 4: Contributions (3-4 bullet points)                    │
│  ─────────────────────────────────────────────────────────────────  │
│  • "We present a lock-free slab allocator..."                      │
│  • "We demonstrate 2x speedup in parallel sorting..."              │
│  • "We provide comprehensive benchmarks..."                         │
│                                                                      │
│  Paragraph 5: Roadmap (1 paragraph)                                │
│  ─────────────────────────────────────────────────────────────────  │
│  • "The rest of this paper is organized as follows..."             │
│  • Section 2: Background...                                        │
│  • Section 3: Design...                                            │
│  • etc.                                                             │
└─────────────────────────────────────────────────────────────────────┘
```

---

### 3.4.3 Related Work (1-1.5 pages)

```
STRUCTURE:
┌─────────────────────────────────────────────────────────────────────┐
│  Section 2.1: Memory Allocators (3-4 paragraphs)                   │
│  ─────────────────────────────────────────────────────────────────  │
│  Paragraph on general-purpose allocators:                           │
│  • jemalloc, tcmalloc, Hoard                                       │
│  • Their design philosophies                                        │
│  • How VstHelper differs                                           │
│                                                                      │
│  Paragraph on embedded allocators:                                  │
│  • TLSF, Robin Hood hashing                                         │
│  • Focus on real-time constraints                                  │
│                                                                      │
├─────────────────────────────────────────────────────────────────────┤
│  Section 2.2: Parallel Sorting (2-3 paragraphs)                    │
│  ─────────────────────────────────────────────────────────────────  │
│  • Traditional parallel sort algorithms                            │
│  • Recent advances (2015+)                                         │
│  • VstHelper's hybrid approach                                     │
│                                                                      │
├─────────────────────────────────────────────────────────────────────┤
│  Section 2.3: Embedded Databases (2-3 paragraphs)                  │
│  ─────────────────────────────────────────────────────────────────  │
│  • LMDB, RocksDB architecture                                       │
│  • Their memory management strategies                              │
│  • Gap that VstHelper fills                                        │
│                                                                      │
├─────────────────────────────────────────────────────────────────────┤
│  Section 2.4: Lock-free Data Structures (2 paragraphs)             │
│  ─────────────────────────────────────────────────────────────────  │
│  • Michael-Scott queues                                             │
│  • Lock-free hash tables                                           │
│  • Application to allocators                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

### 3.4.4 Design (2-3 pages)

```
STRUCTURE:
┌─────────────────────────────────────────────────────────────────────┐
│  Section 3.1: Design Goals (1 paragraph)                            │
│  ─────────────────────────────────────────────────────────────────  │
│  • List 4-5 design goals with rationale                            │
│  • "First, we prioritize allocation latency..."                    │
│  • "Second, we require thread-safety without serialization..."     │
│                                                                      │
├─────────────────────────────────────────────────────────────────────┤
│  Section 3.2: Memory Layer (with figures)                          │
│  ─────────────────────────────────────────────────────────────────  │
│  [Figure 1: Memory Pool Architecture]                               │
│  • Explain bin structure                                            │
│  • Explain page group management                                   │
│                                                                      │
│  [Code Snippet: Rent operation]                                     │
│  • Walk through the code                                            │
│  • Highlight CAS operation                                         │
│                                                                      │
│  [Figure 2: Bitmask tracking]                                       │
│  • Visualize free slot bitmask                                      │
│  • Explain Log2 computation                                         │
│                                                                      │
│  Analysis:                                                          │
│  • Time complexity: O(1) for rent/return                            │
│  • Space complexity: 64 slots per page                             │
│                                                                      │
├─────────────────────────────────────────────────────────────────────┤
│  Section 3.3: Threading Model                                       │
│  ─────────────────────────────────────────────────────────────────  │
│  • Explain RWLock design                                            │
│  • Why spin-wait before OS wait                                     │
│  • Trade-offs discussed                                            │
│                                                                      │
├─────────────────────────────────────────────────────────────────────┤
│  Section 3.4: Sorting Algorithm                                     │
│  ─────────────────────────────────────────────────────────────────  │
│  • Parallel QuickSort with hybrid optimization                     │
│  • Threshold selection rationale                                    │
│  • Load balancing strategy                                          │
└─────────────────────────────────────────────────────────────────────┘
```

---

### 3.4.5 Evaluation (2-3 pages)

```
STRUCTURE:
┌─────────────────────────────────────────────────────────────────────┐
│  Section 4.1: Experimental Setup (0.5 page)                        │
│  ─────────────────────────────────────────────────────────────────  │
│  Hardware:                                                          │
│  • CPU: Intel i9-12900K (or equivalent)                           │
│  • RAM: 32GB DDR5                                                   │
│  • Storage: Samsung 980 Pro NVMe SSD                               │
│                                                                      │
│  Software:                                                          │
│  • OS: Windows 11 / Ubuntu 22.04 LTS                              │
│  • .NET SDK 8.0                                                    │
│  • BenchmarkDotNet 0.13.x                                          │
│                                                                      │
│  Baselines:                                                         │
│  • System allocator (malloc/free)                                 │
│  • jemalloc 5.3.0                                                  │
│  • SQLite 3.41.0                                                   │
│  • LMDB 0.9.29                                                     │
│                                                                      │
├─────────────────────────────────────────────────────────────────────┤
│  Section 4.2: Microbenchmarks (1 page)                             │
│  ─────────────────────────────────────────────────────────────────  │
│  [Figure 3: Allocation Latency Comparison]                          │
│  • Bar chart comparing latencies                                    │
│  • X-axis: Operation count                                         │
│  • Y-axis: Latency (ns)                                            │
│                                                                      │
│  [Figure 4: Scalability with Threads]                              │
│  • Line chart                                                      │
│  • X-axis: Thread count                                             │
│  • Y-axis: Throughput (Mops/s)                                     │
│                                                                      │
│  Key findings summarized:                                          │
│  • "VstHelper achieves 2.3x faster allocation..."                 │
│  • "Scales linearly up to 8 threads..."                            │
│                                                                      │
├─────────────────────────────────────────────────────────────────────┤
│  Section 4.3: Macrobenchmarks (1 page)                             │
│  ─────────────────────────────────────────────────────────────────  │
│  [Figure 5: End-to-end Throughput]                                  │
│  • Mixed workload comparison                                       │
│                                                                      │
│  [Figure 6: Latency Distribution]                                   │
│  • CDF plot showing latency percentiles                             │
│                                                                      │
├─────────────────────────────────────────────────────────────────────┤
│  Section 4.4: Discussion (0.5 page)                                │
│  ─────────────────────────────────────────────────────────────────  │
│  • Interpret results                                               │
│  • Address any surprising findings                                 │
│  • Compare with hypotheses                                         │
└─────────────────────────────────────────────────────────────────────┘
```

---

### 3.4.6 Conclusion (0.5 page)

```
STRUCTURE:
┌─────────────────────────────────────────────────────────────────────┐
│  • Restate the problem                                             │
│  • Summarize contributions (3-4 points)                           │
│  • Highlight key results (with numbers)                             │
│  • State implications for the field                                │
│  • Future work directions (2-3 points)                             │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 4. Benchmark & Đo Lường

### 4.1 Setup Chi Tiết

```yaml
# System Configuration Document

hardware:
  cpu:
    model: "Intel Core i9-12900K"
    cores: 16 (8P + 8E)
    threads: 24
    base_clock: 3.2 GHz
    boost_clock: 5.2 GHz
    
  memory:
    capacity: "32 GB"
    type: "DDR5-4800"
    channels: "Dual"
    
  storage:
    primary: "Samsung 980 Pro 1TB NVMe"
    sequential_read: "7000 MB/s"
    sequential_write: "5100 MB/s"

software:
  os: "Windows 11 Pro 22H2"
  dotnet: ".NET 8.0 SDK"
  compiler: "Roslyn 4.x"
  build_config: "Release"
  
environment:
  # Disable background processes
  background_services: "minimal"
  
  # Pin to specific cores for consistency
  cpu_affinity: "all"
  
  # Disable turbo boost for repeatable results
  turbo_boost: "disabled"
  
  # Warm up period
  warmup_iterations: 30
  
  # Measurement iterations  
  measure_iterations: 100
```

### 4.2 Benchmark Cases Chi Tiết

#### 4.2.1 Memory Allocation Benchmark

```csharp
// Case 1: Sequential Allocation
[Params(1000, 10000, 100000, 1000000)]
public int OperationCount { get; set; }

[Benchmark]
public void SequentialAlloc_RentReturn()
{
    for (int i = 0; i < OperationCount; i++)
    {
        var slot = MemoryPool.Rent(256);
        MemoryPool.Return(slot);
    }
}

// Case 2: Concurrent Allocation (multiple threads)
// [Measure with ThreadStatic per thread]

// Case 3: Random Size Allocation
[Params(16, 32, 64, 128, 256, 512, 1024, 2048)]
public int AllocationSize { get; set; }
```

#### 4.2.2 Sorting Benchmark

```csharp
[Params(1000, 10000, 100000, 1000000, 10000000)]
public int ArraySize { get; set; }

[Benchmark(Baseline = true)]
public void SequentialQuickSort()
{
    var entries = GenerateEntries(ArraySize);
    SequentialQuickSort(entries, 0, entries.Length - 1);
}

[Benchmark]
public void ParallelQuickSort()
{
    var entries = GenerateEntries(ArraySize);
    KeyIndexer.Sort(entries);
}

[Benchmark]
public void ArraySort() // Baseline
{
    var entries = GenerateEntries(ArraySize);
    Array.Sort(entries);
}
```

#### 4.2.3 Database Workload Benchmark

```csharp
[ParamsAllValues]
public WorkloadType Workload { get; set; }

// Workload definitions:
// InsertOnly: Sequential inserts
// PointQuery: Random reads
// Mixed: 70% read / 30% write
// UpdateHeavy: 50% read / 50% update

[Benchmark]
public void VstHelper_ProcessWorkload()
{
    foreach (var op in GenerateWorkload(Workload))
    {
        switch (op.Type)
        {
            case OpType.Insert: Insert(op.Key, op.Value); break;
            case OpType.Read: Read(op.Key); break;
            case OpType.Update: Update(op.Key, op.Value); break;
        }
    }
}

// Compare with baselines
[Benchmark]
public void SQLite_ProcessWorkload() { /* SQLite implementation */ }

[Benchmark]
public void LMDB_ProcessWorkload() { /* LMDB implementation */ }
```

### 4.3 Output Format

```json
{
  "benchmark_name": "MemoryAllocator_SequentialAlloc",
  "timestamp": "2026-05-07T10:30:00Z",
  "environment": {
    "cpu": "Intel i9-12900K",
    "os": "Windows 11",
    "dotnet": "8.0.100"
  },
  "results": {
    "operation_count": 1000000,
    "allocators": {
      "system": {
        "mean_ns": 45.2,
        "median_ns": 42.1,
        "stddev_ns": 8.3,
        "p95_ns": 58.0,
        "p99_ns": 72.5
      },
      "vsthelper": {
        "mean_ns": 12.3,
        "median_ns": 11.8,
        "stddev_ns": 2.1,
        "p95_ns": 15.2,
        "p99_ns": 18.7
      }
    },
    "speedup": 3.67,
    "statistical_significance": {
      "p_value": 0.0001,
      "confidence_interval_95": [3.45, 3.89]
    }
  }
}
```

---

## 5. Cấu Trúc Bài Báo

### 5.1 Template cho IEEE/ACM

```latex
\documentclass[conference]{IEEEtran}
% Hoặc cho ACM:
%\documentclass[sigconf]{acmart}

\title[VstHelper: Lock-free Allocator]{VstHelper: A Lock-free Slab Allocator\\for High-Performance Embedded Databases}

\author{
  \IEEEauthorblockN{Your Name}
  \IEEEauthorblockA{University Name\\
    Department\\
    Email: you@email.com}
}

\begin{document}

\begin{abstract}
% 150-250 words
\end{abstract}

\begin{IEEEkeywords}
embedded database, memory allocator, lock-free, parallel sorting
\end{IEEEkeywords}

\section{Introduction}
% 1 page

\section{Background and Related Work}
% 1-1.5 pages

\section{Design}
% 2-3 pages

\section{Implementation}
% 1 page

\section{Evaluation}
% 2-3 pages

\section{Discussion}
% 0.5 page

\section{Conclusion}
% 0.5 page

\section*{Acknowledgment}

\begin{thebibliography}{99}
% 20-30 references
\end{thebibliography}

\end{document}
```

### 5.2 Word Count Guide

| Section | Target Words | Notes |
|---|---|---|
| Abstract | 150-250 | Standalone summary |
| Introduction | 800-1000 | 1 page |
| Background | 1000-1500 | 1-1.5 pages |
| Design | 2000-3000 | 2-3 pages |
| Implementation | 500-800 | 1 page |
| Evaluation | 1500-2500 | 2-3 pages |
| Discussion | 400-600 | 0.5 page |
| Conclusion | 300-500 | 0.5 page |
| **Total** | **7000-10000** | **8-10 pages** |

### 5.3 Figure Guidelines

```
┌─────────────────────────────────────────────────────────────────────┐
│  Figure Requirements                                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Format:                                                             │
│  • PDF or EPS (vector graphics) for print                          │
│  • PNG/SVG for web                                                   │
│  • Minimum 300 DPI                                                  │
│  • Font size: 8-10pt for axis labels                                │
│                                                                      │
│  Style:                                                              │
│  • Use consistent colors (colorblind-friendly palette)            │
│  • Include gridlines where appropriate                              │
│  • Clear axis labels with units                                     │
│  • Legend if multiple series                                         │
│                                                                      │
│  Caption:                                                            │
│  • Start with "Figure X: "                                           │
│  • Be descriptive: "Figure 3: Allocation latency comparison..."    │
│  • 1-2 sentences                                                    │
│                                                                      │
│  References in text:                                                │
│  • "...as shown in Figure 3..."                                    │
│  • "...see Figure 2 for details..."                                 │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 6. Tài Liệu Tham Khảo

### 6.1 Must-Read Papers

#### Memory Allocators

| # | Paper | Year | Why Important |
|---|---|---|---|
| 1 | Berger, E. D., et al. "Hoard: A Lock-Free Memory Allocator" | 2000 | Lock-free allocator foundation |
| 2 | G后者, J. "jemalloc: A General Purpose Allocator" | 2006 | Production-grade allocator |
| 3 | Evans, J. "A Scalable Concurrent malloc(3) implementation for FreeBSD" | 2006 | jemalloc original paper |
| 4 | Serna, F. "The Memory Management in the .NET Runtime" | 2012 | .NET GC internals |
| 5 | Gohariker, M. et al. "TCMalloc: Thread-Caching Malloc" | 2007 | Google allocator |

#### Parallel Sorting

| # | Paper | Year | Why Important |
|---|---|---|---|
| 6 | Musapat et al. "Parallel Quicksort using Fork-Join" | 2011 | Parallel QuickSort analysis |
| 7 | Sanders, J. "Fast Parallel Sorting Under Linux" | 2009 | Practical parallel sorting |
| 8 | Cederman, D. "QuickSplit: A Work-Optimized Parallel Quicksort" | 2013 | Load balancing in QuickSort |

#### Embedded Databases

| # | Paper | Year | Why Important |
|---|---|---|---|
| 9 | Howard, H. et al. "RocksDB Paper" | 2014 | LSM-tree for SSDs |
| 10 | Symas, O. "LMDB: Lightning Memory-Mapped Database" | 2015 | MVCC + B+Tree design |

### 6.2 How to Find More Papers

```
Search Strategies:

1. Google Scholar Alerts
   - Set up alerts for: "lock-free allocator", "slab allocator"
   - Weekly digest to inbox

2. Citation Chaining
   - Start with key paper (e.g., Hoard)
   - Click "Cited by" for newer papers
   - Click "References" for older foundational work

3. Top Venues for This Research
   - USENIX ATC, OSDI, SOSP (systems)
   - VLDB, SIGMOD, ICDE (databases)
   - PPoPP, SPAA (parallel computing)

4. Preprints
   - arXiv: cs.DC (distributed computing)
   - arXiv: cs.PL (programming languages)
```

---

## 7. Công Cụ Hỗ Trợ

### 7.1 Writing Tools

| Tool | Purpose | Alternative |
|---|---|---|
| **LaTeX** | Typesetting (IEEE/ACM format) | Overleaf (online) |
| **Zotero** | Reference management | Mendeley, EndNote |
| **Grammarly** | Grammar checking | LanguageTool |
| **Overleaf** | Collaborative LaTeX | ShareLaTeX |
| **Draw.io** | Architecture diagrams | Lucidchart |

### 7.2 Benchmarking Tools

| Tool | Purpose | Cost |
|---|---|---|
| **BenchmarkDotNet** | .NET performance testing | Free |
| **perf (Linux)** | CPU profiling | Free |
| **Intel VTune** | Performance profiler | Free for personal |
| **dotMemory** | Memory profiling | Free trial |
| **dotTrace** | .NET tracing | Free trial |

### 7.3 Visualization Tools

| Tool | Purpose | Alternative |
|---|---|---|
| **Python + Matplotlib** | Charts/graphs | R + ggplot2 |
| **Google Charts** | Interactive charts | Plotly |
| **Graphviz** | Architecture diagrams | Mermaid |

### 7.4 Project Management

```
┌─────────────────────────────────────────────────────────────────────┐
│  Recommended Setup                                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Code Repository:                                                    │
│  • GitHub Private Repository                                         │
│  • Include benchmark code                                           │
│  • MIT License (for citation)                                        │
│                                                                      │
│  Paper Writing:                                                      │
│  • Overleaf for LaTeX                                               │
│  • Share with advisor                                                │
│                                                                      │
│  Notes:                                                              │
│  • Notion hoặc Obsidian                                             │
│  • Organize by: Papers, Ideas, Results, Writing                     │
│                                                                      │
│  Schedule:                                                          │
│  • Weekly meetings với advisor                                       │
│  • GitHub Issues for task tracking                                   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 8. Timeline

### 8.1 10-Week Plan (Detailed)

```
┌─────────────────────────────────────────────────────────────────────┐
│  Week 1: Literature Review                                          │
├─────────────────────────────────────────────────────────────────────┤
│  Days 1-2: Read 10 foundational papers (Hoard, jemalloc, etc.)      │
│  Days 3-4: Read 5-10 additional papers                              │
│  Days 5-7: Write reading notes, identify research gap               │
│                                                                      │
│  Deliverable: Reading notes + Research gap summary                  │
├─────────────────────────────────────────────────────────────────────┤
│  Week 2: Research Questions & Experiment Design                     │
├─────────────────────────────────────────────────────────────────────┤
│  Days 1-2: Define 3 research questions with hypotheses              │
│  Days 3-4: Design experiment matrix                                  │
│  Days 5-7: Setup benchmark environment (hardware, tools)            │
│                                                                      │
│  Deliverable: RQ document + Experiment plan                         │
├─────────────────────────────────────────────────────────────────────┤
│  Week 3-4: Benchmarking                                              │
├─────────────────────────────────────────────────────────────────────┤
│  Days 1-3: Implement benchmarks in BenchmarkDotNet                   │
│  Days 4-5: Run microbenchmarks (allocation, sorting)                │
│  Days 6-7: Run macrobenchmarks (database workloads)                  │
│                                                                      │
│  Deliverable: Raw benchmark data + Preliminary results              │
├─────────────────────────────────────────────────────────────────────┤
│  Week 5: Analysis & Statistical Rigor                               │
├─────────────────────────────────────────────────────────────────────┤
│  Days 1-2: Statistical analysis of results                          │
│  Days 3-4: Generate figures and tables                               │
│  Days 5-7: Verify reproducibility                                    │
│                                                                      │
│  Deliverable: Analyzed results + Figures ready for paper            │
├─────────────────────────────────────────────────────────────────────┤
│  Week 6-7: First Draft                                               │
├─────────────────────────────────────────────────────────────────────┤
│  Days 1-2: Write Methods & Evaluation sections                       │
│  Days 3-4: Write Design section                                     │
│  Days 5-6: Write Introduction & Related Work                       │
│  Day 7: Write Abstract & Conclusion                                 │
│                                                                      │
│  Deliverable: Complete first draft (8-10 pages)                    │
├─────────────────────────────────────────────────────────────────────┤
│  Week 8: Revision Round 1                                           │
├─────────────────────────────────────────────────────────────────────┤
│  Days 1-4: Address advisor feedback                                  │
│  Days 5-7: Self-review for clarity and technical accuracy           │
│                                                                      │
│  Deliverable: Revised draft                                          │
├─────────────────────────────────────────────────────────────────────┤
│  Week 9: Polish                                                      │
├─────────────────────────────────────────────────────────────────────┤
│  Days 1-2: Grammar, spelling, formatting check                      │
│  Days 3-4: Refine figures and tables                                │
│  Days 5-7: Final proof-reading                                      │
│                                                                      │
│  Deliverable: Polished draft ready for submission                   │
├─────────────────────────────────────────────────────────────────────┤
│  Week 10: Submission                                                 │
├─────────────────────────────────────────────────────────────────────┤
│  Days 1-2: Final format check                                        │
│  Days 3-4: Create supplementary materials (code, data)               │
│  Days 5-7: Submit to target venue                                   │
│                                                                      │
│  Deliverable: Submitted paper                                        │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 9. Checklist Hoàn Thành

### 9.1 Before Writing

- [ ] Read 20+ relevant papers
- [ ] Take detailed notes for each paper
- [ ] Identify unique contribution
- [ ] Define 3 research questions
- [ ] Write hypotheses for each RQ

### 9.2 During Benchmarking

- [ ] Setup reproducible environment
- [ ] Document hardware/software config
- [ ] Run minimum 30 iterations per test
- [ ] Calculate statistical significance (p-value)
- [ ] Generate all figures with proper labels
- [ ] Verify results are repeatable

### 9.3 Writing Checklist

#### Abstract
- [ ] 150-250 words
- [ ] Problem, solution, results, impact
- [ ] Key numbers included
- [ ] Self-contained (understandable without paper)

#### Introduction
- [ ] Hook in first sentence
- [ ] Clear problem statement
- [ ] 3-4 concrete contributions
- [ ] Roadmap included
- [ ] No citations in first paragraph

#### Related Work
- [ ] 15+ citations
- [ ] Organized by theme, not paper-by-paper
- [ ] Clearly state how your work differs
- [ ] Include recent papers (last 5 years)

#### Design
- [ ] Architecture diagram included
- [ ] Key code snippets with explanations
- [ ] Time/space complexity analysis
- [ ] Design decisions justified
- [ ] Limitations acknowledged

#### Evaluation
- [ ] Clear experimental setup
- [ ] All baselines documented
- [ ] Statistical significance reported
- [ ] Results compared to baselines
- [ ] Negative results discussed

#### Writing Quality
- [ ] No grammar/spelling errors
- [ ] Consistent terminology
- [ ] Active voice preferred
- [ ] Paragraphs flow logically
- [ ] Figures have proper captions
- [ ] Tables have proper headers

### 9.4 Final Check Before Submission

- [ ] Follow venue template exactly
- [ ] Check page limits
- [ ] Verify all references are complete
- [ ] Check figure resolution
- [ ] Review author guidelines
- [ ] Create author bio/statement (if required)
- [ ] Check conflict of interest disclosure
- [ ] Upload supplementary materials

---

## 10. Mẫu Email Gửi Advisor

```markdown
Subject: Paper Progress Update - Week [X]

Dear [Advisor Name],

I wanted to share my progress on the VstHelper paper.

## Completed This Week:
- Finished literature review (read 15 papers on lock-free allocators)
- Identified research gap: lack of .NET-specific lock-free allocators
- Designed 3 research questions

## Current Status:
- Benchmarking setup is 80% complete
- Preliminary results suggest 2x speedup in allocation

## Questions for You:
1. Should we focus more on comparison with jemalloc or tcmalloc?
2. Is the parallel sorting contribution strong enough?

## Next Week Plan:
- Complete benchmarking
- Start writing first draft

Please let me know if you'd like to schedule a meeting this week.

Best regards,
[Your Name]
```

---

## 11. Khi Gặp Khó Khăn

### Common Issues & Solutions

| Issue | Solution |
|---|---|
| **Can't find enough related work** | Search broader (embedded systems, real-time OS, IoT databases) |
| **Benchmark results not significant** | Increase sample size, check for outliers, use paired t-test |
| **Paper too long** | Cut related work (10 refs instead of 15), merge paragraphs |
| **Paper too short** | Add more detailed evaluation, more comparisons |
| **Results don't support hypothesis** | Revisit hypothesis, explain unexpected results in Discussion |
| **Advisor wants major changes** | Schedule meeting, clarify expectations, prioritize changes |

### When to Ask for Help

```
┌─────────────────────────────────────────────────────────────────────┐
│  Reach out when:                                                     │
│                                                                      │
│  ✓ 2+ weeks stuck on same problem                                   │
│  ✓ Benchmark producing inconsistent results                         │
│  ✓ Major disagreement with advisor on direction                     │
│  ✓ Need help interpreting statistical results                       │
│  ✓ Deadline is approaching and off track                            │
│                                                                      │
│  Don't wait when:                                                    │
│  ✓ Minor technical issue (solve yourself first)                      │
│  ✓ Writing style feedback (can use Grammarly)                        │
│  ✓ Minor results interpretation (can discuss with peers)           │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 12. Venue Selection Guide

### Top Venues for This Research

| Venue | Acceptance Rate | Focus | Notes |
|---|---|---|---|
| **ACM TOMPECS** | ~30% | Embedded systems, real-time | Good fit for embedded focus |
| **IEEE TPDS** | ~25% | Parallel and distributed | Strong on parallel algorithms |
| **USENIX ATC** | ~15% | Practical systems | High impact, competitive |
| **VLDB** | ~20% | Databases | Top database venue |
| **ACM SIGMOD Record** | ~40% | Database systems | Shorter papers, good for novel ideas |

### Alternative Venues

| Venue | Focus | Notes |
|---|---|---|
| **ICPADS** | Parallel and distributed systems | Regional but respected |
| **ICPP** | Parallel processing | Good for parallel sorting |
| **HPCC** | High performance computing | Broad HPC focus |
| **SC** | Supercomputing | Very competitive |
| **Springer JSC** | Journal, less time pressure | Good if rejected from conferences |

---

## 13. Sau Khi Nộp

### 8-12 Weeks: Review Process

```
┌─────────────────────────────────────────────────────────────────────┐
│  Review Timeline (typical)                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Week 0: Submit                                                      │
│       │                                                              │
│       ▼                                                              │
│  Week 1-4: Area Chair assigns reviewers                              │
│       │                                                              │
│       ▼                                                              │
│  Week 4-8: Reviewers read and write comments                         │
│       │                                                              │
│       ▼                                                              │
│  Week 8-10: Rebuttal period (if allowed)                             │
│       │                                                              │
│       ▼                                                              │
│  Week 10-12: Decision notification                                   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Possible Outcomes

| Decision | Action |
|---|---|
| **Accept** | Celebrate! Prepare camera-ready version |
| **Minor Revision** | Address reviewer comments, typically 1-2 weeks |
| **Major Revision** | Significant changes required, new review cycle |
| **Reject** | Learn from feedback, submit to another venue |

### Camera-Ready Checklist

- [ ] Address all reviewer comments
- [ ] Format according to camera-ready guidelines
- [ ] Sign copyright form
- [ ] Upload final PDF
- [ ] Prepare presentation (if conference)
- [ ] Submit supplementary materials (code, data)

---

> **Ghi chú cuối cùng**: Quy trình này là hướng dẫn, không phải công thức cứng nhắc. Điều chỉnh theo yêu cầu cụ thể của advisor, deadline, và đặc thù của đề tài nghiên cứu.

---

*Tài liệu được tạo cho dự án VstHelper - Embedded NoSQL Database Engine*
*Phiên bản: 1.0 - Ngày: Tháng 5, 2026*
