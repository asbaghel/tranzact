# Approach Document
## 1. Initial Understanding

We have to create a PDF generator that can handle 1000 request per sec with hight constraint on Memory mangement as pupeteer uses around 400 MB of memory. The PDF generator should also be using latest data instead of using the stale data during bulk generation

The challenging part was handling the memory and cost both at the same time.

## 2. Assumptions & Clarifying Questions
- ~30,000 PDFs/month (~1,000/day) Traffic is uniformly distributed 
- Stricter solution is needed for Tamper proof PDFs
- Is partial inconsistency within the batch acceptable as the data can change during bulk PDF generation
- For how long these PDFs be stored as this will increase the cost


## 3. Capacity Planning & Math
> Show your calculations. How did you arrive at your resource
requirements?
> Consider: memory, CPU, storage, network bandwidth, queue depth.

| Parameter               | Value                |
|-------------------------|--------------------|
| PDFs/month              | 30,000             |
| PDFs/day                | 1,000              |
| Peak single requests    | 1000/min           |
| Bulk requests           | 5–10/min           |
| PDFs per bulk           | Up to 100          |
| Time per PDF            | 2–3 sec (avg = 2.5 sec) |
| Memory per PDF          | ~400 MB            |

- Average Load

1000 PDFs/day ≈ 42 PDFs/hour ≈ 0.012 PDFs/sec

- Peak Load
Single requests:
1000 requests/min = ~16.7 PDFs/sec
Bulk requests:
10 bulk/min × 100 PDFs = 1000 PDFs/min = ~16.7 PDFs/sec

Total Peak:
~33 PDFs/sec (worst case combined)

Required Concurrency

33 PDFs/sec × 2.5 sec = ~82.5 ≈ 83 concurrent PDFs

Memory Requirement

83 × 400 MB ≈ 33,200 MB ≈ 32.5 GB RAM

Concurrency Target

~6–10 concurrent PDFs 

Memory Needed: 10 × 400 MB = 4 GB + overhead = 8 GB

At 60% utilization (Safe limit to avoid OOM) 1 instance can handle 3-4 PDF concurrently 

for 10 we need 3 workers

CPU Requirement

Each PDF:

~0.5–1 vCPU

For 10 concurrent PDFs:
~5–10 vCPU total , 3 x t3.large = 6vCPU

| Parameter                | Value                         |
|--------------------------|-------------------------------|
| Average PDF size         | 500 KB                        |
| Monthly PDFs             | 30,000                        |
| Monthly storage          | 30,000 × 500 KB ≈ 15 GB       |
| With ZIPs + overhead     | 0–25 GB/month                 |
| Storage cost (S3)        | ~$0.023/GB → ~$1/month        |


Queue Depth

1000 PDFs/min = ~16.7/sec

We process 10 PDFs concurrently → ~4 PDFs/sec


Incoming = 16.7/sec
Processing = 4/sec
Backlog growth = ~12.7/sec


~750 queued jobs in 1 min so ~1000 messages safely




## 4. Design Decisions (minimum 3)

### Decision 1: [Controlled concurrency (2PDFs/worker)]
**Alternatives considered:** Allow scalling when we have bulk request
**Chosen approach:** Have a controlled conqurrency of 10 PDFs per second
**Why:** This is help save the cost nof system and also prevent OOM erros 
**Tradeoff accepted:**  Bulk jobs take longer


### Decision 2: [Digital signatures ]

**Alternatives considered:** Using crypto graphic hashesh
**Chosen approach:** Use Digital signatures
**Why:**   This is a standard way to prevent tampering and crypto hashes have No identity verification. Someone can modify PDF and recompute hash
**Tradeoff accepted:**  Slightly more complex implementation



We are evaluating your submission across four dimensions. Here is exactly what we
look for:
### Decision 3: [Pre-signed URLs for downloads ]
**Alternatives considered:**  Direct download from server
**Chosen approach:** Pre-signed URLs fordownloads 
**Why:**   Handles unreliable internet and Allows partial downloads
**Tradeoff accepted:**  Need  expiry / refresh mechanism

...

## 5. AI Usage Log
> Public links : https://drive.google.com/file/d/1ZzUtCvM5iQJAW032oKk6Tzqrk8k8wvu6/view?usp=sharing
> Chat link: https://chatgpt.com/share/69c0e2df-f218-8004-b95b-9dd3fc978df0

## 6. Weaknesses & Future Improvements
>Bulk Job Latency

Generating 100 PDFs sequentially can take 4–5 minutes; users with poor internet may face incomplete downloads.

Limited Horizontal Scaling

To stay under ₹12,500/month, the architecture assumes a small number of EC2 instances.

>Future Improvements
Advanced Worker Scaling & Orchestration
Use Kubernetes or Fargate to dynamically scale workers based on queue length and memory availability.
>Cost Optimization

Use spot instances or serverless PDF workers for low-cost scaling.
Archive older PDFs to cheaper storage tiers (S3 Glacier or equivalent).

10x Scaling 

This would need increating the memory and CPU and also have horizonatal scaling 
We can use HTML-to-PDF Lightweight Libraries like PDFKit to save memory


## 7. One Thing the Problem Statement Didn't Mention
Monitoring, Observability, and Alerting for the PDF Microservice

PDF generation is memory-intensive (~400MB per PDF) and can easily cause OOM crashes under high concurrency.
Bulk jobs can fail silently if workers crash, leaving users with partial or inconsistent PDFs.

Metrics collection – queue length, worker memory usage, PDF generation time, number of retries.

Adding monitoring ensures reliable operations, prevents silent failures, and supports future scaling or optimizations, which is crucial for a memory-heavy service like this.

## 8. Cost Estimation

Concurrent Request Targeted

10 concurrent PDFs --> 3 instance of 8GB 


Memory Needed: 10 × 400 MB = 4 GB + overhead (OS, Node) = 8 GB

At 60% utilization (Safe limit to avoid OOM) 1 instance can handle 3-4 PDF concurrently 

10/3 = ~3 instance

| Parameter                | Value                         |
|--------------------------|-------------------------------|
| Average PDF size         | 500 KB                        |
| Monthly PDFs             | 30,000                        |
| Monthly storage          | 30,000 × 500 KB ≈ 15 GB       |
| With ZIPs + overhead     | 0–25 GB/month                 |
| Storage cost (S3)        | ~$0.023/GB → ~$1/month        |

Redis 
Each job entry: ~1–2 KB (job ID, status, timestamps)
Memory: 100 jobs × 2 KB = ~200 KB → negligible

Bulk PDF jobs
10 bulk jobs at peak × 100 PDFs × 50 bytes = 50 KB
cache.t3.micro: 0.5 GB RAM → ~$16–20/month

| Component             | Qty        | Cost        |
|-----------------------|-----------|------------|
| Worker (t3.large, 8GB)| 3         | ~$120      |
| Redis (small)         | 1         | ~$16       |
| S3 Storage            | 20 GB     | ~$1–2      |
| Queue (SQS)           | -         | <$1        |

Total: $137/month ✅ within budget
