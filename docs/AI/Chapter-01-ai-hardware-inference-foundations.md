# Chapter 1 — AI Hardware & Inference Foundations

> **A layman-first guide to understanding GPU, VRAM, bandwidth, PCIe, NVIDIA drivers, CUDA, vLLM, tokens, KV cache, batching, and where Kubernetes eventually fits.**

---

## 📚 Table of Contents

1. [What is AI inference?](#1--what-is-ai-inference)
2. [CPU vs GPU](#2--cpu-vs-gpu)
3. [GPU, VRAM, and memory bandwidth](#3--gpu-vram-and-memory-bandwidth)
4. [Where does 272 GB/s come from?](#4--where-does-272-gbs-come-from)
5. [`nvidia-smi`](#5--nvidia-smi)
6. [Reading `nvidia-smi`](#6--reading-nvidia-smi)
7. [CPU ↔ RAM bandwidth](#7--cpu--ram-bandwidth)
8. [PCIe — the bridge to the GPU](#8--pcie--the-bridge-to-the-gpu)
9. [Why CPU offloading can be slower](#9--why-cpu-offloading-can-be-slower)
10. [Integrated GPU vs discrete GPU](#10--integrated-gpu-vs-discrete-gpu)
11. [Driver vs CUDA](#11--driver-vs-cuda)
12. [What is inside CUDA?](#12--what-is-inside-cuda)
13. [What is vLLM?](#13--what-is-vllm)
14. [Model weights](#14--model-weights)
15. [Quantization](#15--quantization)
16. [Tokens](#16--tokens)
17. [Prefill vs decode](#17--prefill-vs-decode)
18. [KV cache](#18--kv-cache)
19. [Prefix caching](#19--prefix-caching)
20. [Batching](#20--batching)
21. [Why memory bandwidth matters for LLMs](#21--why-memory-bandwidth-matters-for-llms)
22. [Multi-GPU](#22--multi-gpu)
23. [Where Kubernetes fits](#23--where-kubernetes-fits)
24. [My lab machine](#24--my-lab-machine)
25. [One complete mental model](#25--one-complete-mental-model)
26. [Quick revision card](#26--quick-revision-card)

---

# 1. 🧠 What is AI inference?

**Inference** means using an already-trained AI model to answer a new request.

```text
You type a question
        ↓
The model processes the prompt
        ↓
The model predicts the next token
        ↓
Then the next token
        ↓
Then the next token
        ↓
You receive the answer
```

> [!IMPORTANT]
> **Punch line:**  
> **Training = the model learns.**  
> **Inference = the trained model uses what it learned to answer you.**

The model may contain billions of learned numbers called **parameters** or **weights**. During inference, the computer repeatedly reads those weights and performs huge amounts of math.

That is why hardware matters.

---

# 2. ⚙️ CPU vs GPU

A **CPU — Central Processing Unit** is a flexible general-purpose processor.

It runs things like:

- Windows or Linux
- browsers
- databases
- applications
- networking
- storage logic

A **GPU — Graphics Processing Unit** is built to perform **many similar calculations in parallel**.

> [!TIP]
> **Layman story — Workers**
>
> Think of a CPU as a **small team of very skilled workers** who can do many different jobs.
>
> Think of a GPU as a **huge team of workers** who are extremely good at doing the same kind of math at the same time.

Large language models perform enormous amounts of matrix math, so GPUs are very useful.

> [!IMPORTANT]
> **Punch line:**  
> **CPU = flexible general-purpose work**  
> **GPU = massive parallel math**

This does **not** mean “GPU is better than CPU.” They are designed for different jobs.

---

# 3. 🧱 GPU, VRAM, and memory bandwidth

These three concepts are different:

| Concept | Simple meaning |
|---|---|
| **GPU compute** | How much math the GPU can perform |
| **VRAM capacity** | How much data the GPU can keep nearby |
| **Memory bandwidth** | How fast data moves between VRAM and the GPU |

> [!TIP]
> **Layman story — The kitchen**
>
> - **GPU = chefs**
> - **VRAM = kitchen counter**
> - **Memory bandwidth = how fast ingredients reach the chefs**
>
> Great chefs are not enough if the counter is tiny or the ingredients arrive too slowly.

### What is VRAM?

**VRAM — Video Random Access Memory** is high-speed memory located close to a discrete GPU.

During LLM inference, VRAM commonly holds:

```text
VRAM
│
├── Model weights
├── KV cache
└── Temporary tensors / working data
```

For your RTX 4060:

```text
VRAM capacity  ≈ 8 GB
VRAM bandwidth ≈ 272 GB/s theoretical
```

> [!IMPORTANT]
> **Punch line:**  
> **Capacity = How much can I hold?**  
> **Bandwidth = How fast can I move it?**

---

# 4. 🛣️ Where does 272 GB/s come from?

It is easy to hear “VRAM is fast because it is physically close to the GPU.”

That is **partly true**, but distance is **not how we calculate the bandwidth number**.

For the RTX 4060:

```text
GDDR6 effective data rate = 17 Gbit/s per pin
Memory bus width          = 128 bits
```

Calculation:

```text
17 Gbit/s × 128 bits = 2176 Gbit/s

2176 ÷ 8 = 272 GB/s
```

We divide by 8 because:

```text
8 bits = 1 byte
```

> [!TIP]
> **Layman story — Highway**
>
> - **Physical distance** = how close the warehouse is
> - **Bus width** = number of highway lanes
> - **Memory speed** = how fast traffic moves
> - **Bandwidth** = total cargo moved every second

> [!IMPORTANT]
> **Punch line:**  
> The short physical distance helps engineers build a fast connection, but **bandwidth comes mainly from memory data rate × bus width**.

---

# 5. 🔍 `nvidia-smi`

`nvidia-smi` stands for:

**NVIDIA System Management Interface**

Run:

```bash
nvidia-smi
```

It shows information such as:

- GPU model
- NVIDIA driver version
- CUDA compatibility reported by the driver
- VRAM usage
- GPU utilization
- temperature
- power
- GPU processes

> [!IMPORTANT]
> **Memory line:**  
> `nvidia-smi` = **“Show me the NVIDIA GPU status.”**

### Is it only for NVIDIA?

**Yes.** It is an NVIDIA-specific tool.

### Windows only?

**No.** It is commonly used on both Windows and Linux.

### Does Windows/Linux install it automatically?

**No.** It normally comes with the NVIDIA driver/software package.

> [!NOTE]
> If `nvidia-smi` works, that is strong evidence that the **OS + NVIDIA driver can communicate with the GPU**.

But it does **not** prove that:

- a container can use the GPU
- Kubernetes can schedule the GPU
- PyTorch can use the GPU
- vLLM can use the GPU

Those are higher layers.

---

# 6. 📊 Reading `nvidia-smi`

Your machine showed roughly:

```text
GPU:            NVIDIA GeForce RTX 4060
Driver:         560.94
CUDA Version:   12.6
VRAM:           ~8188 MiB
```

### Memory-Usage

```text
0 MiB / 8188 MiB
```

means:

```text
Used VRAM / Total VRAM
```

### GPU-Util

```text
GPU-Util = 0%
```

means the GPU was basically idle **at that moment**.

It does **not** mean the GPU is slow or broken.

### `nvidia-smi dmon -s u`

```bash
nvidia-smi dmon -s u
```

Example:

```text
gpu   sm   mem
0      0     0
```

For a beginner:

```text
sm  = GPU compute activity
mem = GPU memory activity
```

> [!CAUTION]
> **Common trap:**  
> `mem = 0%` does **not** mean your GPU has 0 GB/s bandwidth.  
> It means the memory subsystem was not busy during that sample.

### CUDA Version trap

If `nvidia-smi` says:

```text
CUDA Version: 12.6
```

that does **not necessarily mean the full CUDA Toolkit 12.6 is installed**.

> [!IMPORTANT]
> **Punch line:**  
> `nvidia-smi` mainly tells us about the **driver and GPU visibility**.  
> It does not prove the whole CUDA development/toolkit stack is installed.

---

# 7. 🧠 CPU ↔ RAM bandwidth

Your machine has:

```text
CPU: AMD Ryzen 9 9950X
RAM: 64 GB total
     2 × 32 GB G.SKILL
Configured speed: DDR5-4800
```

The rough theoretical maximum for dual-channel DDR5-4800 is:

```text
One channel:
4800 MT/s × 8 bytes ≈ 38.4 GB/s

Two channels:
38.4 × 2 ≈ 76.8 GB/s theoretical
```

Your measured Windows result was:

```text
Memory Performance: 52054.72 MB/s
```

or about:

```text
52.1 GB/s measured
```

> [!IMPORTANT]
> **Punch line:**  
> **Theoretical bandwidth = hardware ceiling**  
> **Measured bandwidth = what a real test actually achieved**

Real results are lower because of timings, overhead, controller behavior, and workload effects.

---

# 8. 🌉 PCIe — the bridge to the GPU

Now we connect the CPU/RAM world to the GPU/VRAM world.

```text
CPU / RAM
    ↓
   PCIe
    ↓
GPU / VRAM
```

**PCIe — Peripheral Component Interconnect Express** is a high-speed general-purpose connection used by devices such as:

- GPUs
- NVMe SSDs
- network cards
- accelerators

PCIe has:

```text
Generation = speed capability of each lane
Lane count = how many lanes exist
```

Your RTX 4060 reported:

```text
PCIe Gen 4
x8 lanes
```

So:

```text
PCIe 4.0 x8
≈ 15.75 GB/s theoretical per direction
```

> [!TIP]
> **Layman story — Highway**
>
> - **PCIe generation = how fast each lane can carry traffic**
> - **x8 / x16 = how many lanes the highway has**

Now compare your three paths:

| Path | Approximate bandwidth |
|---|---:|
| CPU ↔ RAM | **52 GB/s measured** |
| RAM ↔ RTX 4060 over PCIe 4.0 x8 | **15.75 GB/s theoretical** |
| RTX 4060 ↔ VRAM | **272 GB/s theoretical** |

> [!IMPORTANT]
> **Major AI lesson:**  
> Keep frequently used model data in **GPU-local VRAM** whenever possible.

---

# 9. 🐢 Why CPU offloading can be slower

Suppose a model is too large for your 8 GB VRAM.

One option is:

```text
Model
│
├── Part in VRAM
└── Part in system RAM
```

But data stored in RAM must cross PCIe:

```text
Fast:
GPU ↔ VRAM

Slower:
GPU ↔ PCIe ↔ RAM
```

> [!TIP]
> **Layman story — Chef and pantry**
>
> VRAM is the **counter beside the chef**.
>
> System RAM is the **pantry in another room**.
>
> The chef *can* walk to the pantry, but doing it repeatedly slows the work down.

Alternatives include:

- smaller model
- quantized model
- GPU with more VRAM
- CPU offloading
- multiple GPUs

---

# 10. 🖥️ Integrated GPU vs discrete GPU

Your PC has two GPUs:

```text
1. AMD Radeon Graphics integrated into Ryzen 9 9950X
2. NVIDIA GeForce RTX 4060 discrete GPU
```

### Integrated GPU — iGPU

```text
AMD iGPU
   ↕
System DDR5 RAM
```

It shares the system-memory architecture.

### Discrete GPU — dGPU

```text
RTX 4060
   ↕
Dedicated GDDR6 VRAM
```

It has its own local high-speed memory.

> [!IMPORTANT]
> **Punch line:**  
> **iGPU = built into the CPU/platform and usually shares RAM**  
> **dGPU = separate GPU with dedicated VRAM**

### Can AMD + NVIDIA memory simply be combined?

No.

You cannot normally do:

```text
AMD memory + NVIDIA 8 GB VRAM
= one larger GPU
```

They are separate devices with separate compute ecosystems.

```text
NVIDIA → CUDA
AMD    → ROCm / AMD stack
```

---

# 11. 🧩 Driver vs CUDA

This distinction is foundational.

```text
AI Application
      ↓
     CUDA
      ↓
NVIDIA Driver
      ↓
 NVIDIA GPU
```

### NVIDIA Driver

The driver lets the operating system **recognize, control, and communicate with the NVIDIA GPU**.

### CUDA

**CUDA — Compute Unified Device Architecture** is NVIDIA's software platform for running general-purpose compute workloads on NVIDIA GPUs.

> [!IMPORTANT]
> **Best memory line in this chapter:**  
> **DRIVER = Can I talk to the GPU?**  
> **CUDA = Can I compute on the GPU?**

> [!TIP]
> **Layman story — Language**
>
> - **GPU = worker**
> - **Driver = translator/controller that lets the OS communicate with the worker**
> - **CUDA = the work-instruction system used to give that worker compute jobs**

---

# 12. 🧰 What is inside CUDA?

CUDA is not just one program.

```text
CUDA
│
├── Programming model
│   └── How developers describe parallel GPU work
│
├── Runtime
│   └── Helps applications launch and manage GPU operations
│
├── Compiler
│   └── Converts CUDA code into GPU-executable code
│
├── Libraries
│   └── Pre-built optimized math and AI operations
│
└── GPU development tools
    └── Profiling, debugging, and performance analysis
```

> [!IMPORTANT]
> **Infrastructure lesson:**  
> You do **not** need to become a CUDA programmer to become strong in AI infrastructure.
>
> You need to understand **where CUDA sits, what depends on it, and how to troubleshoot the stack**.

---

# 13. 🚀 What is vLLM?

This is where many people get confused.

First:

> [!IMPORTANT]
> **The model is the brain. vLLM is not the model.**

A simple stack:

```text
User request
     ↓
    vLLM
     ↓
PyTorch / optimized GPU kernels
     ↓
    CUDA
     ↓
NVIDIA Driver
     ↓
NVIDIA GPU
     ↕
    VRAM
```

### What problem does vLLM solve?

One user is easy:

```text
One user
   ↓
Python program
   ↓
Model
   ↓
GPU
   ↓
Answer
```

But imagine 100 users:

```text
User 1   ─┐
User 2   ─┤
User 3   ─┤
...       ├──→ ?
User 100 ─┘
```

Now someone must manage:

- which requests run together
- how to keep the GPU busy
- how much VRAM each request uses
- KV cache
- batching
- returning tokens to the right user
- serving the model through an API

That is where vLLM comes in.

> [!TIP]
> **Layman story — Restaurant**
>
> - **Model = chef**
> - **GPU = kitchen equipment**
> - **VRAM = kitchen counter**
> - **vLLM = restaurant manager**
>
> The manager does not make the chef smarter.  
> The manager makes the restaurant **serve many customers efficiently**.

### Without a serving engine

```text
Customer 1 arrives
      ↓
Chef finishes entire order
      ↓
Customer 2 arrives
      ↓
Chef finishes entire order
```

### With vLLM

```text
Many customer requests
        ↓
       vLLM
        ↓
organizes inference work
        ↓
keeps GPU productively busy
        ↓
returns each result correctly
```

> [!IMPORTANT]
> **Punch line:**  
> **vLLM is an inference-serving engine that helps run LLMs efficiently, especially for many concurrent requests.**

### Where PyTorch fits

| Layer | Job |
|---|---|
| **vLLM** | Inference-serving engine |
| **PyTorch / optimized kernels** | Model and tensor computation |
| **CUDA** | NVIDIA GPU compute platform |
| **NVIDIA Driver** | Controls/communicates with GPU |
| **GPU** | Performs calculations |
| **VRAM** | Holds weights, KV cache, working data |

### vLLM and Kubernetes

```text
Kubernetes = WHERE the workload runs
vLLM       = HOW inference requests are served efficiently
```

Example:

```text
Users
  ↓
Service / Load Balancer
  ↓
Kubernetes
  ↓
vLLM Pod
  ↓
Model
  ↓
CUDA
  ↓
GPU ↔ VRAM
```

Kubernetes may decide:

> “Put this vLLM Pod on a node with an NVIDIA GPU.”

vLLM then decides:

> “How do I efficiently serve all these inference requests on that GPU?”

---

# 14. 🧠 Model weights

A trained neural network contains learned numbers called **weights**.

A **7B** model has roughly:

```text
7 billion parameters
```

If each parameter takes 2 bytes:

```text
7 billion × 2 bytes ≈ 14 GB
```

A 70B model:

```text
70 billion × 2 bytes ≈ 140 GB
```

These are simplified **weight-only** estimates.

Real inference also needs memory for:

- KV cache
- temporary tensors
- runtime buffers
- allocator overhead

> [!IMPORTANT]
> **Punch line:**  
> A model that looks like “14 GB of weights” may need **more than 14 GB of VRAM** in real operation.

---

# 15. 📦 Quantization

Quantization stores model numbers using fewer bits.

```text
Higher precision
      ↓
More memory

Lower precision / quantization
      ↓
Less memory
```

You may hear:

```text
FP16
BF16
INT8
INT4
4-bit
```

> [!TIP]
> **Layman story — Packing a suitcase**
>
> Full-precision values are like packing everything in large boxes.
>
> Quantization is like packing the same trip into smaller containers so more fits in the suitcase.

> [!IMPORTANT]
> **Punch line:**  
> Quantization can make a model **smaller and easier to serve**, but there can be quality and performance trade-offs.

---

# 16. 🔤 Tokens

LLMs do not usually generate an entire sentence at once.

They generate **tokens**.

A token may be:

- a whole word
- part of a word
- punctuation
- another text unit

Very simplified:

```text
Prompt
  ↓
Token 1
  ↓
Token 2
  ↓
Token 3
  ↓
...
```

> [!IMPORTANT]
> **Punch line:**  
> LLM inference is **token-by-token generation**, not “write the whole paragraph instantly.”

---

# 17. ⏱️ Prefill vs decode

LLM inference has two important phases.

### Prefill

The model processes the prompt.

```text
Entire prompt
     ↓
Model processes input tokens
     ↓
Attention state created
     ↓
First output token becomes possible
```

A common metric:

```text
TTFT = Time To First Token
```

### Decode

Then the model generates the answer token by token:

```text
Token 1
  ↓
Token 2
  ↓
Token 3
```

A common metric:

```text
TPOT = Time Per Output Token
```

> [!IMPORTANT]
> **Memory line:**  
> **Prefill = understand/process the prompt**  
> **Decode = generate the answer token by token**

---

# 18. 🗃️ KV cache

During inference, transformers calculate **keys** and **values** for previous tokens.

Instead of recomputing them again for every new token, the system stores them in the **KV cache**.

```text
Conversation tokens
        ↓
Keys + Values calculated
        ↓
Stored in KV cache
        ↓
Reused for later tokens
```

This speeds up generation but consumes VRAM.

```text
VRAM
│
├── Model weights
├── KV cache
└── Runtime / temporary data
```

> [!TIP]
> **Layman story — Notes on a desk**
>
> Imagine reading a long case file.
>
> Instead of rereading the entire file every time you answer the next question, you keep useful notes beside you.
>
> Those notes save time — but they take desk space.

> [!IMPORTANT]
> **Production lesson:**  
> A model may fit in VRAM for one short request but run out of memory with **many users or long conversations**, because KV cache grows.

---

# 19. ♻️ Prefix caching

Many users may send the same beginning of a prompt.

Example:

```text
"You are the company's customer-support assistant..."
```

If that exact prefix was already processed, an inference engine may reuse cached state.

```text
Common prefix
    ↓
Process once
    ↓
Cache reusable state
    ↓
Reuse later
```

> [!IMPORTANT]
> **Punch line:**  
> Prefix caching avoids doing the **same prefill work again and again**.

---

# 20. 📚 Batching

A GPU is built for parallel work.

Instead of handling requests completely one by one:

```text
Request A
then B
then C
```

an inference engine can organize work together:

```text
Request A ─┐
Request B ─┼──→ batch ──→ GPU
Request C ─┘
```

This can improve:

```text
Throughput = total work processed over time
```

Examples:

- tokens/second
- requests/second

But there is a trade-off.

Waiting for larger batches may improve throughput while increasing individual request latency.

> [!IMPORTANT]
> **Punch line:**  
> Production inference is a balancing act between **throughput, latency, VRAM usage, and concurrency**.

---

# 21. 🚚 Why memory bandwidth matters for LLMs

During inference, the GPU repeatedly reads model data from VRAM.

Return to the kitchen:

```text
GPU compute = chefs
VRAM        = nearby ingredients
Bandwidth   = conveyor belt
```

Even powerful chefs wait if ingredients arrive too slowly.

That is why GPU comparisons should not use VRAM capacity alone.

Two GPUs can both have 24 GB of VRAM but have very different:

- compute capability
- memory bandwidth
- memory type
- interconnect
- supported precision
- power envelope

> [!IMPORTANT]
> **Punch line:**  
> **VRAM size tells you how much fits. Bandwidth helps determine how quickly the GPU can consume it.**

---

# 22. 🔗 Multi-GPU

If one GPU is too small, a model can sometimes be split across multiple GPUs.

```text
Large model
   │
   ├── GPU 1
   └── GPU 2
```

But now a new problem appears:

```text
GPU 1 ↔ GPU 2
```

The GPUs must communicate.

Possible communication paths include:

- PCIe
- NVLink
- NVSwitch
- network interconnects in multi-node systems

> [!IMPORTANT]
> **Punch line:**  
> **Single-GPU question:** Do I have enough VRAM?  
> **Multi-GPU question:** How do I split the model *and* how fast can the GPUs communicate?

Later this leads into concepts such as tensor parallelism and multi-node inference.

---

# 23. ☸️ Where Kubernetes fits

Kubernetes does **not** make the GPU faster.

Its job is orchestration.

A future GPU Kubernetes stack looks roughly like:

```text
Kubernetes Pod
      ↓
requests nvidia.com/gpu
      ↓
Kubernetes Scheduler
      ↓
GPU-capable worker node
      ↓
NVIDIA Device Plugin / GPU Operator
      ↓
NVIDIA Container Toolkit
      ↓
vLLM / PyTorch / CUDA
      ↓
NVIDIA Driver
      ↓
GPU + VRAM
```

> [!IMPORTANT]
> **Punch line:**  
> Kubernetes sits **above** the hardware and driver layers.

If `nvidia-smi` fails on the Linux node, starting with Kubernetes scheduling would be the wrong troubleshooting layer.

### Troubleshooting order

```text
Hardware visible?
      ↓
Driver healthy?
      ↓
Container can access GPU?
      ↓
Kubernetes advertises GPU?
      ↓
Pod requests GPU?
      ↓
CUDA/framework sees GPU?
      ↓
vLLM healthy?
```

This is the same layer-by-layer troubleshooting approach used in networking.

---

# 24. 🖥️ My lab machine

Current hardware:

```text
AMD Ryzen 9 9950X
        │
   16 CPU cores
        │
        ▼
64 GB DDR5 RAM
DDR5-4800 configured
~52 GB/s measured
```

Integrated graphics:

```text
AMD Radeon iGPU
      ↕
shares system-memory architecture
```

Discrete AI GPU:

```text
CPU / System RAM
      │
PCIe 4.0 x8
~15.75 GB/s theoretical
      │
      ▼
NVIDIA RTX 4060
      │
~272 GB/s theoretical
      │
      ▼
~8 GB GDDR6 VRAM
```

### Three bandwidth paths to remember

| Path | Bandwidth |
|---|---:|
| **CPU ↔ RAM** | ~52 GB/s measured |
| **RAM ↔ NVIDIA GPU** | ~15.75 GB/s theoretical |
| **NVIDIA GPU ↔ VRAM** | ~272 GB/s theoretical |

> [!IMPORTANT]
> These three paths explain a surprising amount of AI-inference behavior.

---

# 25. 🧭 One complete mental model

### Hardware/data path

```text
MODEL ON DISK
     ↓
SYSTEM RAM
     ↓
PCIe
     ↓
GPU VRAM
     ↓
GPU COMPUTE
```

### Inference software path

```text
User Request
     ↓
vLLM
     ↓
PyTorch / optimized GPU kernels
     ↓
CUDA
     ↓
NVIDIA Driver
     ↓
GPU
     ↕
VRAM
     ↓
Tokens generated
```

### Later, Kubernetes wraps around it

```text
                 Kubernetes
                     │
              schedules vLLM Pod
                     │
                     ▼
vLLM → PyTorch → CUDA → Driver → GPU ↔ VRAM
```

---

# 26. 📝 Quick revision card

> [!IMPORTANT]
> ### If you remember only these lines:
>
> **GPU** = does massively parallel math  
> **VRAM** = fast local GPU memory  
> **Bandwidth** = how fast data moves  
> **PCIe** = bridge between CPU/RAM and GPU  
> **Driver** = lets OS talk to NVIDIA GPU  
> **CUDA** = lets compute software use NVIDIA GPU  
> **Model weights** = learned numbers in the neural network  
> **vLLM** = efficiently serves LLM inference requests  
> **KV cache** = saves previous attention state but consumes VRAM  
> **Batching** = combines work to keep GPU busy  
> **Kubernetes** = decides where the workload runs

---

## 🎯 Final architecture memory line

```text
Kubernetes = WHERE the workload runs
vLLM       = HOW inference is served efficiently
CUDA       = HOW software computes on NVIDIA GPU
Driver     = HOW software controls/talks to the GPU
GPU        = DOES the math
VRAM       = HOLDS the model and active working data
```

---

> [!NOTE]
> This chapter intentionally uses simple language first. Deeper topics such as CUDA kernels, tensor parallelism, NVLink, GPU Operator, continuous batching, PagedAttention internals, and production vLLM architecture can be added in later chapters.
