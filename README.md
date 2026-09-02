# Evidence-based local coding agents on a 2019 Mac Pro and 32 GB M2 Max

Status: public discussion release, 2026-09-02.

This report answers a practical question: **which local model, inference
configuration, and coding harness should someone actually try on hardware like
ours?** It reports measured results, including failures, and a repeatable way to
produce comparable evidence on another machine.

This is not a universal model leaderboard. It is a deployment guide derived
from fixed tool-use and coding tests on two specific machines.

## Start here: what to run

If your hardware and workload are similar, these are the configurations our
evidence supports.

| Goal | Recommended route | Evidence | Practical decision |
|---|---|---|---|
| Dependable general coding on the Mac Pro | **Qwen3.6-35B-A3B UD-Q4_K_S + Pi or Claude Code** | Both harnesses passed 3/3 identical coding repetitions; approximately 25--33 generated tok/s | Use as the default local route |
| Coding-specialist work | **Qwen3-Coder-Next Q4_K_M + Claude Code** | 3/3; approximately 16.7 generated tok/s at the speed gate | Use when coding specialization matters; do not assume Pi is equivalent |
| Consistent dense-model alternative | **Qwen3.8 27B UD-Q4_K_XL + Pi or Claude Code** | Both harnesses passed 3/3; approximately 9.4--9.6 generated tok/s | Slower, but polished and dependable in the bounded task |
| Non-Qwen diversity route | **Nemotron 3 Nano 30B-A3B + Pi** | 2/3; approximately 22 generated tok/s | Useful promoted alternative; retain a watchdog |
| Small work on a 32 GB M2 Max | **Qwen3.5 4B MLX 4-bit + Pi** | 2/3 coding passes and 9/10 later context-ladder passes | Use for bounded, plugged-in laptop work—not as the always-on service |
| Supervised high-capacity repair candidate | **gpt-oss-120B MXFP4 + Claude Code** | 1/3 strict passes, but 3/3 functions passed all 17 tests | Review and repair; do not route unattended by default |

Routes we would not put into unattended service from this evidence:

- Devstral 24B: too slow on one GPU and failed the substantial coding task
  after split-GPU acceleration.
- Gemma 4 31B: structured tools worked, but approximately 2.05 tok/s was below
  our long-run threshold.
- Mistral Small 4 119B-A6B: approximately 15.3 tok/s, but neither harness
  obtained a structured tool call.
- Qwen3-Coder 30B with OpenCode or Qwen Code: correct work appeared in some
  runs, but completion loops and error-handling defects made it unreliable.
- Slotstream 0.2.1 with Qwen3.8-Flash-Next: the model fit, but generation was
  approximately 1.1--1.5 tok/s and the endpoint rejected `tools` and
  `tool_choice`.

## The machines and what their memory numbers mean

| Machine | CPU memory | GPU memory | Tested inference role |
|---|---:|---:|---|
| 2019 Intel Mac Pro | **192 GB system/CPU RAM** | W5700X **16 GB discrete VRAM** plus Radeon Pro 580X **8 GB discrete VRAM** | Always-available server; GGUF through ToshLLM/llama.cpp |
| 14-inch M2 Max MacBook Pro | **32 GB unified memory**, shared by CPU and GPU | No separate VRAM; 30-core integrated GPU | Opportunistic local worker; MLX models |

The Mac Pro does **not** have 192 GB of GPU or unified memory. In this report,
`D24` means verified split placement across two physically separate VRAM pools:
16 GB and 8 GB. It is not one native 24 GB allocation. Large MoE models also
used the 192 GB system/CPU RAM for explicitly recorded CPU expert placement.
Fitting a model in system RAM proves capacity, not GPU residency, latency, or
interactive speed.

For every reported D24 result we required the actual server log to register
both GPUs. The MacBook's unified-memory results are a different architecture
and should not be compared as though they were VRAM-equivalent.

## Software and storage under test

- Mac Pro backend: ToshLLM's bundled `llama-server`, build 10495, commit
  `3dc7285b4`.
