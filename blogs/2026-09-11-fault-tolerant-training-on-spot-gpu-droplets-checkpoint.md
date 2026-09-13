---
title: "Fault-Tolerant Training on Spot GPU Droplets: Checkpoint, Resume, and Keep Your Run Alive"
url: "https://www.digitalocean.com/community/tutorials/spot-gpu-droplets-fault-tolerance"
date: "2026-09-11"
author: "Anish Singh Walia"
feed_url: "https://www.digitalocean.com/rss/community/tutorials.atom"
---
Spot GPU Droplets give you self-serve, hourly access to NVIDIA HGX B300 and AMD Instinct MI350X/MI355X GPUs on DigitalOcean, with the price locked when you create the Droplet. The one design rule is that the Droplet can be reclaimed. This tutorial builds the checkpoint-and-resume loop that makes a training job shrug that off, then runs the drill for real on an MI355X.
