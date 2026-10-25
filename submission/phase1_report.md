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
| IaaS VM | <!--answer:vm_cold_ms-->241.8<!--/answer--> | <!--answer:vm_warm_median_ms-->211.4<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->2020.6<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->636.6<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->739.0<!--/answer--> | <!--answer:function_warm_median_ms-->637.6<!--/answer--> |

Include your histogram or plot of the warm distribution below (an image, or the ASCII chart
`measure.py` prints). This is read by a human, not the grader:

<!--answer:latency_plot-->
Phase 1 — deployment measurements
------------------------------------------------------------
[OK  ] vm        cold=  241.8ms  warm_median=  211.4ms
            #############
[OK  ] cloudrun  cold= 2020.6ms  warm_median=  636.6ms
            ########################################
[OK  ] function  cold=  739.0ms  warm_median=  637.6ms
            ########################################
------------------------------------------------------------
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
The VM does not create anything automatically; it has only one container running on a single instance, so its hard limit is fixed by the resources of the --machine-type=e2-micro. In contrast, both Cloud Run and Cloud Functions automatically spin up new container instances as traffic spikes. Their hard limit comes from the --max-instances=3 flag I configured during deployment. Once 3 instances are fully loaded, neither platform creates more, and incoming requests queue or drop.
<!--/answer-->

**Deployment effort.** What did you have to build, configure and maintain to get each of the
three running? (~100 words)

<!--answer:deployment_effort-->
The VM took by far the most effort: I had to create a custom VPC and subnet, set up ingress firewall rules for ports 22 and 8080, launch the instance, grant IAM permissions to pull images, SSH in, install Docker, authenticate with Artifact Registry, and run the container manually. Cloud Run took moderate effort because I had to write a multi-stage Dockerfile, build the image with Cloud Build, push it to Artifact Registry, and deploy the service. Cloud Functions was the easiest: I only needed the Python source file and requirements, leaving image creation, server configuration, and networking to Google.
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

With Cloud Run and your Function both deployed, run `gcloud run services list` and paste the
**complete output** here, unedited:

<!--answer:run_services_list-->
SERVICE: echo
REGION: us-central1
URL: https://echo-426580260498.us-central1.run.app
LAST DEPLOYED BY: service-426580260498@gcf-admin-robot.iam.gserviceaccount.com
LAST DEPLOYED AT: 2026-10-25T17:59:32.853897Z
SERVICE: echo-bot
REGION: us-central1
URL: https://echo-bot-426580260498.us-central1.run.app
LAST DEPLOYED BY: eugeniehuang.fr@gmail.com
LAST DEPLOYED AT: 2026-10-25T17:53:02.353456Z
<!--/answer-->

Now explain what that output tells you about how your Function is actually being run, why its
cold start resembles Cloud Run's so closely, and what — if anything — genuinely differs between
the two deployment paths. (~120 words)

<!--answer:convergence_explanation-->
The output shows both echo-bot and echo listed as Cloud Run services, with echo deployed by the gcf-admin-robot service account. This proves that Cloud Functions gen 2 is literally running on Cloud Run under the hood. When I deploy the function, Cloud Build automatically turns the code into a container image and deploys it as a Cloud Run service. Because both use the exact same underlying Cloud Run infrastructure and container runtime, their scale-from-zero process and warm routing performance are practically identical (~637ms). The only real difference is who handles containerization: with Cloud Run I had to write and maintain the Dockerfile myself, while Cloud Functions builds and manages the container image automatically using Buildpacks.
<!--/answer-->
