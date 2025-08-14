# 🚀 Day 2 - Building Your First Self-Healing Kubernetes System with Flux

**Yesterday, you learned the antidote to configuration drift. Today, you’ll put it into practice.**

Welcome back to GitOps-Days. [Day 1](https://github.com/ahmedmuhi/GitOps-Days/blob/main/Day-1-What-really-is-GitOps.md) gave you the concepts, vocabulary, and four principles behind self-healing infrastructure.
In short, configuration drift is what happens when the live state of your cluster no longer matches what’s in Git - for example, when someone makes a quick kubectl fix in production and forgets to commit it. GitOps solves that by making Git the single source of truth and letting your cluster reconcile itself automatically.

If any of those terms or ideas sound unfamiliar, you can [catch up on Day 1 here](https://github.com/ahmedmuhi/GitOps-Days/blob/main/Day-1-What-really-is-GitOps.md) before continuing.

In this session, you’ll use **[Flux](https://fluxcd.io/)** - a GitOps operator - to create a local Kubernetes cluster that stays in sync with your repository, detects drift, and restores the intended state automatically.

## 🗺️ Your Hands-On Journey (\~1 hour)

* Set up a local Kubernetes lab environment (\~15 mins)
* Create a Git repository as your single source of truth (\~5 mins)
* Install Flux and connect it to your cluster (\~10 mins)
* Deploy an application through Git commits (\~10 mins)
* Break things on purpose and watch Flux fix them (\~15 mins)

No cloud accounts. No complex setup. Just Docker and a few free tools.

## 🎯 By the End, You’ll Have

* A real Kubernetes cluster with Flux keeping it in sync with your repo
* Automated deployments triggered by pushes to Git
* Continuous drift detection and correction in under a minute
* A system that can recover from mistakes without your intervention

Ready to turn yesterday’s ideas into a live, self-healing system? Let’s get started.

## 🧰 Preparing Your Workspace

Before we dive into building your self-healing system, let’s set up the **essential building blocks** it relies on. These tools form the foundation for everything you’ll do today - once they’re in place, Flux will be able to work its GitOps magic.

Later in this lab, we’ll install **[Flux](https://fluxcd.io/)** - the GitOps operator that keeps your cluster in sync with Git - but first, we need this solid groundwork.

> [!TIP]
> Already have Docker, kind, kubectl, and Git installed?
> [Skip ahead to “Set Up Your Git Repository”](#📂-set-up-your-git-repository).

> [!IMPORTANT]
> This lab requires approximately **4 GB of free RAM** and **2 GB of disk space**.

### The Tools You’ll Need

**1. Docker** - Runs the containers that become your Kubernetes nodes.
[Install Docker](https://docs.docker.com/get-docker/)
**Minimum version:** **24.0**

**2. kind** - Creates your local “playground” by spinning up a Kubernetes cluster inside Docker.
[Install kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
**Minimum version:** **0.25.0**

**3. kubectl** - Your command-line tool for inspecting and managing Kubernetes resources.
[Install kubectl](https://kubernetes.io/docs/tasks/tools/)
**Minimum version:** **1.32**

**4. Git** - Your single source of truth, where you declare what should be running in your cluster.
[Install Git](https://git-scm.com/downloads)
**Minimum version:** **2.40**

### ✅ Checkpoint: Verify Your Setup

**Why this matters:**
Each lab step includes a checkpoint like this. If something fails later, knowing exactly which step last worked makes troubleshooting easier - without retracing your entire setup.

Run the following commands to confirm your tools are ready:

```shell
docker --version
kind --version
kubectl version --client --short
git --version
```

**Pass criteria:**
You should see version numbers that match or exceed the minimums above.

With your tools confirmed, you’re ready to create the Git repository that will control everything in your self-healing Kubernetes system.

## 📂 Set Up Your Git Repository

Your Git repository is the **single source of truth** Flux will watch. Every change you push here will be automatically applied to your Kubernetes cluster - which is why this step is critical to building your self-healing system.

⏱️ **Time needed:** \~3 minutes

### 1️⃣ Fork this Repository

Go to [`https://github.com/ahmedmuhi/GitOps-Days`](https://github.com/ahmedmuhi/GitOps-Days) and click **Fork** in the top-right corner.
On the next screen, click **Create fork**.

💡 *Why this matters:*
Flux needs **write access** to the repository it’s watching so it can apply your changes. Forking creates a copy in your own GitHub account where you can commit freely - without affecting the original project.

Your fork will live at:

```
https://github.com/YOUR-USERNAME/GitOps-Days
```

### 2️⃣ Clone Your Fork Locally

> **Important:** Replace `YOUR-USERNAME` in the commands below with your actual GitHub username before running them.

```shell
git clone https://github.com/YOUR-USERNAME/GitOps-Days.git
cd GitOps-Days
```

### 3️⃣ Checkpoint: Confirm Your Local Repo Is Linked to Your Fork

**Why this matters:**
Flux will only work if your local repository is connected to your own fork. Otherwise, you’ll be pushing changes to the wrong place - and Flux won’t see them.

Run:

```shell
git remote -v
```

Expected output:

```
origin  https://github.com/YOUR-USERNAME/GitOps-Days.git (fetch)
origin  https://github.com/YOUR-USERNAME/GitOps-Days.git (push)
```

* ✅ This means you have a fork in your account, cloned locally, with write access.
* ❌ If you see `ahmedmuhi` instead of your username, you cloned the original repo by mistake. Delete the folder and clone your fork instead.

**Checkpoint complete:** You now have a GitOps-ready repository with full write access, linked locally, and ready for Flux to watch.

Next: we’ll create the Kubernetes cluster where your self-healing system will run.

## 🔄 Creating Your Kubernetes Environment

Now that your Git repository is ready, it’s time to build the **playground that Flux will soon protect from drift**.
This Kubernetes cluster will be the stage where your GitOps workflow comes to life - continuously pulling configuration from Git and applying it automatically.

⏱️ **Time needed:** \~2 minutes

We’ll use **kind** (*Kubernetes in Docker*) - a tool that runs a full Kubernetes cluster inside Docker containers. It’s fast, lightweight, and perfect for local experiments.

### Creating the Cluster

**Choose one** of the following commands based on your operating system:

**Windows (PowerShell or Windows Terminal)**

```shell
kind create cluster --name gitops-loop-demo
```

**macOS/Linux (Terminal)**

```shell
kind create cluster \
  --name gitops-loop-demo
```

This spins up a fully functional Kubernetes cluster in about a minute, using the latest stable Kubernetes version supported by kind.

### ✅ Checkpoint: Confirm Your Cluster Is Ready

**Why this matters:**
Before we bring in Flux, we need to be sure the cluster is running and ready to accept workloads. If the cluster isn’t healthy, Flux can’t do its job.

Run:

```shell
kubectl get nodes
```

**Pass criteria:**

* `STATUS` shows **Ready**.
* The `AGE` is only a few minutes (freshly created).

Example output:

```
NAME                             STATUS   ROLES           AGE   VERSION
gitops-loop-demo-control-plane   Ready    control-plane   1m    v1.32.x
```

If you see `NotReady`, wait a few seconds and try again.

**Quick troubleshooting tips:**

* Make sure Docker is running:

  ```shell
  docker ps
  ```
* Check that your system has at least 4 GB of free RAM and 2 GB of disk space.
* If the cluster remains `NotReady`, delete and recreate it:

  ```shell
  kind delete cluster --name gitops-loop-demo
  kind create cluster --name gitops-loop-demo --image kindest/node:v1.32+
  ```

### What You’ve Just Built

* ✅ A fully functional Kubernetes cluster running locally
* ✅ A safe playground for your GitOps experiments
* ✅ The environment Flux will soon manage, keeping it in sync with your repository

> [!TIP]
> When you’re done with the lab, you can delete the cluster to free resources:
>
> ```shell
> kind delete cluster --name gitops-loop-demo
> ```

## 🚀 Deploy Flux to Your Cluster and Connect It to Your Repo

Now that you have the Flux CLI installed and your cluster running, let’s:

1. **Install Flux’s controllers into your cluster**
2. **Tell Flux which Git repository to watch** (your fork on GitHub)

⏱️ **Time needed:** \~3-4 minutes

### Install the Flux controllers

Run:

```shell
flux install
```

This installs Flux’s components (Source Controller, Kustomize Controller, etc.) into the `flux-system` namespace of your cluster. These controllers will be responsible for pulling configuration from Git and reconciling your cluster to match.

### Connect Flux to your Git repository

Next, tell Flux to watch **your fork** on GitHub. Use the HTTPS URL to your fork so Flux in the cluster can fetch it directly.

```shell
flux create source git gitops-loop-demo \
  --url=https://github.com/YOUR-USERNAME/GitOps-Days.git \
  --branch=main \
  --interval=30s
```

> **Important:** Replace `YOUR-USERNAME` with your actual GitHub username. This must be **your fork**, not the original repo.
> Flux’s Source Controller will check this repo every 30 seconds for changes.

### ✅ Checkpoint: Confirm Flux is running and watching your repo

1. **Check that Flux components are healthy:**

   ```shell
   flux check
   ```

   Example output when healthy:

   ```shell
   ► checking prerequisites
   ✔ kubectl 1.32.0 >=1.20.0
   ✔ Kubernetes 1.32.2 >=1.20.0
   ✔ prerequisites checks passed
   ► checking controllers
   ✔ source-controller: deployment ready
   ✔ kustomize-controller: deployment ready
   ✔ helm-controller: deployment ready
   ✔ notification-controller: deployment ready
   ✔ all checks passed
   ```

2. **Verify your Git source is registered and ready:**

   ```shell
   flux get sources git
   ```

   Example output when ready:

   ```shell
   NAME                URL                                              READY   STATUS                                                              AGE
   gitops-loop-demo    https://github.com/YOUR-USERNAME/GitOps-Days     True    stored artifact for revision 'main@sha1:123abc456def...'             1m
   ```

**Pass criteria:**

* `flux check` shows all controllers as `✔ ... ready`
* `flux get sources git` shows your source with `READY` = `True`

**Checkpoint complete:** Flux is now installed in your cluster and watching your GitHub fork.
In the next section, we’ll tell Flux **what** to deploy from that repository.

## 💡 Before You Copy the Example Files

I update this repository frequently - adding lessons, improving examples, and fixing issues.
It’s a good idea to occasionally **sync your fork** with the upstream repo so you get those updates.

However, if you keep your changes directly in the shared `examples/` folder, **syncing could overwrite your work**. To avoid that:

* **Always copy** the example files into your own folder under `student-work/YOUR-USERNAME/`
* Make changes **only** in your own folder
* Point Flux at *your* folder, not the shared examples

This way, syncing upstream changes will **never overwrite your personal work**.

### ⚠️ A note about re-copying

If you repeat a lesson in the future and re-copy files from `examples/` into a folder that **already exists**, your existing files will be overwritten.

**How to avoid overwriting yourself:**

* If you want a fresh start, delete or rename your old folder first
* Or create a new subfolder (e.g., `student-work/YOUR-USERNAME/day2-v2`) and copy into that

**Next:** Now that you know how to keep your work safe, let’s copy the Day 2 example into your workspace and tell Flux what to deploy.

## 📦 Tell Flux What to Deploy

Now that your workspace is set up and you’ve copied the example into your own folder under `student-work/YOUR-USERNAME/`, it’s time to tell Flux **what** to deploy.

**⏱️ Time needed:** \~2 minutes

### 🛑 Important

Do **not** point Flux at the shared `examples/` folder.
If you do, any time you sync your fork with upstream changes, your work could be overwritten.

Always point Flux to your **own workspace folder**, e.g.:

```
./student-work/YOUR-USERNAME/day2/hello
```

### Create a Kustomization

Run:

```bash
flux create kustomization hello-app \
  --source=GitRepository/gitops-loop-demo \
  --path="./student-work/YOUR-USERNAME/day2/hello" \
  --prune=true \
  --interval=1m
```

**Flags explained:**

* `--path` → the folder in your repo where your manifests live (**your** workspace, not examples)
* `--prune=true` → remove cluster resources when the file is deleted from Git
* `--interval=1m` → check for drift and reconcile every minute

### ✅ Checkpoint: Confirm the Kustomization is Ready

```bash
flux get kustomizations
```

Example when ready:

```
NAME        READY  MESSAGE                                      REVISION               SUSPENDED
hello-app   True   Applied revision: main@sha1:123abc456def...  main@sha1:123abc...    False
```

**Checkpoint complete:** You’ve just given Flux its marching orders - a Kustomization that points to your workspace folder in Git.

Here’s the thing most people don’t expect: the moment you created that Kustomization, Flux didn’t wait for a commit. It immediately pulled the manifests from your folder and applied them to the cluster.

That means your very first GitOps-powered deployment has already happened in the background. Let’s go see what Flux just installed for you…

## 👀 See Your First Flux Deployment

In the last few minutes, Flux has been busy. Remember the folder in `student-work/YOUR-USERNAME/...` that you pointed Flux to? Let’s see what it found there and what it did with it.

**⏱️ Time needed:** \~5 minutes

### What Flux Discovered in Your Repository

When you created the Kustomization, you told Flux to watch your workspace folder. Inside that folder, Flux found:

```
student-work/YOUR-USERNAME/day2/hello/
├── namespace.yaml      # Creates the 'hello' namespace
├── deployment.yaml     # Defines pods running the web server
└── service.yaml        # Exposes the app on port 80
```

These YAML files define a simple “Hello World” web application. Flux pulled them from your GitHub fork and **deployed them automatically** to your cluster.

### See What Flux Created

Check your cluster for these resources:

```bash
kubectl get pods,svc -n hello
```

You should see something like:

```
NAME                         READY   STATUS    RESTARTS   AGE
pod/hello-65d4c4d5c9-xz7vp   1/1     Running   0          2m

NAME            TYPE        CLUSTER-IP      PORT(S)   AGE
service/hello   ClusterIP   10.96.x.x       80/TCP    2m
```

> **Note:** The `AGE` column should show only a few minutes - that’s how you know Flux just created them.

### The Automatic Deployment in Action

Think about this:

* ❌ You didn’t run `kubectl apply`
* ❌ You didn’t manually trigger a deployment
* ✅ Yet your application is running in the cluster

Flux:

1. Pulled your manifests from GitHub
2. Applied them to the cluster
3. Got everything running without your intervention

### Access Your Running Application

Forward the service port to your local machine:

```bash
kubectl port-forward -n hello svc/hello 8080:80
```

Then open [http://localhost:8080](http://localhost:8080) in your browser.

🎉 **There it is!** Your Hello World app, running in Kubernetes, deployed entirely by Flux.

**Checkpoint complete:** Flux is actively deploying workloads from your workspace folder in your GitHub fork.

Next, we’ll have some fun: we’ll *break* something in the cluster on purpose and watch Flux notice the drift - and put it back automatically.

## 🔨 Breaking Things (For Science!)

Now that Flux is watching your cluster, let’s try what would normally cause a **3 a.m. wake-up call** - and watch the system heal itself automatically.

We’ll run **two mini-labs** simulating real-world drift:

1. **The Quick Fix** - scaling a deployment manually
2. **The Accident** - deleting everything in the namespace

### 🧪 Mini-Lab 1: The Emergency Scale

**Scenario:**
A teammate, in a hurry during an incident, manually scales down your app.

**Action:**
Run this command to set the replicas to zero:

```bash
kubectl scale deployment hello -n hello --replicas=0
```

Then watch what happens live:

```bash
kubectl get deployment hello -n hello -w
```

> The `-w` flag means “watch” - you’ll see updates as they happen.

**Expected Result:**
Within \~30-60 seconds, you’ll see something like:

```
hello   0/1   0   0   10m30s   # Your manual change
hello   0/1   1   0   10m35s   # Flux starts fixing
hello   1/1   1   1   10m40s   # Back to normal!
```

**Why it happens:**
Flux detected that the cluster no longer matched what’s in Git (1 replica), so it reconciled it back automatically.

**Checkpoint ✅**
Deployment is back to **1/1 replicas**. Your manual change didn’t stick - Git’s declared state wins.

Press `Ctrl+C` to stop watching.

### 🧪 Mini-Lab 2: The Catastrophic Delete

**Scenario:**
Someone accidentally deletes the entire namespace.

**Action:**
Run:

```bash
kubectl delete namespace hello
```

Then watch Flux rebuild everything:

```bash
watch kubectl get all -n hello
```

**Expected Result:**
Over the next \~60 seconds you’ll see:

* Namespace recreated
* Deployment, service, and pods restored
* App back to its original state

**Why it happens:**
Flux sees the namespace and all its resources are missing, so it reapplies the manifests from Git until the cluster matches Git again.

**Checkpoint ✅**
Namespace **hello** and all resources are running exactly as defined in your repo. Drift = gone.

Press `Ctrl+C` when you see everything running.

### 📊 Why It Took \~60 Seconds

When you set up Flux, you configured:

* **Source check interval**: every 30 s (checks Git)
* **Reconciliation interval**: every 60 s (checks for drift)

See it in the events log:

```bash
flux events --for Kustomization/hello-app
```

Sample output:

```
Reconciliation finished in 1.2s, next run in 1m0s
```

### 💡 The Takeaway

Those “quick fixes” and “accidental deletes” that used to linger?
**They literally cannot persist now.**

Flux keeps your cluster locked to what’s in Git:

* Accidental deletions → rebuilt
* Manual scaling → reverted
* Any config drift → corrected

## 🔧 How Flux Works Under the Hood (Optional Deep Dive)

You've built a self-healing system and seen it work beautifully. For the rest of your GitOps journey (Days 3-5), you'll explore more advanced patterns with Flux. Understanding Flux's architecture now will make those advanced concepts much clearer.

**⏱️ Time needed:** ~5 minutes (or skip to Day 2 Complete if you're satisfied!)

### The Two-Controller Architecture

When you ran `flux install`, you deployed several controllers into your cluster. Let's focus on the two main ones that powered everything you just saw:

![Flux Architecture Diagram](assets/images/flux-architecture.png)

Looking at the diagram, you can see:

1. **Source Controller** (top) - Your Git watchdog
   - Connects to Git repositories (GitHub, GitLab, etc.)
   - Pulls changes every 30 seconds (remember our `--interval=30s`?)
   - Stores the files for other controllers to use

2. **Kustomize Controller** (middle) - Your cluster enforcer
   - Takes files from Source Controller
   - Applies them to your cluster
   - Detects and fixes drift every 60 seconds

But how do these controllers know what to watch and what to deploy? That's where those commands you ran become important...

### The Instructions You Gave Flux

Those two `flux create` commands weren't just setup steps-they created specific instructions for each controller.

**First, you created a GitRepository resource:**
```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: gitops-loop-demo
spec:
  interval: 30s          
  url: https://github.com/YOUR-USERNAME/GitOps-Days.git
  ref:
    branch: main
```

This GitRepository tells the Source Controller: "Watch this specific GitHub repository. Check it every 30 seconds. If you find new commits on the main branch, download the files." Think of it as a subscription-the Source Controller now subscribes to your repository.

**Then, you created a Kustomization resource:**
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: hello-app
spec:
  interval: 1m           
  sourceRef:
    kind: GitRepository
    name: gitops-loop-demo
  path: "./examples/day2-gitops-loop-demo/clusters/local/apps/hello"
  prune: true
```

This Kustomization tells the Kustomize Controller: "Look at the files the Source Controller downloaded from 'gitops-loop-demo'. Specifically, go to the hello app path. Apply whatever YAML you find there to the cluster. Check every minute if reality matches those files. Oh, and if files get deleted from Git, delete the corresponding resources too (that's what prune means)."

These two resources create a complete GitOps pipeline-from Git to cluster, automatically.

### How Controllers Collaborate During Reconciliation

Let's trace what happened during your scaling test to see this collaboration:

The Source Controller runs its check every 30 seconds:
- "Time for my regular check of the Git repository"
- "Still showing replicas: 1 in the deployment.yaml"
- "No changes to download, but I'll keep the files ready"

Meanwhile, the Kustomize Controller runs its reconciliation every 60 seconds:
- "Let me compare what's in the cluster to what Source Controller has"
- "Wait, the cluster shows 0 replicas but the files say 1"
- "That's drift! Applying the correct state now..."
- "Done. Deployment back to 1 replica"

This separation is powerful-one controller focuses on Git, the other on the cluster. They collaborate through those GitRepository and Kustomization resources you created, each doing their specialized job.

That's the engine behind your self-healing infrastructure. Simple, elegant, and now revealed.

Ready to celebrate what you've built?

## 🏆 Day 2 Complete: You Built the Impossible

You just did something most Kubernetes users only dream about-you built infrastructure that heals itself. Not in theory. Not in a demo. On your actual laptop, with real code.

### Your GitOps Transformation

**You started this morning with:**
- Just Docker and some tools
- Hope that self-healing was possible
- Memories of drift-induced headaches

**You're ending with:**
- A Kubernetes cluster that refuses to drift
- Flux watching your Git repository like a guardian
- Proof that 3am fixes can't persist
- Deep understanding of how GitOps delivers its magic

### What You Actually Built

- ✅ A complete GitOps pipeline (Git → Flux → Cluster)
- ✅ Automatic deployment from Git commits
- ✅ Self-healing that fixes drift in <60 seconds
- ✅ Your first taste of "kubectl-free" life

And if you did the deep dive, you also understand the two-controller architecture powering it all.

### The "Never Again" List

After today, you'll never again have to:
- ❌ Wonder if production matches Git
- ❌ Run `kubectl apply` for deployments
- ❌ Worry about undocumented manual changes
- ❌ Fear that someone's "quick fix" will persist
- ❌ Lose sleep over configuration drift

This isn't wishful thinking-you proved it works.

### The Journey So Far

**Day 1**: You understood why GitOps exists and how it works  
**Day 2**: You built proof that it actually works  
**Day 3**: Next, you'll take this to the cloud with AKS

### Your GitOps Capabilities

You can now:
- Explain GitOps with concrete examples (because you've seen it work)
- Deploy applications without touching kubectl
- Trust that your clusters will self-heal

### Tomorrow: Cloud Scale

Day 3 takes your local success to the cloud. You'll face new challenges:
- Deploy GitOps on Azure Kubernetes Service (AKS)
- Handle cloud-specific challenges
- Implement production-ready patterns
- See how this scales beyond your laptop

But here's the thing: the hard part is done. You understand GitOps. Tomorrow is just applying it at scale.

> [!TIP]
> Before tomorrow, try experimenting:
> - Change the replicas in your Git repo and watch Flux update it
> - Break things in new ways-Flux will keep fixing them!

### You Did It! 🎉

From "drift anxiety" to "drift immunity" in one day. That's not just learning-that's transformation.

Your clusters will never be the same. Neither will your weekends.

**See you tomorrow for Day 3!**

[Continue to Day 3: GitOps in the Cloud →](link)

---

*P.S. Proud of what you built? Share your success! #GitOpsDays*