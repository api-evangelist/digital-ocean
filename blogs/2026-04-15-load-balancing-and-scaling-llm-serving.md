---
title: "Load Balancing and Scaling LLM Serving"
url: "https://www.digitalocean.com/blog/load-balancing-scaling-llm-serving"
date: "2026-04-15T19:03:31.807Z"
author: "Mohammad Ashar Khan"
feed_url: "https://www.digitalocean.com/rss/blog.atom"
---
<p>Load balancing for LLMs is fundamentally different from <a href="/products/load-balancers">load balancing</a> for traditional services like web servers, APIs, or databases. Prompt caching is the reason. Prompt caching typically <a href="/blog/advanced-prompt-caching">cuts input token costs by 50-90%</a> and can reduce Time to First Token (TTFT) latency by up to 80%, but those gains assume your request lands on the replica that already has the relevant prefix cached. Under naive round-robin load balancing across N replicas, that probability is 1/N. The cache hit rate that made caching so attractive at one replica degrades almost linearly as your fleet grows.</p>
<p>Solving this requires rethinking how requests are routed at the infrastructure level. This article covers the load balancing strategies and specialized routers that preserve cache efficiency at scale, starting with why standard approaches fall short and progressing to precise, cache-aware routing techniques.</p>
<details class="collapsible" open="">

<h2 id="inferencing-engines"><a href="#inferencing-engines">Inferencing engines</a><a class="hash-anchor" href="#inferencing-engines"></a></h2>

<p>To achieve large-scale inferencing, we use inference engines. These engines simplify the complexities of serving LLMs and offer improved resource utilization on the underlying GPUs. They also enable higher concurrency and allow for customization to suit diverse inference workloads, such as real-time chat completions and long-form document summarization. Noteworthy engine options include <a href="https://vllm.ai" rel="ugc nofollow noopener" target="_blank">vLLM</a>, <a href="https://sglang.io" rel="ugc nofollow noopener" target="_blank">SGLang</a>, and <a href="https://developer.nvidia.com/tensorrt" rel="ugc nofollow noopener" target="_blank">TensorRT</a>.</p>
<p>The inferencing process is largely consistent across different engines. Sending an HTTP request to an engine initiates a standard sequence of steps.</p>
<ol>
<li><strong>Prefill Phase:</strong></li>
</ol>
<ul>
<li>The input prompt is first converted into token IDs using the model’s tokenizer.</li>
<li>Requests are grouped into batches for efficient concurrent processing by the engine.</li>
<li>During this initial processing, special Key (K) and Value (V) tensors are computed.</li>
<li>This phase concludes after the first forward pass, resulting in the generation of the first output token.</li>
</ul>
<ol start="2">
<li><strong>Decode Phase:</strong></li>
</ol>
<ul>
<li>This phase involves an auto-regressive loop, continuing until an end-of-sequence token is generated or the maximum sequence length is reached.</li>
<li>For every subsequent token generated, the K and V tensors are incrementally updated.</li>
</ul>
<p><strong>Optimizations:</strong></p>
<ul>
<li>To improve efficiency and avoid redundant computation, the engine uses advanced techniques like Paged Attention and Prefix Caching. These methods employ and access K/V tensors efficiently from GPU memory, especially when new requests might share common prefixes.</li>
</ul>
<p><em>NOTE: This is a highly simplified description of token processing; in reality, inference engines perform much more significant optimizations, such as Fusing Tensor operations, capturing CUDA graphs, and tuning for various batch sizes.</em></p>
<p>For conversational workloads with a Large Language Model (LLM), every new message requires sending the entire conversation history to the engine. To improve efficiency, the use of pre-computed Key-Value (KV) caches reduces the “prefill” stage by reusing older caches. This ultimately decreases the TTFT.</p>
</details>
<details class="collapsible" open="">

<h2 id="routing-in-homogeneous-instances"><a href="#routing-in-homogeneous-instances">Routing in homogeneous instances</a><a class="hash-anchor" href="#routing-in-homogeneous-instances"></a></h2>

