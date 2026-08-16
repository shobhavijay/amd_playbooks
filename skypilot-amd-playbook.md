# Moving a GPU workload between two vclusters with SkyPilot on AMD MI300X

The exact commands used to build two GPU-isolated virtual clusters on a single AMD node
and make SkyPilot transition a running workload from one to the other automatically.

**Result achieved:** managed GPU job (`MI300:1`) running in `team-a`, tenant blocked and
pod deleted, SkyPilot recovered it into `team-b` unattended — `#RECOVERIES: 1`, host
healthy, no manual relaunch.

**Environment:** 8× MI300X VF (gfx942) · Ubuntu 24.04 · k3s v1.36 · SkyPilot 0.13 ·
vcluster 0.36 · single node.

---

## 1. Base stack

### Confirm the host sees the GPUs

Before Kubernetes enters the picture, confirm the operating system itself sees the
hardware — every physical GPU listed with its card series.

```bash
# ROCm check — expect all 8 GPUs listed
rocm-smi --showproductname
```

### k3s

```bash
curl -sfL https://get.k3s.io | sh -
mkdir -p ~/.kube && sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown "$USER": ~/.kube/config
export KUBECONFIG=~/.kube/config
kubectl get nodes -o wide
```

### AMD device plugin

Kubernetes doesn't know about GPUs until something advertises them as an extended resource
called `amd.com/gpu`. The **AMD GPU Operator** is the heavier, more complete option — a
Helm release bringing cert-manager, driver lifecycle, a node labeller and metrics — and is
the right answer for production.

The **AMD device plugin** is lightweight and works for this use case: a single DaemonSet
that does one job — advertise the GPUs — and assumes ROCm is already installed on the host.
Most cloud AMD images ship ROCm preinstalled. That's the path used here, and everything
downstream was validated on it.

```bash
# AMD device plugin + SkyPilot's accelerator label
kubectl create -f https://raw.githubusercontent.com/ROCm/k8s-device-plugin/master/k8s-ds-amdgpu-dp.yaml
kubectl label node "$(kubectl get nodes -o name | head -1 | cut -d/ -f2)" \
  skypilot.co/accelerator=mi300 --overwrite

# verify amd.com/gpu is allocatable
kubectl get nodes -o json | python3 -c "import sys,json;\
[print(n['metadata']['name'],n['status']['allocatable'].get('amd.com/gpu')) \
for n in json.load(sys.stdin)['items']]"
```
```
0    8
```

```bash
# SkyPilot — socat is required for its Kubernetes port-forward mode
sudo apt update && sudo apt install -y python3-venv python3-pip socat
python3 -m venv ~/sky-venv && source ~/sky-venv/bin/activate
pip install --upgrade pip && pip install "skypilot[kubernetes]"
sky check k8s
sky gpus list --infra k8s

# vcluster CLI
curl -L -o vcluster "https://github.com/loft-sh/vcluster/releases/latest/download/vcluster-linux-amd64" \
  && sudo install -c -m 0755 vcluster /usr/local/bin && rm -f vcluster
```
```
MI300  1, 2, 4, 8   8 of 8 free
```

---

## 2. Two vclusters, 4 GPUs each

Both settings in the values file are required: `sync.fromHost.nodes` so SkyPilot can see
the GPUs (it reads the node view), `NodePort` so the SkyPilot controller pod can reach the
vcluster APIs (a localhost proxy is unreachable from inside a pod).

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

kubectl config use-context default          # create from the host context
for t in team-a team-b; do
  vcluster create $t -n $t --connect=false -f /tmp/vcluster-vals.yaml
  kubectl --context default -n $t create quota gpu-cap --hard=requests.amd.com/gpu=4