- MacBook primary backend: oMLX 0.6.3rc3; controlled comparison with
  Rapid-MLX 0.13.3 and Slotstream 0.2.1.
- Harnesses: Claude Code 2.1.248 for the later pinned worker matrix, Pi 0.84.4,
  OpenCode 1.18.20, and Qwen Code 0.22.3.
- MacBook active model storage: a directly attached 1 TB case-sensitive APFS
  SSD. Network storage was tested separately and was not used as the final
  active serving path.

Exact model artifacts, revisions, quantizations, hashes, chat templates,
context, placement, and per-attempt branches were retained in the research
repository. This public summary includes enough configuration to act on the
results without reproducing the entire private run history.

## What we actually tested

We treated the deployable unit as:

`machine × exact model artifact × backend configuration × harness × task`

Candidates moved through these gates in order:

1. **Artifact gate:** identify the exact checkpoint, revision, quantization,
   license, architecture support, and chat template.
2. **Placement gate:** load the model and verify actual GPU and CPU placement.
3. **Speed gate:** run a bounded request. Below 3 generated tok/s, preserve the
   capacity result but defer long coding tests.
4. **Tool gate:** force structured file and shell work through the real harness.
5. **Coding gate:** implement a fixed duration parser against 17 immutable
   tests and an exact result contract.
6. **Reliability gate:** run three identical repetitions. Promotion requires
   at least 2/3 complete passes.
7. **Independent acceptance:** verify unchanged tests and protected files,
   allowed changed paths, execution of the exact verifier, parseable result
   evidence, an independent test rerun, and normal harness termination.
8. **Human review:** separately assess correctness, completeness,
   repairability, and mergeability.

The three test workloads were:

| Workload | What it measures | What it does not prove |
|---|---|---|
| Forced tool probe | API/template/harness can produce and execute structured file and shell calls | General coding quality |
| Smoke-v2 duration parser | Small implementation, correction from real test failures, exact paths, evidence, and safe termination | Large-system architecture or very long autonomy |
| BenchBoard | A substantial Django app with data models, validation, CRUD, filters, sorting, comparison, APIs, CSV, demo data, migrations, tests, documentation, and an exact finish contract | Performance on every real repository |
| Context ladder | Retrieval of unique facts from the beginning, middle, and end of a filled context, structured tools, exact artifact, verifier, and termination | Long multi-turn coding at the same nominal context |

A strict PASS required correct code **and** trustworthy autonomous completion.
Correct code at the wrong path, a changed evaluator, missing evidence, skipped
verifier, or a completion loop remained an objective FAIL. We then reviewed
those failures separately so a low-repair near-miss was not confused with
absent or broken code.

## Mac Pro repeated coding results

Unless noted, these smoke-v2 rows used a 65,536-token context and verified D24
placement. `CPU-MoE` means experts were additionally placed in system/CPU RAM.
Speed is observed server generation or sustained agent generation, not prompt
processing.

