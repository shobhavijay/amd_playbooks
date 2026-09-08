# Moving a running AI agent between clusters on AMD MI300X

**Result:** a multi-step AI agent, mid-batch, relocated from one virtual cluster to
another on an 8×MI300X node — resuming at the next unit of work with nothing
reprocessed, verified against four independent artifacts.

**Validated on:** 8× AMD Instinct MI300X (gfx942) · Ubuntu 24.04 · k3s v1.36 ·
SkyPilot 0.13 · vcluster 0.36 · vLLM (ROCm) · single node.

---

## What this demonstrates

An agent works through a batch of recipe photographs. For each one it reads the
image with a vision model, researches the dish, generates a PDF, and indexes it.

Mid-batch, the cluster it is running in is blocked and the agent is killed
outright — no graceful drain. SkyPilot brings it up in the other cluster, where
it **skips the work already finished and continues**.

The interesting half is not the relocation. SkyPilot does that in six lines of
YAML. The interesting half is that the agent remembers anything at all: a
relocated agent is a brand-new process with no knowledge that a predecessor
existed. Left alone it would restart the batch and re-buy every inference call
already paid for.

Two design decisions make the difference:

- **The unit that moves holds no GPU.** The agent is an HTTP client of
  `vllm serve`. The model servers stay put, one set per tenant. Killing a pod
  that owns no GPU never exercises the driver's device-reclaim path.
- **The unit that moves holds no state.** Progress, vectors and outputs live in
  services outside both clusters, reachable over TCP from either.

```
                    ┌──────────────────────────────────────┐
   HOST CLUSTER     │  Postgres    Chroma     MinIO        │  state store
                    │  checkpoints vectors    PDFs         │  external
                    └──────────────────────────────────────┘
                              ▲                  ▲
                              │                  │  same endpoints
                              │                  │  from either tenant
   ┌──────────────────────────┴──┐   ┌───────────┴──────────────┐
   │ vcluster team-a             │   │ vcluster team-b          │
   │   vLLM  text(2) vision(1)   │   │   vLLM  text(2) vision(1)│
   │   agent  0 GPU  ───────────────────►  agent  0 GPU         │
   └─────────────────────────────┘   └──────────────────────────┘
              8 × MI300X, 6 allocated to vLLM, 2 spare
```

---

## 1. Base stack

### Get the code onto the box

The agent source ships as a tarball. Copy it to the MI300X host and unpack it:

```bash
# from your laptop
scp culinary-archivist-agent.tar.gz <user>@<mi300x-host>:~/

# on the host
tar -xzf ~/culinary-archivist-agent.tar.gz
cd ~/CulinaryArchivist_vLLM_ray_skypilot
```

Every command from here on is run from inside that directory — the steps below
use relative paths like `./k8s/state-plane-gate.sh` and `docker build .`.

### Confirm the host sees the GPUs

```bash
rocm-smi --showproductname          # expect all 8 listed
```

### k3s

```bash
curl -sfL https://get.k3s.io | sh -
mkdir -p ~/.kube && sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown "$USER": ~/.kube/config
export KUBECONFIG=~/.kube/config
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bash_profile
kubectl get nodes -o wide
```

The `KUBECONFIG` export is not optional. `kubectl` is a k3s symlink that
otherwise reads `/etc/rancher/k3s/k3s.yaml`, which will never contain the
vcluster contexts created in section 2.

### AMD device plugin

Kubernetes does not know about GPUs until something advertises them as the
extended resource `amd.com/gpu`. The lightweight device plugin does exactly that
and assumes ROCm is already on the host, which most AMD cloud images provide.

```bash
kubectl create -f https://raw.githubusercontent.com/ROCm/k8s-device-plugin/master/k8s-ds-amdgpu-dp.yaml
kubectl label node "$(kubectl get nodes -o name | head -1 | cut -d/ -f2)" \
  skypilot.co/accelerator=mi300 --overwrite

kubectl get nodes -o json | python3 -c "import sys,json;\
[print(n['metadata']['name'], n['status']['allocatable'].get('amd.com/gpu')) \
for n in json.load(sys.stdin)['items']]"
```
```
0    8
```

### SkyPilot and vcluster

`socat` is required, or `sky check` reports Kubernetes disabled.

