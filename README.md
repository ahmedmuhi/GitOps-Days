# GitOps-Days ⏲️🚀

> **“GitOps: Infrastructure that heals itself.”**

## 👋 Welcome

Welcome to **GitOps-Days**, a hands-on learning journey designed specifically to take you step-by-step from understanding core GitOps concepts to building automated, self-healing Kubernetes infrastructure. Together, we'll replace manual cluster updates and stressful troubleshooting with automation you can trust.

**If you've ever…**

* Checked your cluster and found it no longer matched what’s in Git—with no clear trail of how it got out of sync,
* Spent hours untangling a well-intended fix that saved production but left your system undocumented and harder to trust, or
* Wished your infrastructure could detect when it’s out of sync—and fix itself automatically...

**Then GitOps-Days is exactly where you need to be.**

### 🟢 Project Status – May 2025

* ✅ **Days 1–3 published**
* 🔧 **Day 4 (Policies & Multi-Tenancy)** in progress
* 🧪 **Day 5 (GitOps for Disaster Recovery)** planned

> Want to follow updates or contribute? [⭐ Star this repo](#) or [🍴 open a pull request](#)

## 🧑‍💻 Who This Is For

This series is for developers, DevOps engineers, platform teams, and system architects who want to:

* Understand why GitOps is more than scripting `kubectl apply` on a schedule
* Build self-healing infrastructure that automatically responds to change
* Go from exploring GitOps concepts to deploying it in real-world clusters

## 🗺️ Your Learning Journey

| Day | Theme                                                            | What You'll Learn                                                                       | What You'll Build                                        |
| --- | ---------------------------------------------------------------- | --------------------------------------------------------------------------------------- | -------------------------------------------------------- |
| 1   | **Why Your Kubernetes Cluster Drifts — and How GitOps Fixes It** | Understand the four GitOps principles and how they restore trust in your infrastructure | A clear mental model for how GitOps replaces manual ops  |
| 2   | **Building Your First Self-Healing Kubernetes System with Flux** | Set up Flux, connect it to Git, and test how it detects and corrects drift              | A self-healing Kubernetes cluster running on your laptop |
| 3   | **Scale GitOps to the Cloud with AKS**                           | Adapt your GitOps flow for Azure Kubernetes Service and production environments         | A production-grade GitOps setup on AKS                   |
| …   | **Go Beyond the Basics**                                         | Explore policies, observability, secrets, and multi-team collaboration                  | Enterprise-grade GitOps workflows and guardrails         |

Each day builds on the last—giving you not just concepts, but working infrastructure you can grow and trust.

## ⚙️ How This Series Works

1. **One hour per day.** Each session is designed to fit into a focused learning block—ideal for daily upskilling.
2. **Concept first, then hands-on.** You'll start with a clear explanation of *why* the pattern matters—then implement it yourself.
3. **Rooted in real practice.** The exercises reflect production-grade GitOps patterns used by modern teams—not simplified classroom examples.
4. **Open and iterative.** This series is a GitOps project itself: contributions are welcome, and every update flows through Git.

## ✅ Prerequisites

Before you begin, make sure you have the following:

* **Docker (version ≥ 24):** Used to run Kubernetes clusters locally via tools like `kind`
* **`kubectl` (version ≥ 1.27):** Required to interact with your Kubernetes clusters
* **A GitHub account and basic Git fluency:** Comfortable with `git clone`, opening pull requests, and navigating repositories

> 🧭 *New to Kubernetes?* Start with [this guided introduction](https://kubernetes.io/docs/tutorials/) to get up to speed—then come back here when you're ready to begin.

## 🚀 Start Your GitOps Journey

### 📘 [Day 1 – Why Your Kubernetes Cluster Drifts (and How GitOps Fixes It)](https://github.com/ahmedmuhi/GitOps-Days/blob/main/Day-1-What-really-is-GitOps.md)

Learn the four foundational GitOps principles, how they solve real-world infrastructure drift, and why Git is more than just version control—it’s your new source of operational truth.

### 🧪 [Day 2 – Building Your First Self-Healing Kubernetes System with Flux](https://github.com/ahmedmuhi/GitOps-Days/blob/main/Day-2-Building-Your-First-GitOps-Loop.md)

Get hands-on: install Flux, connect it to Git, and see your cluster automatically detect drift and return to the desired state without manual intervention.

### ☁️ Day 3 – Scale GitOps to the Cloud with AKS

Apply the same GitOps patterns from Day 2 to a production-grade environment on Azure Kubernetes Service.
*Coming soon — stay tuned!*

## 👥 Join the GitOps Community

If this guide has helped you, consider:

* ⭐ **Starring** this repository to stay updated
* 🍴 **Forking** it to customize your own learning path or contribute
* 🧪 **Trying it in your own team**—then sharing your feedback or improvements via pull request

This repo is built using GitOps principles—every update, fix, and improvement is tracked, reviewed, and versioned.

## 🎯 Final Call to Action

**Ready to build infrastructure that keeps itself in sync?**
[Start with Day 1 →](https://github.com/ahmedmuhi/GitOps-Days/blob/main/Day-1-What-really-is-GitOps.md)

– Ahmed