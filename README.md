# GitOps-Days ⏲️🚀
*The hands-on journey from manual Kubernetes chaos to automated, self-healing infrastructure*

> **"GitOps gives you infrastructure that fixes itself."**

Ever had a 3 AM call because someone made a direct change to production? Or spent hours figuring out why your cluster state doesn't match your Git repo? GitOps solves these problems by creating a system where your Git repository automatically drives what's running in your cluster—with no manual intervention required.

**This field guide will take you from GitOps concepts to implementation in just a few hours.**

> **Who is this for?**  
> Developers, platform engineers, and DevOps practitioners who know the basics of containers/Kubernetes and want a structured path—from "What *is* GitOps?" to production-grade patterns.

---

## 🗺️ Your Learning Journey

| Day | Theme | What You'll Learn | Practical Outcome |
|-----|-------|------------------|-------------------|
| 1 | **Why clusters drift & what GitOps solves** | The four GitOps principles and how they eliminate configuration drift | You'll understand exactly how GitOps differs from traditional CI/CD |
| 2 | **Run your first GitOps loop locally** | Setting up Flux, connecting to Git, testing self-healing | You'll have a working self-healing system on your laptop |
| 3 | **Run the same GitOps flow on AKS** | Adapting GitOps for cloud environments | You'll deploy the same pattern to a production-grade environment |
| … | *More days coming soon* | Advanced patterns, policy, observability, disaster recovery | You'll build enterprise-ready GitOps workflows |

Each day builds on the previous one, creating a complete learning path from concept to production-ready implementation.

---

## ⚙️ How This Series Works

1. **One hour per day.** Every session fits in a lunch break.  
2. **Theory ➜ Practice.** You read *why* first, then build it yourself.  
3. **Real-world focused.** No toy examples—these are the same patterns used in production.
4. **Community-driven.** PRs are welcome—this repo itself follows GitOps.

---

## ✅ Prerequisites

- Docker ≥ 24  
- `kubectl` ≥ 1.27  
- Basic comfort with `git clone`, GitHub, and pull requests

> *New to Kubernetes?* Work through [this 20‑min crash course](https://kubernetes.io/docs/tutorials/) first, then come back.

---

## 🚀 Begin Your GitOps Journey

### 📘 [Day 1 – What Really Is GitOps?](https://github.com/ahmedmuhi/GitOps-Days/blob/main/Day-1-What-really-is-GitOps.md)  
Discover why GitOps exists, what problems it solves, and the principles that make it work. This foundation will change how you think about infrastructure management.

### 🧪 [Day 2 – Build Your First Self-Healing System](https://github.com/ahmedmuhi/GitOps-Days/blob/main/Day-2-Building-Your-First-GitOps-Loop.md)  
Get hands-on: install Flux, connect it to Git, watch your cluster automatically sync with your repository, and witness it recover from intentional "breaks" without manual intervention.

### ☁️ Day 3 – Scale to Production with AKS
Take your GitOps skills to the cloud! Using the same patterns from Day 2, you'll deploy a production-grade GitOps system on Azure Kubernetes Service.  
*Coming soon!*

---

## 🔄 What's New?

- ✅ **Day 2 fully rewritten:** Clearer structure, step-by-step validation, practical drift tests, and detailed controller explanations  
- 🔜 **Day 3 coming soon:** Run the same GitOps patterns on Azure Kubernetes Service (AKS)

---

## 👥 Join the GitOps Community

If you find this guide helpful, consider:
- ⭐ Starring this repository
- 🍴 Forking it to contribute improvements
- 👁️ Watching for updates as we add more days and examples

---

Ready to build infrastructure that maintains itself? [Start with Day 1!](https://github.com/ahmedmuhi/GitOps-Days/blob/main/Day-1-What-really-is-GitOps.md)

– Ahmed