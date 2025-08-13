# GitOps-Days ⏲️🚀

Commit, Reconcile, Ship.
*Learn declarative, pull-based delivery that self-heals.*

## 👋 Welcome

Run Kubernetes the Git-first way. Changes land in Git, and the cluster follows. When trouble shows up, revert cleanly to a previous state. If that’s the predictability you’ve been missing after last-minute tweaks and forgotten settings, you’ll feel right at home here.

## 🚀 Start here

Pick the path that fits how you like to learn.

* **I want the idea first.** Read **[Day 1: What really is GitOps?](https://github.com/ahmedmuhi/GitOps-Days/blob/main/Day-1-What-really-is-GitOps.md)**
* **I learn by doing.** Go to **[Day 2: Building Your First Self-Healing System](https://github.com/ahmedmuhi/GitOps-Days/blob/main/Day-2-Building-Your-First-Self-Healing-System.md)**
* **I am ready for cloud.** Jump to **[Day 3: Production GitOps on AKS with GitHub Actions](https://github.com/ahmedmuhi/GitOps-Days/blob/main/Day-3-Production-GitOps-on-AKS-with-GitHub-Actions.md)**

What you need: Git, Docker, kubectl. Day 3 also needs an Azure account and Azure CLI.

> **Quick terms**
>
> * **Desired state**: what Git says the cluster should look like.
> * **Drift**: when the cluster no longer matches Git.
> * **Controller**: software in the cluster, like Flux or Argo CD, that keeps it matching Git.
> * **Self-healing**: the controller restores the cluster to what is in Git.

## 🎯 What you will achieve

* Keep clusters aligned with Git and recover fast when things change.
* Build a self-healing loop locally and see it fix deliberate breaks.
* Repeat the same loop in Azure Kubernetes Service with production-minded patterns.
* Plan a simple CI job that feeds the loop with fresh images. *(Day 4, planned)*
* Map multi-environment and secrets patterns you can adopt. *(Day 5, planned)*

## 🗺️ Your learning path

**Status:** Days 1–3 ready now. Days 4–5 planned.

| Day                                                                                                                   | Focus                                                | What You'll Build                    |
| --------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------- | ------------------------------------ |
| [**Day 1**](https://github.com/ahmedmuhi/GitOps-Days/blob/main/Day-1-What-really-is-GitOps.md)                        | **Understand:** what GitOps is and why it matters    | A clear mental model                 |
| [**Day 2**](https://github.com/ahmedmuhi/GitOps-Days/blob/main/Day-2-Building-Your-First-Self-Healing-System.md)      | **Build:** your first self-healing system with Flux  | Local GitOps loop that auto-corrects |
| [**Day 3**](https://github.com/ahmedmuhi/GitOps-Days/blob/main/Day-3-Production-GitOps-on-AKS-with-GitHub-Actions.md) | **Scale:** take the same loop to AKS                 | Production-ready cloud deployment    |
| **Day 4**                                                                                                             | **Operate:** real-world patterns and troubleshooting | Robust GitOps workflows              |
| **Day 5**                                                                                                             | **Advance:** tool choices and next steps             | Your GitOps roadmap                  |

## 🛠️ What you need

* Docker Desktop or Docker Engine
* kubectl
* Git and a GitHub account
* Flux CLI
* Azure CLI and an Azure subscription for Day 3
* A typical laptop with at least a few gigabytes of RAM for the local labs

> **Tested with**
> Current stable releases of Docker, kubectl, kind, Flux, and Azure CLI. If you hit a version snag, open an issue and we will help.

## 🔄 Stay up to date (sync your fork)

We improve this course often. To pull updates without re-forking:

```bash
# inside your local clone of your fork
git remote add upstream https://github.com/ahmedmuhi/GitOps-Days.git
git fetch upstream
git checkout main
git merge upstream/main    # or: git rebase upstream/main
git push origin main
```

Prefer a stable snapshot? Check out a tag, for example:

```bash
git fetch --tags
git checkout v0.3
```

## 🗂️ Repo map

```
/Day-1-What-really-is-GitOps.md
/Day-2-Building-Your-First-Self-Healing-System.md
/Day-3-Production-GitOps-on-AKS-with-GitHub-Actions.md
/examples/day2/...   # manifests for the local lab
/examples/day3/...   # manifests for the AKS lab
```

## ❓ FAQ

* **How do Helm charts fit in?** Put the chart, version, and values in Git. The controller applies it and keeps it up to date.
* **What about secrets?** Use an external manager or encrypted manifests so nothing sensitive lives in plain text.
* **Do I need to delete my fork to get updates?** No. Use the sync steps above.
* **What if I break the cluster during labs?** Reconciliation brings it back. Worst case, recreate the kind cluster and reapply Flux.

## 💬 Join the conversation

This series is actively evolving.

* Found an issue or a typo? [Open an issue](https://github.com/ahmedmuhi/GitOps-Days/issues)
* Have suggestions? Share them in an issue or pull request
* Sharing your progress? Use #GitOpsDays

## 🧭 Roadmap

* **Day 4**: CI pipeline that feeds the loop, image promotion, and basic rollout gates
* **Day 5**: Multi-environment layout and common secrets patterns
* Future topics will follow what helps learners most

## 📚 Additional resources

* [Flux documentation](https://fluxcd.io/flux/)
* [CNCF GitOps Working Group](https://opengitops.dev/)
* [Kubernetes tutorials](https://kubernetes.io/docs/tutorials/)

---

Ready to begin? Start with **[Day 1: What really is GitOps?](https://github.com/ahmedmuhi/GitOps-Days/blob/main/Day-1-What-really-is-GitOps.md)**.