| Exact model route | Placement | Harness | Strict passes and 95% Wilson CI | Functional code | Speed | Deployment interpretation |
|---|---|---|---:|---:|---:|---|
| Qwen3.6-35B-A3B UD-Q4_K_S | D24 | Claude Code | **3/3; 100% (43.9--100%)** | 3/3 | ~25--33 tok/s | Promoted default |
| Qwen3.6-35B-A3B UD-Q4_K_S | D24 | Pi | **3/3; 100% (43.9--100%)** | 3/3 | ~33 tok/s tool gate | Promoted default; disposable container worked well |
| Qwen3.8 27B UD-Q4_K_XL | D24 | Claude Code | **3/3; 100% (43.9--100%)** | 3/3 | ~9.4--9.6 tok/s | Promoted dense route |
| Qwen3.8 27B UD-Q4_K_XL | D24 | Pi | **3/3; 100% (43.9--100%)** | 3/3 | ~9.4--9.6 tok/s | Promoted; no observed container penalty |
| Qwen3-Coder-Next official Q4_K_M | D24 + CPU-MoE | Claude Code | **3/3; 100% (43.9--100%)** | 3/3 | ~16.7 tok/s | Promoted coding-specialist route |
| Qwen3-Coder-Next official Q4_K_M | D24 + CPU-MoE | Pi | **1/3; 33.3% (6.1--79.2%)** | 2/3 | ~16.7 tok/s | Do not substitute Pi for Claude on this evidence |
| Nemotron 3 Nano 30B-A3B Q4_K_M | D24 + CPU-MoE | Claude Code | **1/3; 33.3% (6.1--79.2%)** | 1/3 | 21.981 tok/s | High variance; not promoted |
| Nemotron 3 Nano 30B-A3B Q4_K_M | D24 + CPU-MoE | Pi | **2/3; 66.7% (20.8--93.9%)** | 3/3 | 21.981 tok/s | Promoted diversity route; one correct-code timeout |
| gpt-oss-120B official MXFP4 | D24 + CPU-MoE | Claude Code | **1/3; 33.3% (6.1--79.2%)** | 3/3 | 16.710 tok/s | Strong logic, weak path/result discipline |
| gpt-oss-120B official MXFP4 | D24 + CPU-MoE | Pi | **0/3; 0% (0--56.1%)** | 2/3 | 16.710 tok/s | Supervised evidence only |
| Qwen3-Coder 30B-A3B Q4_K_M | D24 | OpenCode | **1/3; 33.3% (6.1--79.2%)** | 3/3 | 9.546 tok/s sustained | Two correct runs looped until the watchdog |

The Qwen3-Coder-Next artifact was the official four-shard Q4_K_M release,
pinned to revision `6f77dc161d902cad878f6b23eddcdbe0ad1ebde0`, totaling
48,410,992,032 bytes. The gpt-oss-120B artifact was the official ggml-org
63,387,346,208-byte MXFP4 release. These large models fit because system RAM
held CPU-resident MoE data; neither result means the full model occupied D24.

### How much confidence should 3/3 create?

Not much statistical certainty by itself. The interval column uses a two-sided
95% Wilson score interval for the strict binary result. With only three trials:

- 3/3 estimates 100%, but its interval is **43.9--100%**;
- 2/3 estimates 66.7%, with **20.8--93.9%**; and
- 1/3 estimates 33.3%, with **6.1--79.2%**.

These repetitions are best used as a screening experiment: 3/3 rules a route
in for more work, while repeated distinct failures rule it out for unattended
use. They are not precise estimates of production success probability. The
same machine and task were held fixed, so the trials sample only a narrow
operating condition.

Across the matched five-model Claude/Pi panel above, Claude produced 11/15
strict passes (73.3%; Wilson interval 48.0--89.1%) and Pi produced 9/15 (60.0%;
35.7--80.2%). Both produced functionally verified code in 13/15 runs. This
pooled view is descriptive only: it mixes models with strong model-by-harness
interactions, and the intervals overlap substantially. It supports testing the
pair, not declaring one universal harness winner.

The Qwen3.6 Claude repetitions also show why raw pass rate is incomplete. All
three passed, but agent duration ranged from 113 to 698 seconds (median 235),
tool calls ranged from 8 to 28 (median 11), and observed end-of-run generation
ranged from 25.7 to 31.4 tok/s (median 29.6). Publish dispersion and individual
repetitions rather than only an average.

### Harness choice changed the winner

| Held model | Claude Code | Pi | Conclusion |
|---|---:|---:|---|
| Qwen3.6 35B-A3B | 3/3 | 3/3 | Either harness is supported by this test |
| Qwen3.8 27B | 3/3 | 3/3 | Either harness; Pi isolation cost no observed reliability |
| Qwen3-Coder-Next | **3/3** | 1/3 | Use Claude Code |
| Nemotron 3 Nano | 1/3 | **2/3** | Use Pi |
| gpt-oss-120B | 1/3 strict, 3/3 functional | 0/3 strict, 2/3 functional | Claude is better, but neither is unattended-ready |