<p>If we are running the same model on <code>n</code> independent inference engines, the router policies typically include several popular options, such as:</p>
<div class="table-wrapper"><table>
<thead>
<tr>
<th><strong>Strategy</strong></th>
<th><strong>Description</strong></th>
<th><strong>Drawbacks</strong></th>
</tr>
</thead>
<tbody>
<tr>
<td><strong>Random or round robin</strong></td>
<td>Each request is sent to a randomly selected engine or in a sequential, round robin manner.</td>
<td>Suboptimal performance and inconsistent results because random routing hinders the effective utilization of engine-specific KV caches.</td>
</tr>
<tr>
<td><strong>Consistent hashing</strong></td>
<td>Uses a “sticky session” or user_id routing, ensuring requests from the same user consistently hit the same engine.</td>
<td>While an improvement over random, the first request from a new user may land on any engine, potentially an engine without the required prompt’s KV cache. This is better suited for long conversational workloads.</td>
</tr>
<tr>
<td><strong>Cache-aware load balancing</strong></td>
<td>Routes requests to the engine with the maximum prompt prefix overlap, unless the load is uneven, in which case it routes to minimize imbalance.</td>
<td>If the engines lack support for KV events, routing decisions are made solely based on the request. This can lead to routing a request to an engine whose cache has been invalidated, making the decision inaccurate.</td>
</tr>
</tbody>
</table>
</div><p>The standard approach for load balancing is typically <strong>cache-aware load balancing</strong>. A more sophisticated version of this is known as <strong>precise prefix cache-aware routing</strong>. In this advanced strategy, the router captures <strong>KV cache events</strong> emitted by the engine. This information is then used to make routing decisions, specifically directing the request to the engine that offers the greatest overlap with the existing prefix cache.</p>
<p>To highlight the impact of routing on inference performance, we can compare a precise-prefix-cache-aware routing strategy with a standard k8s service employing a round-robin or random policy.</p>
<p>Benchmarking details and results, provided by the LLM-d community, are available on the LLM-d routing <a href="https://github.com/llm-d/llm-d/blob/main/guides/precise-prefix-cache-aware/README.md#benchmarking" rel="ugc nofollow noopener" target="_blank">benchmark page</a>.</p>
<p><img alt="image alt text" src="https://doimages.nyc3.cdn.digitaloceanspaces.com/002Blog/Load%20balancing%20and%20scaling%20LLM%20Serving/Load%20Balancing%20and%20Scaling%20LLMs%20-%20Image%201.png" /></p>
<p>Cache-aware routing achieves an improvement in throughput of up to 108% for the same hardware configuration and workload. To understand cache-aware routing, we will start with its simpler form before progressing to a more advanced routing technique.</p>
<p>Cache-aware routing fundamentally relies on a data structure, often a <strong>Radix tree</strong>, that facilitates rapid insertion, removal, and prefix matching. The methodology is straightforward: a separate radix tree is maintained for each engine instance. When a new request arrives, its prompt is extracted. The system then iterates through all the instance-specific radix trees to identify the one with the longest matching prefix. This instance is then selected to handle the request. Additionally, the new prompt is inserted into the radix tree of the chosen instance for future routing considerations.</p>
<p>To prevent excessive radix tree growth, it implements a Least Recently Used (LRU) policy for purging its contents. Our routing strategy is not solely based on cache; we also incorporate a load-balancing mechanism that employs a balance threshold. This dynamic approach switches between pure load balancing and cache-aware routing because relying only on cache-aware routing has proven to be suboptimal for ensuring even load distribution across all instances.</p>
<p>The current design stores the entire prefix cache state within the router, without direct feedback from the individual engines regarding their actual prefix cache status. This lack of synchronization is why this method is not always accurate.</p>
<p>Inaccuracies can occur in two main ways:</p>
<ol>
<li><strong>Stale Router State (Engine Evicts Cache):</strong> An engine might evict a prefix cache due to memory pressure, yet the router’s radix tree still incorrectly lists it as present. Consequently, the router routes the request to that engine, forcing the entire K/V cache to be re-computed.</li>
<li><strong>Stale Engine State (Router Evicts Cache):</strong> Conversely, the router might evict a cache entry based on an LRU policy, but the actual K/V cache might still exist within the engine.</li>
</ol>
<p>Due to these inconsistencies, a more precise, cache-aware routing algorithm is often necessary.</p>
<p>In precise prefix cache aware routing, the router is able to make a more informed routing decision because it receives up-to-date K/V cache values from each engine via K/V cache events. This method differs from the simpler cache aware routing because it requires the router to first tokenize the input text prompt before performing the prefix match using a radix tree. While this adds a computation step during the routing process, the overall approach offers significant speed advantages and is much more precise, as tokenization is faster than the prefill or decoding phases.</p>
<p>To achieve high availability (HA) and independent scaling of the router, an external source like Redis can be used. Alternatively, a mesh architecture backed by Conflict-Free Replicated Data Types (CRDTs) can be employed. This approach allows each router to maintain a replicated data structure (the “tree”) such that they can independently handle KV events and process requests without needing to maintain individual, separate trees. This inherent capability enables the horizontal scaling of the router layer.</p>
</details>
<details class="collapsible" open="">