```bash
sudo apt update && sudo apt install -y python3-venv python3-pip socat
python3 -m venv ~/sky-venv && source ~/sky-venv/bin/activate
pip install --upgrade pip && pip install "skypilot[kubernetes]"
sky check k8s

curl -L -o vcluster \
  "https://github.com/loft-sh/vcluster/releases/latest/download/vcluster-linux-amd64" \
  && sudo install -c -m 0755 vcluster /usr/local/bin && rm -f vcluster
```

Keep `sky-venv` for SkyPilot alone. Installing unrelated packages into it can
upgrade a shared dependency past SkyPilot's pins and break the CLI.

---

## 2. Two virtual clusters

Both values in the file are required: `sync.fromHost.nodes` so SkyPilot can see
the GPUs (it reads the node view), and `NodePort` so the SkyPilot controller pod
can reach the vcluster APIs — a localhost proxy is unreachable from inside a pod.

```bash
cat > /tmp/vcluster-vals.yaml <<'EOF'
sync:
  fromHost:
    nodes:
      enabled: true
      selector:
        all: true
controlPlane:
  service:
    spec:
      type: NodePort
EOF

kubectl config use-context default
for t in team-a team-b; do
  vcluster create $t -n $t --connect=false -f /tmp/vcluster-vals.yaml
  kubectl --context default -n $t create quota gpu-cap --hard=requests.amd.com/gpu=4
done
```

Build kubeconfigs against the node IP and assigned NodePort:

```bash
NODE_IP=$(kubectl --context default get nodes \
  -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
NP_A=$(kubectl get svc team-a -n team-a -o jsonpath='{.spec.ports[?(@.port==443)].nodePort}')
NP_B=$(kubectl get svc team-b -n team-b -o jsonpath='{.spec.ports[?(@.port==443)].nodePort}')

vcluster connect team-a -n team-a --server="https://$NODE_IP:$NP_A" --print > /tmp/kc-a.yaml
vcluster connect team-b -n team-b --server="https://$NODE_IP:$NP_B" --print > /tmp/kc-b.yaml

cp ~/.kube/config ~/.kube/config.bak
KUBECONFIG=~/.kube/config:/tmp/kc-a.yaml:/tmp/kc-b.yaml \
  kubectl config view --flatten > /tmp/merged && mv /tmp/merged ~/.kube/config
```

**Gate — three contexts, two distinct API servers:**

```bash
kubectl config get-contexts
kubectl --context vcluster_team-a_team-a_default get nodes
kubectl --context vcluster_team-b_team-b_default get nodes
```

If TLS objects to the node IP:
`kubectl config set-cluster <name> --insecure-skip-tls-verify=true`

---

## 3. Register both with SkyPilot

The jobs controller performs the recovery, so it must run on the **host**, never
inside a tenant you intend to break.

```bash
mkdir -p ~/.sky
cat > ~/.sky/config.yaml <<'EOF'
kubernetes:
  allowed_contexts:
    - default
    - vcluster_team-a_team-a_default
    - vcluster_team-b_team-b_default
jobs:
  controller:
    resources:
      infra: k8s/default
      cpus: 4+
EOF
```

Now merge in the pod shaping the agent needs — downward-API identity, the state
plane secret, the shared mount, and the image pull policy:

```bash
./k8s/apply-sky-config.sh          # idempotent; --check to verify only
sky check k8s
```
```
Allowed contexts:
├── default: enabled.
├── vcluster_team-a_team-a_default: enabled.
└── vcluster_team-b_team-b_default: enabled.
```

---

## 4. The external state store

Three services on the **host** cluster: Postgres for checkpoints and the work
ledger, Chroma for vectors, MinIO for generated PDFs. They hold no GPUs and cost
roughly 3 vCPU and 6 GB.

They live on the host for the same reason the controller does — state that must
survive a blocked tenant cannot live inside one.

```bash
./k8s/state-plane-gate.sh
```

The script generates credentials, applies the manifests, waits for rollout,
creates the bucket, and — the part that matters — proves both vclusters can
reach the services:

```
6. Reachability from inside each vcluster
  ✗ team-a → postgres via dns (postgres.archivist-state.svc.cluster.local)
  ✓ team-a → postgres via clusterip (<clusterip>)
```

