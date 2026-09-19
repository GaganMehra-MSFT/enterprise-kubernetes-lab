Chapter 1 — AI Hardware and Inference Foundations

A beginner-friendly explanation of how CPU, RAM, GPU, VRAM, PCIe, NVIDIA drivers, CUDA, and LLM inference fit together.

1. Start with the big picture

When we say AI inference, we mean using an already-trained model to answer a new request.

For example:

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

The model may contain billions of learned numbers called parameters or weights. During inference, the computer must repeatedly read those weights and perform a huge number of mathematical operations. This is why hardware matters.

A useful first mental picture is:

CPU ── RAM

GPU ── VRAM

The CPU has normal system memory. The GPU has fast local memory called VRAM. For modern AI workloads, both the amount of VRAM and the speed at which the GPU can read it are extremely important.

2. CPU versus GPU — why use a GPU for AI?

A CPU — Central Processing Unit is designed to be very flexible. It runs the operating system, applications, databases, browsers, and thousands of different kinds of instructions.

A GPU — Graphics Processing Unit was originally designed to perform many similar calculations at the same time for graphics. That same parallel design happens to be extremely useful for AI.

A simple analogy is:

CPU = a small group of very skilled workers
GPU = a huge group of workers doing similar math in parallel

Large language models perform enormous amounts of matrix multiplication and related operations. GPUs are very good at this kind of parallel work.

This does not mean a GPU is simply "better than a CPU." They are designed for different jobs. The CPU still coordinates the system, prepares work, handles networking and storage, and performs many tasks that are not good GPU workloads.

3. GPU, VRAM, and memory bandwidth

Three terms should always be kept separate:

GPU compute       = how much calculation the GPU can perform
VRAM capacity     = how much data the GPU can keep locally
Memory bandwidth  = how quickly data can move between VRAM and GPU

Think of a kitchen:

GPU              = chefs
VRAM             = the kitchen counter
Memory bandwidth = how quickly ingredients reach the chefs

You can have excellent chefs, but if the counter is too small, the ingredients do not fit. You can also have a huge counter, but if ingredients reach the chefs too slowly, the chefs spend time waiting.

What is VRAM?

VRAM — Video Random Access Memory is the high-speed memory located close to a discrete GPU. The word "video" comes from the history of GPUs, but modern AI workloads use VRAM for much more than graphics.

During LLM inference, VRAM is commonly used for:

VRAM
│
├── Model weights
├── KV cache
└── Temporary tensors / working data

The model weights are usually the largest fixed piece. KV cache grows with active requests and context length and becomes very important when serving many users.

Capacity versus bandwidth

These are different measurements.

For the RTX 4060 used in this lab:

VRAM capacity  ≈ 8 GB
VRAM bandwidth ≈ 272 GB/s theoretical

The first number answers "How much can I store?"

The second answers "How quickly can I move it?"

4. Where does 272 GB/s come from?

It is tempting to say that VRAM is fast simply because it sits physically close to the GPU. Physical closeness certainly helps with signal quality, latency, power, and the ability to build a very fast interface, but distance is not how bandwidth is calculated.

For the RTX 4060, the approximate memory specifications are:

GDDR6 effective data rate = 17 Gbit/s per pin
Memory bus width          = 128 bits

The calculation is:

17 Gbit/s × 128 bits = 2176 Gbit/s

2176 ÷ 8 = 272 GB/s

Why divide by 8? Because there are 8 bits in one byte.

A useful highway analogy is:

Physical distance = how close the warehouse is
Bus width         = number of highway lanes
Memory data rate  = speed/carrying rate of each lane
Bandwidth         = total cargo moved per second

So the short physical path helps engineers build the fast memory system, but memory data rate × bus width gives us the bandwidth number.

5. nvidia-smi — the first NVIDIA GPU health check

nvidia-smi stands for:

NVIDIA System Management Interface

It is NVIDIA's command-line utility for inspecting NVIDIA GPUs.

nvidia-smi

A simple memory line is:

nvidia-smi = "Show me the NVIDIA GPU status."

