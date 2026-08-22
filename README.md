# DigitalOcean H100 GPU Droplets: How to Get NVIDIA H100 Compute on Demand, Which Plan to Pick, and Whether It's Worth It (With Reserved Pricing Breakdown and Setup Walkthrough)

If you've been shopping around for H100 GPU capacity lately, you already know the landscape is messy. Some providers quote one price and bill you for another. Others gate H100 access behind enterprise sales calls, quota applications, or week-long waits. Then there's the question of whether you actually need an H100 at all — or whether an A100, H200, or even an L40S would do the job for a fraction of the cost.

This piece walks through everything I learned while digging into **digitalocean h100** options: what the plans actually cost after the August 2026 price update, how the on-demand and reserved tiers compare, where DigitalOcean's H100 sits relative to AWS, Lambda, and RunPod, and what kind of workloads make sense on this hardware. I'll also cover the billing quirks that catch people off guard — the powered-off billing rule alone has surprised more than a few developers on Reddit.

## What Is the NVIDIA H100, and Why Are People Hunting for It?

The H100 is NVIDIA's Hopper-architecture data center GPU. It ships with 80 GB of HBM3 memory, 640 Tensor Cores, and 128 RT Cores, and it's built for the kind of workloads that broke older hardware: training large language models, running high-throughput inference, fine-tuning, and HPC simulations. NVIDIA claims up to 30x inference acceleration over the previous generation, and real-world benchmarks from independent testers put H100 training throughput at roughly 2.4x to 3x faster than A100 on mixed-precision workloads.

The H100 also supports FP8, which matters more than it sounds. FP8 lets you halve memory and bandwidth usage for calculations that can tolerate lower precision, which is most of modern LLM training and inference. That translates directly into either bigger models on the same hardware or lower cost per token served.

So when someone searches for "digitalocean h100," they're usually trying to answer one of these questions:

- Where can I get H100 capacity without an enterprise sales process?
- How much does H100 actually cost per hour on a developer-friendly cloud?
- Is DigitalOcean's H100 offering competitive with Lambda, RunPod, or AWS?
- Can I fine-tune a 7B or 70B model on a single H100, or do I need the 8x node?
- What's the catch with the pricing — are there hidden costs?

Let's work through each of those.

## DigitalOcean H100: The Plan Lineup After the August 2026 Price Update

DigitalOcean raised on-demand GPU prices on August 1, 2026, citing strong demand for advanced GPU capacity. The H100 went from $3.39/GPU/hour to $4.41/GPU/hour on-demand. The 8x H100 node went from roughly $27.12/hour to $35.28/hour. Reserved pricing was adjusted too, and that's where the real savings live if you have a sustained workload.

Here's the full H100 lineup as it stands now:

| Plan | GPU | GPU Memory | Droplet vCPUs | Droplet Memory | Boot Disk | Scratch Disk | On-Demand Price | 12-Month Reserved | Purchase |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| NVIDIA HGX H100 (1x) | 1x H100 | 80 GB | 20 | 240 GiB | 720 GiB NVMe | 5 TiB NVMe | $4.41/GPU/hr | $3.26/GPU/hr | [Get H100 1x Droplet](https://bit.ly/DigitaLocean) |
| NVIDIA HGX H100 (8x) | 8x H100 | 640 GB | 160 | 1,920 GiB | 2,046 GiB NVMe | 40 TiB NVMe | $35.28/hr ($4.41/GPU/hr) | $26.08/hr ($3.26/GPU/hr) | [Get H100 8x Droplet](https://bit.ly/DigitaLocean) |

A few things worth noting on these numbers. First, the per-GPU rate is the same whether you take 1 GPU or 8 — there's no volume discount baked into the on-demand price for the 8x node. The 8x just gives you 8 GPUs worth of capacity, NVLink interconnect between them, and a much larger scratch disk for staging training data.

Second, the 12-month reserved tier drops the per-GPU rate from $4.41 to $3.26, which is about a 26% discount. DigitalOcean also advertises a 3-year reserved commitment that goes as low as $1.91/GPU/hour, but that one requires a sales conversation — you can't self-serve it from the console.

Third, both configurations are HGX/SXM form factor, not PCIe. That matters if you're comparing against providers like Spheron or Vast.ai, where the cheaper H100 listings are often PCIe cards with lower memory bandwidth. SXM gives you full NVLink bandwidth between GPUs, which is what you want for multi-GPU training of large models.

If you want to see the live console and current availability before committing, you can 👉 [check current H100 pricing and regions on DigitalOcean](https://bit.ly/DigitaLocean).

## How DigitalOcean H100 Compares to Other GPU Clouds

Pricing alone doesn't tell the whole story. Billing model, form factor, and powered-off rules all change the effective cost. Here's how DigitalOcean's H100 stacks up against the other names that come up in H100 shopping comparisons:

| Provider | H100 Form Factor | On-Demand $/hr | Spot $/hr | Billing Granularity | Bills While Powered Off? |
| --- | --- | --- | --- | --- | --- |
| DigitalOcean | HGX/SXM | $4.41 | None | Per second (5-min min) | Yes |
| Lambda Labs | SXM | ~$2.49–$3.44 | None | Per hour | No |
| RunPod (Secure Cloud) | SXM | ~$2.99 | Available | Per minute | No |
| Nebius | SXM | $3.85 | $2.15 (preemptible) | Per hour | No |
| AWS (p5 instances) | SXM | ~$2.74+ | Limited | Per second | No |
| Spheron (PCIe) | PCIe | $2.01 | N/A | Per minute | No |
| Spheron (SXM5) | SXM5 | $4.06 | $1.49 | Per minute | No |
| Vast.ai (verified hosts) | Mostly PCIe | ~$1.50–$2.27 | ~$0.90–$1.60 | Per hour | No |

On raw on-demand price, DigitalOcean's H100 at $4.41/hr sits in the upper-middle of the pack. Lambda and Spheron PCIe are cheaper, RunPod is cheaper, and AWS is roughly comparable once you factor in p5 instance minimums. Where DigitalOcean's H100 gets expensive is the powered-off billing rule — and that's the part that catches people.

## The Powered-Off Billing Rule: The One Catch You Need to Understand

Here's the thing that surprised a Reddit user who spun up an 8x H100 droplet for half a day of tinkering and got a $290 bill: DigitalOcean bills GPU Droplets at the full rate whether they're running or powered off. Powering off a Droplet does not stop billing. The only way to stop the meter is to destroy the Droplet entirely.

The reasoning, per DigitalOcean's docs, is that the GPU and compute resources stay reserved on the hypervisor even when the Droplet isn't running. That makes sense from an infrastructure standpoint — they can't reassign your H100 to someone else while you have it powered off — but it changes the math significantly for anyone used to per-minute or per-hour billing that stops when the instance is idle.

Let's make this concrete with a single H100 at the new $4.41/hr rate:

- **24/7 utilization**: $4.41 × 24 × 30 = $3,175/month
- **8 hours/day active, Droplet kept alive**: still $3,175/month (you're paying for 24 hours of reservation)
- **8 hours/day active, Droplet destroyed after each session**: $4.41 × 8 × 30 = $1,058/month, plus whatever provisioning time costs you

For comparison, on a per-minute platform like Spheron or RunPod, the same 8-hours-a-day pattern costs roughly $482/month at Spheron's $2.01/hr PCIe rate, with no idle charges.

This doesn't make DigitalOcean a bad choice. It makes DigitalOcean a specific choice — the right one when you're running GPUs at high utilization, when you're already deep in the DO ecosystem, or when you want the simplicity of a single vendor for your whole stack. It's the wrong choice if you're doing bursty training runs and leaving Droplets alive between sessions out of convenience.

The fix is simple: destroy your Droplets when you're done, not power them off. Save your checkpoints to Spaces or a block volume first, then destroy. Re-provision when you need it again. Provisioning takes 2–5 minutes, which is a small price for not paying for 16 hours of idle H100 time.

## When Does the H100 Actually Make Sense Over Alternatives?

Not every workload needs an H100. DigitalOcean's GPU Droplets lineup includes H200, B300, AMD MI300X, MI325X, MI350X, L40S, RTX 4000 Ada, and RTX 6000 Ada — so the question is really about matching hardware to workload.

Here's how I'd think about it:

**Single H100 (1x, 80 GB)** is the right pick for:
- Fine-tuning 7B to 13B parameter models with LoRA or QLoRA
- Inference on models up to ~70B parameters with quantization (FP8 or INT4)
- RAG pipelines with embedding models and a smaller reranker
- Experimentation and prototyping where you want H100 performance but don't need multi-GPU scaling

**8x H100 node (640 GB total)** is the right pick for:
- Pre-training or continued pre-training of LLMs from scratch
- Full-parameter fine-tuning of 30B+ models
- Distributed training with model parallelism across GPUs (NVLink helps here)
- High-throughput production inference with continuous batching on large models
- HPC workloads like molecular dynamics or CFD that scale across GPUs

**When to pick H200 instead of H100**: if your model needs more than 80 GB of GPU memory and you don't want to split across multiple GPUs. H200 has 141 GB of HBM3e with 76% more memory and 43% more bandwidth than H100. It's the better call for serving Llama 3 405B or similar frontier-scale models on a single GPU. DigitalOcean prices H200 at $4.47/hr on-demand — only $0.06 more than H100 — so the upgrade is almost free if you need the memory.

**When to pick B300 instead of H100**: B300 is Blackwell Ultra, with 288 GB of memory per GPU and significantly higher FP8 throughput. It's overkill for most teams, but if you're training frontier-scale models or running very high-throughput inference, it's the top of the lineup. DigitalOcean offers B300 only on contract pricing — you'll need to talk to sales.

**When to pick L40S or RTX 4000/6000 Ada instead**: if your workload is inference on smaller models, graphics rendering, video processing, or 3D modeling. L40S at $1.57/hr is roughly a third the cost of an H100 and handles 7B model inference comfortably. RTX 4000 Ada at $0.76/hr is the cheapest GPU in the lineup and works for light inference and development work.

## Getting Started: Spinning Up Your First H100 Droplet

If you've decided H100 is the right fit, the setup is straightforward. DigitalOcean ships GPU Droplets with pre-installed Python, PyTorch, CUDA, and the NVIDIA container toolkit, so you can be running workloads within minutes of provisioning.

The basic flow looks like this:

1. **Create the Droplet** from the DigitalOcean console or via the API. Pick the H100 plan, choose a region (New York, Atlanta, or Toronto for GPU Droplets), and select an AI/ML-ready image. The 1-Click model marketplace also has prebuilt images for Llama 3.1, Stable Diffusion, and other popular models if you want to skip the setup.

2. **SSH in and verify the GPU** with `nvidia-smi`. You should see the H100 with 80 GB of memory and current driver version.

3. **Install your framework of choice**. PyTorch with CUDA support is pre-installed, but you may want to add vLLM for inference serving, Unsloth for fast fine-tuning, or Hugging Face Transformers for model loading.

4. **Stage your data** onto the scratch disk. The 5 TiB NVMe scratch disk is where you want training data and model weights — it's much faster than pulling from block storage or Spaces on every epoch.

5. **Run your workload**. For training, use `torchrun` or your framework's distributed launcher. For inference, vLLM with continuous batching will give you the best throughput per dollar.

6. **Destroy the Droplet when done**. Save checkpoints to Spaces or a persistent volume first, then destroy — don't just power off, or you keep paying.

DigitalOcean's docs have a full walkthrough for setting up the NVIDIA container toolkit, running Docker for GPU workloads, and installing Miniconda for Python environment management. There's also a community tutorial for running Jupyter Lab on a GPU Droplet if you prefer notebook-style development.

If you want to skip the manual setup entirely, you can 👉 [launch an H100 Droplet with 1-Click AI/ML images from DigitalOcean](https://bit.ly/DigitaLocean) and be running in minutes.

## Cost Scenarios: What Does an H100 Actually Cost You Per Month?

Let's run some numbers on the new August 2026 pricing, because the per-hour rate only tells you part of the story. Your actual monthly cost depends on utilization pattern and whether you're using on-demand or reserved pricing.

**Scenario 1: Sustained training run, 24/7 for two weeks**
- 1x H100, on-demand: $4.41 × 24 × 14 = $1,482
- 1x H100, 12-month reserved: $3.26 × 24 × 14 = $1,096
- 8x H100, on-demand: $35.28 × 24 × 14 = $11,854
- 8x H100, 12-month reserved: $26.08 × 24 × 14 = $8,763

For a two-week training run, reserved pricing saves you about 26% — meaningful but not transformative. The bigger lever is right-sizing: do you actually need 8x H100, or will 1x H100 with model parallelism across a couple of runs do the job?

**Scenario 2: Daily inference, 8 hours/day, Droplet destroyed nightly**
- 1x H100, on-demand: $4.41 × 8 × 30 = $1,058/month
- 1x H100, 12-month reserved: $3.26 × 8 × 30 = $782/month

This is the cost-conscious pattern. You're paying for actual GPU time, and the reserved tier brings it down further.

**Scenario 3: Daily inference, 8 hours/day, Droplet kept alive (the expensive mistake)**
- 1x H100, on-demand: $4.41 × 24 × 30 = $3,175/month
- 1x H100, 12-month reserved: $3.26 × 24 × 30 = $2,347/month

Same workload, three times the cost. This is the pattern to avoid.

**Scenario 4: Bursty experimentation, 2 hours/day**
- 1x H100, on-demand, destroyed each session: $4.41 × 2 × 30 = $265/month
- 1x H100, on-demand, kept alive: $4.41 × 24 × 30 = $3,175/month

For bursty work, the destroy-after-session pattern is the only one that makes sense on DigitalOcean. If your sessions are short and unpredictable, a per-minute platform like RunPod or Spheron may be a better fit.

## Who Should Actually Use DigitalOcean H100?

After digging through all this, here's how I'd characterize the fit:

**DigitalOcean H100 is a good fit for**:
- Teams already on DigitalOcean for the rest of their stack (DOKS, Spaces, Managed Databases, App Platform) who want consolidated billing and a single control plane
- Sustained training or inference workloads at high utilization, where the powered-off billing rule doesn't bite
- Developers who want H100 access without an enterprise sales process and are willing to manage Droplet lifecycle themselves
- Teams that need NVLink between GPUs for multi-GPU training and want the simplicity of a managed cloud over bare-metal providers
- Anyone who values the DigitalOcean documentation, community tutorials, and developer experience — it's genuinely better than most competitors

**DigitalOcean H100 is not a good fit for**:
- Bursty or intermittent workloads where you'd be paying for idle GPU time
- Teams that need spot/preemptible pricing to hit cost targets (DigitalOcean doesn't offer spot on H100)
- Workloads where the absolute lowest per-hour rate is the deciding factor — Lambda, Spheron PCIe, and Vast.ai all beat DigitalOcean on raw price
- Teams that need regions outside North America (GPU Droplets are currently only in New York, Atlanta, and Toronto)
- Anyone who wants to power off instances between sessions and not pay — that pattern doesn't work here

## Reserved Pricing: When It's Worth Committing

The 12-month reserved tier at $3.26/GPU/hour is a 26% discount over on-demand. The 3-year reserved tier at $1.91/GPU/hour is a 57% discount. These are substantial, but they require actual commitment.

The math is straightforward: if you're going to run an H100 for more than roughly 9 months of sustained usage, the 12-month reserved tier pays for itself even if you have idle months. If you're confident in 2+ years of sustained demand, the 3-year tier is dramatically cheaper — but you're locked in, and you'll need to talk to sales to set it up.

For most teams, the on-demand tier is the right starting point. Run on-demand for a month, see what your actual utilization looks like, then decide whether to commit to a reserved tier. The price increase from $3.39 to $4.41 on August 1 made the reserved tier more attractive than it was before — the gap between on-demand and reserved widened from about $0.13/GPU/hour to $1.15/GPU/hour.

If you're ready to explore reserved pricing or talk to sales about volume discounts, you can 👉 [reach DigitalOcean sales through the GPU Droplets page](https://bit.ly/DigitaLocean).

## Common Questions About DigitalOcean H100

**Can I use DigitalOcean signup credits for GPU Droplets?**
Generally no. Promotional credits typically don't cover GPU usage — DigitalOcean bills GPU charges directly to your payment method. Don't assume a $200 signup credit will fund your H100 experimentation.

**What's the minimum charge on a GPU Droplet?**
Per-second billing with a 60-second minimum, or $0.01, whichever is higher. There's also a 5-minute round-up mentioned in some docs — short evaluation runs under 5 minutes will bill as 5 minutes.

**What regions are H100 Droplets available in?**
New York, Atlanta, and Toronto. All North American. If you need EU or APAC regions for latency or data residency, you'll need a different provider.

**Is there an uptime SLA?**
Yes, 99% uptime SLA for GPU Droplets. That's lower than the 99.99% you'd see on AWS or GCP for some services, but reasonable for GPU capacity.

**Can I run Kubernetes with GPU Droplets?**
Yes — DigitalOcean Kubernetes (DOKS) supports GPU node pools, and GPU Droplets integrate with the rest of the DO ecosystem (VPC, Load Balancers, Spaces, Managed Databases). This is one of the genuine advantages over bare-metal providers.

**Do I need to install CUDA drivers myself?**
No. AI/ML-ready images come with CUDA, PyTorch, and the NVIDIA container toolkit pre-installed. You can SSH in and start running workloads immediately.

**What's the difference between H100 and H200 on DigitalOcean?**
H200 has 141 GB of HBM3e memory versus H100's 80 GB of HBM2e — 76% more memory and 43% more bandwidth. H200 is priced at $4.47/hr on-demand, only $0.06 more than H100. If you need the memory for larger models, H200 is almost the same price. If you don't, H100 is fine.

## The Bottom Line on DigitalOcean H100

DigitalOcean's H100 offering is a solid, developer-friendly way to get Hopper GPU capacity without enterprise sales friction. The August 2026 price increase to $4.41/hr on-demand put it firmly in the mid-to-upper range on raw price, but the 12-month reserved tier at $3.26/hr and the 3-year tier at $1.91/hr keep it competitive for sustained workloads. The powered-off billing rule is the real cost lever — manage your Droplet lifecycle aggressively, and the per-hour rate is what you pay; leave Droplets alive between sessions, and you're paying 3x what you should.

For teams already on DigitalOcean, the H100 lineup is a natural extension of the platform they're already using. For teams shopping purely on price, Lambda, Spheron, and RunPod all have cheaper H100 options — but none of them offer the same integrated ecosystem of managed Kubernetes, databases, storage, and networking that DigitalOcean does.

If you're running sustained training or inference, comfortable destroying Droplets between sessions, and want H100 capacity without a sales call, 👉 [DigitalOcean H100 GPU Droplets are worth a look](https://bit.ly/DigitaLocean). Spin one up, run a real workload, and see if the platform fits your team. The per-second billing means a quick evaluation costs less than a coffee.