Pi was therefore a first-class harness, not merely a lightweight compromise.
It matched Claude on the two most stable general routes and improved Nemotron.
It was not universally better: Qwen3-Coder-Next produced 3/3 with Claude and
1/3 with Pi in this experiment. With only three repetitions per arm, that is an
operational selection signal rather than a statistically conclusive universal
effect.
There is no meaningful model ranking independent of the harness and its tool
contract.

OpenCode and Qwen Code supplied useful negative evidence. OpenCode produced
correct Qwen3-Coder code in all three smoke repetitions, but two agents kept
working after completion until a 30-minute watchdog stopped them. Qwen Code's
probe denied a shell-redirection write; the model ignored the error and falsely
claimed success. A platform must judge executed effects and termination—not
the final prose response.

## Speed, placement, and quality are separate axes

Two matched examples show what the Mac Pro's second discrete GPU changed.

| Model | W5700X 16 GB only | Verified D24 split | Quality outcome |
|---|---:|---:|---|
| Devstral 24B Q4_K_M | ~0.95 tok/s sustained | ~6 tok/s sustained; 12.9 tok/s short gate | Substantial task still failed |
| Qwen3-Coder 30B Q4_K_M | 1.304 tok/s gate | 9.546 tok/s sustained | Became interactive; full BenchBoard quality still failed |

The extra 8 GB VRAM pool produced roughly six- to seven-fold improvements in
these routes and changed them from impractical to interactive. It did not make
either model follow the specification. Publish capacity, placement, speed,
correctness, and completion behavior as distinct fields.

## Substantial-task evidence

The 17-test smoke task selects promising routes; it is not a substitute for a
larger implementation. BenchBoard exposed failures that short tasks did not.

| Model | Context | Time / generation | Model tests | Independent result | Review |
|---|---:|---:|---:|---:|---|
| Qwen3.6 35B-A3B + Claude | 131,072 | 2,911 s; ~24.7 tok/s; ~96K prompt and ~57K generated tokens | 65 passed | 6/13 evaluator checks; finish contract failed | Substantial but moderate repair |
| Qwen3.8 27B + Claude, best run | 131,072 | 12,335 s; 7.504 tok/s | 90 passed | **11/13** evaluator checks; finish contract passed | Strongest large-task output; two narrow integration fixes |
| Qwen3.8 27B + Claude, replication | 131,072 | 12,844 s; 7.691 tok/s | 74 passed | 9/13 evaluator checks; finish contract passed | Useful, but more repair than the best run |
| Devstral 24B + Claude | 65,536 | 7,212 s; ~6 tok/s sustained | 24 discovered; verifier failed | Specification and finish contract failed | Not mergeable |

Qwen3.8's best run was the most convincing large-task application despite
being slower than Qwen3.6. This is why the recommendation table separates
speed from output quality. The result is also a warning: 3/3 on a small fixed
task does not guarantee specification completeness on a multi-hour build.

## Prompt processing: what we measured

Yes, prompt processing was measured, but not uniformly in every historical
cell. Early long-agent reports often retained aggregate generation speed and
token counts without a separately comparable prefill rate. Later backend tests
explicitly recorded prompt size, cold and warm time to first token (TTFT),
prefix reuse, and decode rate. We do not fill missing prefill values by
inference.

### Raw MacBook model benchmarks

These are direct MLX benchmark measurements on the 32 GB M2 Max, not end-to-end
agent results. `pp` is prompt processing and `tg` is token generation.

| MLX 4-bit artifact | Benchmark | Prompt processing | Generation | Peak memory |
|---|---|---:|---:|---:|
| `mlx-community/Qwen3.8-27B-4bit` | pp1024 / tg128 | 83.4 tok/s | 11.6 tok/s | 15.85 GB |
| `mlx-community/Qwen3.8-27B-4bit` | pp4096 / tg128 | 89.7 tok/s | 13.4 tok/s | 17.12 GB |
| `mlx-community/Qwen3.6-35B-A3B-4bit` | pp1024 / tg128 | 553.0 tok/s | 75.5 tok/s | 19.35 GB |
| `mlx-community/Qwen3.6-35B-A3B-4bit` | pp4096 / tg128 | 613.9 tok/s | 81.9 tok/s | 20.09 GB |