A pod inside a vcluster resolves DNS through the vcluster's own CoreDNS, which
knows nothing of host-cluster services, so the ClusterIP path is the expected
one. The script detects which works and prints settings accordingly. ClusterIPs
are stable for the life of the Service; re-run the script if you recreate them.

Point the agent at the external state store when you are ready:

```bash
./k8s/state-plane-gate.sh --wire      # reversible with --unwire
```

Deploying and wiring are separate on purpose: running the gate to check
connectivity should never silently change which backend the next run uses.

---

## 5. The agent image

The agent runs as a Kubernetes pod, and a pod starts from an image. The agent's code
and its Python packages have to be built into an image first. The image has to
be on the host before either vcluster can start a pod from it.

```bash
sudo docker build -t culinary-archivist:skypilot .
sudo docker save culinary-archivist:skypilot | sudo k3s ctr images import -
sudo k3s ctr -n k8s.io images ls | grep culinary        # gate
```


---

## 6. Directories, weights, corpus

```bash
sudo mkdir -p /srv/archivist-state/data/recipes /srv/hf-cache
sudo chmod -R 777 /srv/archivist-state /srv/hf-cache
df -h /srv                                   # ~150 GB free
```

Pre-pull the weights so the vLLM pods do not race to download the same 72B into
a shared cache. Use a separate virtualenv — not `sky-venv`:

```bash
python3 -m venv ~/hf-venv && ~/hf-venv/bin/pip install -U huggingface_hub
export HF_HOME=/srv/hf-cache
tmux new -s hf -d "~/hf-venv/bin/hf download Qwen/Qwen2.5-72B-Instruct && \
                   ~/hf-venv/bin/hf download Qwen/Qwen2.5-VL-7B-Instruct"
```

Then the corpus:

```bash
sudo cp data/*.jpeg /srv/archivist-state/data/recipes/
```

To lengthen the batch — useful if you want a comfortable window to trigger the
move — replicate it. Each copy gets a unique request ID stamped into the image,
so the models process genuinely distinct inputs rather than serving one from
cache:

```bash
python3 multiply_images.py 3 --base data --out data_x3 --clean
sudo cp data_x3/*.jpeg /srv/archivist-state/data/recipes/
```

---

## 7. vLLM in both tenants

These pods hold their GPUs for their whole lifetime and **do not** fail over,
which keeps the device-reclaim path out of the demo entirely.

```bash
kubectl --context vcluster_team-a_team-a_default apply -f k8s/vllm-tenant.yaml
kubectl --context vcluster_team-b_team-b_default apply -f k8s/vllm-tenant.yaml
```

The tenant label the agent's banner reads — the only value that differs between
the two clusters:

```bash
kubectl --context vcluster_team-a_team-a_default -n default \
  create configmap tenant-info --from-literal=TENANT_NAME=team-a
kubectl --context vcluster_team-b_team-b_default -n default \
  create configmap tenant-info --from-literal=TENANT_NAME=team-b
```

**Gate — all four pods `1/1`, both tenants answering:**

```bash
kubectl --context default get pods -A | grep vllm

for T in team-a team-b; do echo "== $T"; kubectl --context vcluster_${T}_${T}_default \
  -n vllm run curl-$RANDOM --rm -i --restart=Never --quiet --image=curlimages/curl:latest \
  --command -- sh -c "curl -sf http://vllm-text.vllm.svc.cluster.local:8000/v1/models | head -c 80; echo
                      curl -sf http://vllm-vision.vllm.svc.cluster.local:8001/v1/models | head -c 80"; done
```

The 72B takes minutes to load; the readiness probe allows for it.

**GPU budget:** 2 (text, TP=2) + 1 (vision) = 3 per tenant, so 6 of 8 committed
under the per-tenant quota, with 2 spare. The agent needs none.

Embeddings run on CPU inside the agent. Encoder-architecture embedding models
(BERT-family, such as `bge-m3`) are not supported by vLLM's V1 ROCm attention
backend; a decoder-architecture embedding model would work on GPU if you prefer.

---

## 8. Verify the agent logic without a cluster

No GPU, no Kubernetes, no vLLM. Any Python 3.10+:

