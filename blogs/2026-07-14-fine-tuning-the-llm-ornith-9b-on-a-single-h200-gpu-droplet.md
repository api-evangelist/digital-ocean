---
title: "Fine-tuning the LLM Ornith 9b on a single H200 GPU Droplet: cost, latency, and serving overhead"
url: "https://www.digitalocean.com/community/tutorials/ornith-9b-fine-tuning-cost-benchmark"
date: "2026-07-14"
author: "James Skelton"
feed_url: "https://www.digitalocean.com/rss/community/tutorials.atom"
---
We fine-tuned Ornith-1.0-9B to output structured reasoning summaries instead of raw chain-of-thought, using LLaMA-Factory on a single DigitalOcean H200 GPU Droplet. Full training cost ($84.88), TTFT and throughput benchmarks at 1/5/20 concurrent requests, and a direct comparison against the unmodified base model.