It can show information such as:

GPU model
Driver version
CUDA compatibility reported by the driver
VRAM usage
GPU utilization
Temperature
Power
Processes using the GPU

Is nvidia-smi only for NVIDIA?

Yes. It is an NVIDIA tool. AMD and Intel have their own GPU tooling.

Is it Windows-only?

No. It is commonly used on both Windows and Linux. That matters because production Kubernetes GPU worker nodes are often Linux servers.

Does it come preinstalled with the operating system?

No. It normally arrives with the NVIDIA driver/software package rather than being a standard Windows or Linux command.

What does this prove?

If nvidia-smi works, it is strong evidence that the operating system and NVIDIA driver can communicate with the NVIDIA GPU.

It does not yet prove that Kubernetes, a container, PyTorch, or vLLM can use the GPU.

Think in layers:

Can Windows/Linux see the GPU?
        ↓
      Driver
        ↓
Can a container see the GPU?
        ↓
 NVIDIA Container Toolkit
        ↓
Can Kubernetes schedule the GPU?
        ↓
 NVIDIA Device Plugin / GPU Operator
        ↓
Can the AI framework use it?
        ↓
      CUDA

6. Understanding the important nvidia-smi fields

On the lab machine, nvidia-smi identified:

NVIDIA GeForce RTX 4060
Driver Version: 560.94
CUDA Version: 12.6
Memory: ~8188 MiB

Memory-Usage

If the output shows:

0 MiB / 8188 MiB

read it as:

Used VRAM / Total VRAM

So this GPU has roughly 8 GB of dedicated VRAM.

GPU-Util

GPU-Util tells us how busy the GPU compute engines are at that moment.

GPU-Util = 0%

means the GPU was essentially idle at that instant. It does not mean the GPU is slow or broken.

nvidia-smi dmon -s u

This command continuously samples utilization:

nvidia-smi dmon -s u

An idle system may show:

gpu   sm   mem
0      0     0

For a beginner mental model:

sm  = GPU compute activity
mem = GPU memory subsystem activity

mem = 0% does not mean the GPU has 0 GB/s of bandwidth. It means the memory subsystem was not busy during that sample.

Important CUDA-version trap

If nvidia-smi says:

CUDA Version: 12.6

that does not necessarily mean CUDA Toolkit 12.6 is installed.

It primarily tells us the CUDA compatibility level supported by that NVIDIA driver.

The driver and the CUDA Toolkit are separate concepts.

7. System RAM and CPU memory bandwidth

The lab machine has:

CPU: AMD Ryzen 9 9950X
RAM: 64 GB total
     2 × 32 GB G.SKILL
Configured memory rate: DDR5-4800

The two DIMMs are installed across Channel A and Channel B, giving a dual-channel memory configuration.

A rough theoretical calculation for DDR5-4800 is:

One channel:
4800 MT/s × 8 bytes ≈ 38.4 GB/s

Two channels:
38.4 × 2 ≈ 76.8 GB/s theoretical

The Windows System Assessment Tool measured:

Memory Performance: 52054.72 MB/s

which is about:

52.1 GB/s measured

This is lower than the 76.8 GB/s theoretical maximum, which is normal. Real systems have protocol overhead, timings, controller behavior, workload effects, and other inefficiencies.

The lesson is important:

Theoretical bandwidth is the hardware ceiling. Measured bandwidth is what a real workload actually achieved.

8. PCIe — the bridge between CPU/RAM and GPU/VRAM

The next path is different from CPU-to-RAM and GPU-to-VRAM.

CPU / RAM
    ↓
   PCIe
    ↓
GPU / VRAM

PCIe — Peripheral Component Interconnect Express is a general-purpose high-speed interconnect used by devices such as GPUs, NVMe drives, network adapters, and accelerators.

It has both generations and lane counts.

PCIe generation = speed/capability of each lane
x8, x16, etc.   = number of lanes

A highway analogy works well:

PCIe generation = how capable each lane is
Lane count       = how many lanes exist

The RTX 4060 in this lab reported:

PCIe generation: 4
Link width:      x8