```bash
python3 -m venv .venv
grep -E "^(langgraph|python-dotenv)" requirements.txt | sed 's/ *#.*//' > /tmp/tr.txt
./.venv/bin/pip install -r /tmp/tr.txt

./.venv/bin/python tests/test_failover_semantics.py
./.venv/bin/python tests/test_cli_failover.py
```

These cover state-path rooting, endpoint precedence, deterministic thread ids,
durable checkpointing, credential redaction, and — the part the demo turns on —
that a unit of work left pending by an abrupt kill is driven to completion rather
than reported finished.

Two further suites exercise the external state store directly and skip cleanly
without one:

```bash
kubectl --context default -n archivist-state port-forward svc/postgres 5433:5432 &
export ARCHIVIST_PG_HOST=127.0.0.1 ARCHIVIST_PG_PORT=5433 \
  ARCHIVIST_PG_PASSWORD="$(kubectl --context default -n archivist-state get secret \
    archivist-state-secrets -o jsonpath='{.data.POSTGRES_PASSWORD}' | base64 -d)"

./.venv/bin/python tests/test_postgres_checkpointer.py   # cross-process resume
./.venv/bin/python tests/test_postgres_manifest.py       # claims and fencing
```

---

## 9. Launch

```bash
source ~/sky-venv/bin/activate
sky jobs launch -n archivist archivist.sky.yaml --env ARCHIVIST_RUN_ID=demo-01
```

The relocation policy is the whole of what SkyPilot needs from you:

```yaml
resources:
  any_of:
    - infra: k8s/vcluster_team-a_team-a_default
    - infra: k8s/vcluster_team-b_team-b_default
  job_recovery:
    max_restarts_on_errors: 3
```

**Read the banner before trusting anything downstream.** It is printed at
startup, so it appears at the top of the job log:

```bash
timeout 90 sky jobs logs -n archivist 2>&1 | grep "║" | head -20
```
```
║  TENANT      : team-b                                              ║
║  RUN ID      : demo-01                                             ║
║  checkpoints : postgres  <clusterip>:5432/archivist                ║
║  vectors     : chroma server  <clusterip>:8000                     ║
║  blobs       : s3  http://<clusterip>:9000/archivist               ║
║  GPUs held   : none — this process is a vLLM *client*              ║
```

`checkpoints : sqlite …` means the external state store is not wired and the run
will prove nothing. `GPUs held: none` is the zero-GPU premise holding.

Record the starting tenant. `any_of` is a candidate list, not a priority order —
the job may start in either.

---

## 10. Trigger the move

A move requires **two** conditions, not one: end the current attempt, **and**
prevent the replacement landing back where it started. Blocking alone moves
nothing, because quotas gate admission of new pods and never evict running ones.
Killing alone relaunches into the same tenant.

Detect the tenant rather than assuming it:

```bash
unset VICTIM VCTX
for T in team-a team-b; do
  CTX="vcluster_${T}_${T}_default"
  if kubectl --context "$CTX" -n default get pods 2>/dev/null | grep -q archivist; then
    VICTIM=$T; VCTX=$CTX
  fi
done
POD=$(kubectl --context "$VCTX" -n default get pods -o name | grep archivist)
echo "victim=$VICTIM  pod=$POD"
```

**All three must be non-empty before continuing.** Then block, then kill:

```bash
kubectl --context default -n "$VICTIM" create quota block-all --hard=pods=0
kubectl --context "$VCTX" -n default delete "$POD" --grace-period=0 --force
```

Note `pods=0`, not a GPU quota — the agent requests zero GPUs, so a
`requests.amd.com/gpu=0` quota would not block it. The quota goes on the host
namespace, where synced pods are actually admitted.

Leave the tenant's API server reachable. Pausing a vcluster removes its API and
the controller can no longer clean up.

---

## 11. Verify the move

Four independent artifacts. Any one is suggestive; together they are conclusive.

**One agent, in the other tenant:**

```bash
for ctx in vcluster_team-a_team-a_default vcluster_team-b_team-b_default; do
  echo "== $ctx"; kubectl --context $ctx get pods -A 2>/dev/null | grep archivist
done
```
```
== vcluster_team-a_team-a_default
default   archivist-3-af7b1d31-head   1/1   Running   0   2m10s
== vcluster_team-b_team-b_default
```

