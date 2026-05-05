---
title: "Building a Robust Documentation Agent with DigitalOcean Gradient AI Platform"
url: "https://www.digitalocean.com/blog/documentation-agent"
date: "2026-04-13T16:59:45.687Z"
author: "Anna Lushnikova"
feed_url: "https://www.digitalocean.com/rss/blog.atom"
---
<p>At DigitalOcean, documentation has always been a priority. Developers come to our docs to get unstuck, and the faster they find what they need, the better. Traditional docs pages work, but they require users to know which page to visit, scan for the relevant section, and map generic instructions to their specific setup. That process takes minutes (or longer) when it could take seconds.</p>
<p>So we built an AI documentation assistant. Ask a question in plain language, get an answer with working links and ready-to-use commands. Simple enough as a demo. Getting it production-ready was a different story. It took us several iterations to reach an agent we were confident enough to ship. The LLM could generate plausible-sounding answers from day one. Knowing whether those answers were grounded, and keeping them grounded after every model update and prompt change — that’s where we spent most of our time.</p>
<p>This post covers what we built, how we validated it, and the specific decisions that moved our metrics from “not great” to “ready for launch.” We’ll walk through prompt engineering, evaluation pipelines, and the CI/CD glue that holds it all together. Throughout, we relied on DigitalOcean’s Agentic Inference Cloud so we could focus on product behavior instead of stitching together inference, RAG, and evaluation tooling ourselves — one place to run agents, attach knowledge bases, and measure quality, with scale when we need it, and straightforward operational patterns.</p>
<details class="collapsible" open="">

<h2 id="architecture"><a href="#architecture">Architecture</a><a class="hash-anchor" href="#architecture"></a></h2>

<p><a href="/products/gradient/platform">Gradient™ AI Platform</a> is the control plane and runtime for standing up production AI agents without assembling pieces by hand. In one place you can attach a knowledge base, define an agent, and tune how it behaves: pick an LLM (managed or open-source), set temperature and top P, choose retrieval behavior, and edit the system prompt. The goal is to get from an empty project to a working agent in minutes, not to wire up inference, RAG, and evaluation from scratch.</p>
<p>A Gradient AI Agent is the concrete thing you end up with: a configured AI agent aimed at a specific job (for us, docs Q&amp;A), with that stack of choices baked in. Think of it as the deployed product built on top of the Gradient™ AI Inference Cloud building blocks — not a separate platform, but the named, production-shaped instance you operate and evaluate.</p>
<p>We chose an <a href="https://docs.digitalocean.com/reference/api/reference/agent-inference/" rel="ugc nofollow noopener" target="_blank">Agent</a> for the docs assistant because of the product fit. We need task-oriented Q&amp;A with grounding and follow-up behavior that fits an assistant, not a raw completion API.</p>
<p>We run two interfaces to the agent, each solving a different problem.</p>
<h3 id="1-direct-script-embedding"><a href="#1-direct-script-embedding">1. Direct script embedding</a><a class="hash-anchor" href="#1-direct-script-embedding"></a></h3>
<p>The first approach is the simplest one. We embedded a <a href="https://docs.digitalocean.com/products/gradient-ai-platform/how-to/use-agents/" rel="ugc nofollow noopener" target="_blank">Gradient AI JavaScript snippet</a> directly into the DigitalOcean documentation site. This went live early as a low-cost way to get the agent in front of users fast. It generates a token to authenticate with the Gradient AI Agent.</p>
<h3 id="2-api-gateway-and-proxy-for-the-askdocs-in-cloud"><a href="#2-api-gateway-and-proxy-for-the-askdocs-in-cloud">2. API Gateway and Proxy for the AskDocs in Cloud</a><a class="hash-anchor" href="#2-api-gateway-and-proxy-for-the-askdocs-in-cloud"></a></h3>
<p>The second approach is more involved. An API gateway routes requests to an internal proxy service that sits between the user and the Gradient AI Agent. Authentication here relies on the user’s existing DigitalOcean session rather than a generated token.</p>
<p>The proxy turned out to be worth the extra complexity. Because it handles streaming responses from the Gradient AI Agent, we can measure TTFT (time to first token) at the proxy layer and get real latency numbers without instrumenting the agent itself.</p>
<p>Beyond latency measurement, running the proxy as an internal service gave us several things we didn’t have to build from scratch:</p>
<ul>
<li>
<p>Logging, golden signals metrics, and access to internal libraries that follow company standards</p>
</li>
<li>
<p>Rate limiting, auth, and load balancing, all reused from existing internal infrastructure</p>
</li>
<li>
<p>A translation layer between the frontend contract and the Gradient Agent API, so either side can change independently</p>
</li>
</ul>
<p>We deploy multiple proxy instances across regions, each paired with its own Gradient AI Agent. The agents are provisioned through Terraform, which eliminates the configuration drift that creeps in when you set things up by hand through a UI. If one proxy goes down, the load balancer redirects traffic to another instance. No single point of failure.</p>
<p><img alt="image alt text" src="https://doimages.nyc3.cdn.digitaloceanspaces.com/002Blog/Documentation%20Chatbot%20with%20Gradient%20AI%20Platform/image-1.png" /></p>
</details>
<details class="collapsible" open="">