So the link is:

PCIe 4.0 x8

PCIe 4.0 provides roughly 1.97 GB/s per lane in each direction, giving approximately:

1.97 × 8 ≈ 15.75 GB/s theoretical per direction

Now compare the three paths:

CPU ↔ RAM             ≈ 52 GB/s measured
RAM ↔ RTX 4060        ≈ 15.75 GB/s theoretical over PCIe 4.0 x8
RTX 4060 ↔ local VRAM ≈ 272 GB/s theoretical

This comparison explains a major AI architecture principle:

Keep frequently used model data in GPU-local VRAM whenever possible.

If the GPU must repeatedly fetch large amounts of data from system RAM across PCIe, the PCIe path can become a bottleneck.

9. Why CPU offloading can be slower

Suppose a model needs more memory than the GPU has.

You might keep some data in VRAM and some in system RAM:

Model
│
├── Part in VRAM
└── Part in system RAM

The GPU can read local VRAM through its very fast memory interface, but data in system RAM must travel over PCIe.

Conceptually:

Fast path:
GPU ↔ VRAM

Slower path:
GPU ↔ PCIe ↔ system RAM

This is one reason CPU offloading can make an otherwise too-large model runnable while reducing performance.

Alternatives when a model does not fit include using a smaller model, quantizing the model, using a GPU with more VRAM, offloading to system RAM, or distributing the model across multiple GPUs.

10. Integrated GPU versus discrete GPU

The lab PC has two GPUs:

1. AMD Radeon Graphics integrated into the Ryzen 9 9950X
2. NVIDIA GeForce RTX 4060 discrete GPU

The integrated AMD GPU is part of the CPU/platform and primarily uses system memory.

The NVIDIA RTX 4060 is a separate discrete graphics card with dedicated GDDR6 VRAM.

AMD integrated GPU
       ↕
   System DDR5 RAM

NVIDIA RTX 4060
       ↕
 Dedicated GDDR6 VRAM

This is why a Windows tool reporting something like "2 GB" for the integrated GPU should not automatically be interpreted as 2 GB of physical dedicated VRAM chips. Integrated graphics commonly use reserved and shared system memory.

Can AMD and NVIDIA GPUs coexist?

Yes. Windows can use both devices at the same time.

For example:

AMD iGPU   → desktop/display work
RTX 4060   → CUDA/AI workload

Can their memory simply be added together?

No.

You cannot normally say:

AMD GPU memory + NVIDIA 8 GB VRAM = one larger AI GPU

They are separate devices with different software ecosystems and separate memory architectures.

NVIDIA's main compute ecosystem is CUDA. AMD's is commonly associated with ROCm and related APIs. A CUDA application cannot simply move half of its CUDA work onto the AMD GPU.

Both devices can exchange data through the wider system, but that is very different from behaving as one accelerator.

11. Driver versus CUDA

This distinction is one of the most important foundations in GPU computing.

A simple memory line is:

DRIVER = Can the operating system talk to the GPU?
CUDA   = Can a compute application use the NVIDIA GPU for work?

The stack looks like this:

AI Application
      ↓
     CUDA
      ↓
NVIDIA Driver
      ↓
 NVIDIA GPU

NVIDIA driver

The driver is the software layer that understands the NVIDIA hardware and allows the operating system to control and communicate with it.

This is why nvidia-smi working is primarily a driver/hardware visibility check.

CUDA

CUDA — Compute Unified Device Architecture is NVIDIA's software platform for general-purpose computation on NVIDIA GPUs.

It gives software a way to use GPU parallelism for tasks beyond graphics, including AI, simulation, image processing, and scientific computing.

The GPU may already be visible to Windows or Linux, but an AI framework still needs a compute ecosystem that knows how to launch and manage GPU computation. CUDA provides that NVIDIA-specific environment.

12. What is inside CUDA?

CUDA is not one executable. It is an ecosystem.

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

You do not need to become a CUDA programmer to become good at AI infrastructure.

An infrastructure engineer needs to understand where CUDA sits, what depends on it, how versions and drivers interact, and how to troubleshoot when the GPU is visible at one layer but unavailable at another.

