# AMD Playbooks

Field-tested playbooks for running AI infrastructure on AMD Instinct GPUs.

Each playbook is a set of commands that were actually executed on real hardware, with the
outputs they produced. Where something failed first, the working approach is the one
documented — along with the specific reason it's needed.

## Playbooks

### [Moving a GPU workload between two vclusters with SkyPilot](skypilot-amd-playbook.md)

Build two GPU-isolated virtual clusters on a single AMD node and have SkyPilot transition
a running GPU workload from one to the other automatically, with no manual intervention.

Covers: k3s, the AMD device plugin (`amd.com/gpu`), SkyPilot on Kubernetes,
[vcluster](https://www.vcluster.com) multi-tenancy with per-tenant GPU quotas, and the
managed-job failover sequence.

**Result:** an `MI300:1` managed job running in `team-a` recovered into `team-b`
unattended — `#RECOVERIES: 1`, GPU driver clean, no manual relaunch.

**Validated on:** 8× MI300X VF (gfx942) · Ubuntu 24.04 · k3s v1.36 · SkyPilot 0.13 ·
vcluster 0.36 · single node.

### [Moving a running AI agent between clusters](agent-failover-playbook.md)

Take the same two-vcluster setup further: run a real LangGraph agent as a managed job,
kill it mid-batch, and have it resume in the other cluster without redoing work it had
already paid for.

Covers: externalising agent state to Postgres, Chroma and MinIO outside both clusters,
LangGraph checkpointing with `durability="sync"`, a claim ledger with attempt counts, and
how to tell a genuine resume apart from a silent restart.

**Result:** 9 recipes completed across the two clusters after a forced failover, with
`max_attempt = 1` on every one — no photograph processed twice, roughly 54 inference calls
against a 72B model not re-bought.

The agent source is in [`culinary-archivist-agent.tar.gz`](culinary-archivist-agent.tar.gz);
step 1 of the playbook copies it onto the host.

**Validated on:** 8× MI300X VF (gfx942) · Ubuntu 24.04 · k3s · SkyPilot managed jobs ·
vcluster 0.36 · Qwen2.5-72B + Qwen2.5-VL-7B · single node.

## Notes

Version-sensitive: the commands reflect the tool versions listed with each playbook.
Check current upstream docs for AMD GPU Operator charts, SkyPilot accelerator naming, and
vcluster configuration schema before relying on them in a different environment.