<h2 id="inference-infrastructure"><a href="#inference-infrastructure">Inference Infrastructure</a><a class="hash-anchor" href="#inference-infrastructure"></a></h2>

<p>When building automation with AI, an important question comes up: what kind of inference layer should you put behind that implementation?</p>
<p>Serverless Inference gives direct access to foundation models for straightforward, high-speed, ephemeral completion calls, with automatic scaling and pay-per-token pricing — predictable when your workload is “call model, get text.”</p>
<p>AI agents address a different shape of workload: multi-step, conversational, retrieval-grounded flows. Under the hood they use the same ephemeral completion calls; what changes is the agent platform — injecting RAG context, tuning parameters such as temperature, rewriting queries, and wiring steps together — so the product emphasis is orchestration and agentic behavior.</p>
<p>Agents are the most direct way to keep answers tied to your domain: they add knowledge-base grounding and orchestration on top of Serverless Inference, so you get retrieval, session-shaped flows, and managed RAG wiring in one product path with fewer moving parts.</p>
<p>In practice that showed up as defaults we did not have to build or operate separately. Retrieved chunks are re-ranked for relevance before the model answers, so grounding improves without wiring a custom ranker. The Gradient Agent keeps session context across turns, which makes follow-up questions behave like a conversation instead of a fresh completion each time. When we indexed documentation, the platform handled cleaning and chunking automatically, which cut manual ingestion work and kept retrieval behavior steadier across a large doc set.</p>
<p>Under the hood, the Gradient AI Agent still runs on managed inference — the platform abstracts the exact serving stack so we focus on prompts, retrieval, and evaluation. Teams that need full control over serving (custom images, pinning hardware, or bespoke scaling) may still prefer GPU Droplets, Bare Metal GPUs, or DigitalOcean Kubernetes (DOKS) with their own inference layer; that is a valid alternative when “managed agent endpoint” is not the right tradeoff.</p>
<p>For us, the combination of a managed agent + KB + evaluations matched the bar for speed of iteration and operational simplicity.</p>
</details>
<details class="collapsible" open="">

<h2 id="data-driven-approach-to-validation"><a href="#data-driven-approach-to-validation">Data Driven Approach to Validation</a><a class="hash-anchor" href="#data-driven-approach-to-validation"></a></h2>

