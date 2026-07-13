---
title: "How to Set Up a Private Droplet and Bastion Host with Cloud Firewalls"
url: "https://www.digitalocean.com/community/tutorials/how-to-setup-private-droplet"
date: "2026-07-01"
author: "Anish Singh Walia"
feed_url: "https://www.digitalocean.com/rss/community/tutorials.atom"
---
A Private Droplet has no public network interface, so you can't SSH to it directly. The standard way in is a **bastion host** (jump host): a small Droplet with a public IP, in the same VPC, that you connect through.
