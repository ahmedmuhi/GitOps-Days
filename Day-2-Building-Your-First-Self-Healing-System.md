# 🚀 Day 2 – Building Your First Self-Healing Kubernetes System with Flux

Yesterday we explored the antidote to one of the most frustrating problems in Kubernetes operations: **configuration drift**.
That moment when the running system doesn’t match what you think is deployed - maybe because someone made a “quick fix” in production and never committed it, or a setting silently changed without explanation. In the old world, that meant 3 a.m. firefights, scrambling through logs, and hoping you could piece the cluster back together.

GitOps changes that story. By making Git your single source of truth and letting the cluster reconcile itself, drift can’t persist - and today, you’ll see that in action.

## 🏁 Welcome Back to GitOps-Days

If any of the terms from Day 1 - *desired state*, *pull model*, or the *four GitOps principles* - aren’t fresh in your mind, you can [review them here](./Day-1-What-really-is-GitOps.md) before diving in.

Today, we’ll use **[Flux](https://fluxcd.io/)** - a GitOps operator - to build a live, breathing system that syncs with your repo, detects drift, and heals itself automatically. You’ll turn yesterday’s concepts into something you can watch work in real time.

## 🗺️ Your Hands-On Journey (\~1 hour)

You’ll:

1. Set up a local Kubernetes lab environment (\~15 min)
2. Create a Git repository as your single source of truth (\~5 min)
3. Install Flux and connect it to your cluster (\~10 min)
4. Deploy an application entirely through Git commits (\~10 min)
5. Break things on purpose and watch Flux fix them (\~15 min)

No cloud accounts. No complex setup. Just Docker and a few free tools.

## 🎯 By the End, You’ll Have

* A real Kubernetes cluster with Flux keeping it in sync with your repo
* Automated deployments triggered by pushes to Git
* Continuous drift detection and correction in under a minute
* A system that can recover from mistakes without your intervention

**Ready to turn the pain of drift into the peace of self-healing?**
Let’s get started.

## 🧰 Preparing Your Workspace

Before we dive into building your self-healing system, let’s set up the **essential building blocks** it relies on. These tools form the foundation for everything you’ll do today - once they’re in place, Flux will be able to work its GitOps magic.

Later in this lab, we’ll install **[Flux](https://fluxcd.io/)** - the GitOps operator that keeps your cluster in sync with Git - but first, we need this solid groundwork.

> [!TIP]
> Already have Docker, kind, kubectl, and Git installed?
> [Skip ahead to “Set Up Your Git Repository”](#📂-set-up-your-git-repository).

> [!IMPORTANT]
> You'll need at least **4 GB of free RAM** and **2 GB of disk space** for the local cluster.

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
In GitOps, Flux constantly pulls from a Git repository to know what you cluster should look like. To make changes that Flux will actually see and apply, you need **write access** to that repository. You can't commit to someone else's repo, so forking creates you own copy in your GitHub account - one you control completely.

Your fork will live at:

```
https://github.com/YOUR-USERNAME/GitOps-Days
```

### 2️⃣ Clone Your Fork Locally

```shell
git clone https://github.com/YOUR-USERNAME/GitOps-Days.git
cd GitOps-Days
```

> [!IMPORTANT]
> Replace `YOUR-USERNAME` with your actual GitHub username before running the command.
> If you copy it exactly as show above without replacing it, the command will fail.

### 3️⃣ Checkpoint: Confirm Your Local Repo Is Linked to Your Fork

Run:

```shell
git remote -v
```

Expected output:

```
origin  https://github.com/YOUR-USERNAME/GitOps-Days.git (fetch)
origin  https://github.com/YOUR-USERNAME/GitOps-Days.git (push)
```

✅ This means:

* You have a fork in your GitHub account.
* You've cloned it locally.
* Your local repo is connected to your fork, so pushes will go to a place you control

❌ If you see `ahmedmuhi` instead of your username, you cloned the original repo by mistake. Delete the folder and clone your fork instead - Flux won't work otherwise.

**Checkpoint complete:** You now have a GitOps-ready repository with full write access, linked locally, and ready for Flux to watch.

Next: we’ll create the Kubernetes cluster where your self-healing system will run.

## 🔄 Creating Your Kubernetes Environment

Your Git repository is ready - now it's time to prepare the **stage** where GitOps will actually perform it's magic.
We'll spin up a local Kubernetes cluster using **[kind](https://kind.sigs.k8s.io/)** (*Kubernetes in Docker*), which runs an entire cluster inside Docker containers.

Why kind? For GitOps experiments, it's perfect:

* ⚡ **Fast** - launches in about a minute
* 🔄 **Rebuildable** - tear down and recreate easily to start fresh
* 🧪 **Safe** - keeps your work local so you can experiment without touching production

This gives you a fast feedback loop: test your GitOps setup locally, confirm it works, and only then consider running it in production.

⏱️ **Time needed:** \~2 minutes

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
Before we bring in Flux, we need to be sure the cluster is running and ready to accept workloads. If the cluster isn’t healthy, you'll run into errors later.

Run:

```shell
kubectl get nodes
```

**Pass criteria:**

* `STATUS` shows **Ready**.
* The `AGE` is just a few minutes (freshly created).

Example output:

```
NAME                             STATUS   ROLES           AGE   VERSION
gitops-loop-demo-control-plane   Ready    control-plane   1m    v1.32.x
```

If you see `NotReady`:

1. Wait a few seconds and try again.
2. Make sure Docker is running:
    ```shell
    docker ps
    ```
3. Ensure you have a least **4 GB RAM** and **2 GB disk** free
4. If still stuck, recreate the cluster:

    ```shell
    kind delete cluster --name gitops-loop-demo
    kind create cluster --name gitops-loop-demo
    ```

### What You’ve Just Built

* ✅ A fully functional Kubernetes cluster running locally
* ✅ A safe, disposable playground for your GitOps experiments
* ✅ The environment Flux will soon manage, keeping it in sync with your Git repo

> [!TIP]
> When you’re done with the lab, you can delete the cluster to free resources:
>
> ```shell
> kind delete cluster --name gitops-loop-demo
> ```

## 🚀 Deploy Flux to Your Cluster and Connect It to Your Repo

Your cluster is up and running - now it's time to bring in the **director** of our GitOps pla: **Flux**.

Flux is not a single binary that “just runs” - when you install it, it deploys several **specialized controllers** in your cluster.
These controllers live together in their own namespace (`flux-system`) and work like a team of automation agents:

* **Source Controller** - pulls manifests from your Git repository
* **Kustomize Controller** - applies those manifests to your cluster
* **Helm Controller** - manages Helm releases (if you use them)
* **Notification Controller** - sends alerts and status events

Together, they ensure that what's running in your cluster always matches what's in Git - continuously and automatically.

⏱️ **Time needed:** \~3-4 minutes

### Install the Flux controllers

Run:

```shell
flux install
```

This sets up all the core Flux components insdie the `flux-system` namespace. From here on, Flux will be applying your declared state and fixing any drift.

### Connect Flux to your Git repository

Next, we tell Flux **Source Controller** which Git repository to watch. This must be **your fork on GitHub**, not the original.

```shell
flux create source git gitops-loop-demo \
  --url=https://github.com/YOUR-USERNAME/GitOps-Days.git \
  --branch=main \
  --interval=30s
```

> [!IMPORTANT]
> Replace `YOUR-USERNAME` with your actual GitHub username.
> We use the HTTPS URL so Flux can pull directly from GitHub
> The `--interval=30s` means Flux will check for changes every 30 seconds.

### ✅ Checkpoint: Confirm Flux is healthy and watching your repo

1. **Check that Flux components are healthy:**

   ```shell
   flux check
   ```

   Example whenall is healthy:

   ```shell
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

**What this means:**
The Source Controller has made a copy of your GitHub fork and stored it locally inside your cluster. It will refresh this cached copy every 30 seconds (or whatever interval you set), so your cluster always has the latest version of your repo ready to use.
Later, when we tell Flux *what* to deploy, the Kustomize Controller will read those files from this local cache - not directly from GitHub - and deploy them to your cluster.

**Checkpoint complete:** 
Flux is now **installed in your cluster** and **watching your GitHub fork**.
Before we tell Flux *what* to deploy from that repository, there's one last bit of setup to keep your work safe.

## 💡 Before You Copy the Example Files

I update this repository frequently - adding lessons, improving examples, and fixing issues.
If you make changes directly in the shared `examples/` folders and later *sync your fork* with the upstream repo, those changes could be overwritten.

To prevent that:

* **Create your own workspace folder** under `student-work/YOUR-USERNAME`
* **Copy** the example files you'll be working on into that folder
    (e.g., copy the Day 2 `hello` app example there)
* Make changes **only** inside your workspace folder
* Later, when we tell Flux what to deploy, you'll point it to *your* folder - not the shared examples

This way, syncing upstream changes will never overwrite your personal work.

### 📋 Commands to Create Your Workspace and Copy the Example

**macOS/Linux (Terminal):**

```shell
mkdir -p student-work/YOUR-USERNAME/Day2
cp -r examples/days/clusters/local/apps/hello student-work/YOUR-USERNAME/day2/
```

**Windows (PowerShell):**

```shell
New-Item -ItemType Directory - Path "student-work\YOUR-USERNAME\day2" -Force
Copy-Item -Recurse examples\day2\clusters\local\apps\hello student-work\YOUR-USERNAME\day2\
```

> [!IMPORTANT]
> Replace `YOUR-USERNAME` with your actual GitHub username before running these commands.

### ⚠️ A note About Re-copying

If you repeat a lesson in the future and re-copy files from `examples/` into a folder that **already exists**, your previous work will be overwritten.

**How to avoid overwriting yourself:**

* If you want a fresh start, delete or rename your old folder first
* Or create a new subfolder (e.g., `student-work/YOUR-USERNAME/day2-v2`) and copy into that

**Next:** We'll copy the Day 2 example into your workspace and tell Flux what to deploy - and when we do that, something will happen immediately that surprises most people on their first run.

## 📦 Tell Flux What to Deploy

You've got Flux installed, your repository linked, and you know how to keep your work safe.
Now it's time to give Flux a very specific instruction: **what to deploy** from your Git repo into the cluster.

This is done by creating a **Kustomization** - a Kubernetes object that Flux treats like marching orders.
It tells Flux:

* **Where** in your Git repository to find your application configuration manifests
* **How often** to check them
* **What to do** when something changes (or is removed)

**⏱️ Time needed:** \~2 minutes

### 🛑 Important: Point to Your Own Workspace

Do **not** point Flux at the shared `examples/` folder.
If you do, the next time you sync your fork with upstream changes, your work could be overwritten.

Instead:

* Work only inside your **own** folder under `./student-work/YOUR-USERNAME/`
* Point Flux to *your* folder, not the shared examples

For example:

```shell
./student-work/YOUR-USERNAME/day2/hello
```

### 🛠️ Create a Kustomization

Now tell Flux to deploy from your workspace folder:

Run:

```bash
flux create kustomization hello-app \
  --source=GitRepository/gitops-loop-demo \
  --path="./student-work/YOUR-USERNAME/day2/hello" \
  --prune=true \
  --interval=1m
```

**Flags explained:**

* `--path` → The folder in your repo where your manifests live
* `--prune=true` → Remove cluster resources when the file is deleted from Git
* `--interval=1m` → Check for drift and reconcile every minute

### ✅ Checkpoint: Confirm the Kustomization is Ready

```bash
flux get kustomizations
```

Example when ready:

```
NAME        READY  MESSAGE                                      REVISION               SUSPENDED
hello-app   True   Applied revision: main@sha1:123abc456def...  main@sha1:123abc...    False
```

**Checkpoint complete:** You’ve just given Flux its marching orders.

Here’s the part that surprises most people: the moment you created that Kustomization, Flux didn’t wait for you to push a commit. It immediately pulled the manifests from your folder and applied them to the cluster.
Your very first GitOps-powered deployment has already happened in the background.

**Next:** Let’s go see what Flux just installed for you.

## 👀 See Your First Flux Deployment

The moment you created your Kustomization, Flux got to work.
It didn't wait for a new commit - it immediately pulled the manifests from your workspace folder and applied them to your cluster.

Let's see exactly what happened.

**⏱️ Time needed:** \~5 minutes

### 🔍 What Flux Discovered in Your Repository

When you created the Kustomization, you told Flux to watch your workspace folder. Inside that folder, Flux found:

```
student-work/YOUR-USERNAME/day2/hello/
├── namespace.yaml      # Creates the 'hello' namespace
├── deployment.yaml     # Defines pods running the web server
└── service.yaml        # Exposes the app on port 80
```

These YAML files define a simple “Hello World” web application. The Kustomize Controller read them from the cached copy of your repository that Flux's Source Controller originally fetch from your GitHub Repository, and **deployed them automatically** to your cluster.

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

That's because Flux:

1. Pulled your manifests from GitHub
2. Applied them to the cluster
3. Got everything running without any manual steps

### Access Your Running Application

Forward the service port to your local machine:

```bash
kubectl port-forward -n hello svc/hello 8080:80
```

Then open [http://localhost:8080](http://localhost:8080) in your browser.

🎉 **There it is!** Your Hello World app, running in Kubernetes, deployed entirely by Flux.

### ✅ Checkpoint complete:

Flux is actively deploying workloads from your workspace folder in your GitHub fork.
Next, let's go beyond simply watching Flux deploy what's already there, we'll make a real change in Git, push it, and watch Flux pick it up and apply it to the cluster automatically.

## ✏️ Make Your First GitOps-Driven Change

So far, Flux has deployed what was alreay in your workspace folder.
Now, let's prove that **pushing a Change to Git** is all it takes to update your cluster.

**⏱️ Time needed:** \~5 minutes

### 1️⃣ Edit Your Deployment

Open:

```shell
student-work/YOUR-USERNAME/day2/hello/deployment.yaml
```

Find:

```yaml
spec:
  replicas: 1
```

... and change it to:

```yaml
spec:
  replicas: 3
```

This tells Kubernetes to run three pods instead of one.

### 2️⃣ Commit and Push Your Change

> [!TIP]
> You can do this because you forked the repository earlier
> If you had cloned the original repo, the `git push` command below would fail - and Flux wouldn't see your change.

Run:

```shell
git add student-work/YOUR-USERNAME/day2/hello/deployment.yaml
git commit -m "Scale hello app to 3 replicas"
git push
```

### 3️⃣ Watch Flux Reconcile

Flux checks your Git source every 30 seconds.
Let's watch it notice the new commit and update being applied to your cluster:

```shell
kubectl get deployment hello -n hello -w
```

You'll see the replica count change from 1 → 3 within a minute.

### ✅ Checkpoint: Git Changes = Live Changes

Flux just:

1. Pulled your updated repo from GitHub (via the Source Controller cache)
2. Saw that the desired state now had `replicas: 3`
3. Applied the change (via the Kustomize Controller) to your cluster automatically

No `kubectl apply`. No manual deploys.
Just Git → Flux → Cluster, exactly as GitOps promises.

Next, we’ll put that self-healing promise to the test: we’ll *break* something in the cluster on purpose and watch Flux notice the drift - and put it back automatically.

## 🔨 Breaking Things (For Science!)

Seeing Flux deploy you app automatically is great - but the **real proof** of a self-healing system is how it responds when things inevitably go wrong.
Let's test that resilience with **two real-world drift simulations**.

### 🧪 Mini-Lab 1: The Emergency Scale

**Scenario:**
A teammate is in the middle of an incident. Under pressure, they bypass Git and run a quick fix in the cluster - scaling the app manually.

**Action:**
Scale the deployment from 3 to 5:

```bash
kubectl scale deployment hello -n hello --replicas=5
```

Now watch what happens live:

```bash
kubectl get deployment hello -n hello -w
```

> [!TIP]
> The `-w` flag means “watch” - you’ll see updates as they happen.

**Expected Result:**
Within \~30-60 seconds, you’ll see something like:

```
hello   3/3   3   3   15m
hello   5/5   5   5   15m15s   # Manual change applied
hello   3/3   3   3   15m40s   # Flux detects drift and fixes it
```

**Why it happens:**
Flux's Kustomize Controller compared the cluster's live state to your Git-defined desired state (3 replicas).
Seeing a mismatch, it reconciled the deployment back to what's in Git - no questions asked.

**Checkpoint ✅**
Your deployment is back at **3/3 replicas**.
Manula changes didn't stick - **Git's declared state wins**.

Press `Ctrl+C` to stop watching.

### 🧪 Mini-Lab 2: The Catastrophic Delete

**Scenario:**
A worst-case accident - the entire `hello` namespace is deleted.

**Action:**
Run:

```shell
kubectl delete namespace hello
```

Then watch Flux rebuild everything:

```shell
watch kubectl get all -n hello
```

**Expected Result:**
Over the next \~60 seconds you’ll see:

* Namespace recreated
* Deployment, service, and pods restored
* App fully backonline

**Why it happens:**
The namespace and its resources are still defined in Git.
When the Kustomize Controller notices they're missing, it reapplies the manifests from the Source Controller's latest copy of your repo until the cluster matches Git again.

**Checkpoint ✅**
Namespace **hello** and all resources are running exactly as defined in your repo. Drift is gone.

Press `Ctrl+C` when you see everything running again.

### 📊 Why It Took \~60 Seconds

When you set up Flux, you configured:

* **Source check interval** → every 30 s (updates the cached Git copy)
* **Reconciliation interval** → every 60 s (compares cluster vs. Git and applies fixes)

See the timing in the events log:

```bash
flux events --for Kustomization/hello-app
```

Sample output:

```
Reconciliation finished in 1.2s, next run in 1m0s
```

### 💡 The Takeaway

With GitOps:

* Accidental deletions → **rebuilt**
* Manual scaling → **reverted**
* Any config drift → **corrected**

**Manual fixes that don't exist in Git literally cannot persist.**
The cluster is always brought back to the declared state - automatically, continuously, and reliably.

## 🏆 Day 2 Complete: You Built a Self-Healing Cluster

Today, you didn't just learn about GitOps - you proved it works.

### What You Achieved

* **A local Kubernetes cluster** running on kind + Docker
* **Flux installed** and watching your personal GitHub fork
* **Automatic depoyments** from Git commits
* **Drift detection and correction** in under a minute
* Confidence that quick fixes and accidental deletes **can't persist**

### Your Journey Today

1. Prepared your workspace with Docker, kind, kubectl, and Git
2. Forked and cloned your own GitOps repo
3. Installed Flux and connected it to your fork
4. Deployed your first app with Kustomization
5. Made a Git change and watched Flux apply it automatically
6. Broke things on purpose, and watched Flux heal them

### What It Matters

You now have a working Git → Flux → Cluster pipeline.
Your cluster state is no longer guesswork, it's **enforced reality**

### Up Next - Day 3: GitOps in the Cloud

Tomorrow, we take your local success to **Azure Kubernetes Service (AKS)**:

* Cloud-specific GitOps patterns
* Managing screts securely
* Production-ready deployment flows

THe core GitOps loop stays the same - Git defines, Flux enforces, but the stage gets bigger.

> [!TIP]
> 
> Between now and day 3, try:
>
> * Changing the `replicas` in your Git repo and watch Flux update it
> * Break things in new ways - Flux will keep fixing them

**See you tomorrow for Day 3!**

[Continue to Day 3: GitOps in the Cloud →](https://github.com/ahmedmuhi/GitOps-Days/blob/main/Day-3-Production-GitOps-on-AKS-with-GitHub-Actions.md)

> [!NOTE]
> *Proud of what you built? Share your success! #GitOpsDays*