<p>Reliability, scalability, and other traditional development best practices are important, but in AI development, they are worthless if the agent itself is consistently hallucinating or fails to solve the problem it was designed to address.</p>
<h3 id="metrics"><a href="#metrics">Metrics</a><a class="hash-anchor" href="#metrics"></a></h3>
<p>It became obvious early that changing prompts or agent settings without measuring outcomes was a waste of time.</p>
<p>There are several metrics you could track. Some metrics rely on the LLM as a judge and require more time and resources to run, while others are fast, inexpensive, and deterministic.</p>
<p>We settled on a mix of LLM-judged and deterministic metrics:</p>
<p><strong>Correctness</strong> (LLM as a judge)</p>
<p>An LLM judge evaluates whether the response is factually accurate, which catches open-domain hallucinations — fabricated information that isn’t grounded in any source document.</p>
<p><strong>Ground truth adherence</strong> (LLM as a judge)</p>
<p>Measures whether the response is semantically equivalent to a known-good reference answer. Correctness asks “is this true?” Ground truth adherence asks “is this the right answer to this specific question?” A response can be factually correct but miss what the user was actually asking about.</p>
<p><strong>Time to first token</strong> (deterministic)</p>
<p>Tells us whether the agent is responding within an acceptable window — users notice latency before they notice anything else.</p>
<p><strong>URL correctness</strong> (deterministic)</p>
<p>Extracts every DigitalOcean documentation link from the response using regex and verifies each one returns a successful HTTP response. Our agent is supposed to point users to the right docs page. Dead links undermine that immediately.</p>
<p>To ship our Product Documentation agent, we chose <strong>80% ground truth adherence and 95% correctness</strong> (testing details available upon request) as our release bar — thresholds we picked after weighing accuracy risk, user impact, and what our evaluation pipeline could support reliably.</p>
<p>Getting to those numbers required multiple iterative adjustments. Here is what moved each metric.</p>
<h4 id="correctness-and-ground-truth-adherence">Correctness and ground truth adherence</h4>
<p>Correctness and ground truth adherence were built through layered prompt engineering:</p>
<ul>
<li>
<p>Product identification with keyword mappings to route questions to the right product context (see Prompt Engineering).</p>
</li>
<li>
<p>Chunk validation that blocks cross-product blending at retrieval time. Cross-product blending (e.g., answering an App Platform question with Kubernetes docs) is explicitly forbidden. This is a guardrail against RAG returning semantically similar but factually wrong context.</p>
</li>
<li>
<p>Retrieval failure awareness that prevents the agent from denying features that exist when retrieval is simply missed. The prompt contains a sophisticated heuristic to distinguish “the docs don’t cover this” from “retrieval just failed”.</p>
</li>
</ul>
<p>We also switched the <a href="https://docs.digitalocean.com/products/gradient-ai-platform/how-to/test-agents/#retrieval-settings" rel="ugc nofollow noopener" target="_blank">retrieval method from rewrite to sub-queries</a> for better chunk relevance on complex questions. The golden datasets went through numerous passes to fix factually wrong ground truth (like monitoring webhooks and GPU model errors), disambiguate vague questions by adding product context, and add a dedicated validation set that verifies the agent asks clarifying questions on ambiguous terms.</p>
<p>We tuned the model to temperature 0.1 with k=10 retrieval and added strict response constraints: no exhaustive lists; when the documentation states something plainly, answer in the same direct terms; for pricing and plans, use what appears in the retrieved docs or link readers to the official pricing page when those details are not in context; and default to Control Panel steps unless the question asks for the API.</p>
<p>We also overhauled the golden dataset itself: correcting inaccurate answers, removing unanswerable community questions where long answers didn’t match the agent’s concise style, reducing question ambiguity, and adding per-product topic tags for granular tracking.</p>
<h4 id="ttft-time-to-first-token">TTFT (time to first token)</h4>
<p>Three changes moved this metric:</p>
<ul>
<li>
<p>Reduced the retrieval parameter <code>k</code> from 10 to 5 (fewer chunks to process before generating)</p>
</li>
<li>
<p>Shortened the system prompt by removing redundant sections and condensing the Special Cases block to stay under the 10k character limit</p>
</li>
<li>
<p>Added the “no exhaustive lists” directive so the model starts outputting a concise answer sooner instead of planning a long enumeration</p>
</li>
</ul>
<h4 id="url-correctness">URL correctness</h4>
<p>URL correctness was hardened separately from correctness by building a custom URL correctness metric and scorer that extracts URLs from agent responses and validates they resolve. We also iteratively hardened the extraction regex to handle edge cases: trailing punctuation, trailing <code>&gt;</code> characters, right square brackets, and periods that are legitimately part of a URL path.</p>
<p>On the prompt side, we added explicit rules:</p>
<ul>
<li>
<p>Use URLs exactly as they appear in documentation chunks</p>
</li>
<li>
<p>Never fabricate, infer, or extend URLs</p>
</li>
<li>
<p>Each URL appears once per response</p>
</li>
<li>
<p>Format as Markdown: <a href="URL" rel="ugc nofollow noopener" target="_blank">text</a></p>
</li>
<li>
<p>Exceptions: URLs may be omitted for clarifications, out-of-scope queries, or “no documentation” responses</p>
</li>
</ul>
<p>We also verified the knowledge base itself (via OpenSearch chunk inspection notebooks) to confirm it only contained <code>docs.digitalocean.com</code> URLs and not community or third-party domains, eliminating a source of invalid URLs at the retrieval layer.</p>
<h3 id="golden-datasets"><a href="#golden-datasets">Golden Datasets</a><a class="hash-anchor" href="#golden-datasets"></a></h3>
<p>Our evaluations require “golden datasets” of question-and-answer pairs that we consider correct. For example, a real golden dataset that we use looks like:</p>
<div class="code-label" title=""></div>
<pre><code>question: How do I rotate my DigitalOcean API tokens securely?
answer: &gt;
  Create a new API token on the API page, update your applications to      
  use the new token, then revoke the old token. This ensures continuous service   
  while maintaining security by eliminating access through the old token.