<h2 id="dis-aggregated-serving-for-large-sequence-lengths"><a href="#dis-aggregated-serving-for-large-sequence-lengths">Dis-aggregated serving for large sequence lengths</a><a class="hash-anchor" href="#dis-aggregated-serving-for-large-sequence-lengths"></a></h2>

<p>Arithmetic intensity is the ratio of arithmetic operations to memory operations for a particular hardware. This ratio is key in determining whether an operation is constrained by computation power (compute-bound) or memory access speed (memory-bound).</p>
<p><strong>Prefill Stage (No Batching):</strong></p>
<ul>
<li>All tokens in the prompts are processed in one forward pass.</li>
<li>Predominantly compute bound.</li>
</ul>
<p><strong>Decode Stage (Per-Token Generation):</strong></p>
<ul>
<li>Tokens are generated sequentially, one at a time.</li>
<li>Even with batching applied, the decode stage remains memory-bound because the KV cache must be updated after the generation of each individual token of a sequence before the next token can be produced.</li>
</ul>
<p>Optimal performance requires selecting hardware based on workload characteristics: higher TFLOPs are needed for efficient prefill, while high memory bandwidth is crucial for faster decode operations. This is because different hardware exhibits varying arithmetic intensity.</p>
<p>The final consideration is the method for transferring the calculated KV cache from the pre-fill engines to the decode engines, as the latter requires this cache to generate subsequent tokens. Crucially, the KV cache transfer must be executed with maximum speed, minimizing any additional memory movement.</p>
<p>Several solutions exist for transferring KV cache, including <a href="https://github.com/NVIDIA/nccl" rel="ugc nofollow noopener" target="_blank">NCCL (Nvidia)</a>, <a href="https://github.com/ROCm/rccl" rel="ugc nofollow noopener" target="_blank">RCCL (AMD)</a>, <a href="https://github.com/ai-dynamo/nixl" rel="ugc nofollow noopener" target="_blank">NVIDIA NIXL</a>, <a href="https://github.com/kvcache-ai/Mooncake" rel="ugc nofollow noopener" target="_blank">Mooncake TE (Transfer Engine)</a>, and <a href="https://github.com/uccl-project/uccl/tree/main/p2p" rel="ugc nofollow noopener" target="_blank">UCCL P2P</a>. For a comprehensive comparison and performance analysis of these different KV cache transfer engines, a benchmark is <a href="https://uccl-project.github.io/posts/kv-transfer-engine/#benchmarking-setup" rel="ugc nofollow noopener" target="_blank">available here</a>.</p>
<p>Disaggregated serving, while possible, is not always the most efficient choice for every request due to the overhead of KV transfer. Although the KV transfer uses advanced, high-speed technologies like RDMA, its bandwidth is still lower than the on-GPU memory bandwidth. Therefore, disaggregated serving is generally recommended for larger models with demanding workloads, specifically those with long sequences, such as a 100K Input Sequence Length (ISL) and a 1K Output Sequence Length (OSL). The ratio between Prefill and Decode nodes should be dynamically adjusted based on the specific ISL/OSL workload.</p>
<p>For smaller ISL, prefill/decode (P/D) dis-aggregation can be entirely omitted. In these cases, the decode worker can handle both the prefill and decode operations. We usually have a threshold that dynamically switches between proper P/D and non dis-aggregate serving.</p>
<p>The routing strategies described here assume KV cache state remains local to individual replicas. The logical next step (and where the industry is actively headed) is a shared cache layer accessible across replicas, likely backed by a high-bandwidth CPU DRAM pool. This would allow any replica to serve a cache hit regardless of which replica originally computed it. The challenge is latency: moving KV tensors across a network boundary is significantly slower than reading from local GPU VRAM, even with RDMA. Until that tradeoff is resolved at acceptable latency for production workloads, session affinity and prefix-aware routing remain the practical state of the art.</p>
<p>Performance metrics and savings (including TTFT and token costs) are based on specific benchmarks; actual results vary by configuration and are not guaranteed. References to third-party tools (e.g., vLLM, SGLang) are for informational purposes and do not imply any endorsement. This content is provided “as-is” for educational use and does not constitute a technical warranty or a binding Service Level Agreement (SLA).</p>
</details>
