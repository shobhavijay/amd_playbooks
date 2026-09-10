# The Invisible Cost of Restarts in the AI Era

When you restart a traditional software service, you lose a few seconds of
uptime.

When you restart an AI agent, you lose those seconds too — but you also re-buy
every single expensive inference calculation it had already made.

In traditional enterprise software, an interruption costs you the downtime, and
the meter stops when service is restored. With autonomous agents, the outage is
only the first line on the invoice. Recovery adds a second — the inference the
failed process had already paid for, bought again — and no uptime dashboard will
ever show it to you.

## The reality of "silent restarts"

I recently tested an agentic data pipeline running on an AMD 8×MI300X
node, split into two isolated virtual clusters. The agent's task was to process a
batch of 39 visual assets. For every single asset it makes nine distinct
LLM calls against a 72-billion-parameter model.

Six items into the batch, I abruptly pulled the plug — a hard infrastructure
deletion, zero warning to the agent , no graceful shutdown.

Our orchestration layer (SkyPilot) performed beautifully. It detected the
failure, relaunched the agent in the second cluster, and had it running again in
less than two minutes. From an infrastructure standpoint, a flawless automated
failover.

It was that fast for a reason worth understanding: the compute never moved. Both
clusters already had their own model servers running and warm. The only thing
that relocated was the agent process itself, which holds no GPUs at all. Nothing
had to be provisioned, and no 72B model had to be loaded.

But there was a financial catch.

## The 54-call blind spot

SkyPilot can move a process. It cannot move that process's memory.

The fresh agent that woke up in the second cluster had no context of what its
predecessor had accomplished. Left to its default behaviour, it did the most
logical thing a computer program can do: it started again at item number one.

Six items were already finished and sitting in object storage. A blind restart
meant duplicating 54 token-heavy LLM calls to recreate files that already
existed.

Now scale that. If you are processing thousands of documents and your system
hits a routine network hiccup three-quarters of the way through a batch, the
recovery costs you significantly more than the outage ever did. If you are using
cost-effective spot instances — where interruptions are part of the daily
bargain — you can lose more to redone work than the discount ever saved you.

## The strategic danger: invisible financial leakage

Here is the part that should concern any engineering leader. **A batch that
silently restarts from scratch looks identical to a batch that cleanly resumed.**

- It generates the same logs.
- It delivers the same successful business deliverables.
- It prints the same "success" message at the finish line.

Traditional monitoring does not flag double-spent compute. You cannot catch this
on an uptime graph. You find it when the invoice arrives, months later, if you
find it at all.

Two architectural flaws in my own system would have caused exactly this. Neither
would have failed a product demo. Both would have quietly drained the inference
budget.

The first: the agent recognised only *graceful* interruptions. It knew how to
pause for a human question and resume afterwards, and that path was well tested.
It did not know how to be killed mid-step — and when that happened it reported
the item finished and moved on.

The second: checkpoints were being written asynchronously, which is the right
default for throughput and the wrong one for a process that can be terminated
without warning. Work was reported as saved a moment before it actually was.

I found both by reading the recovery path before the test, not by discovering
them in a bill.

## The fix: engineering for disposable compute

Plugging this leak meant treating the agents as entirely disposable while
treating their progress as permanent. Two non-negotiable rules:

1. **Decoupled state.** The agent's progress is written synchronously to a
   database that lives outside both clusters. If the server dies, the progress
   lives.
2. **Mandatory audit check.** Before a revived agent makes its first inference
   call, it queries external storage and asks: what is already done? Anything
   finished is skipped.

With those rules in place, the failover behaved the way it should. The agent
woke up in the second cluster, recognised that six items were complete, and
picked up at item seven. Nine items finished across the two clusters, and the
ledger recorded exactly one attempt against every one of them — no asset was
processed twice.

The outage divided the work. It did not duplicate the cost.

## The executive takeaway

Moving AI work from a failed environment to a live one is a solved operational
problem. Infrastructure tooling makes it easy.

Financial continuity is a different problem, it is not solved by default, and it
is where the money goes. If your teams are deploying autonomous workflows at
scale, three questions are worth asking:

- Are our agents saving progress outside the environment that can fail?
- Do our agents assume work needs doing, or do they check whether it is already
  done?
- Are we counting attempts per item — and is that number anywhere a human can
  see it?

An AI agent that silently starts over has not failed loudly. It has failed in the
most expensive way available to it: invisibly, and entirely on your bottom line.

## A note on scale

This was a deliberately small, controlled test: 39 assets, one agent, two virtual
clusters on a single node. It is not a benchmark, and it does not tell you what
your organisation is currently losing.

What it does establish is that the failure mode is on by default and invisible by
default. Neither property depends on the size of the workload. A relocated agent
has no memory of its predecessor whether the batch holds 39 items or 39,000, and
a silent restart writes the same logs as a clean resume at any scale.

The exposure, though, does scale. The larger the batch and the later the
interruption lands, the more completed work gets purchased a second time. A small
test was enough to surface the problem, which is by far the cheapest place to
find it.

So the question worth taking away is not whether this happens at your scale. It
is whether anyone in your organisation would be able to tell if it did.

---

*Run on 8×MI300X, two virtual clusters, Qwen2.5-72B and Qwen2.5-VL-7B, SkyPilot
managed jobs. GPU time provided by the AMD Developer Program. Full technical
playbook and agent source:
[github.com/shobhavijay/amd_playbooks](https://github.com/shobhavijay/amd_playbooks).*
