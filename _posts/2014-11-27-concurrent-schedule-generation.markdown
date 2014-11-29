---
layout: post
title:  "Using concurrency in university schedule generation"
tags: python threading multiprocessing winston classtime

published: false
---


OLDPOST

---

The multiprocessing approach seemed to affect performance based on two factors: the working set size and the number of worker processes used. I did some good 'ol research to look for an optimal combination of the two.

> **working set size:** the number of "best" schedules which are kept around at any one time. A larger working set means less chance of mistakenly discarding a potentially great schedule.

For each working set size, I varied the number of workers used, ran the benchmark (`tests/test_schedule_generator.py`), and compared those times to the synchronous approach.

Working set of 50 schedules
---
**Synchronous approach: 3.8 seconds with a working set of 50**
![pool50sched-gen-perf](https://cloud.githubusercontent.com/assets/1522678/4966148/2dafa904-67af-11e4-9d35-146f86c5e342.png)

Working set of 200 schedules
--
**Synchronous approach: 12 seconds with a working set of 200**
![pool200sched-gen-perf](https://cloud.githubusercontent.com/assets/1522678/4966149/2db3a2c0-67af-11e4-8f3e-cab8ae605106.png)

Interestingly, the second chart shows that it's possible to **quadruple** the working set size and *still improve performance by about 25%*. This requires at least 20 processes, and at least 32 for optimal performance. That's a lot, but we can potentially pull it off in production:

- we'll have a maximum of 5 web processes
- we'll be limited to 255 processes+threads per dyno
- each web process can handle one schedule generation request at a time
- worst case: all web processes are serving a generation simultaneously
  - that's 5 + 5*32 = 165 processes, which leaves us with 255-165 = 90 processes left over for other tasks.

