# GitOps-Days

Learn GitOps by building it, one day at a time. Each day you build something and watch how it behaves.

<p align="center">
  <img src="./assets/images/gitops-reconciliation-loop.png" width="70%" alt="The GitOps reconciliation loop: Git stores the desired state, the cluster runs the actual state, and a controller reconciles the cluster whenever the two differ — whether from a commit or from drift.">
</p>

## What you'll build

Over the series you'll build a GitOps setup from nothing: a cluster that pulls its configuration from Git, notices when reality stops matching the repo, and puts itself back. You start on your laptop and finish on a cloud cluster, with every change along the way made through Git.

You'll get the most from this if you're comfortable with basic Kubernetes — you can apply a manifest and know roughly what a pod and a deployment are. If Kubernetes itself is new, start with the official [Kubernetes tutorials](https://kubernetes.io/docs/tutorials/) first and come back; if you already run Flux or Argo in production, the early days will feel familiar and you may want to skim to the later ones.

## Start here

New here? Start with [Day 1](./Day-1-What-really-is-GitOps.md) and go in order — each day builds on the one before.

| | Day | What you'll do |
|---|---|---|
| 🧠 | [Day 1 – What really is GitOps?](./Day-1-What-really-is-GitOps.md) | Build the mental model: desired state, drift, reconciliation. |
| 🛠️ | [Day 2 – Building Your First Self-Healing System](./Day-2-Building-Your-First-Self-Healing-System.md) | Build it locally: a kind cluster that repairs itself when you break it. |
| ☁️ | [Day 3 – GitOps on AKS: Same Loop, Bigger Stage](./Day-3-GitOps-on-AKS-Self-Healing-Cloud-Scale.md) | Take it to the cloud: the same loop, production-grade, on Azure. |
| 🏭 | [Day 4 – Production GitOps Patterns: Closing the Loop](./Day-4-Production-GitOps-Patterns.md) | Close the loop: a CI pipeline that feeds Git, and the repo patterns teams scale with. |

The series is ongoing — new days are added as they're written.

Comfortable with Kubernetes already and just want to see it work? [Day 2](./Day-2-Building-Your-First-Self-Healing-System.md) stands on its own. Ready for cloud from the start? [Day 3](./Day-3-GitOps-on-AKS-Self-Healing-Cloud-Scale.md) does too.

<details><summary>New to GitOps terminology?</summary>

A few terms that come up throughout the series:

- **Desired state** — what Git says your cluster should look like.
- **Drift** — when your cluster no longer matches Git.
- **Controller** — software in the cluster (like Flux or Argo CD) that watches Git and applies changes.
- **Self-healing** — the controller restoring the cluster to match Git, automatically.

Day 1 covers these in depth.

</details>

## Staying up to date

**Using GitHub (no local clone):**

1. Open *your fork* on GitHub.
2. Click **Sync fork** (or **Fetch upstream**) → **Update branch**.
3. Your fork is now in sync with the latest changes.

<details><summary>Using the CLI (optional)</summary>

```bash
# inside your local clone
git remote add upstream https://github.com/ahmedmuhi/GitOps-Days.git
git fetch upstream
git checkout main
git merge upstream/main    # or: git rebase upstream/main
git push origin main
```

</details>

## FAQ

**Do I need to re-fork to get updates?**
No — use the sync steps above.

**I broke my local lab — now what?**
Good news: this is exactly what self-healing is for. Flux's reconciliation loop should correct the drift automatically. If things are too far gone, recreate the kind cluster and reapply Flux. [Day 2](./Day-2-Building-Your-First-Self-Healing-System.md) walks through recovery in detail.

**I have a question that's not listed here.**
[Open an issue](https://github.com/ahmedmuhi/GitOps-Days/issues) — we'll add it to the FAQ.

## What's next

Days 1–4 are live. More days are on the way.

Have a topic you'd like covered? [Vote or suggest in the issues](https://github.com/ahmedmuhi/GitOps-Days/issues).

## Join the conversation

GitOps-Days is evolving, and your feedback shapes what comes next.

- Found a bug or typo? [Open an issue](https://github.com/ahmedmuhi/GitOps-Days/issues)
- Have ideas or improvements? Send a pull request
- Sharing your journey? Post with `#GitOpsDays`

## Additional resources

- [Flux documentation](https://fluxcd.io/flux/)
- [OpenGitOps / CNCF GitOps Working Group](https://opengitops.dev/)
- [Kubernetes tutorials](https://kubernetes.io/docs/tutorials/)

## About the author

Built by [Ahmed Muhi](https://github.com/ahmedmuhi), Microsoft Azure MVP and cloud migration architect with 20+ years in IT. Principal Consultant at LAB³, specialising in Azure migrations, solution architecture, and Infrastructure as Code.
