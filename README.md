# GitOps-Days 🔄🚀

Your cluster broke. Git fixed it.
*A hands-on series that takes you from zero GitOps to a self-healing Kubernetes setup — starting local, finishing in the cloud.*

## Who is this for?

You know your way around Kubernetes — pods, deployments, `kubectl` — but you're still pushing changes by hand and hoping nothing drifts overnight. Maybe you've heard of Flux or Argo CD but haven't built a reconciliation loop end to end.

This series is for you. No GitOps experience needed — just curiosity and a terminal.

## What you'll walk away with

By the end of this series you won't just know what GitOps is — you'll have built it, broken it, and watched it fix itself.

- **Understand why clusters drift** — and set up a system that makes drift correct itself before you even notice.
- **Build a self-healing loop on your laptop** — a local Flux-powered reconciliation cycle you can experiment with safely.
- **Take that same loop to Azure Kubernetes Service** — with production patterns, CI integration, and real-world structure.
- **Roll back with confidence** — because every change lives in Git, recovery is a revert, not a firefight.

## Start here — pick your path

> Three days available now. More on the way — see [What's next](#whats-next).

There's more than one way into GitOps. Pick the entry point that fits how you learn best.

🧠 **I want to understand first**
Start with [Day 1: What really is GitOps?](./Day-1-What-really-is-GitOps.md) — build a clear mental model of desired state, drift, and reconciliation before touching a terminal.

🛠️ **I learn by doing**
Jump to [Day 2: Building Your First Self-Healing System](./Day-2-Building-Your-First-Self-Healing-System.md) — spin up a local cluster, install Flux, and watch it heal itself in real time.

☁️ **I'm ready for cloud-scale**
Head to [Day 3: Production GitOps on AKS with GitHub Actions](./Day-3-GitOps-on-AKS-Self-Healing-Cloud-Scale.md) — take the same loop to Azure Kubernetes Service with CI and production structure.

> **Prerequisites vary by day** — each guide lists exactly what you need at the top. In general: Git, Docker, and `kubectl` for local labs; add the Azure CLI for Day 3.

<details><summary>New to GitOps terminology?</summary>

A few terms that come up throughout the series:

- **Desired state** — what Git says your cluster should look like.
- **Drift** — when your cluster no longer matches Git.
- **Controller** — software in the cluster (like Flux or Argo CD) that watches Git and applies changes.
- **Self-healing** — the controller restoring the cluster to match Git, automatically.

Day 1 covers these in depth.

</details>

## 🔄 Stay up to date (sync your fork)

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

## ❓ FAQ

**Do I need to re-fork to get updates?**
No — use the sync steps above.

**I broke my local lab — now what?**
Good news: this is exactly what self-healing is for. Flux's reconciliation loop should correct the drift automatically. If things are too far gone, recreate the kind cluster and reapply Flux. [Day 2](./Day-2-Building-Your-First-Self-Healing-System.md) walks through recovery in detail.

**I have a question that's not listed here.**
[Open an issue](https://github.com/ahmedmuhi/GitOps-Days/issues) — we'll add it to the FAQ.

## 🗺️ What's next

Days 1–3 are live. Here's where the series is heading:

- **Day 4** — CI pipelines that feed the loop, image promotion, rollout gates
- **Day 5** — Multi-environment layouts and secrets patterns

Have a topic you'd like covered? [Vote or suggest in the issues](https://github.com/ahmedmuhi/GitOps-Days/issues).

## 💬 Join the conversation

GitOps-Days is evolving, and your feedback shapes what comes next.

- Found a bug or typo? [Open an issue](https://github.com/ahmedmuhi/GitOps-Days/issues)
- Have ideas or improvements? Send a pull request
- Sharing your journey? Post with `#GitOpsDays`

## 📚 Additional resources

- [Flux documentation](https://fluxcd.io/flux/)
- [OpenGitOps / CNCF GitOps Working Group](https://opengitops.dev/)
- [Kubernetes tutorials](https://kubernetes.io/docs/tutorials/)

## ✍️ About the author

Built by [Ahmed Muhi](https://github.com/ahmedmuhi), Microsoft Azure MVP and cloud migration architect with 20+ years in IT. Principal Consultant at LAB³, specialising in Azure migrations, solution architecture, and Infrastructure as Code.