product: api
reasoning: moderate
source: synthetic
type: how_to_configuration
</code></pre>
<p>The above example contains a question, ground truth answer, and additional metadata such as:</p>
<ul>
<li>
<p>Product area/type - allows us to zoom in on product domains that are not performing well or need more test coverage.</p>
</li>
<li>
<p>Reasoning - gives visibility into the level of difficulty for an evaluation, which is useful when looking at results over time. For light questions, we expect higher results since they are “easier” to answer. For heavy questions, we expect more variability over time since these questions are more nuanced and have higher complexity.</p>
</li>
<li>
<p>Source - the origin of the question such as human-generated or synthetic.</p>
</li>
</ul>
<p>Our golden datasets are sent to the <a href="https://docs.digitalocean.com/products/gradient-ai-platform/how-to/evaluate-agents/" rel="ugc nofollow noopener" target="_blank">Gradient AI Platform</a> where metric results are computed. Evaluations use an LLM-as-a-Judge approach with multiple judges running OpenAI GPT-4o. The judges employ Chain-of-Thought (CoT) reasoning to generate a numeral score between 0 and 1. For example, a golden dataset could return a correctness score of .66 meaning 2 out of 3 judges found the response to be correct.</p>
<p>Once results are created, the metrics and metadata are sent as telemetry via OTLP to our observability cluster for future observability and analysis.</p>
<h3 id="creating-datasets"><a href="#creating-datasets">Creating Datasets</a><a class="hash-anchor" href="#creating-datasets"></a></h3>
<p>There are several ways to create golden datasets.</p>
<ol>
<li>
<p><strong>Human-generated</strong> - Subject matter experts manually write question-and-answer pairs. While this is the most reliable method to create golden datasets, we quickly realized it was extremely time-consuming for teams.</p>
</li>
<li>
<p><strong>Using existing content</strong> - DigitalOcean has resources, such as <a href="https://docs.digitalocean.com/support/" rel="ugc nofollow noopener" target="_blank">support</a> and <a href="/community">community</a>, which provide gold-standard datasets based on customer support materials. This approach maintains quality while reducing manual effort, however there was still human-work necessary to identify good examples.</p>
</li>
<li>
<p><strong>Synthetic generation</strong> - LLMs generated pairs using detailed prompts to produce synthetic datasets. The process involved prompting an LLM to crawl product documentation pages 2-3 levels deep and generating multiple diverse QA pairs. While the answers could introduce bias from the LLM, the tradeoff around accuracy is made up by the speed of generation. Humans are still needed to review synthetically generated datasets.</p>
</li>
</ol>
<p>Since the golden datasets are used by both automated systems and humans in various roles — PMs, engineers, and managers — we chose YAML format for readability due the responses having multiple lines and code. The YAML files are then converted to CSV and fed into the Gradient evaluations.</p>
<p>The generated datasets are submitted as pull requests on GitHub to a dedicated team for review and approval. This approach allowed us to add high-quality responses quickly from subject matter experts with minimal effort.</p>
<h3 id="running-evaluations"><a href="#running-evaluations">Running Evaluations</a><a class="hash-anchor" href="#running-evaluations"></a></h3>
<p>Our agent evaluations are used during development and in our CI/CD pipeline on GitHub Actions. During development, engineers can experiment with changes such as prompt updates and run evaluations locally. When changes are merged, our CI/CD pipeline executes evaluations automatically before deployment. We also periodically run our full evaluation suite once a day to verify our documentation agent continues to operate as we expect.</p>
<p><img alt="image alt text" src="https://doimages.nyc3.cdn.digitaloceanspaces.com/002Blog/Documentation%20Chatbot%20with%20Gradient%20AI%20Platform/image-6.png" /></p>
</details>
<details class="collapsible" open="">

<h2 id="agent-configuration-decisions"><a href="#agent-configuration-decisions">Agent Configuration Decisions</a><a class="hash-anchor" href="#agent-configuration-decisions"></a></h2>

