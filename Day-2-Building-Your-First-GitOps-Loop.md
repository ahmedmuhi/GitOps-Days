# 🚀 Day 2 – Building Your First Self-Healing Kubernetes System with Flux

**Yesterday you learned the antidote to configuration drift. Today, you build it.**

Welcome back to GitOps-Days! Day 1 transformed how you think about Kubernetes operations—from fighting fires to preventing them. You discovered how GitOps eliminates the gap between Git and your cluster, learned the four principles that make infrastructure self-healing, and gained the vocabulary to implement real change.

Now it's time to see these principles in action. You'll build your first GitOps loop using Flux, watching it automatically detect and fix drift in real-time. No more theory—just hands-on proof that your clusters really can heal themselves.

### 🗺️ Your Hands-On Journey Today

In the next 60-70 minutes, you'll:

* Set up a local Kubernetes lab environment
* Create your Git repository as the source of truth
* Install Flux and watch it come alive
* Deploy an application using only Git commits
* Deliberately break things (for science!)
* Watch your system detect and fix itself automatically

By the end, you'll have tangible proof that the promise of self-healing infrastructure is real—running right on your laptop.

### 🎯 What You'll Build

This isn't a demo or a simulation. You'll create:
- A real Kubernetes cluster (local, but fully functional)
- A working GitOps pipeline powered by Flux
- Automatic deployment triggered by Git commits
- Self-healing that fixes drift within 60 seconds
- Confidence that you can implement this anywhere

Remember that frustration from Day 1—when manual changes caused drift? Today you'll deliberately cause drift and watch it disappear. That 3am emergency fix that haunted your documentation? Today you'll see why it can't persist in a GitOps world.

Ready to build infrastructure that refuses to stay broken? Let's begin!

## 🧰 Preparing Your Workspace

Before we build your self-healing cluster, let's ensure you have the right tools. Each one serves a specific purpose in creating the GitOps loop you learned about yesterday—no magic, just the right software for the job.