**The agent continued rather than restarted.** Two banners with different
`TENANT` and identical `RUN ID`, and a skip line:

```bash
timeout 120 sky jobs logs -n archivist 2>&1 | grep -E "TENANT|RUN ID|already completed"
sky jobs queue          # #RECOVERIES: 1
```

**The ledger — the artifact a restart cannot imitate:**

```bash
kubectl --context default -n archivist-state exec -i deploy/postgres -- \
  psql -U archivist -d archivist -c "
SELECT tenant, count(*) FILTER (WHERE status='ok') AS done,
       max(attempt) AS max_attempt, min(claimed_at) AS first
  FROM run_manifest WHERE run_id='demo-01' GROUP BY tenant ORDER BY first;"
```
```
 tenant | done | max_attempt |             first
--------+------+-------------+-------------------------------
 team-b |    6 |           1 | 2026-09-02 23:10:28.241757+00
 team-a |    3 |           1 | 2026-09-02 23:17:11.77199+00
```

One run id, two clusters, and `max_attempt = 1` throughout: **nothing was
processed twice.** Ask which items went where and the two lists share no entry —
the work was divided by the outage, not repeated across it.

```bash
kubectl --context default -n archivist-state exec -i deploy/postgres -- \
  psql -U archivist -d archivist -c "
SELECT tenant, string_agg(file, ', ' ORDER BY updated_at) AS items
  FROM run_manifest WHERE run_id='demo-01' AND status='ok' GROUP BY tenant;"
```

**The outputs, told by a third system.** The gap in timestamps is the move:

```bash
MC_PW=$(kubectl --context default -n archivist-state get secret archivist-state-secrets \
  -o jsonpath='{.data.MINIO_ROOT_PASSWORD}' | base64 -d)
kubectl --context default -n archivist-state run mc-ls-$RANDOM --rm -i --restart=Never \
  --quiet --image=minio/mc:latest --command -- sh -c \
  "mc alias set l http://minio:9000 archivist '$MC_PW' >/dev/null && mc ls l/archivist"
```

Why this verification matters: **a batch that silently restarted looks exactly
like one that resumed** — same logs, same outputs, same success message, same
recovery count. The only difference is inference paid for twice, and nothing in
ordinary output counts it. The attempt column is what makes the distinction
observable.

---

## 12. Reset and teardown

Between runs, clear the block and give the batch a fresh identity. The ledger is
keyed on `(run_id, item)`, so a new run id is a clean run with history intact:

```bash
kubectl --context default -n team-a delete quota block-all --ignore-not-found
kubectl --context default -n team-b delete quota block-all --ignore-not-found
sky jobs launch -n archivist archivist.sky.yaml --env ARCHIVIST_RUN_ID=demo-02
```

To reuse the same run id, truncate instead:

```bash
kubectl --context default -n archivist-state exec -i deploy/postgres -- \
  psql -U archivist -d archivist -c \
  "TRUNCATE run_manifest, checkpoints, checkpoint_writes, checkpoint_blobs;"
```

Full teardown:

```bash
sky jobs cancel -a -y; sky down -a -y
kubectl --context vcluster_team-a_team-a_default delete -f k8s/vllm-tenant.yaml
kubectl --context vcluster_team-b_team-b_default delete -f k8s/vllm-tenant.yaml
kubectl --context default delete -f k8s/state-plane.yaml
kubectl --context default delete ns archivist-state
vcluster delete team-a -n team-a && vcluster delete team-b -n team-b
```

Leave `/srv/hf-cache` alone unless you want to re-download the weights.

---

## Acknowledgements

GPU time for this work was provided by the **AMD Developer Program**, whose
access to 8×MI300X hardware made the multi-tenant failover testing possible.
Thank you.

Built on [SkyPilot](https://github.com/skypilot-org/skypilot),
[vcluster](https://www.vcluster.com), [vLLM](https://github.com/vllm-project/vllm),
[LangGraph](https://github.com/langchain-ai/langgraph) and the
[ROCm k8s device plugin](https://github.com/ROCm/k8s-device-plugin).

---

## Version note

Version-sensitive throughout: SkyPilot's configuration schema, vcluster's values
schema, and vLLM's ROCm engine all move quickly. The commands here reflect the
versions listed at the top. Check current upstream documentation before relying
on them in a different environment.
