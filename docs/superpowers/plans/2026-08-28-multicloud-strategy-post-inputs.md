# Inputs — multicloud strategy post (measured 2026-08-28)

## Same-claim diffs

Source: `Smana/crossplane-configuration` repo, `examples/` directory, fetched from
GitHub `main` on 2026-08-28 via `gh api repos/Smana/crossplane-configuration/contents/<path>`.

Reading note: in both pairs, every `-`/`+` line above `apiVersion:` is a YAML
**comment** (documentation prose). The actual claim surface differs only in
`metadata.name` for the InferenceService pair, and in `metadata.name` plus the
optional `backup` block for the SQLInstance pair — the `backup` block being the
entire cloud-specific surface of that composition (gs:// vs s3://), set
deliberately in the GCP example so the golden render proves something.

### inferenceservice (aws vs gcp)

```diff
--- inferenceservice-basic.yaml	2026-08-28 09:21:24
+++ inferenceservice-gcp.yaml	2026-08-28 09:21:24
@@ -1,23 +1,26 @@
 ---
-# Basic InferenceService — smallest viable model.
-# Minimal claim: HF model, 1 GPU, defaults applied (min=1 always-warm per
-# SPEC-001), no gateway, no LoRA.
+# GCP InferenceService example — exercises the composition-gcp.yaml branch.
 #
-# `model.preload.enabled: true` is NOT optional here, despite defaulting to
-# false. The serving pod is always launched with `--model /models/<revision>`
-# (a path on the shared S3 Files mount — never an HF repo id), and its
-# CiliumNetworkPolicy allows egress to kube-dns and NOTHING else. So a serving
-# pod cannot reach HuggingFace even in principle: with no preload Job, /models
-# stays empty and vLLM fails to start, permanently.
+# Identical in shape to inferenceservice-basic.yaml, and deliberately so: the
+# claim is cloud-neutral and nothing below says "gcp". Which cloud renders is
+# decided by the EnvironmentConfig the installed Composition selects, which is
+# what apis/inferenceservice/kcl/main.k's `_cloud` reads.
 #
-# The only claim that may leave preload off is one whose
-# `weightsFileSystem.subPath` points at a directory ANOTHER claim already
-# preloaded — that is the deliberate weight-sharing case, and it is why the
-# default is false rather than true.
+# scripts/render_check.py resolves this file to composition-gcp.yaml via the
+# "-gcp" filename marker — see composition_for().
+#
+# What the golden pins that the AWS goldens cannot: a per-claim
+# GCPWorkloadIdentity granting the serving ServiceAccount read access to the
+# weights bucket. AWS needs no such resource, because the EFS CSI driver
+# authenticates at the CONTROLLER and serving pods just mount (ADR-0004, and
+# the note in main.k where the per-claim EPI was dropped). Cloud Storage FUSE
+# authenticates as the MOUNTING POD's own ServiceAccount instead, so without
+# this the mount succeeds and every read 403s — a failure that looks like a
+# broken volume and is actually IAM.
 apiVersion: cloud.ogenki.io/v1alpha1
 kind: InferenceService
 metadata:
-  name: xplane-qwen3-8b-basic
+  name: xplane-qwen3-8b-gcp
   namespace: llm
 spec:
   model:
```

(The rest of both files — the whole `spec:` — is byte-identical: 34 vs 37
lines, all added lines are comments plus the name change.)

### sqlinstance (aws vs gcp)

```diff
--- sqlinstance-basic.yaml	2026-08-28 09:21:25
+++ sqlinstance-gcp.yaml	2026-08-28 09:21:25
@@ -1,23 +1,36 @@
 ---
-# Basic SQLInstance Example
+# GCP SQLInstance example — exercises the composition-gcp.yaml branch.
 #
-# Minimal PostgreSQL database configuration with CloudNativePG.
-# This creates a simple database cluster with sensible defaults.
+# The claim is cloud-neutral: nothing below says "gcp". Which cloud renders is
+# decided by which Composition is installed on the cluster (aws-0 installs
+# composition-aws.yaml, gcp-0 installs composition-gcp.yaml), and
+# apis/sqlinstance/kcl/main.k's `_cloud` branch reads that from the
+# EnvironmentConfig rather than from the claim. That is the whole point of the
+# API being neutral, and it is why this file is byte-for-byte something you
+# could also apply on AWS.
 #
-# For a complete example with all features, see sqlinstance-complete.yaml
-
+# scripts/render_check.py cannot use the claim's content to pick a Composition
+# either, for the same reason — it resolves this file to composition-gcp.yaml
+# via the "-gcp" filename marker. See render_check.py's composition_for().
+#
+# `backup` is set deliberately: it is the ENTIRE cloud-specific surface of this
+# Composition. Without it, the AWS and GCP renders are identical and this
+# fixture would prove nothing. With it, the golden file pins:
+#   - ObjectStore destinationPath  gs://  (not s3://)
+#   - ObjectStore credentials      googleCredentials.gkeEnvironment: true
+#                                  (not s3Credentials.inheritFromIAMRole)
+#   - identity                     one GCPWorkloadIdentity, bucket-scoped
+#                                  (not Role + Policy + Attachment + PodIdentityAssociation)
 apiVersion: cloud.ogenki.io/v1alpha1
 kind: SQLInstance
 metadata:
-  name: my-database
+  name: xplane-gcp-db
   namespace: demo
 spec:
-  # Minimum required configuration (all three are required by the XRD)
-  instances: 1          # Number of PostgreSQL instances; 1 = no HA, fine for dev
-  size: small           # Options: small, medium, large
-  storageSize: 20Gi     # Persistent storage size
+  instances: 1
+  size: small
+  storageSize: 20Gi
 
-  # Create a database and user
   databases:
     - name: myapp
       owner: myapp-user
@@ -25,4 +38,9 @@
   roles:
     - name: myapp-user
       comment: "Application database user"
-      superuser: false  # Required; grant superuser only when genuinely needed
+      superuser: false
+
+  backup:
+    schedule: "0 2 * * *"
+    bucketName: ogenki-435905-ogenki-cnpg-backups
+    retentionPolicy: "30d"
```

## Cost per million tokens

| Cloud | GPU | Instance | $/h (on-demand) | tok/s | $/Mtok |
|---|---|---|---|---|---|
| AWS EKS | 1x NVIDIA L4 (24 GB) | g6.2xlarge (8 vCPU / 32 GiB), eu-west-3 | $1.24095 | PENDING | PENDING |
| GCP GKE | 1x NVIDIA L4 (24 GB) | g2-standard-8 (8 vCPU / 32 GB), europe-west4 | $0.8972 | PENDING | PENDING |

Measurement window: PENDING — not yet measured. Target model:
`xplane-qwen3-8b` = Qwen/Qwen3-8B, quantization fp8, 32k context,
KV-offload 16 GB, maxNumSeqs 32, 1 GPU
(`cloud-native-ref/apps/base/ai/llm/qwen3-8b.yaml`).

### Why tok/s is PENDING (attempted 2026-08-28)

The platform's VictoriaMetrics (vmsingle on each cluster, exposed at
`vm.priv.aws.ogenki.io` / `vm.priv.gcp.ogenki.io` per
`observability/base/victoria-metrics-k8s-stack/httproute-vmsingle.yaml` +
per-cloud `private_domain_name` tfvars) was unreachable from the authoring
machine: the only Tailscale profile present is a different (work) tailnet, both
hostnames fail DNS/TCP (`curl` → HTTP 000 on both `/api/v1/query` and
`/select/0/prometheus/...`), and no ogenki kubeconfig/AWS/GCP credentials exist
locally. Numbers were NOT estimated or fabricated.