13. Where PyTorch and vLLM fit

This is one of the easiest places to get confused because several software layers sit between the user and the GPU.

Start with the simplest idea:

The model is the brain. vLLM is not the model. vLLM is the serving engine that helps run that model efficiently for inference.

A very simple stack is:

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

What problem does vLLM solve?

Imagine that one person wants to experiment with a model locally. A simple Python program can load the model and send work to the GPU:

One user
   ↓
Python program
   ↓
Model
   ↓
GPU
   ↓
Answer

That can work well for learning. But now imagine 100 users sending prompts at the same time:

User 1   ─┐
User 2   ─┤
User 3   ─┤
...       ├──→ ?
User 100 ─┘

Someone now has to manage questions such as:

Which requests should be processed together?
How do we keep the GPU busy?
How much GPU memory is each request consuming?
How do we manage the KV cache?
How do we return generated tokens to the correct user?
How do we expose the model through an API?

That is the kind of problem an inference-serving engine such as vLLM is designed to solve.

A useful one-line definition is:

vLLM is an inference engine that serves large language models efficiently by managing requests, batching, KV cache, and GPU memory.

Restaurant analogy

Think of an AI-serving system as a restaurant:

Model = chef
GPU = kitchen equipment
VRAM = kitchen counter / nearby workspace
vLLM = restaurant manager

Without a good serving manager, the kitchen could work inefficiently:

Customer 1 arrives
      ↓
Chef completes entire order
      ↓
Customer 2 arrives
      ↓
Chef completes entire order
      ↓
...

With vLLM, requests can be organized so the expensive GPU is kept productive:

Many customer requests
        ↓
       vLLM
        ↓
organizes inference work
        ↓
keeps GPU busy efficiently
        ↓
returns each result to the right requester

The important idea is not that vLLM makes the model smarter. It helps serve the model more efficiently.

Where PyTorch fits

PyTorch is an AI/ML computation framework. It provides higher-level tensor and model operations so developers do not have to manually write raw CUDA code for every calculation.

A useful separation of responsibilities is:

vLLM
= inference-serving engine

PyTorch / optimized kernels
= performs model/tensor computation

CUDA
= NVIDIA GPU compute platform

NVIDIA Driver
= communicates with and controls the NVIDIA hardware

GPU
= performs the calculations

VRAM
= holds model weights, KV cache, and working data

So:

vLLM does not replace CUDA.
vLLM does not replace the NVIDIA driver.
vLLM sits much higher in the software stack.

vLLM and the KV cache

Suppose a user has a conversation:

User: My name is Kavisha. Explain Kubernetes.
AI:   ...
User: Now explain Services.

During generation, the model stores attention-related state for earlier tokens in the KV cache so it does not have to recompute all previous attention information from scratch for every new token.

That cache consumes VRAM:

1 active request   → some KV cache
10 active requests → more KV cache
100 active requests → potentially much more KV cache

vLLM is designed to manage this memory efficiently. One of the ideas it became well known for is PagedAttention.

At a beginner level, think of PagedAttention as a better way to organize KV-cache memory so that VRAM is not wasted in large unused chunks.

A hotel-room analogy helps:

Poor memory management:
Guest A reserves 10 rooms but uses only 4.
The remaining 6 cannot be used by anyone else.

Better memory management:
Allocate smaller chunks as needed.
Unused capacity can serve other guests.

The real implementation is more technical, but the architectural lesson is simple:

Efficient KV-cache management allows more useful inference work to fit into limited GPU memory.

vLLM and batching

A GPU is efficient when it has enough parallel work. If several users submit requests:

User A → "What is Kubernetes?"
User B → "What is CUDA?"
User C → "What is vLLM?"

A serving engine can organize work from multiple active requests so the GPU does not unnecessarily process each request in complete isolation.

Conceptually:

Request A ─┐
Request B ─┼──→ vLLM scheduling / batching ──→ GPU
Request C ─┘

This can improve throughput:

Throughput = how much total work the system can process over time