<p>Once evaluations and metrics were in place, it became possible to iterate on areas important to us, such as correctness and speed, using objective measurements instead of one-off tests and vibes.</p>
<h3 id="prompt-engineering"><a href="#prompt-engineering">Prompt Engineering</a><a class="hash-anchor" href="#prompt-engineering"></a></h3>
<p>From our evaluations, we noticed that the primary area that kept our ground truth adherence metric below 80% and our correctness metric below 95% was that the LLM would decide on its own what ambiguous words meant or how to give suggestions to perform an action. For example, consider the following prompt:</p>
<blockquote>
<p>How do I create a Droplet on DigitalOcean?</p>
</blockquote>
<p>Based on our results using Gradient evaluations, we discovered that our agent would vary how it responded to this question over multiple evaluation runs. Sometimes the agent would return responses to create Droplets on the Control Panel, other times it would suggest the Public API or doctl. Since the agent lives in the Control Panel, we added the following to our prompt to help ensure that ambiguous requests responded with Control Panel instructions:</p>
<div class="code-label" title=""></div>
<pre><code>- **If public API or automation is NOT specified in the question:**
  - Use ONLY Control Panel chunks in your primary answer
  - If only public API chunks are retrieved, ask clarifying question: &quot;Would you like to know how to do this in the
    Control Panel or via the API?&quot;
  - You may mention API/CLI methods in the Recommendation section as alternative automation options
</code></pre>
<p>We also discovered that our golden datasets were too ambiguous with prompts such as:</p>
<blockquote>
<p>How do I automate deleting files older than 30 days?</p>
</blockquote>
<p>Questions like these would return varying responses since files can live on a variety of different DigitalOcean products. This prompted us to make two changes:</p>
<ol>
<li>
<p>We updated our golden dataset questions to be more specific. In this case, we changed the above prompt to be “How do I automate deleting files older than 30 days <strong>on Spaces</strong>”.</p>
</li>
<li>
<p>We added the following to our agent prompt:</p>
</li>
</ol>
<div class="code-label" title=""></div>
<pre><code>**If uncertainty exists**, ask clarifying question: &quot;Are you asking about [Product A] or [Product B]?&quot;
</code></pre>
<p>Finally, we found that certain words in prompts would return inconsistent results. The following prompt adjustment helped our agent relate different words to specific products. In cases where words are too ambiguous, we added a specific clarification prompt to help improve our agent performance.</p>
<div class="code-label" title=""></div>
<pre><code>Identify keywords that map to specific products:
t
- &quot;app&quot; / &quot;deployment&quot; / &quot;build&quot; → App Platform
- &quot;volume&quot; → Block Storage
- &quot;registry&quot; / &quot;container&quot; / &quot;DOCR&quot; → Container Registry
- &quot;droplet&quot; / &quot;VM&quot; / &quot;server&quot; → Droplets
- &quot;workflow&quot; / &quot;gradient&quot; / &quot;notebook&quot; / &quot;gen ai&quot; → Gradient AI Platform
- &quot;1-click&quot; → Marketplace
- &quot;function&quot; / &quot;serverless&quot; → Serverless Functions
- &quot;spaces&quot; / &quot;bucket&quot; → Spaces
- These words alone are ambiguous → ask clarification:
  - &quot;image&quot;
  - &quot;plan&quot;
  - &quot;quota&quot;
  - &quot;storage&quot;
  - &quot;subscription&quot;
  - &quot;tag&quot;

</code></pre>
<h3 id="experimentation"><a href="#experimentation">Experimentation</a><a class="hash-anchor" href="#experimentation"></a></h3>
<p>Because LLM responses are non-deterministic, we leaned on the metrics framework to compare configurations instead of eyeballing outputs.</p>
<ul>
<li><strong>Splitting the knowledge base (KB).</strong> We divided our single OpenSearch knowledge base (all product documentation) into smaller, product-scoped KBs. The hypothesis: smaller, scoped doc sets would produce more relevant results at retrieval time. It didn’t play out that way. We saw worse performance with increased complexity of managing multiple KBs with overlapping content.</li>
</ul>
<p><img alt="image alt text" src="https://doimages.nyc3.cdn.digitaloceanspaces.com/002Blog/Documentation%20Chatbot%20with%20Gradient%20AI%20Platform/image-3.png" /></p>
<p>Due to the non-deterministic nature of the results, we ran the same set of tests multiple times to obtain a reliable average. Each point is the overall correctness for a full test run.</p>
<p>Yellow = baseline (one shared KB, one agent). Green = one agent, four product KBs. Blue = shared KB plus four product KBs</p>
<ul>
<li><strong>Dedicated agents per knowledge base.</strong> Next, we checked whether correctness improved if each KB got its own dedicated agent. We hypothesized that giving each smaller, product-scoped knowledge base its own dedicated child agent under a main orchestrator would narrow retrieval and improve correctness compared to a single agent on one common KB.</li>
</ul>
<p>This one showed results: roughly ~4% correctness improvement. That’s meaningful, and it might be an approach worth trying in production, but it comes with real costs. Maintaining one agent per product significantly increases operational overhead. So we haven’t committed to that path; one agent + one common KB is still what we run today.</p>
<p><img alt="image alt text" src="https://doimages.nyc3.cdn.digitaloceanspaces.com/002Blog/Documentation%20Chatbot%20with%20Gradient%20AI%20Platform/image-4.png" /></p>
<p>Yellow = baseline (one KB, one agent). Green = four small KBs, each with its own agent, no shared KB. Blue = four agents on small KBs plus a shared KB on the main agent.</p>
<p>None of these decisions would have been possible without having evaluation metrics in place. All of the changes above used our data driven approach to measure, adjust, and measure again to ensure that we meaningfully improved our agent’s performance.</p>
</details>
<details class="collapsible" open="">

