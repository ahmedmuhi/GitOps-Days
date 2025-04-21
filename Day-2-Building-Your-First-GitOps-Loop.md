## 🚀 Day 2: Building Your First GitOps Loop — Flux CD on Your Laptop  

Welcome back!

[Yesterday](https://github.com/ahmedmuhi/GitOps-Days/blob/main/Day-1-What-really-is-GitOps.md), we unpacked the *what* and *why* of GitOps.  
You saw how GitOps isn't just "YAML in Git," but a powerful operational model based on four clear principles: declarative state, version control, pull-based delivery, and continuous reconciliation.

Today, it's time to make that theory real.

We're going to break Kubernetes—twice. And we'll watch GitOps put it back together. Automatically.

You'll create a local Kubernetes cluster, connect it to a GitHub repository, and experience the GitOps loop firsthand. Within an hour, you'll have Flux CD (our GitOps agent for today) running inside a KIND cluster, syncing to a GitHub repo, and healing your changes without you lifting a finger.

You'll see:
- The GitOps loop come alive—on your screen.
- Drift repaired instantly.
- Operational guardrails in action.

This isn't just about installing tools—it's about experiencing how GitOps *feels* when it's running. Today is your crash-test: manual failure, automatic repair, and a moment of "ohhh… now I get it."

Let's build.

## 🏗️ Setting the Stage

First, we'll need three key components for our experiment:
1. A GitHub repository (your source of truth)
2. A local Kubernetes cluster (your runtime environment)
3. Flux (the GitOps controller that connects them)

### 📋 Prerequisites

Before we begin, make sure you have these tools installed:

- **Docker**: Powers our local Kubernetes environment
- **Kind**: Creates a lightweight Kubernetes cluster
- **kubectl**: Lets you interact with the cluster
- **Git**: For interacting with your GitHub repository
- **Flux CLI**: We'll install this during the tutorial

Missing anything? Quick installation links:
- [Docker](https://docs.docker.com/get-docker/)
- [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
- [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)
- [Git](https://git-scm.com/downloads)

### 🧱 Step 1: Create Your Source of Truth Repository

In GitOps, your Git repository is the single source of truth. For this lab, you'll need your own GitHub repository that you can modify.

#### Why You Need to Fork the Repository

For GitOps to truly demonstrate its power, you need to make changes to the repository and watch Flux automatically apply them. This requires:
- A repository you can push changes to
- A clean, isolated environment for your experiments
- The ability to commit, push, and see the GitOps loop in action

Let's set up your repository:

1. Visit [https://github.com/ahmedmuhi/GitOps-Days](https://github.com/ahmedmuhi/GitOps-Days)
2. Click the "Fork" button in the top right to create your own copy
3. Clone your fork to your machine:

```bash
git clone https://github.com/YOUR-USERNAME/GitOps-Days.git
cd GitOps-Days
```

> 💡 **What's in this repository?**  
> The repository contains example manifests for our demo application and the configuration files we'll need for our GitOps experiment.

### 🧱 Step 2: Create a Kubernetes Cluster

Next, we'll spin up a small Kubernetes cluster using [kind](https://kind.sigs.k8s.io/).

Kind (Kubernetes IN Docker) runs Kubernetes inside a Docker container—ideal for quick, disposable clusters like this one.

Let's look at the provided configuration file first:

```yaml
# examples/day2-gitops-loop-demo/kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
```

This is a minimal configuration that creates a single-node Kubernetes cluster.

Now, let's create the cluster:

```bash
kind create cluster --name gitops-loop-demo --config examples/day2-gitops-loop-demo/kind-config.yaml
```

When it's ready, confirm everything is working:

```bash
kubectl get nodes
```

You should see something like:

```
NAME                          STATUS   ROLES           AGE   VERSION
gitops-loop-demo-control-plane Ready    control-plane   1m    v1.27.x
```

### 📦 Understanding Our Demo Application

For this lab, we're using a simple pre-built container called `nginxdemos/hello:plain-text`. This container:

- Runs a lightweight NGINX web server
- Displays a simple "Hello World" page
- Requires no custom code or building

We chose this application specifically because it lets us focus on the GitOps patterns without getting distracted by application development details. The immediate visual feedback will make it easy to confirm that our GitOps loop is working.

### 🧱 Step 3: Examine Our Application Manifests

Take a moment to look at what's in the example application folder:

```bash
ls -la examples/day2-gitops-loop-demo/clusters/local/apps/hello/
```

You'll find two key manifests:

#### 📄 The Deployment Manifest

This tells Kubernetes what to run:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
        - name: hello
          image: nginxdemos/hello:plain-text
          ports:
            - containerPort: 80
```

> 🧠 This instructs Kubernetes to run one copy of the `nginxdemos/hello` container and expose port 80.

#### 📄 The Service Manifest

This creates a network endpoint for the application:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello
spec:
  selector:
    app: hello
  ports:
    - port: 80
      targetPort: 80
```

> 🧠 This Service creates a consistent internal network address that routes traffic to our application pods. It's how other components or users can connect to our application.

#### 📂 Understanding the Folder Structure

The manifests follow a structure commonly used in GitOps repositories:

```
clusters/
└── local/
    └── apps/
        └── hello/
            ├── deployment.yaml
            └── service.yaml
```

| Folder | Purpose |
|--------|---------|
| `clusters/` | Where you declare cluster-specific configurations |
| `local/` | The environment name (could be dev/staging/prod in real scenarios) |
| `apps/hello/` | Where application-specific configurations live |

### 🧭 Where We Stand

Let's pause and take inventory:

✅ You've forked the **GitHub repository** to serve as your source of truth  
✅ You've spun up a **Kubernetes cluster** using Kind  
✅ You've examined the **manifests** for our simple application  
🟡 You haven't applied anything to the cluster yet (`kubectl apply`)  
🟡 No GitOps automation is running yet

Everything is still manual.  
And that's intentional.

You've built a clean, declarative foundation—ready for Flux to take over from here.

## 🔁 From *Manual* to *GitOps*: What Flux Actually Does  

You now have everything a Kubernetes engineer needs:

* **A forked GitHub repository** with your configuration  
* **A running Kubernetes cluster** on your machine
* **Two YAML manifests** (`deployment.yaml` and `service.yaml`) ready to deploy

In the traditional workflow, you would now type:

```bash
kubectl apply -f examples/day2-gitops-loop-demo/clusters/local/apps/hello
```

But we're **not** going to do that. Instead, your **cluster will configure itself** once Flux is connected to your GitHub repository.

> *GitOps is commit-driven operations*: you change **Git**, the cluster changes **itself**.  

No more one-off `kubectl apply` commands; every change is versioned, reviewed, and continuously reconciled.

This inversion of control is the essence of **GitOps**.

### Meet Flux — the Agent that Watches Git  

To make this happen, we need something **inside the cluster** that can:  

1. **Watch** your GitHub repository  
2. **Apply** whatever configuration it finds there  
3. **Keep everything in sync — forever**  

That's what **Flux** does.

Flux is a small bundle of Kubernetes controllers.  
It doesn't just install your app once; it **keeps the desired state alive**.  
If something drifts, it fixes it. If something changes in Git, it reacts.

### 🧩 How Flux Knows What to Do

Flux doesn't magically apply your app—it works from **two Kubernetes resources** that you define.

Let's break them down:

#### 📦 First: Tell Flux *Where to Look*
That's the job of a `GitRepository`.

It's a Kubernetes object you'll create that says:

> "Hey Flux, clone this GitHub repo—every 30 seconds—and keep it cached inside the cluster."

Here's what this resource looks like:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: gitops-demo
spec:
  url: https://github.com/YOUR-USERNAME/GitOps-Days.git
  branch: main
  interval: 30s
```

💡 **This resource just pulls.**  
- It clones your repo and updates it regularly
- It gives the repo a name: `gitops-demo`
- It does *not* apply anything yet

#### 🧭 Next: Tell Flux *What to Apply*

That's the job of a `Kustomization`.

This second Kubernetes object says:

> "Inside that repo you just cloned, go into this specific folder, and apply what you find there—on a loop."

In our case, that folder contains our hello app:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: hello-app
spec:
  sourceRef:
    kind: GitRepository
    name: gitops-demo
  path: ./examples/day2-gitops-loop-demo/clusters/local/apps/hello
  prune: true
  interval: 1m
```

Let's walk through what this means:

- `sourceRef.name: gitops-demo` — matches the GitRepository you defined above  
- `path: ./examples/day2-gitops-loop-demo/clusters/local/apps/hello` — tells Flux where your YAML files live  
- `interval: 1m` — re-applies it every minute  
- `prune: true` — removes anything from the cluster that no longer exists in Git

💡 **This resource triggers the loop.**  
Once you apply it, Git becomes your live control plane.

### 🤔 Why Do We Need Two Resources?

You might wonder: "Why not just have one big Flux object? Why split it into two?"

The answer is **separation of concerns**.

Here's why Flux separates "pulling" from "applying":

- You can **pull from one repo** and apply multiple folders (dev/staging/prod)
- You can **watch several repos**, each with its own purpose (infra/apps/secrets)
- You can **sync different parts** on different intervals
- You can **delegate ownership** of folders to different teams

This separation gives you flexibility, composability, and clean Git hygiene.

### 📁 So Where Do These Resources Come From?

You might notice we haven't included instructions for creating these Flux resources yet. That's intentional.

In practice, you don't manually write these resources. The Flux CLI will generate them for you in the next section. We're showing them to you now so you understand:
- What Flux will create
- How these resources tell Flux where to look
- How the GitOps loop gets established

These resources will only exist in your Kubernetes cluster, not in your Git repository. They're what connects Flux to your GitHub repo.

## 🚀 Install Flux & Wire It Up

It's time to experience the real power of GitOps! 

You've set up your GitHub repository with application manifests.
You've created your Kubernetes cluster.
Now you'll connect them using Flux—and watch the magic happen.

In this section, you'll:
- Install the Flux CLI and controllers
- Connect Flux to your GitHub repository
- Watch as your application deploys automatically
- See the GitOps loop in action for the first time

Let's go!

### 🛠️ 4.1: Install the Flux CLI

The Flux CLI is the tool you'll use to:

- Install Flux into your cluster  
- Generate and apply GitOps resources
- Check status, reconcile manually, and more  

#### 📦 On macOS or Linux:

```bash
curl -s https://fluxcd.io/install.sh | sudo bash
```

Verify installation:

```bash
flux --version
```

#### 🪟 On Windows:

You can install the CLI using [Chocolatey](https://chocolatey.org/):

```powershell
choco install fluxcd
```

Or download the binary directly from the [Flux GitHub Releases](https://github.com/fluxcd/flux2/releases).

### ✅ 4.2: Check That Everything Is Ready

Before installing anything, run:

```bash
flux check --pre
```

This checks:

- Is `kubectl` available?
- Is your cluster reachable?
- Is your Flux CLI version compatible?

If all green, you're good to go.

### 📦 4.3: Install Flux Controllers into Your Cluster

Now let's install the Flux engine into your Kubernetes cluster:

```bash
flux install
```

This installs several controllers into a dedicated namespace called `flux-system`, including:

- **source-controller** – fetches content from Git  
- **kustomize-controller** – applies manifests to the cluster  
- **notification-controller** – sends alerts (optional, not used yet)  
- **helm-controller** – manages Helm charts (also not used yet)

### 🔍 4.4: Confirm That Flux Is Running

Check the pods:

```bash
kubectl get pods -n flux-system
```

You should see output like:

```
NAME                                  READY   STATUS    RESTARTS   AGE
source-controller-xxxxx               1/1     Running   0          10s
kustomize-controller-xxxxx            1/1     Running   0          10s
...
```

At this point, **Flux is alive but idle**.  
It's running inside your cluster, but it has no Git repo to watch yet.

Let's connect it to your GitHub repository.

### ✍️ 4.5: Create the GitRepository Resource

Now we'll tell Flux to watch your GitHub repository.

Run this command, replacing YOUR-USERNAME with your actual GitHub username:

```bash
flux create source git gitops-demo \
  --url=https://github.com/YOUR-USERNAME/GitOps-Days.git \
  --branch=main \
  --interval=30s
```

This creates a GitRepository resource in your cluster that:
- Points to your forked repository
- Checks for changes every 30 seconds

Verify it's working:

```bash
flux get sources git
```

You should see your repository listed as "Ready".

> 💡 **Note**: For simplicity, we're using a public GitHub repository without authentication. In production environments, you would typically use SSH keys or tokens for private repositories.

### ✍️ 4.6: Create the Kustomization Resource

Now tell Flux what to apply from that repo:

```bash
flux create kustomization hello-app \
  --source=GitRepository/gitops-demo \
  --path="./examples/day2-gitops-loop-demo/clusters/local/apps/hello" \
  --prune=true \
  --interval=1m
```

This creates a Kustomization resource that:
- References the GitRepository you just created
- Points to the folder with your application manifests
- Checks for changes and applies them every minute
- Removes resources that no longer exist in Git (prune)

> 🧠 **Important Note**: Unlike the GitRepository and Kustomization examples we looked at earlier, we're now *directly applying* these resources to your cluster instead of saving them as YAML files. Flux itself will operate based on these cluster resources.

### 👀 4.7: Watch Flux Deploy Your App

Now the exciting part—watch Flux deploy your application automatically:

```bash
flux get kustomizations
```

Look for:

```
NAME       READY   STATUS     AGE
hello-app  True    Applied    1m
```

Now check if your app is running:

```bash
kubectl get pods
kubectl get svc hello
```

You should see the `hello` pod running, and a Service called `hello`.

To verify it works, access the application:

```bash
kubectl port-forward svc/hello 8080:80
```

Then open [http://localhost:8080](http://localhost:8080) in your browser.

🎉 **Boom!** Hello from NGINX—without you running `kubectl apply`.

### 🧠 4.8: A Moment of Reflection

Let think about this for a moment:

✅ You didn't directly apply the Deployment or Service  
✅ You didn't trigger a pipeline or CI/CD job  
✅ Flux automatically detected your manifests in GitHub and applied them

Your Kubernetes cluster is now **continuously syncing** with your GitHub repository.

That's GitOps.  
You've just set up your first GitOps loop and watched it in action.

### 🔮 What's Next?

Now that Flux is watching your GitHub repository, let's see what happens when things drift.

In the next section, we'll break the cluster on purpose—and watch Flux heal it automatically.
That's the power of **continuous reconciliation**.

## 🔄 Flux in Action: Two Quick Self-Healing Drifts  

So far, you've seen GitOps take the wheel:  
You connected Flux to your GitHub repository—and it automatically deployed your application.

Now let's take it one step further.  
Let's break the cluster.  
Just a little.

And watch Flux put it back.

### 🧪 Drift #1 — Scale the App to Zero  

Flux believes `replicas: 1` is the truth—because that's what's in your GitHub repository.  
So what happens if we scale it down to zero?

Let's try it:

```bash
kubectl scale deployment/hello --replicas=0
```

Now monitor what happens:

```bash
# If you have the watch command:
watch kubectl get deployment/hello

# If you don't have watch, run this command repeatedly:
kubectl get deployment/hello
```

You'll see `0/0` pods briefly... then within a minute, Flux will restore it to `1/1`.

🧠 **What just happened?**  
Flux noticed that the live state didn't match what was in GitHub—and it quietly put things back.

### 🧨 Drift #2 — Delete the App Completely  

Let's go a bit further. What if someone accidentally deleted the whole app?

```bash
kubectl delete deployment/hello service/hello
```

Now monitor what happens:

```bash
# With watch:
watch kubectl get deployment,service

# Without watch (run repeatedly):
kubectl get deployment,service
```

Flux won't panic. It'll just re-read the manifests from GitHub—and reapply them.

Within a minute or less, you'll see both the Deployment and Service return.

🧠 **GitHub is still the truth.**  
Even if something vanishes, Flux restores it. That's reconciliation in action.

### 🧠 So What Did You Just Witness?

These two moments—scaling to zero and deleting everything—weren't just tests.

They were **proof** that Git is no longer just a backup. It's **active infrastructure**.

You now have a cluster that watches for drift and corrects it.  
No alerts. No panic. Just *quiet restoration*.

Your YAML is no longer just "config." It's reality—because Flux enforces it.  
That's the power of declarative systems. That's the heart of GitOps.

## ✅ Wrapping Up: What You Just Built (And Why It Matters)

Take a moment to congratulate yourself 🎉

You didn't just run a bunch of commands today. You built a system.  
A loop. A contract between GitHub and your cluster.

Let's step back and look at what that really means:

- You defined your application's desired state in GitHub  
- You connected Flux to continuously monitor your repository  
- You watched your cluster pull and apply configurations automatically  
- You deliberately broke things—twice—and watched the system heal itself

That's not scripting. That's not a tutorial.  
That's **infrastructure with a memory**.

You've just experienced the difference between:

> *"I hope this is running the way I left it…"*  
> and  
> *"I know exactly what state we should be in—because it's written in GitHub."*

What you saw today is the foundation of modern platform operations:  
Declarative configuration. Continuous reconciliation. Git as the single source of truth.

It wasn't theoretical. You built it. You watched it work. You even watched it recover.

That's GitOps, in action.

### 🗺️ Coming Up on Day 3  

Next, we'll take what you've built and expand it into more advanced GitOps patterns:

- Separate environments (`dev` / `staging` / `prod`) using folder-based structures  
- Walk through real-world promotion flows  
- Introduce drift-tolerant overrides (like `flux suspend`) for safe debugging  
- Compare Flux with Argo CD
- Explore GitOps security best practices

By the end of Day 3, you'll be thinking like a platform engineer—and Git will be thinking for you.

Ready?

**We'll see you there.**