The MoE Qwen3.6 benchmark was dramatically faster than the similarly sized
dense Qwen3.8 artifact on this machine. These short raw numbers are useful for
screening, but the coding tests still decide whether a harness/model route is
trustworthy.

### Controlled prefix-cache measurement

We held the exact 15 GB Qwen3.8 27B MLX 4-bit artifact, a 65,536-token context,
a 5.3K-token prompt, and the Pi task fixed.

| Backend configuration | Cold TTFT | Exact-repeat TTFT | Prefix-extension TTFT | Decode | Pi task |
|---|---:|---:|---:|---:|---:|
| oMLX 0.6.3rc3 control | 87.9 s | 19.8 s | 19.9 s | 11.2--11.7 tok/s | **PASS, 115.8 s** |
| Rapid-MLX 0.13.3, 2 GiB prefix cache | 65.3 s | **0.7 s** | **1.2 s** | ~11.3 tok/s | **PASS, 69.3 s** |
| Rapid-MLX 0.13.3, cache disabled | 65.6 s | 66.6 s | 65.5 s | 11.0--11.4 tok/s | **PASS, 202.6 s** |

TTFT includes more than pure prompt evaluation, so it should not be relabeled
as prefill tok/s. The controlled comparison nevertheless isolates the useful
effect: prefix reuse made the Pi task approximately **2.9× faster** than Rapid
without cache and **1.7× faster** than the oMLX control while decode speed was
nearly unchanged. Coding agents repeatedly resend growing, mostly unchanged
histories, so prompt caching can matter more than a modest decode-rate win.

An earlier Mac Pro gpt-oss-20B pilot did retain complete aggregate counters:
83,808 uncached prompt tokens in 234.954 seconds (**356.7 prompt tok/s**) and
39,689 generated tokens in 780.151 seconds (**50.9 generated tok/s**). That is
evidence that the collection method worked, not a cross-model winner: it used
a different model, single-GPU profile, and earlier task.

The result format now has separate fields for prompt tokens, prompt seconds,
prefill tok/s, TTFT, generated tokens, and decode tok/s. New submissions should
populate them whenever the backend exposes trustworthy counters.

## MacBook small-model results

The MacBook matrix used oMLX at 65,536 context for the primary small-model
coding comparison.

The oMLX runner required `iogpu.wired_limit_mb=28672` after reboot so macOS
could wire enough of the 32 GB unified-memory pool for these tests. That value
is part of this machine's test configuration, not a blanket recommendation:
leave operating-system headroom, verify memory pressure and swap, and reset or
lower it for ordinary laptop use if needed.

| Model and harness | Tool gate | Smoke-v2 | Decision |
|---|---:|---:|---|
| Qwen3.5 4B + Pi | PASS | **2/3** | Promoted small local route |
| Qwen3.5 4B + Claude Code | PASS | 1/3 | Not promoted |
| Qwen3.5 9B + Pi | PASS | 1/3 | Not promoted |
| Qwen3.5 9B + Claude Code | PASS | 0/3 | Not promoted |
| Gemma 4 12B + either harness | FAIL | Skipped | No structured tool call |
| Phi-4 Mini + either harness | FAIL | Skipped | No structured tool call |

The larger 9B model was not more autonomous than 4B. Family, tuning, template,
and harness fit mattered more than parameter count in this slice.

Qwen3.6 completed a harder MacBook smoke task in approximately 2.6 minutes,
but a later BenchBoard session grew to 109,142 tokens against a 65,536-token
server limit and looped on one test. Short-task success did not make it a safe
large-task laptop route.

## Context capacity actually tested