<h2 id="top-3-must-dos-when-creating-an-ai-agent-for-production"><a href="#top-3-must-dos-when-creating-an-ai-agent-for-production">Top 3 Must-Dos When Creating an AI Agent for Production</a><a class="hash-anchor" href="#top-3-must-dos-when-creating-an-ai-agent-for-production"></a></h2>

<p>The work that made our agent reliable enough to ship wasn’t the work we expected to spend time on. It was provisioning agents through Terraform instead of clicking through a UI, building golden datasets before we had a single prompt worth testing, and wiring evaluations into CI so that no change went live without proof it helped. Here’s what that looks like in practice:</p>
<h3 id="1-treat-your-agent-like-a-production-system"><a href="#1-treat-your-agent-like-a-production-system">1. Treat your agent like a production system</a><a class="hash-anchor" href="#1-treat-your-agent-like-a-production-system"></a></h3>
<p>If it’s user-facing, it needs monitoring, redundancy, and infrastructure-as-code — the same bar as every other service in your stack. We use Terraform for agent provisioning and multi-region deployment for availability. The agent is a service, not a side project.</p>
<h3 id="2-define-good-and-measure-it"><a href="#2-define-good-and-measure-it">2. Define “good” and measure it</a><a class="hash-anchor" href="#2-define-good-and-measure-it"></a></h3>
<p>Pick metrics that match your product requirements, build golden datasets, and run evaluations. Don’t rely on assumptions or intuition — our first real evaluation runs surfaced correctness problems on questions we thought were easy. Spot-checking a handful of prompts is not a substitute for an automated evaluation suite.</p>
<h3 id="3-only-ship-changes-that-improve-the-numbers"><a href="#3-only-ship-changes-that-improve-the-numbers">3. Only ship changes that improve the numbers</a><a class="hash-anchor" href="#3-only-ship-changes-that-improve-the-numbers"></a></h3>
<p>Continuously run evaluations against a production-like environment, and only make changes to prompts, retrieval, or models when the metrics indicate improvement. Automate this loop so quality doesn’t depend on manual process.</p>
</details>
<details class="collapsible" open="">

<h2 id="build-and-scale-ai-applications-on-digitalocean"><a href="#build-and-scale-ai-applications-on-digitalocean">Build and scale AI applications on DigitalOcean</a><a class="hash-anchor" href="#build-and-scale-ai-applications-on-digitalocean"></a></h2>

<p>We built this agent* on <a href="/products/gradient">DigitalOcean’s Agentic Inference Cloud</a>, which pairs inference-optimized compute with the full-stack cloud to support production AI — managed databases, object storage, Kubernetes, networking, and more. <a href="/products/gradient/platform">Gradient AI Platform</a> is where the agent came together, with built-in knowledge bases, evaluations, and the iteration loop that got us to launch.</p>
<p>Teams like <a href="http://Character.ai" rel="ugc nofollow noopener" target="_blank">Character.ai</a> and Workato run inference at scale on the same infrastructure, and they’re sharing how at <a href="/deploy">Deploy on April 28 in San Francisco</a>.</p>
<p>*This agent is designed to navigate the DigitalOcean docs like a pro, but it’s not an oracle. It can still make mistakes. Use the code and advice provided as a guide, and always keep your best architectural judgment in the driver’s seat.</p>
</details>