For LLM serving, you may see measures such as:

tokens per second
requests per second

Batching is not free. Larger or more aggressive batches may improve throughput while increasing memory use or individual request latency. Production serving is therefore a balancing problem.

vLLM versus a model

Do not confuse these:

Llama / Qwen / Mistral etc.
= the model

vLLM
= software used to serve a compatible model efficiently

The model contains the learned weights. vLLM manages how inference requests use those weights and the underlying GPU resources.

vLLM versus Ollama

Both can be used to run language models, but their common use cases feel different.

A beginner-friendly mental model is:

Ollama
→ very convenient for running models locally and experimenting

vLLM
→ designed strongly around high-performance model serving and concurrent inference

This is not a statement that one is universally "better." The correct choice depends on the use case. For an AI-infrastructure learning path focused on production inference, vLLM is especially important to understand.

vLLM versus Kubernetes

Kubernetes and vLLM solve different problems.

Kubernetes
= manages WHERE the workload runs

vLLM
= manages HOW LLM inference is served efficiently inside that workload

For example:

Users
  ↓
Load Balancer / Service
  ↓
Kubernetes
  ↓
vLLM Pod
  ↓
Model
  ↓
CUDA
  ↓
NVIDIA GPU ↔ VRAM

Kubernetes might decide:

"Place this vLLM Pod on a worker node that has an NVIDIA GPU available."

Then vLLM handles a different concern:

"I have many inference requests. How can I use the GPU and its memory efficiently to serve them?"

That distinction becomes extremely important when we later build GPU Kubernetes clusters.

vLLM memory line

Kubernetes = WHERE the inference workload runs
vLLM       = HOW requests are served efficiently
CUDA       = HOW software performs NVIDIA GPU computation
Driver     = HOW the OS/software stack controls the NVIDIA GPU
GPU        = DOES the math
VRAM       = HOLDS the model and active working data

14. What are model weights?

A trained neural network contains learned numerical parameters called weights.

For a model described as 7B, the "B" means roughly 7 billion parameters.

If parameters are stored using 2 bytes each, a very rough weight-only calculation is:

7 billion × 2 bytes ≈ 14 GB

A 70B model at the same 2-byte representation would require roughly:

70 billion × 2 bytes ≈ 140 GB

These are simplified weight-only estimates. Real inference also needs memory for KV cache, runtime buffers, temporary tensors, allocator overhead, and sometimes additional copies.

This explains why an 8 GB GPU cannot simply load every model at full precision.

15. Quantization — making models smaller

One option is quantization.

The idea is to represent model numbers using fewer bits.

Very simplified:

Higher precision
      ↓
More memory per parameter

Lower precision / quantization
      ↓
Less memory per parameter

You may eventually encounter terms such as:

FP16
BF16
INT8
INT4
4-bit quantization

The architectural lesson is more important than memorizing names:

Quantization trades numerical precision for lower memory use and often better serving efficiency.

This can allow a model to fit on a smaller GPU, although the quality and performance trade-offs depend on the model and quantization method.

16. Tokens — what an LLM actually generates

An LLM does not normally generate an entire sentence in one operation.

Text is represented as tokens, which may be whole words, parts of words, punctuation, or other text units depending on the tokenizer.

Inference roughly looks like:

Prompt
  ↓
Model predicts token 1
  ↓
Model predicts token 2
  ↓
Model predicts token 3
  ↓
...

This token-by-token behavior is why inference performance is often discussed using measures such as time to first token and time per output token.

17. Prefill versus decode

LLM inference has two important phases.

Prefill

During prefill, the model processes the input prompt.

Entire prompt
     ↓
Model processes input tokens
     ↓
Creates internal attention state
     ↓
First output token becomes possible

Prefill can be compute-intensive, especially for long prompts.

A common user-facing measure is TTFT — Time To First Token: how long the user waits before the first generated token appears.

Decode

After the first token, the model enters decode.

Token 1
  ↓
Token 2
  ↓
Token 3
  ↓
Token 4

Decode is sequential at the token level: each next token depends on the state created so far.

