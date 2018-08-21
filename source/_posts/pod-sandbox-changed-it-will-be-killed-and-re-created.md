---
title: 'Pod sandbox changed, it will be killed and re-created.'
date: 2018-05-21 08:46:48
categories:
tags: [postgres, k8s]
---

It all started at 6PM Friday night when we were in the middle of our new office warming party - as any admin can tell you it's best time to have a major outage. Our postgres main was down and all the alarms went crazy. We run everything on k8s so we started looking at the `kubectl get event` logs and found this cryptic event: `Pod sandbox changed, it will be killed and re-created.`. 
Googled that and it seemed that it means that the docker server inside the node was down, restarted and then because the "sandbox changed" it restarted all the pods inside it including our beloved pg main. We confirmed that docker crashed by looking at the `journalctl` inside that k8s node (btw, we run on GKE using Container Optimized OS).

Why did the docker crash? Well after some more grepping we've found this nice little script here: [health-monitor.sh](https://github.com/kubernetes/kubernetes/blob/fe36fdde6c87bae720f2b45579e56f07692c3ec0/cluster/gce/gci/health-monitor.sh#L29) (it got changed recently). What that does is that it kills docker after waiting 60s for it to respond. That echo is what we saw in the `journalctl` logs `echo "Docker daemon failed!"` and that is why we knew it was that script that killed docker.

Why did the docker not respond? As it turns out our postgres was doing it's daily base backup at 6PM and because the backup itself was not throttled it consumed lots of CPU and I/O - presumably making docker unresponsive.

To fix it it we throttled our backup script to use less IO and it seemed to fix the problem. Here's how we did it: https://github.com/wal-e/wal-e#controlling-the-i-o-of-a-base-backup

Hope it helps others out there. Happy hacking!