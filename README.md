# GitOps-Days ⏲️🚀  
*A step-by-step field guide to mastering GitOps with Kubernetes*

> **Who is this for?**  
> Developers, platform engineers, and DevOps practitioners who know the basics of containers/Kubernetes and want a structured path—from “What *is* GitOps?” to production-grade patterns.

---

## 🗓️ Day-by-Day Overview

| Day | Theme | Outcome |
|-----|-------|---------|
| 1 | **Why clusters drift & what GitOps solves** | You’ll see why YAML-in-Git is *not* enough and learn the four principles that anchor every GitOps tool. |
| 2 | **Run your first GitOps loop locally** | Install Flux with the CLI, connect it to Git, and watch your cluster auto-heal from drift—live and declarative. |
| 3 | **Run the same GitOps flow on AKS (Azure Kubernetes Service)** | Take the same repo to the cloud and apply the loop in a production-grade environment. *Coming soon!* |
| … | *More days pending* | Advanced patterns, policy, observability, disaster recovery. |

---

## ⚙️ How this series works

1. **One hour per day.** Every session fits in a lunch break.  
2. **Theory ➜ Practice.** You read *why* first, then feel it in a hands-on lab.  
3. **Built for iteration.** PRs are welcome—this repo itself follows GitOps.

---

## ✅ Prerequisites

- Docker ≥ 24  
- `kubectl` ≥ 1.27  
- Basic comfort with `git clone`, GitHub, and pull requests

> *New to Kubernetes?* Work through [this 20‑min crash course](https://kubernetes.io/docs/tutorials/) first, then come back.

---

## 🚀 Start Here

### 📘 [Day 1 – What really is GitOps?](https://github.com/ahmedmuhi/GitOps-Days/blob/main/Day-1-What-really-is-GitOps.md)  
Foundational theory and principles—learn why GitOps exists and what it solves.

### 🧪 [Day 2 – Run Your First GitOps Loop](https://github.com/ahmedmuhi/GitOps-Days/blob/main/Day-2-Building-Your-First-GitOps-Loop.md)  
Hands-on: install Flux, connect it to Git, watch reconciliation in real time, and test recovery from drift.

### ☁️ Day 3 – Run the Same GitOps Flow on AKS  
You’ve mastered GitOps locally.  
Next, we take the **same repository**, **same setup**, and run it on **Azure Kubernetes Service (AKS)**.  
Coming soon!

---

## 🔄 What’s New?

- ✅ **Day 2 fully rewritten:** clearer structure, validation steps, drift tests, and controller breakdown  
- 🔜 **Day 3 coming soon:** run the same GitOps loop on Azure (AKS)

---

Happy GitOps-ing!  
– Ahmed