Decode can become strongly influenced by memory bandwidth because the model repeatedly needs to access large amounts of model data while generating tokens.

A common measure is TPOT — Time Per Output Token.

A useful beginner summary is:

Prefill = process the prompt
Decode  = generate the answer token by token

18. KV cache — why conversations consume GPU memory

Transformers use an attention mechanism. During inference, the model computes information commonly referred to as keys and values for previous tokens.

Instead of recomputing all of that information from scratch every time a new token is generated, inference engines store it in the KV cache.

Very simplified:

Conversation tokens
        ↓
Keys + Values calculated
        ↓
Stored in KV cache
        ↓
Reused while generating later tokens

This speeds up generation, but KV cache consumes memory.

So GPU memory is not only:

Model weights

It is more like:

VRAM
│
├── Model weights
├── KV cache
└── Runtime / temporary data

This gives us a very important production inference lesson:

A model may fit into VRAM when serving one short request but run out of memory when serving many long-context users because the KV cache grows.

19. Prefix caching

Many requests can begin with the same long prefix.

For example, a company might send the same system prompt with every request:

"You are the company's customer-support assistant..."

If the inference engine has already processed that exact prefix, prefix caching can allow it to reuse previously calculated state rather than repeating all of the same prefill work.

Conceptually:

Common prompt prefix
        ↓
Process once
        ↓
Cache reusable state
        ↓
Reuse for matching future requests

This can reduce repeated computation and improve latency and throughput when workloads contain reusable prefixes.

20. Batching — keeping the GPU busy

A GPU is designed for parallel work. Serving requests one at a time may leave much of the GPU underutilized.

Batching allows an inference engine to process work from multiple requests together.

Request A ─┐
Request B ─┼──→ GPU batch
Request C ─┤
Request D ─┘

This can significantly improve throughput.

But batching introduces trade-offs. Waiting too long to build a large batch may hurt individual request latency. Production inference systems therefore try to balance:

Throughput
Latency
VRAM usage
Number of concurrent users

This is one reason inference engines such as vLLM are important. Efficient model serving is much more than simply "put the model on a GPU."

21. Why memory bandwidth matters during LLM inference

Once model weights are in VRAM, the GPU repeatedly reads them while performing inference.

Think again about:

GPU compute = workers
VRAM        = nearby warehouse
Bandwidth   = conveyor belt

If the GPU has enormous compute capability but cannot feed data to the compute units quickly enough, the compute units wait.

This is why AI accelerator comparisons cannot be made using VRAM capacity alone.

Two GPUs could both have 24 GB VRAM but have very different:

Compute capability
Memory bandwidth
Memory type
Interconnect
Power envelope
Supported precision formats

For LLM inference, both capacity and bandwidth matter.

22. Multi-GPU — when one GPU is not enough

If a model is too large for a single GPU, one option is to distribute it across multiple GPUs.

Conceptually:

Large model
   │
   ├── GPU 1
   └── GPU 2

But adding GPUs creates a new problem:

The GPUs now need to communicate.

Possible GPU communication paths include PCIe and, on supported NVIDIA data-center systems, technologies such as NVLink and NVSwitch.

This means multi-GPU performance depends not only on the number of GPUs but also on the speed of the interconnect and the way the model is partitioned.

Later we will study concepts such as tensor parallelism and multi-node inference. For now, remember:

One GPU problem:
Do I have enough VRAM?

Multi-GPU problem:
How do I split the model AND how fast can the GPUs communicate?

23. Where Kubernetes eventually enters the picture

Kubernetes does not make a GPU faster. Its job is orchestration.

A future Kubernetes GPU stack will look approximately like this:

Kubernetes Pod
      ↓
Pod requests nvidia.com/gpu
      ↓
Kubernetes scheduler
      ↓
GPU-capable worker node
      ↓
NVIDIA Device Plugin / GPU Operator
      ↓
NVIDIA Container Toolkit
      ↓
CUDA / AI framework / vLLM
      ↓
NVIDIA Driver
      ↓
GPU + VRAM