Configured context is not observed context use. The main Mac Pro tool and
smoke matrix used 65,536-token contexts. The Qwen3.6 and Qwen3.8 long BenchBoard
runs used 131,072. The MacBook baseline and backend comparisons used 65,536.

The separate MacBook context ladder used Qwen3.5 4B + Pi. Each cell required
retrieval of unique markers across the filled prompt, a structured tool call,
an exact `answer.json`, immutable independent tests, and normal termination.
The tier is the workload target; observed full-harness prompt tokens are shown.

| Tier | Rep 1 / Rep 2 | Observed prompt tokens | Agent duration |
|---|---:|---:|---:|
| 16K | PASS / quality FAIL | 19,273 / 22,746 | 54 / 51 s |
| 32K | PASS / PASS | 26,009 / 34,141 | 65 / 92 s |
| 64K | PASS / PASS | 47,457 / 57,632 | 137 / 177 s |
| 96K | PASS / PASS | 72,443 / 70,257 | 234 / 224 s |
| 128K | PASS / PASS | 96,314 / 103,346 | 356 / 410 s |

The calibrated ladder passed **9/10: 90.0%, with a 95% Wilson interval of
59.6--98.2%**. The two 128K trials passed, but 2/2 still has a very wide
34.2--100% interval. The lone 16K failure produced no answer or verifier run
while its matched repetition passed, so it is a stochastic quality failure
rather than a context ceiling. No repeatable failure appeared before the
server's 131,072-token maximum on this bounded retrieval workload. This does
**not** establish that a multi-turn 128K coding session is equally reliable or
pleasant.

## Backend, storage, and oversized-model findings

- **Use direct-attached or internal storage for active weights.** One model
  smoke took 170.5 seconds from SMB versus 19.2 seconds from the internal SSD;
  a later network-backed startup blocked in filesystem reads. Network storage
  is suitable as a durable library, followed by local staging.
- **Cache behavior belongs in the model profile.** Raw decode speed alone hid a
  2.9× end-to-end difference in the controlled Rapid comparison.
- **Unload explicitly.** oMLX cleanly released 17.08 GB at shutdown. A laptop
  worker should unload after its task, and an always-on server should unload
  after an idle period so the machine can serve other uses.
- **Fitting is not serving.** Slotstream loaded a 103.8 GB, 176B-total/6B-active
  Qwen3.8-Flash-Next MLX artifact using partial expert residency. Cold chat
  prefill was 2.54 tok/s and a longer cached prefix reached 8.21 tok/s, but
  decode remained 1.08--1.37 tok/s and the API rejected tool fields.

## A deployment pattern that follows from the evidence

The inference server should not also be the project worker. A practical local
system has five boundaries:

1. **Model host:** an always-available machine runs one explicit model profile
   at a time. A profile pins artifact, backend, template, context, cache,
   placement, and sampling.
2. **Gateway:** LiteLLM supplies stable aliases, API keys, and optional paid
   fallback. It is a router, not the source of truth for GPU scheduling.
3. **Coordinator:** one lease protects the model server, queues requests,
   switches model profiles, enforces time and cost budgets, and unloads models
   after idle time.
4. **Disposable worker:** start one isolated Pi or Claude Code environment per
   task, mount only the task workspace and harness home, collect the patch and
   evidence, then remove it.
5. **Independent verifier:** keep immutable tests and policy outside the
   candidate's writable workspace; enforce allowed paths and rerun tests after
   the worker exits.

A useful initial routing policy is:

| Alias | Model profile | Worker | Policy |
|---|---|---|---|
| `local-general` | Qwen3.6 35B-A3B, D24, 65,536 | Pi | Default bounded local task |
| `local-general-claude` | Same model profile | Claude Code | Same model with the alternate harness |
| `local-coder` | Qwen3-Coder-Next, D24 + CPU-MoE, 65,536 | Claude Code | Coding specialist |
| `local-dense` | Qwen3.8 27B, D24, 65,536 | Pi or Claude Code | Slower consistent alternative |
| `local-diversity` | Nemotron 3 Nano, D24 + CPU-MoE, 65,536 | Pi | Non-Qwen comparison |
| `local-supervised` | gpt-oss-120B, D24 + CPU-MoE, 65,536 | Claude Code | Review-required repair experiment |