### How to measure (from a machine on the ogenki tailnet)

Both clusters run **vmsingle**, so the query path is plain `/api/v1/query`:

```bash
# 1. Discover served models per cluster
curl -sk 'https://vm.priv.aws.ogenki.io/api/v1/query' \
  --data-urlencode 'query=group by (model_name) (vllm:generation_tokens_total)'
curl -sk 'https://vm.priv.gcp.ogenki.io/api/v1/query' \
  --data-urlencode 'query=group by (model_name) (vllm:generation_tokens_total)'

# 2. Peak sustained generation throughput (tokens/s) over the last 30 days
#    (GPUs may be scaled to zero right now — historical window is fine;
#    record the window actually used)
curl -sk 'https://vm.priv.aws.ogenki.io/api/v1/query' \
  --data-urlencode 'query=max_over_time(rate(vllm:generation_tokens_total{model_name="xplane-qwen3-8b"}[10m])[30d:1h])'
# same query against vm.priv.gcp.ogenki.io

# 3. Compute
# cost_per_Mtok = (hourly_cost_usd / (throughput_tok_s * 3600)) * 1_000_000
```

### Sources and caveats

- **AWS $/h**: g6.2xlarge on-demand in eu-west-3 (Paris) = $1.24095/h —
  ec2.shop (data derived from the public AWS Pricing API), queried 2026-08-28
  (`curl 'https://ec2.shop?region=eu-west-3&filter=g6.2xlarge'`).
  For reference: g6.xlarge eu-west-3 = $1.0216/h; g6.2xlarge us-east-1 =
  $0.978/h (instances.vantage.sh, 2026-08-28).
- **GCP $/h**: g2-standard-8 on-demand in europe-west4 (Netherlands) =
  $0.8972/h, **GPU included** (G2 machine prices bundle the L4) —
  gcloud-compute.com/g2-standard-8.html, checked 2026-08-28. Spot in
  europe-west4 = $0.5384/h.
- **Instance choice is inferred, not observed** (clusters unreachable): the
  composition's default pod sizing is requests cpu=2 / memory=16Gi
  (`apis/inferenceservice/kcl/main.k`, `_DEFAULTS`), which does not fit the
  16 GiB nodes (g6.xlarge / g2-standard-4) after system reserves, so
  Karpenter (`instance-family: g6`, `instance-gpu-count: 1`, i.e.
  g6.xlarge–g6.16xlarge) and GKE NAP (ComputeClass `gpu-l4`, `machineFamily:
  g2`, 1x nvidia-l4) both land on the smallest 32 GB single-L4 SKU:
  g6.2xlarge / g2-standard-8. Confirm with `kubectl get nodes -o json | jq`
  on a live scale-up.
- **On-demand vs spot asymmetry**: the table uses on-demand for both clouds
  (per the plan schema), but the platform's actual capacity policy differs —
  the AWS NodePool allows `["spot", "on-demand"]`, while the GKE ComputeClass
  is spot-only with `whenUnsatisfiable: DoNotScaleUp`
  (`infrastructure/gcp-0/computeclass/gpu-l4.yaml`). Any like-for-like $/Mtok
  comparison in the post should say which price basis it uses.
- Regions differ (eu-west-3 vs europe-west4) because that is where each
  cluster actually runs (`opentofu/{aws,gcp}/network/variables.tfvars`).