The important lesson is that Kubernetes sits above the hardware and driver layers.

If nvidia-smi fails on the Linux node, debugging Kubernetes scheduling first would be the wrong layer.

A strong troubleshooting mindset is:

Hardware visible?
      ↓
Driver healthy?
      ↓
Container can access GPU?
      ↓
Kubernetes advertises GPU?
      ↓
Pod requests GPU correctly?
      ↓
CUDA/framework sees GPU?
      ↓
Inference engine healthy?

That is the same layer-by-layer troubleshooting philosophy used in networking and distributed systems.

24. The lab machine as one complete picture

Putting the current hardware together:

                 AMD Ryzen 9 9950X
                        │
                  16 CPU cores
                        │
                        ▼
                 64 GB DDR5 RAM
                 DDR5-4800 configured
                 ~52 GB/s measured
                        │
                        │
             AMD integrated Radeon GPU
             shares system-memory architecture


                 CPU / System RAM
                        │
                  PCIe 4.0 x8
               ~15.75 GB/s theoretical
                        │
                        ▼
               NVIDIA RTX 4060
                        │
                  ~272 GB/s
                        │
                        ▼
                 ~8 GB GDDR6 VRAM

The three bandwidth paths should now be mentally separate:

CPU ↔ RAM             ~52 GB/s measured
RAM ↔ NVIDIA GPU      ~15.75 GB/s theoretical over PCIe
NVIDIA GPU ↔ VRAM     ~272 GB/s theoretical

That single diagram explains a large amount of AI-infrastructure behavior.

25. The complete beginner mental model

If you remember only one picture from this chapter, remember this:

MODEL ON DISK
     ↓
SYSTEM RAM
     ↓
PCIe
     ↓
GPU VRAM

User request
     ↓
vLLM / AI framework
     ↓
CUDA
     ↓
NVIDIA Driver
     ↓
GPU COMPUTE ↔ VRAM
     ↓
TOKENS GENERATED

The software stack can also be viewed from the opposite direction:

User Request
     ↓
vLLM
     ↓
PyTorch / GPU kernels
     ↓
CUDA
     ↓
NVIDIA Driver
     ↓
GPU
     ↕
VRAM

And later Kubernetes will orchestrate the environment around that stack.

26. Quick revision notes

GPU
= performs massively parallel computation

VRAM
= fast local memory used by the GPU

VRAM capacity
= how much GPU-local data can fit

Memory bandwidth
= how quickly GPU and VRAM can exchange data

PCIe
= bridge between the CPU/system-memory world and discrete devices such as the GPU

NVIDIA Driver
= lets the operating system control and communicate with NVIDIA hardware

CUDA
= NVIDIA's compute platform used by applications to execute GPU workloads

nvidia-smi
= first-line NVIDIA GPU visibility/health/status command

Model weights
= learned model parameters that must be available during inference

Quantization
= use lower-precision representations to reduce memory/compute requirements

Token
= unit of text processed/generated by an LLM

Prefill
= process the input prompt

Decode
= generate output token by token

KV cache
= stores attention state from previous tokens so it does not need to be fully recomputed

Prefix cache
= reuses already processed common prompt prefixes

Batching
= combine work from multiple requests to use the GPU more efficiently

vLLM
= an LLM inference-serving engine designed for efficient GPU utilization and memory management

27. Interview-level summary

A concise way to explain the entire chapter is:

An LLM inference system needs both compute and memory. The GPU provides highly parallel compute, while VRAM holds model weights, KV cache, and working data close to the GPU. Local VRAM bandwidth is much higher than CPU-to-GPU PCIe bandwidth, so keeping active model data in VRAM is important for performance. The NVIDIA driver allows the operating system to communicate with the GPU, while CUDA provides the compute platform used by frameworks such as PyTorch and inference engines such as vLLM. During inference, the prompt is handled in a prefill phase and output is generated token by token during decode. KV cache, batching, quantization, and eventually multi-GPU architecture are key tools for improving capacity, latency, and throughput. Kubernetes later adds orchestration and GPU scheduling around this underlying stack.