The harness is intentionally outside the LiteLLM alias. A model can succeed
with one harness and fail with another, so the coordinator must select and
record both. Long jobs also need coordinator-level queueing and leases rather
than an HTTP timeout measured in days.

## How to reproduce this on another machine

Do this before declaring a model usable:

1. Record the machine's system RAM, unified memory, and each discrete VRAM pool
   separately.
2. Pin the model repository, revision, file hashes, quantization, license, and
   chat template.
3. Pin the inference engine and harness versions. Record context, batch sizes,
   weight/KV quantization, cache state, sampling, and actual placement.
4. Measure cold load, prompt tokens and time or TTFT, generated tokens and
   decode rate, peak memory, and end-to-end wall time.
5. Stop before long tests if the route cannot load, cannot emit a forced
   structured tool call, or generates below your usability threshold. Ours was
   3 tok/s.
6. Run the same externally verified coding task at least three times. Do not
   alter the prompt or tests between scored repetitions.
7. Preserve each attempt. Classify infrastructure failures separately from
   completed quality failures; do not rerun quality failures merely to obtain a
   pass.
8. Publish objective acceptance and human review separately. Report useful
   near-misses without changing their strict verdict.

At minimum, a public result should contain:

- privacy-safe machine and memory description;
- model, revision, file hash, format, and quantization;
- backend, template, context, cache, sampling, and placement;
- prompt/prefill and generation/decode metrics where available;
- exact task and verifier hashes;
- tool events, wall time, termination status, and pass/fail reasons; and
- a separate qualitative review of correctness, repairability, and
  mergeability.

### Statistical reporting rules for future contributors

The three-run matrix is a qualification screen. A route intended for regular
use should then receive a larger preregistered confirmation set. We recommend:

- publish every repetition and the denominator; never report only the best
  run or an average;
- report strict success and functional-code success as separate binomial
  outcomes with Wilson intervals;
- hold task, model artifact, server profile, and sampling fixed when comparing
  harnesses, and alternate or randomize run order to reduce thermal and cache
  bias;
- report wall time, prompt processing, decode speed, and tool count as median,
  range, and—once the sample is large enough—interquartile range;
- count watchdog terminations as failures at the timeout limit rather than
  dropping them from latency statistics;
- retain failure categories such as implementation, path, verifier, evidence,
  protocol, and nontermination so readers can choose a repair policy; and
- use at least ten unchanged repetitions to confirm a route that passes the
  three-run screen. Even 10/10 has a 95% Wilson lower bound of only 72.2%, so
  higher-assurance production claims require more evidence and more than one
  task.

Do not pool unlike models into a single harness score without showing the
per-model strata. The descriptive 15-run aggregate above is useful context,
but Qwen3-Coder-Next and Nemotron demonstrate that interaction effects can
reverse the apparent winner.

## What this evidence does and does not establish

- Two machines, a handful of exact artifacts, and three repetitions per smoke
  route cannot produce a universal ranking.
- Tool compatibility, raw speed, small-task reliability, and substantial-task
  quality are different findings.
- A 3/3 smoke result is a strong reason to try a route, not permission to merge
  its next patch without review.
- A strict FAIL can still contain correct, low-repair code. gpt-oss-120B is the
  clearest example: five of six implementations passed all 17 functional tests,
  but only one satisfied the full unattended contract.
- The MacBook 128K result establishes bounded retrieval, not long-agent
  stability at 128K.
- The best result is the complete operational profile, not the model name.

The actionable contribution is the evaluation sequence and reporting schema:
it turns “I use a local LLM to code” into a claim another person can reproduce,
challenge, and extend with a different machine, model, backend, or harness.
