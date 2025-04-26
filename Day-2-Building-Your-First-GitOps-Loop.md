## 🚀 Day 2: Your First GitOps Loop — Running Flux CD on Your Laptop

Welcome back.

[Yesterday](Day-1-What-really-is-GitOps.md), we unpacked the *what* and *why* of GitOps.  
You saw how it's not just "YAML in Git"—but a model built on four principles: declarative state, version control, pull-based delivery, and continuous reconciliation.

You’ve now understood the model.  
Today, you'll experience it—in action.

We’re going to break Kubernetes—twice—and watch GitOps put it back together. Automatically.

In less than an hour, you'll have Flux CD running inside a local Kubernetes cluster (powered by Kind), syncing from your own GitHub repository, and watching for signs of drift.

You'll:
- See the GitOps loop come alive—visually and automatically  
- Trigger real-time recovery after intentional failure  
- Operate through Git instead of your terminal—with clarity and control

This isn't just about installing tools.  
It’s about *feeling the shift* GitOps introduces: from commands to commits, from manual recovery to automatic trust.

Today is your GitOps crash test: manual failure, automatic repair, and a moment of  
> _"Ohhh… now I get it."_

Let’s start the loop.

## 🧰 Get Ready to Run GitOps

Before we launch our cluster or start syncing from Git, let’s make sure your system is ready to go.

You’ll need a few tools installed—nothing unusual, just the standard gear for working with Kubernetes locally.

Here’s what you’ll want to have:

| Tool       | Purpose |
|------------|---------|
| **Docker** | Powers our local Kubernetes cluster (via Kind) |
| **Kind**   | Creates a local Kubernetes cluster inside Docker |
| **kubectl**| Lets you interact with the cluster |
| **Git**    | Connects your machine to your GitHub repo |
| **Flux CLI** | Used to install and manage Flux (we’ll install this later) |

> 💡 If you already have these tools installed, you’re good to go.

Otherwise, here are quick install links:

- [Docker](https://docs.docker.com/get-docker/)
- [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
- [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)
- [Git](https://git-scm.com/downloads)

No need to install Flux yet—we’ll walk through that together in a later step.

## 🗃️ Set Up Your GitOps Repository

In GitOps, everything starts with Git.

Your repository isn’t just where you store manifests—it’s where you declare the **truth** about your system. And that’s what your cluster will sync to, over and over again.

In this lab, we’re going to use a pre-built repository that contains all the files you need for Day 2:  
- A simple demo app  
- The Kubernetes manifests to deploy it  
- A production-style folder structure

But here’s the key: you need your own copy of it.  
You’ll be editing it, committing to it, and watching your cluster respond.

### ✅ Fork the Repository

To set this up:

1. Visit  
   👉 [https://github.com/ahmedmuhi/GitOps-Days](https://github.com/ahmedmuhi/GitOps-Days)  
2. Click the **“Fork”** button (top right)
3. Clone your fork:

```bash
git clone https://github.com/YOUR-USERNAME/GitOps-Days.git
cd GitOps-Days
```

This copy is now your GitOps **source of truth**.  
The cluster won’t care where it came from—only that it stays aligned.

### 💡 What’s Inside?

This repository includes everything you’ll need for today’s exercise:

- ✅ Kubernetes manifests for our demo app  
- ✅ A structured layout that mirrors real-world GitOps setups  
- ✅ The exact files Flux will watch in upcoming steps

We’ll explore this structure shortly. For now, you’ve got your control plane in Git—next, we’ll create one in Kubernetes.

## 🧱 Spin Up a Local Kubernetes Cluster

Now that your GitOps repo is ready, it's time to build something for it to sync with.

We’re going to create a clean, lightweight Kubernetes cluster—right on your laptop—using [Kind](https://kind.sigs.k8s.io/).

Kind stands for **Kubernetes IN Docker**. It’s fast, minimal, and perfect for local testing like this.

> 💡 This cluster will be completely self-contained:  
> You can destroy it at any time with `kind delete cluster --name gitops-loop-demo`  
> and recreate it instantly with the same config.

### 🛠️ Create the Cluster

Let’s spin up your cluster:

```bash
kind create cluster --name gitops-loop-demo
```

This command launches a single-node Kubernetes cluster inside Docker.

📦 No custom config needed.  
🧪 No remote resources.  
Just a fresh, disposable cluster that Flux will soon take over.

### ✅ Check That It’s Working

Once Kind finishes provisioning, run:

```bash
kubectl get nodes
```

You should see something like:

```
NAME                          STATUS   ROLES           AGE   VERSION
gitops-loop-demo-control-plane   Ready    control-plane   1m    v1.27.x
```

That’s it—you’ve got a Kubernetes cluster running locally.

> 🧠 Note: This cluster has just **one node**, and it's both the control plane and the worker node.  
> For production, you’d normally separate those—but for this demo, this is everything we need.

### 🧭 What’s Next?

Your GitOps repo is ready. Your Kubernetes cluster is online.

Now let’s explore what this cluster will run—and how it will learn what “correct” looks like.

## 📦 Explore the App You’ll Deploy

Now that your cluster is up and running, let’s take a look at the app that Flux will eventually manage.

We’re using a simple, no-surprises container:  
[`nginxdemos/hello:plain-text`](https://hub.docker.com/r/nginxdemos/hello)

This app:
- Runs a small NGINX web server
- Displays a static “Hello World” page
- Starts fast, with no custom logic or config

> We picked it on purpose—so you can focus on learning GitOps, not debugging application code.

### 🗂️ Understand the Folder Structure

Navigate to:

```
examples/day2-gitops-loop-demo/clusters/local/apps/hello/
```

This is where your Kubernetes manifests live.

Here's how it's structured:

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
| `clusters/` | All cluster-scoped config lives here |
| `local/` | This is our environment name (you might see `dev`, `staging`, or `prod` in real setups) |
| `apps/hello/` | Config for this app only (in a real repo, you'd often see `apps/api/`, `apps/frontend/`, etc.) |

> 🧠 GitOps isn’t just about putting YAML in Git—it’s about structuring your environments and apps in a way that’s scalable and understandable for teams.

### 📄 What the Manifests Declare

Inside the `hello/` folder, you’ll find two YAML files. Let’s walk through what they do—at a glance.

#### 🛠️ `deployment.yaml`

This manifest tells Kubernetes:
> "Start one pod running this container—and label it ‘hello’ so other components can find it."

Here’s what it includes:
- `replicas: 1` — We want **one** copy of the app running  
- `containers.image` — We’re using `nginxdemos/hello:plain-text`  
- `ports.containerPort: 80` — The app listens on port 80  
- `metadata.labels.app: hello` — This is important: it tags the pod so the **Service** can find it

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

> 💡 Later, when we scale this app down or delete it, we’ll come back to this `replicas: 1` line—and Flux will too.

#### 🛠️ `service.yaml`

This manifest tells Kubernetes:
> "Create a stable network address for the app, and route traffic to any pod labeled `app: hello`."

Key parts:
- `metadata.name: hello` — This will become the internal DNS name for our app
- `selector.app: hello` — Matches pods labeled `app: hello`
- `ports.port: 80` — The service will expose this on port 80

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

### 🧭 Where We Stand (and What’s Coming)

Right now, these files haven’t been applied yet. They’re just sitting in Git—*waiting to be noticed*. 

Soon, Flux will start watching this folder.  
When it sees these manifests, it’ll apply them and keep them running—forever.

Here’s where you stand:
✅ You've forked the **GitHub repository** to serve as your source of truth  
✅ You've spun up a **Kubernetes cluster** using Kind  
✅ You've examined the **manifests** for our simple application  
🟡 You haven't applied anything to the cluster yet (`kubectl apply`)  
🟡 No GitOps automation is running yet

We’re not there *yet*.

> The loop—the part where Git drives the cluster—is still quiet.

That’s intentional.

This is the calm before the loop begins. And we’re ready for it.

## 🔁 Understand How Flux Applies GitOps

So far, you’ve created a Git repository, set up a cluster, and written two manifests.

Normally, at this point in a Kubernetes workflow, you’d reach for this:

```bash
kubectl apply -f path/to/yaml
```

But this is **not** that kind of tutorial.  
We’re not here to push YAML.  
We’re here to let Git drive the cluster.

> 🧠 In GitOps, the cluster is no longer passive.  
> It reads from Git. It watches for changes. It reacts.

So how does that work? That’s where Flux comes in.

### 🤖 Meet Flux — the GitOps Agent

Flux is a set of Kubernetes-native controllers.  
Once installed, it runs *inside your cluster* and handles three core jobs:

1. **Watch a Git repository** for changes  
2. **Pull and apply** what it finds  
3. **Continuously reconcile** your cluster to match what’s in Git

That’s the loop you’ve been learning about.  
And Flux is the tool that makes it real.

> Flux doesn’t just deploy once.  
> It keeps checking Git—and correcting drift—over and over again.

### 🧩 How Does Flux Know What to Do?

Flux works from two Kubernetes resources that you define:

#### 📦 1. `GitRepository`  
> _"Where should I look for configs?"_

This tells Flux which Git repo to watch, which branch to follow, and how often to check it.

Flux doesn’t act on files yet—it just watches, fetches, and keeps the repo cached inside your cluster.

#### 🛠️ 2. `Kustomization`  
> _"Now that you’ve pulled the repo—what should I apply?"_

This tells Flux which folder to apply, how often to reconcile it, and what to do if something’s missing (like enabling **prune**).

Together, these two resources define the **GitOps loop**:

```
GitRepository → [pull the repo]  
Kustomization → [apply what’s inside it]
```

### 🤔 Why Two Resources? Why Not Just One?

It’s a fair question.

Flux separates these roles to give you more flexibility:

| Design Choice | Why It Matters |
|---------------|----------------|
| Pull and apply are separate | You can cache multiple repos or paths without applying everything |
| You can sync different paths on different schedules | Faster updates for apps, slower ones for infra |
| You can delegate ownership cleanly | Teams can own specific folders or apps |
| You can layer environments | A `staging` folder and a `prod` folder can be treated differently |

> This separation is what makes Flux composable.  
> It lets you scale GitOps across teams, clusters, and environments.

### 🧠 Recap: What Flux Needs From You

You don’t need to write controllers.  
You don’t need to build pipelines.

You just need to define:
- 🧭 **Where to look** (`GitRepository`)
- 🗂️ **What to apply** (`Kustomization`)

And then Flux does the rest.

📌 In the next section, we’ll:
- Install Flux in your cluster
- Create these two resources
- Connect the loop

Once that’s done, you’ll commit your YAML—and Flux will take it from there.

Let’s get to it.

## 🛠️ Install Flux and Connect the Loop

You now understand the core idea:  
Flux will watch your GitHub repo, pull changes, and continuously reconcile your cluster.

It’s time to make that happen.

In this section, you'll:

✅ Install Flux into your Kubernetes cluster  
✅ Connect Flux to your GitHub repository  
✅ Define the GitOps loop that will run automatically

Let’s move step-by-step.

### 🧰 Step 1: Install the Flux CLI

You’ll use the Flux CLI to:
- Install Flux into your cluster
- Create GitOps resources
- Check synchronization status

> 💡 The CLI makes setting up GitOps clean, consistent, and fast.

#### 📦 Install on macOS or Linux:

```bash
curl -s https://fluxcd.io/install.sh | sudo bash
```

Then verify installation:

```bash
flux --version
```

#### 🪟 Install on Windows:

Using [Chocolatey](https://chocolatey.org/):

```powershell
choco install fluxcd
```

Or download directly from the [Flux Releases](https://github.com/fluxcd/flux2/releases).

✅ Flux CLI installed—good. Let's move forward.

### ✅ Step 2: Run a Pre-flight Check

Before we install anything inside your cluster, let's confirm everything is ready:

```bash
flux check --pre
```

Flux will check:
- Is `kubectl` available?
- Is your cluster reachable?
- Is your Flux CLI compatible?

> 🧠 **Behind the scenes:** Flux CLI uses your default kubeconfig (usually `~/.kube/config`) and checks your current context.

✅ All green? You’re ready to go.

### 🚀 Step 3: Install Flux Controllers

Now we’ll install Flux itself:

```bash
flux install
```

This installs a small bundle of Kubernetes-native controllers into a namespace called `flux-system`:

| Controller | Role |
|------------|------|
| `source-controller` | Pulls content from Git |
| `kustomize-controller` | Applies manifests to the cluster |

> 💡 You might also see `notification-controller` and `helm-controller`.  
> We won’t use them today—but they’re part of what makes Flux extensible for real-world environments.

✅ Flux is now alive inside your cluster—but it’s not watching Git yet.

### 🔍 Step 4: Confirm Flux is Running

Check the controllers:

```bash
kubectl get pods -n flux-system
```

You should see something like:

```
NAME                                 READY   STATUS    RESTARTS   AGE
source-controller-xxxxx              1/1     Running   0          10s
kustomize-controller-xxxxx           1/1     Running   0          10s
```

✅ Controllers are running—steady and ready.

### 🔗 Step 5: Connect Flux to Your GitHub Repo

Now let’s plug Git into the loop.

#### 📦 5.1 Create the GitRepository Resource

This tells Flux where your repo lives—and how often to check it:

```bash
flux create source git gitops-loop-demo \
  --url=https://github.com/YOUR-USERNAME/GitOps-Days.git \    # ← replace YOUR-USERNAME
  --branch=main \
  --interval=30s
```

> 💡 **Replace YOUR-USERNAME** with your actual GitHub username.

Once created, verify it:

```bash
flux get sources git
```

You should see something like:

```
NAME                READY   STATUS    AGE
gitops-loop-demo    True    Fetched   30s
```

✅ Flux now sees your repo and is keeping a copy cached inside the cluster.

#### 🛠️ 5.2 Create the Kustomization Resource

Now tell Flux **what** to apply:

```bash
flux create kustomization hello-app \
  --source=GitRepository/gitops-loop-demo \
  --path="./examples/day2-gitops-loop-demo/clusters/local/apps/hello" \
  --prune=true \     # remove deleted resources
  --interval=1m
```

What this does:
- Targets the GitRepository you just created
- Watches the `hello/` folder for updates
- Automatically prunes resources if they’re deleted in Git
- Syncs changes every minute

✅ Flux now knows **where to look** and **what to apply**.

### 🧠 Quick Recap

Let’s pause and see what you’ve done:

✅ Installed Flux controllers inside your cluster  
✅ Connected Flux to your GitHub repo  
✅ Pointed Flux at the folder that defines your application  

Your cluster is now **watching Git**—but it hasn’t applied anything yet.

That’s about to change.

Next, let’s watch the GitOps loop come to life.

## 🎬 Watch the GitOps Loop Come to Life

You've done the hard work:  
- You set up your GitOps repo.  
- You spun up your Kubernetes cluster.  
- You installed Flux and connected the loop.

Now it’s time to step back—and watch your system take over.

No `kubectl apply`.  
No manual deployments.

Just Git.  
Just Flux.  
Just continuous reconciliation, alive and working.

Let's watch it happen.

### 🧪 Step 1: Check the Kustomizations

First, let’s see if Flux has already detected your setup:

```bash
flux get kustomizations
```

You should see something like:

```
NAME         READY   STATUS    AGE
hello-app    True    Applied   1m
```

✅ Flux has pulled your Git repository, applied the app configuration, and declared your app *running*—all without you applying a single YAML manually.

### 📦 Step 2: Check if the App Is Running

Now, let's ask Kubernetes directly:

```bash
kubectl get pods
```

You should see:

```
NAME    READY   STATUS    RESTARTS   AGE
hello   1/1     Running   0          1m
```

✅ Your app pod is running, just like it’s declared in Git.

Next, let's check the Service:

```bash
kubectl get svc hello
```

You should see:

```
NAME    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
hello   ClusterIP   10.x.x.x          <none>        80/TCP    1m
```

✅ The Service is ready too—exposing your app inside the cluster.

### 🚪 Step 3: Access the App in Your Browser

To view your app, you’ll port-forward from your local machine to the Service inside your cluster:

```bash
kubectl port-forward svc/hello 8080:80
```

Then, open [http://localhost:8080](http://localhost:8080) in your browser.

You should see:

> **"Welcome to nginx!"**

🎉 And there it is—Hello from NGINX, no `kubectl apply` in sight.

### 🧠 A Moment of Reflection

Let’s pause for a second.

✅ You didn’t manually deploy anything.  
✅ You didn’t push anything into the cluster with a script.  
✅ You didn’t even apply the YAML you wrote.

Instead:
- You connected your cluster to Git
- Flux detected what was already declared there—and brought it to life.

This isn’t just automation.  
This is **continuous, declarative operations**.

And this is only the beginning.

### 🔮 What’s Next?

Now that you’ve seen your GitOps loop come to life, it’s time to test what makes it truly powerful:

What happens when something drifts?  
What happens when a manual change conflicts with what’s declared in Git?

Spoiler:  
> The system will fix itself.

In the next section, we’ll break the cluster on purpose—and watch GitOps bring it back.  
That’s where you’ll see **reconciliation** in action.

Let’s go.

## 🔥 Break the Cluster, Watch It Heal

Your app is live.  
Your GitOps loop is running.  
But how strong is it?

Let’s find out.

We’re going to break your cluster—on purpose—and see how Flux quietly brings it back to the desired state, without you lifting a finger.

### 🧪 Drift Test #1: Scale the App to Zero

In your `deployment.yaml`, you declared `replicas: 1`.

Let’s violate that.

Scale the deployment down to zero manually:

```bash
kubectl scale deployment/hello --replicas=0
```

Then watch:

```bash
watch kubectl get deployment hello
```
_(If you don't have `watch`, just run the `kubectl get` command repeatedly.)_

You’ll see:

- The number of pods drops to 0.
- For a brief moment, your app is gone.

But within about a minute...  
Flux notices the drift.  
And quietly restores your app to `replicas: 1`.

✅ Your cluster is no longer drifting.  
✅ Git's declared state is re-enforced automatically.

### 💥 Drift Test #2: Delete the App Completely

Let’s go further.

Delete the Deployment *and* the Service entirely:

```bash
kubectl delete deployment/hello service/hello
```

Check the status:

```bash
watch kubectl get deployment,service
```

For a few moments, there will be nothing—no pod, no service.

But again, within about a minute...  
Flux re-applies everything from Git.  
Your Deployment comes back.  
Your Service comes back.

Your app comes back.

✅ Git remains the source of truth—even when things break.

### 🧠 A Quiet but Powerful Shift

What you just witnessed wasn’t a "redeploy" or a manual rollback.  
It was **continuous reconciliation** in action.

> **Git declared it.  
> Flux enforced it.  
> Drift was corrected without human intervention.**

And the best part?  
You didn’t even have to notice the drift for it to be corrected.

Your system defended itself.

### 🔮 What’s Next

Now that you've seen GitOps healing drift automatically, you understand the true power of declarative systems.

Day 2 is almost complete.  
Next, we'll reflect on what you built—and show you where we're going next.

Spoiler: it's time to scale this model to multiple environments.

Let’s wrap up Day 2.

## 🎯 Reflect and Wrap Up Day 2

Take a moment to reflect.

What you built today wasn’t just a demo app.  
It wasn’t just some YAML files.

You built a **living system**:
- A Kubernetes cluster that **pulls its own state** from Git
- A Git repository that **declares** how your app should run
- A GitOps agent (Flux) that **watches**, **applies**, and **heals** automatically

You didn’t deploy manually.  
You didn’t fix drift manually.  
You committed, Flux watched, and your system corrected itself.

> 🧠 You shifted from pushing to pulling.  
> You moved from manual ops to declarative ops.  
> You let Git become your operational control plane.

And you experienced, firsthand, what GitOps really feels like when it’s running.

## 📈 Where We’re Heading Next

Today, everything happened inside a local cluster.

Tomorrow, we’re going to **take it to the cloud**.

You’ll spin up a real Kubernetes cluster in Azure (AKS).  
You’ll set up Flux in a production-grade way.  
You’ll organize your GitOps repo for **multiple environments** (dev, staging, prod).  
You’ll even start working with secrets safely—without ever committing them to Git.

All the core concepts you learned today will scale with you.

And by the end of Day 3, you’ll have a cloud-native GitOps system you could show your team—or your boss—with confidence.

**Day 2: complete.  
Day 3: let’s go bigger.**