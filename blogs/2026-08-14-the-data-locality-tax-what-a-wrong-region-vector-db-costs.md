---
title: "The Data Locality Tax: What a Wrong-Region Vector DB Costs Your RAG Pipeline"
url: "https://www.digitalocean.com/community/tutorials/vector-database-latency-rag"
date: "2026-08-14"
author: "Anish Singh Walia"
feed_url: "https://www.digitalocean.com/rss/community/tutorials.atom"
---
Vector database for RAG latency is often geography, not HNSW. A live pgvector test: 1.9 ms same region vs 67 ms NYC to SFO. Measure the path before you tune the index.