done
```

Build kubeconfigs against the node IP + assigned NodePort:

```bash
NODE_IP=$(kubectl --context default get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
NP_A=$(kubectl get svc team-a -n team-a -o jsonpath='{.spec.ports[?(@.port==443)].nodePort}')
NP_B=$(kubectl get svc team-b -n team-b -o jsonpath='{.spec.ports[?(@.port==443)].nodePort}')

vcluster connect team-a -n team-a --server="https://$NODE_IP:$NP_A" --print > /tmp/kc-a.yaml
vcluster connect team-b -n team-b --server="https://$NODE_IP:$NP_B" --print > /tmp/kc-b.yaml

cp ~/.kube/config ~/.kube/config.bak
KUBECONFIG=~/.kube/config:/tmp/kc-a.yaml:/tmp/kc-b.yaml kubectl config view --flatten > /tmp/merged \
  && mv /tmp/merged ~/.kube/config

kubectl --context vcluster_team-a_team-a_default get nodes
kubectl --context vcluster_team-b_team-b_default get nodes
```

Confirm they are two distinct API servers:

```bash
kubectl config view -o jsonpath='{range .clusters[*]}{.name}{" -> "}{.cluster.server}{"\n"}{end}' | grep team
```
```
vcluster_team-a_team-a_default -> https://<node-ip>:30411
vcluster_team-b_team-b_default -> https://<node-ip>:31824
```

If TLS complains about the node IP:
```bash
kubectl config set-cluster <cluster-name> --insecure-skip-tls-verify=true
```

---

## 3. Register both with SkyPilot

The jobs controller must run on the **host**, not inside a tenant — it performs the
recovery, so it cannot live in the tenant being broken.

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

sky check k8s
sky gpus list --infra k8s
```
```
Allowed contexts:
├── default: enabled.
├── vcluster_team-a_team-a_default: enabled.
└── vcluster_team-b_team-b_default: enabled.
```

---

## 4. Launch the GPU job across both tenants

```bash
cat > /tmp/probe-gpu.yaml <<'EOF'
name: gpu-failover-probe
resources:
  accelerators: MI300:1
  any_of:
    - infra: k8s/vcluster_team-a_team-a_default
    - infra: k8s/vcluster_team-b_team-b_default
run: |
  rocm-smi --showuniqueid || true
  echo started on $(hostname)
  while true; do echo hb $(date -u +%T); sleep 10; done
EOF

sky jobs launch -y /tmp/probe-gpu.yaml
sky jobs queue
```

Confirm the controller is on the host, and find which tenant received the job
(`any_of` is a candidate list, not a priority order):

```bash
kubectl --context default get pods -A | grep sky-jobs-controller

for T in team-a team-b; do
  CTX="vcluster_${T}_${T}_default"
  if kubectl --context "$CTX" -n default get pods 2>/dev/null | grep -q gpu-failover-probe; then
    VICTIM=$T; VCTX=$CTX
  fi
done
POD=$(kubectl --context "$VCTX" -n default get pods -o name | grep gpu-failover-probe)
echo "$VICTIM / $VCTX / $POD"
```

---

## 5. The transition

Two commands. The quota stops the tenant accepting new pods while leaving its API server
reachable, so SkyPilot can still clean up; deleting the pod supplies the failure. A quota
alone moves nothing — it only gates admission of new pods, never running ones.

```bash
kubectl --context default -n "$VICTIM" create quota block-all --hard=pods=0
kubectl --context "$VCTX" -n default delete "$POD"
watch sky jobs queue
```

---

## 6. Proof the workload moved

```bash
sky jobs queue
```
```
ID  NAME                REQUESTED    #RECOVERIES  STATUS
1   gpu-failover-probe  1x[MI300:1]  1            RUNNING
```

```bash
kubectl --context default get pods -A | grep -E "^team-"
```
```
team-a   coredns-...-x-team-a                                 1/1  Running
team-a   team-a-0                                             1/1  Running
team-b   coredns-...-x-team-b                                 1/1  Running
team-b   gpu-failover-probe-1-<hash>-head-x-default-x-team-b 1/1  Running
team-b   team-b-0                                             1/1  Running
```
The job pod exists only in `team-b`; `team-a` retains only its control plane and DNS.

```bash
kubectl --context default get quota -A | grep block-all
```
```
team-a   block-all   pods: 2/0
```
The block is in `team-a` — confirming that was the source tenant. (`2/0` = its two
pre-existing pods exceed the new limit of zero; quotas do not evict, they only block new
admissions.)

```bash
dmesg -T | grep -iE "amdgpu|mes|gpu reset|flr" | tail -20
```
No GPU reset or MES errors — the GPU teardown and reallocation left the driver clean.

Recovery timeline from the controller log:
```bash
timeout 20 sky jobs logs --controller 1 > /tmp/ctrl.log 2>&1
grep -iE "recover|not ready|Timed out|launched" /tmp/ctrl.log
```
The log shows SkyPilot detecting the failure, retrying the blocked tenant, timing out
against it, then provisioning on the other — the failover decision, not a lucky placement.

---

## 7. Cleanup

```bash
sky jobs cancel -a -y
kubectl --context default -n "$VICTIM" delete quota block-all
```

---

## Notes that prevent failures

| | |
|---|---|
| `socat` | required, or `sky check` reports Kubernetes disabled |
| `sync.fromHost.nodes` | required, or SkyPilot sees no GPUs in the vcluster (pods still schedule — only detection fails) |
| NodePort + node-IP kubeconfig | required, or the controller pod fails prechecks against `127.0.0.1` |
| Controller on host | required, or breaking the tenant kills the recovery mechanism |
| Block + delete | both required; quota alone doesn't move a running job |
| Don't kill the tenant's API server | recovery stalls in cleanup (`FAILED_CONTROLLER`) |
| `any_of` | candidate list, not priority order — check where the job actually landed |
| `sky check` after config changes | `sky gpus list` reflects the last check only |
| `GPU[0]` in container logs | container-local index; compare **unique IDs** for physical identity |
