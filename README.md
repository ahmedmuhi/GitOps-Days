
# GitOps-Days 🔄🚀

Commit, reconcile, ship.
*Learn the Git-first, pull-based way to run Kubernetes - and let your clusters look after themselves.*

## Welcome

If you’ve ever fixed something in production at the last minute… and then spent the next week wondering what else had changed, you’re not alone.
GitOps gives you a different way: changes go into Git, the cluster follows, and if things drift, they’re put back automatically. When trouble shows up, you can roll back cleanly to a known good state - no guesswork.

That’s the predictability you’ll build in this series.
Let’s start by choosing how you want to begin.

## 🗺️ Start here

There’s more than one way to get into GitOps - pick the path that fits how you like to learn.

* **Want the big picture first?** Start with **[Day 1: What really is GitOps?](./Day-1-What-really-is-GitOps.md)** and build a clear mental model.
* **Learn best by doing?** Jump into **[Day 2: Building Your First Self-Healing System](./Day-2-Building-Your-First-Self-Healing-System.md)** and see it in action.
* **Ready for cloud from the start?** Head to **[Day 3: Production GitOps on AKS with GitHub Actions](./Day-3-Production-GitOps-on-AKS-with-GitHub-Actions.md)** and go straight to a production-grade setup.

> **You’ll need**: Git, Docker, and `kubectl` for Days 1-2. Day 3 also needs an Azure account and the Azure CLI.

### Quick cheat sheet before you dive in

* **Desired state**: what Git says your cluster should look like.
* **Drift**: when your cluster no longer matches Git.
* **Controller**: software in the cluster (like [Flux](https://fluxcd.io/) or [Argo CD](https://argo-cd.readthedocs.io/)) that keeps it matching Git.
* **Self-healing**: the controller restores the cluster to match Git automatically.

## 🎯 What you will achieve

By the end of this series, you will be able to:

* Keep clusters aligned with Git and recover fast when things change.
* Build and run a self-healing loop locally.
* Take the same loop to Azure Kubernetes Service with production patterns.
* Extend GitOps with CI pipelines, multi-environment setups, and secrets management. *(Days 4-5 planned)*

## 🛠️ What you need

For local labs (Days 1-2):

* Docker Desktop or Docker Engine
* `kubectl`
* Git and a GitHub account
* [Flux CLI](https://fluxcd.io/)

For cloud labs (Day 3):

* Azure CLI
* An Azure subscription

> **Tested with**: current stable releases of Docker, `kubectl`, kind, Flux, and Azure CLI. If you hit a version snag, open an issue and we’ll help.

## 🗺️ Your learning path

**Status:** Days 1-3 ready now. Days 4-5 planned.

| Day                                                                  | Focus                                                | What you'll build                    |
| -------------------------------------------------------------------- | ---------------------------------------------------- | ------------------------------------ |
| [**Day 1**](./Day-1-What-really-is-GitOps.md)                        | **Understand**: what GitOps is and why it matters    | A clear mental model                 |
| [**Day 2**](./Day-2-Building-Your-First-Self-Healing-System.md)      | **Build**: your first self-healing system with Flux  | Local GitOps loop that auto-corrects |
| [**Day 3**](./Day-3-Production-GitOps-on-AKS-with-GitHub-Actions.md) | **Scale**: take the same loop to AKS                 | Production-ready cloud deployment    |
| **Day 4**                                                            | **Operate**: real-world patterns and troubleshooting | Robust GitOps workflows              |
| **Day 5**                                                            | **Advance**: tool choices and next steps             | Your GitOps roadmap                  |

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

## 🗂️ Repo map

Everything you need is linked in the learning path above.
If you’re browsing the source, the `/examples/` folder contains the manifests for Day 2 and Day 3 labs.

## ❓ Quick FAQ

**Do I need to re-fork to get updates?**
No - use the sync steps above.

**I broke my local lab - now what?**
Reconciliation will usually fix it. If not, recreate the kind cluster and reapply Flux.

**I have a question that’s not listed here.**
[Open an issue](https://github.com/ahmedmuhi/GitOps-Days/issues) - we’ll add it to the FAQ.

## 💬 Join the conversation

GitOps-Days is evolving, and your feedback matters.

* Found a bug or typo? [Open an issue](https://github.com/ahmedmuhi/GitOps-Days/issues)
* Have ideas or improvements? Send a pull request
* Sharing your journey? Post with `#GitOpsDays`

## 🧭 Roadmap

* **Day 4** - CI pipelines that feed the loop, image promotion, rollout gates
* **Day 5** - Multi-environment layouts and secrets patterns
* Future topics will follow learner needs

## 📚 Additional resources

* [Flux documentation](https://fluxcd.io/flux/)
* [OpenGitOps / CNCF GitOps Working Group](https://opengitops.dev/)
* [Kubernetes tutorials](https://kubernetes.io/docs/tutorials/)

🚀 **Ready to start?** Jump into **[Day 1: What really is GitOps?](./Day-1-What-really-is-GitOps.md)** and begin your GitOps journey.