> **Tip:** Already have Docker, kind, kubectl, and Git installed?  
> [Jump straight to Fork and Clone the GitOps Repository](#fork-and-clone-the-gitops-repository).

> **Note:** This lab requires approximately 4GB of available RAM and 2GB of disk space.

### Required Tools

Here's what you'll need and why:

| Tool    | Min. Version | Purpose | Installation |
| ------- | ------------ | ------- | ------------ |
| Docker  | ≥24.0 | Runs the containers that make up your local Kubernetes cluster | [Install Docker](https://docs.docker.com/get-docker/) |
| kind    | ≥0.25.0 | Creates a Kubernetes cluster on your laptop for testing | [Install kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) |
| kubectl | ≥1.32 | The command-line tool you'll use to interact with Kubernetes | [Install kubectl](https://kubernetes.io/docs/tasks/tools/) |
| Git     | ≥2.40 | Stores and versions your infrastructure configurations | [Install Git](https://git-scm.com/downloads) |

### ✅ Verify Your Setup

Once installed, confirm everything is ready by running these commands:

```bash
docker --version
kind --version
kubectl version --client --short
git --version
```

You should see version numbers matching or exceeding the minimums above. If any command returns an error, revisit its installation guide.

> **Why verify?** Catching setup issues now prevents confusing errors later when you're focused on learning GitOps.

## 📂 Creating Your Source of Truth

Remember from Day 1: Git is your single source of truth in GitOps. But you can't push changes to someone else's repository—you need your own. That's why we start by forking.

**Forking creates your personal copy** of a repository where:
- You have full control to make changes
- Your commits will trigger real deployments in your cluster  
- You can experiment without affecting the original

Let's set this up:

### 1️⃣ Fork the Repository on GitHub

1. Go to [`https://github.com/ahmedmuhi/GitOps-Days`](https://github.com/ahmedmuhi/GitOps-Days)
2. Click the **Fork** button at the top right
3. Click **Create fork**

You now have your own copy at:
```
https://github.com/YOUR-USERNAME/GitOps-Days
```

### 2️⃣ Clone Your Fork Locally

Now bring your repository to your local machine:

```bash
git clone https://github.com/YOUR-USERNAME/GitOps-Days.git
cd GitOps-Days
```

Verify you're working with your fork:
```bash
git remote -v
```

The output should show `origin` pointing to your repository (with YOUR-USERNAME), not the original.

> ⚠️ **Common mistake:** If you see `ahmedmuhi` instead of your username, you cloned the original repository instead of your fork. Delete the folder and clone your fork instead—you'll need write access for the GitOps magic to work!

Perfect! You now have:
- ✅ All the tools needed to run Kubernetes locally
- ✅ Your own Git repository that will control your cluster
- ✅ Everything verified and ready to go

Next, we'll create the Kubernetes cluster where your self-healing system will come to life.

## 🔄 Creating Your Kubernetes Environment

Time to bring your cluster to life! In the next few minutes, you'll have a real Kubernetes cluster running on your laptop—your testing ground for GitOps magic.

This is where you'll see:
- Configuration drift happen in real-time
- Flux (our GitOps operator) automatically fix that drift
- Git truly control your cluster's state

Let's create it:

### 🏗️ Creating the Cluster

Run this command to create a cluster named `gitops-loop-demo`:

> **For Windows:** Use PowerShell or Windows Terminal  
> **For macOS/Linux:** Use Terminal

```bash
kind create cluster \
  --name gitops-loop-demo \
  --image kindest/node:v1.32.2
```

This creates a fully functional Kubernetes cluster in about a minute. We're using a specific version (`v1.32.2`) to ensure everyone gets the same experience.

### 🔍 Verifying Your Cluster

Once creation completes, confirm it's ready:

```bash
kubectl get nodes
```

You should see:
```
NAME                             STATUS   ROLES           AGE   VERSION
gitops-loop-demo-control-plane   Ready    control-plane   1m    v1.32.x
```

**What "Ready" means:** Your cluster is up, running, and waiting for action. This is your blank canvas where GitOps will paint its magic.

> 💡 **Cleanup tip:** When you're done with the lab, delete the cluster with:
> ```bash
> kind delete cluster --name gitops-loop-demo
> ```
> This frees up resources and gives you a clean slate for future experiments.

### 🎯 What You've Just Built

You now have:
- A real Kubernetes cluster (yes, on your laptop!)
- A safe environment to break things and watch them heal
- The foundation for your GitOps experiments

Next, we'll examine the application that Flux will deploy and manage in this cluster. This is where the theory from Day 1 starts to become reality.

## 📦 What Flux Will Deploy

Your forked repository contains a simple "Hello World" web application. When you explore the repository, you'll find:
- A web server that displays a welcome message
- Kubernetes YAML files (namespace, deployment, service)

But here's the exciting part: In a few minutes, Flux will automatically deploy all of this to your cluster—without you running a single `kubectl apply`.

The magic? You won't manually deploy anything. Flux will:
- Detect these YAML files in your Git repository
- Create all the Kubernetes resources they define
- Keep everything in sync automatically

Don't worry about understanding every line of YAML right now. The exciting part is watching GitOps principles come to life.

Once you see Flux work its magic, we'll circle back to explore what happened under the hood. For now, let's understand how Flux makes this automation possible.

Next up: Installing the GitOps engine that makes all this possible.

## 🚀 Activating Your GitOps Engine

Time to bring Flux to life! In the next 5 minutes, you'll install Flux and watch it automatically deploy your application—no `kubectl apply` needed.

### Step 1: Install Flux CLI (30 seconds)

Choose your platform:

**macOS:**
```bash
brew install fluxcd/tap/flux
```

**Linux:**
```bash
curl -s https://fluxcd.io/install.sh | sudo bash
```

**Windows (PowerShell):**
```powershell
choco install fluxcd
```

✅ Verify: `flux --version`

### Step 2: Deploy Flux to Your Cluster (1 minute)

```bash
flux install
```

This installs the GitOps operators that will manage your cluster.

### Step 3: Connect Flux to Your Repository (2 minutes)

Tell Flux where to find your configurations:

```bash
flux create source git gitops-loop-demo \
  --url=https://github.com/YOUR-USERNAME/GitOps-Days.git \
  --branch=main \
  --interval=30s
```
📝 Replace `YOUR-USERNAME` with your GitHub username!

Then tell Flux what to deploy:

```bash
flux create kustomization hello-app \
  --source=GitRepository/gitops-loop-demo \
  --path="./examples/day2-gitops-loop-demo/clusters/local/apps/hello" \
  --prune=true \
  --interval=1m
```

### 🎉 That's It! 

Flux is now:
- Watching your Git repository every 30 seconds
- Automatically deploying any changes
- Keeping your cluster in sync with Git

**No manual deployments. Ever again.**

Ready to see the magic? Let's verify your app is running...

## 🔎 Witnessing GitOps in Action – Your First Automatic Deployment

Remember all that manual `kubectl apply` work from traditional Kubernetes? You just skipped ALL of it. 

While you were setting up Flux, something magical happened in the background: Flux found your application in Git and deployed it automatically. Let's prove it!

### ✅ Verify Your Automatic Deployment

Check what Flux created without any manual intervention:

```bash
kubectl get pods,svc -n hello
```

You should see:
```
NAME                         READY   STATUS    RESTARTS   AGE
pod/hello-65d4c4d5c9-xz7vp   1/1     Running   0          2m

NAME            TYPE        CLUSTER-IP      PORT(S)   AGE
service/hello   ClusterIP   10.96.x.x       80/TCP    2m
```

**Think about what just happened:** You never told Kubernetes about these resources. You never ran `kubectl apply`. Flux detected them in your Git repository and created them automatically!

### 🌐 See Your App Running

Let's access your automatically-deployed application:

```bash
kubectl port-forward -n hello svc/hello 8080:80
```

Open [http://localhost:8080](http://localhost:8080) in your browser. You'll see your Hello World app running!

### 🎉 Celebrate This Moment

You've just experienced the core GitOps promise:
- **Git commit → Automatic deployment** ✓
- **No manual kubectl commands** ✓  
- **No cluster credentials needed outside Flux** ✓
- **Your first self-healing system is LIVE** ✓

But here's the best part: This system doesn't just deploy automatically—it HEALS automatically too.

Ready to have some fun? Let's break things and watch Flux fix them...

## 🔨 Breaking Things (For Science!)

Time for the moment you've been waiting for. You're about to deliberately break your cluster and watch it heal itself. This isn't just a demo—this is exactly how GitOps protects you from configuration drift in production.

### 🧪 Test 1: The Accidental Scale-Down

Remember that colleague who "temporarily" scaled down a deployment? Let's simulate that:

```bash
kubectl scale deployment hello -n hello --replicas=0
```

Now watch what happens:
```bash
kubectl get deployment hello -n hello -w
```

Within 30-60 seconds, you'll see:
```
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
hello   0/0     0            0           10m
hello   0/1     0            0           10m30s
hello   0/1     1            0           10m35s
hello   1/1     1            1           10m40s
```

**Flux just undid your manual change!** No alerts, no panic, no 3am wake-up call. Just automatic correction.

Press `Ctrl+C` to stop watching.

### 🧪 Test 2: The Nuclear Option

Let's get more destructive. Delete EVERYTHING:

```bash
kubectl delete namespace hello
```

This completely removes:
- Your deployment
- Your service  
- All pods
- The entire namespace

Now watch Flux rebuild from scratch:
```bash
watch kubectl get all -n hello
```

Every 2 seconds, you'll see the namespace and all resources being recreated. Within a minute, your application is back as if nothing happened.

Press `Ctrl+C` when you see everything running again.

### 📊 See Flux's Work in Real-Time

Want to see Flux catching and fixing these changes? Check the events:

```bash
flux events --for Kustomization/hello-app
```

You'll see entries like:
```
Reconciliation finished in 1.2s, next run in 1m0s
```

### 💡 What This Means for You

Those manual fixes that drift from Git? **They can't persist anymore.**
- Emergency scaling? Flux restores the declared state
- Accidental deletion? Flux rebuilds automatically  
- Cowboy kubectl commands? Flux reverts them

This is self-healing infrastructure in action. Your Git repository isn't just documentation—it's the law.

Next, let's dive deeper into the GitOps loop and see exactly how Flux detected and fixed your changes...

## 🎯 Understanding What Just Happened

Time to pull back the curtain. You've experienced the magic—now let's understand the mechanics.

### You Just Witnessed All Four GitOps Principles

Remember Day 1's principles? You just saw them ALL in action:

1. **Declarative**: Your YAML files declared the desired state
2. **Versioned & Immutable**: Everything lived in Git with full history
3. **Pulled Automatically**: Flux pulled from Git (you never pushed to the cluster)
4. **Continuously Reconciled**: Your manual changes were auto-corrected within 60 seconds

This wasn't a demo—this is how GitOps actually works.

### The GitOps Loop Revealed

When you ran `flux install`, you deployed a control loop powered by two key resources:

**🔍 GitRepository: Your Git Watchdog**
```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: gitops-loop-demo
spec:
  interval: 30s          # Check Git every 30 seconds
  url: https://...       # Your repository
  ref:
    branch: main
```

**🔧 Kustomization: Your Cluster Enforcer**
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: hello-app
spec:
  interval: 1m           # Reconcile every minute
  sourceRef:
    kind: GitRepository
    name: gitops-loop-demo
  path: "./examples/..."
  prune: true           # Remove deleted resources
```

Here's what happened behind the scenes:

```
[Git Repository] 
    ↓ (Flux detects files)
[Source Controller pulls changes]
    ↓ (Every 30 seconds)
[Kustomize Controller applies to cluster]
    ↓ (Every 60 seconds)
[Your app appears/heals automatically]
```

### Why Your Manual Changes Couldn't Persist

When you scaled to zero or deleted the namespace, here's the timeline:

1. **T+0s**: You break something with kubectl
2. **T+30s**: Source controller checks Git (still says replicas=1)
3. **T+60s**: Kustomize controller compares cluster to Git
4. **T+61s**: Drift detected! Flux reapplies Git configuration
5. **T+62s**: Your "fix" is reverted, cluster matches Git again

This is continuous reconciliation in action. Git always wins.

### The Architecture Behind Your Self-Healing System

Two controllers work together as your GitOps engine:

**Source Controller**
- Watches your Git repository
- Pulls changes when detected
- Stores them for other controllers
- This is why you never needed to give CI/CD cluster access!

**Kustomize Controller**  
- Takes configurations from Source Controller
- Applies them to your cluster
- Detects and fixes drift
- Ensures cluster state = Git state

Remember the security benefit? Your cluster pulled from Git. No external system needed credentials to push changes. This is the pull model protecting you.

### What Flux Actually Deployed

Let's demystify those YAML files in your repository:

1. **Namespace** (`hello`): A logical boundary for your app
2. **Deployment**: Manages your pod with the web server
3. **Service**: Makes your pod accessible on port 80

Simple resources, but the magic is HOW they got there:
- You never ran `kubectl apply`
- Flux found them in Git and deployed them
- When you deleted them, Flux recreated them
- They'll stay in sync with Git forever

No hidden complexity—just GitOps doing what it promises.

---

**You now understand the complete picture.** You've built a self-healing system, broken it on purpose, watched it recover, and learned exactly how it works.

Ready to take this to the cloud? Day 3 awaits...

## 🏆 Day 2 Complete: You Built a Self-Healing System!

Take a moment to appreciate what you've accomplished today. This is a milestone worth celebrating.

### Your Transformation

**You started today with:**
- Theoretical knowledge of GitOps principles
- Questions about how self-healing actually works
- Hope that automation could solve drift

**You finish today with:**
- A working GitOps pipeline on your laptop
- Hands-on proof that self-healing is real
- Confidence that drift is now a solved problem

### What You Actually Built

Look at what's running right now:
- ✅ A Kubernetes cluster that pulls its own configuration
- ✅ Automatic deployment triggered by Git commits
- ✅ Self-healing that fixes drift in under 60 seconds
- ✅ Security through pull-based deployment
- ✅ A system that literally cannot stay broken

You didn't just learn about GitOps—you implemented it.

### The Journey So Far

**Day 1**: You understood why GitOps exists and how it works  
**Day 2**: You built proof that it actually works  
**Day 3**: Next, you'll take this to the cloud with AKS

### Your GitOps Capabilities

You can now:
- Explain GitOps with concrete examples (because you've seen it work)
- Deploy applications without touching kubectl
- Trust that your clusters will self-heal
- Sleep better knowing drift can't persist

### What's Next?

Tomorrow in Day 3, you'll scale this up:
- Deploy GitOps on Azure Kubernetes Service (AKS)
- Handle cloud-specific challenges
- Implement production-ready patterns
- See how this scales beyond your laptop

But tonight? Celebrate. You've built your first self-healing system. That's no small achievement.

**See you tomorrow for Day 3!** 🚀

---

*P.S. Before you go—try one more thing: make a small change to your app in Git and watch it deploy automatically. Pure GitOps magic.*