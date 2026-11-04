# Phase 1 — Comparison Report

## 1. Setup facts

**Your image size**, in MB, from `gcloud artifacts docker images list` (number only, e.g. `142`):

<!--answer:image_size_mb-->
50.2
<!--/answer-->

**The custom VPC and subnet you created** (names only):

<!--answer:vpc_name-->
echo-net
<!--/answer-->

<!--answer:subnet_name-->
echo-subnet
<!--/answer-->

The firewall rules `allow-echo-8080` and `allow-echo-ssh` target only instances carrying the network tag `echo-http`, so they open ports 8080 and 22 on the VM and on nothing else in the network. The image size is 50249659 bytes (about 50.2 MB), well under the 400 MB target thanks to the multi-stage build.

---

## 2. Measurements

| Platform | Cold (ms) | Warm median (ms) |
|---|---|---|
| IaaS VM | <!--answer:vm_cold_ms-->237.1<!--/answer--> | <!--answer:vm_warm_median_ms-->210.4<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->2144.8<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->632.9<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->1868.0<!--/answer--> | <!--answer:function_warm_median_ms-->633.6<!--/answer--> |

<!--answer:latency_plot-->
cat > submission/phase1_report.md <<'EOF'
# Phase 1 — Comparison Report

## 1. Setup facts

**Your image size**, in MB, from `gcloud artifacts docker images list` (number only, e.g. `142`):

<!--answer:image_size_mb-->
50.2
<!--/answer-->

**The custom VPC and subnet you created** (names only):

<!--answer:vpc_name-->
echo-net
<!--/answer-->

<!--answer:subnet_name-->
echo-subnet
<!--/answer-->

The firewall rules allow-echo-8080 and allow-echo-ssh target only instances carrying the network tag echo-http, so they open ports 8080 and 22 on the VM and on nothing else in the network. The image size is 50249659 bytes (about 50.2 MB), well under the 400 MB target thanks to the multi-stage build.

---

## 2. Measurements

| Platform | Cold (ms) | Warm median (ms) |
|---|---|---|
| IaaS VM | <!--answer:vm_cold_ms-->237.1<!--/answer--> | <!--answer:vm_warm_median_ms-->210.4<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->2144.8<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->632.9<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->1868.0<!--/answer--> | <!--answer:function_warm_median_ms-->633.6<!--/answer--> |

<!--answer:latency_plot-->
[OK  ] vm        cold=  237.1ms  warm_median=  210.4ms
            #############
[OK  ] cloudrun  cold= 2144.8ms  warm_median=  632.9ms
            ########################################
[OK  ] function  cold= 1868.0ms  warm_median=  633.6ms
            ########################################
<!--/answer-->

The VM has no meaningful cold start: its container never stops, so the 27 ms gap between cold and warm is noise. Cloud Run and the Function pay about 1.2 to 1.5 s extra on the first request after idle (instance start, image load, gunicorn boot). Their warm latency (about 633 ms) is higher than the VM's (about 210 ms). My hypothesis, not measured separately, is the extra TLS handshake: the VM is queried over plain HTTP while the other two use HTTPS, and the client is in France while everything runs in us-central1.

---

## 3. Per-platform analysis

<!--answer:scaling_behaviour-->
VM: nothing is created automatically. It is one fixed e2-micro (shared vCPU, 1 GB RAM) running one gunicorn worker with 8 threads, so the hard limit comes from the machine size I picked: about 8 concurrent requests, and anything beyond that queues. Cloud Run: the autoscaler creates new container instances (copies of my image) as concurrent requests grow, from 0 up to N; the hard limit is the flag I passed, --max-instances=3. Cloud Function: the same autoscaler creates new function instances (also containers) from 0 up to N; I only set --min-instances=0, so the cap is the platform default rather than a value I chose.
<!--/answer-->

<!--answer:deployment_effort-->
VM (most effort): I built a custom VPC, a subnet and two firewall rules (8080 and 22, scoped by tag), created the e2-micro, granted its service account Artifact Registry read access, SSHed in, installed Docker, authenticated it and ran the container. I also remain responsible for OS patching, restarting Docker and the external IP, which changes after a stop and start. Cloud Run: once the image was in Artifact Registry, a single deploy command was enough, with HTTPS, scaling and a URL included and no OS to maintain. Cloud Function (least effort): one command from source, no Dockerfile at all; Google builds the container. The price is less control over the runtime and a fixed entry-point model.
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

<!--answer:run_services_list-->
SERVICE: echo
REGION: us-central1
URL: https://echo-976585416414.us-central1.run.app
LAST DEPLOYED BY: service-976585416414@gcf-admin-robot.iam.gserviceaccount.com
LAST DEPLOYED AT: 2026-11-04T12:42:01.483284Z

SERVICE: echo-bot
REGION: us-central1
URL: https://echo-bot-976585416414.us-central1.run.app
LAST DEPLOYED BY: soren.faucon@gmail.com
LAST DEPLOYED AT: 2026-11-04T12:38:24.195488Z
<!--/answer-->

<!--answer:convergence_explanation-->
The output shows two Cloud Run services: echo-bot, which I deployed, and echo, deployed by the Cloud Functions service agent (gcf-admin-robot). So a 2nd-gen Cloud Function is not a separate runtime: it is a Cloud Run service whose container Google builds from my source. Its cold start therefore goes through the same steps as my own service: schedule an instance, load the container image, start the process, serve the first request. That is why both cold starts (about 1.9 s and 2.1 s) and both warm medians (633 ms) are so close. What genuinely differs is packaging and programming model: for Cloud Run I write the Dockerfile and gunicorn command and push an image, whereas for the Function I deploy source with an entry point and Google generates the container. The management layer and the abstraction differ, not the platform underneath.
<!--/answer-->
