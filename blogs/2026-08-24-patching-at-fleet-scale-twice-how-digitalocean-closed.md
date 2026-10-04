---
title: "Patching at Fleet Scale, Twice: How DigitalOcean Closed Januscape and the AMD Safe RET Issue Without Customer Impact"
url: "https://www.digitalocean.com/blog/patching-januscape-amd-safe-ret"
date: "2026-08-24"
author: "Tim Lisko"
feed_url: "https://www.digitalocean.com/rss/blog.atom"
---
Setting the stakes In early July, security researcher Hyunwoo Kim discovered Januscape (CVE-2026-53359), a flaw in KVM’s handling of nested virtualization that could allow a malicious guest to escape into the host hypervisor. It was disclosed publicly on July 6 via the Linux oss-security mailing list . For a cloud provider, a guest-to-host escape is the most serious class of vulnerability there is: the hypervisor is the boundary that keeps each customer’s workloads isolated from each other, and from our infrastructure itself.
