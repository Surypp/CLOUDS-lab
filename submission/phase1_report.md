# Phase 1 — Comparison Report

Copy this file to `submission/phase1_report.md`, fill in every answer slot, and commit it:

```bash
cp phase-1-echo-bot/report_template.md submission/phase1_report.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or images freely.

Numbers must **match `submission/phase1_report.json`** exactly — that file is produced by
`measure.py` and is what your figures are checked against. Copy them across; do not retype from
memory.

---

## 1. Setup facts

**Your image size**, in MB, from `gcloud artifacts docker images list` (number only, e.g. `142`):

<!--answer:image_size_mb-->
50
<!--/answer-->

**The custom VPC and subnet you created** (names only):

<!--answer:vpc_name-->
echo-net
<!--/answer-->

<!--answer:subnet_name-->
echo-subnet
<!--/answer-->

---

## 2. Measurements

Copy these from `submission/phase1_report.json`. Cold is `cold_ms`; warm median is
`stats.median`. Numbers only, in milliseconds (e.g. `812.4`).

| Platform | Cold (ms) | Warm median (ms) |
|---|---|---|
| IaaS VM | <!--answer:vm_cold_ms-->257.9<!--/answer--> | <!--answer:vm_warm_median_ms-->211.8<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->1622.1<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->633.9<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->1288.1<!--/answer--> | <!--answer:function_warm_median_ms-->633.5<!--/answer--> |

Include your histogram or plot of the warm distribution below (an image, or the ASCII chart
`measure.py` prints). This is read by a human, not the grader:

<!--answer:latency_plot-->
```
[OK  ] vm        cold=  257.9ms  warm_median=  211.8ms
            #############
[OK  ] cloudrun  cold= 1622.1ms  warm_median=  633.9ms
            ########################################
[OK  ] function  cold= 1288.1ms  warm_median=  633.5ms
            ########################################
```
<!--/answer-->

---

## 3. Per-platform analysis

**Scaling behaviour.** Imagine requests arriving one at a time, then ten at once, then a
hundred. For each of the three platforms, answer concretely:

- Does anything get created automatically to absorb the load — and if so, *what* exactly?
  (One platform here creates nothing at all.)
- What is the hard limit, and where does it come from — a flag you passed yourself, or the
  size of the machine you picked?

(~80 words. Your `--machine-type=e2-micro` and `--max-instances=3` are both concrete answers
to the second question.)

<!--answer:scaling_behaviour-->
For VM: nothing is created automatically, all the requests are queued on the machine. There is a limit however: the machine's size, that depends on --machine-type (e2-micro here, 1 GB of RAM). To be able to handle more workload, the solution could be to add VM(s) by ourselves.

For Cloud Run: Here, container instances are automatically created from the image and the current workload (handled dynamically). An instance can handle multiple requests at the same time. We capped it at 3 (using --max-instances=3). Beyond 3 instances, the requests are queued or refused.

For Cloud Function: Here, instances are created automatically again. This time, only one request per instance. (maxInstanceRequestConcurrency: 1).
<!--/answer-->

**Deployment effort.** What did you have to build, configure and maintain to get each of the
three running? (~100 words)

<!--answer:deployment_effort-->
For VM: we had to build everything ourselves: a custom VPC, a subnet, two firewall rules with a network tag, and an IAM role, so the VM can pull the image. Then we had to connect with SSH, install Docker, authenticate it and start the container. After that, we had to handle the OS and Docker by ourselves.

For Cloud Run: we needed the image first, then only one command. Google handles the servers, HTTPS and scaling. Much easier than VM.

For Cloud Function: only one command with the source code, no Dockerfile. Google builds the image itself. 
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

With Cloud Run and your Function both deployed, run `gcloud run services list` and paste the
**complete output** here, unedited:

<!--answer:run_services_list-->
```
✔
SERVICE: echo
REGION: us-central1
URL: https://echo-687521515164.us-central1.run.app
LAST DEPLOYED BY: service-687521515164@gcf-admin-robot.iam.gserviceaccount.com
LAST DEPLOYED AT: 2026-11-01T06:24:41.697419Z

✔
SERVICE: echo-bot
REGION: us-central1
URL: https://echo-bot-687521515164.us-central1.run.app
LAST DEPLOYED BY: louis.cano@eurecom.fr
LAST DEPLOYED AT: 2026-11-01T06:21:52.712895Z
```
<!--/answer-->

Now explain what that output tells you about how your Function is actually being run, why its
cold start resembles Cloud Run's so closely, and what — if anything — genuinely differs between
the two deployment paths. (~120 words)

<!--answer:convergence_explanation-->
The output shows two Cloud Run services, but we only deployed one with gcloud run: echo-bot (deployed by us) and echo, deployed by gcf-admin-robot. My guess is that Cloud Function gen2 is actually a Cloud Run service created by Google. The deploy output already showed it: the service field points to services/echo and the uri ends with .run.app.

This is why the cold start looks like Cloud Run's. Both runs on the same infrastructure, and a cold start is Cloud Run starting a container instance from zero. And the warm medians are almost the same (2ms difference).

What really differs is the packaging: for Cloud Run we give our own image, for the Function we only give the source code and Google builds the image. However, as we only tried on one code, the gap between 1622.1 and 1288.1 ms is not very reliable.
<!--/answer-->


PS: thanks for giving me more time :)

CANO Louis
