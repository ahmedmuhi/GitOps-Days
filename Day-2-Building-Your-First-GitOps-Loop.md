## 🚀 Day 2 – Your First GitOps Loop: Running Flux on Your Laptop

**Welcome back to GitOps-Days.**

[Yesterday](Day-1-What-really-is-GitOps.md), we looked at why Kubernetes clusters can drift from what’s in Git—and how GitOps helps prevent that. You saw how using Git as the source of truth, and having the cluster pull from it continuously, creates a safer and more predictable system.

Today, you’ll bring that idea to life. You’ll be using **Flux**, an open-source GitOps engine built specifically for Kubernetes. Flux runs inside your cluster and takes care of watching your Git repository, pulling the latest configuration, and applying it continuously. If something drifts—a pod is deleted, a config is changed manually—Flux brings it back to match Git.

In under an hour, you’ll:

* Set up a local Kubernetes cluster
* Install Flux and link it to your GitHub repo
* See changes applied automatically from Git
* Trigger recovery after intentional drift
* Deploy by committing to Git instead of running `kubectl`

You’ll even make a manual change—to see Flux detect and fix the drift on its own.

By the end of today, you'll experience firsthand how committing changes to Git can streamline your workflow, turning manual interventions into automated consistency.

Let’s start the loop.

## 🧰 Get Ready to Run GitOps

Before we launch our cluster or start syncing from Git, let’s make sure your workstation has the usual Kubernetes tooling.

| Tool         | Minimum version | Purpose                                           |
| ------------ | --------------- | ------------------------------------------------- |
| **Docker**   | 24.x            | Provides the container runtime that **kind** uses |
| **kind**     | ≥ 0.23          | Spins up a local Kubernetes cluster inside Docker |
| **kubectl**  | ≥ 1.27          | Lets you interact with the cluster                |
| **Git**      | any recent      | Connects your machine to your GitHub fork         |
| **Flux CLI** | ≥ 2.3.0         | Installs and manages Flux (you’ll add it shortly) |

> 💡 If you're using Docker Desktop (macOS/Windows), version 4.29 or later includes Docker Engine 24+, which is what `kind` and other tools rely on.

> 💡 Already have these installed? Skip ahead. Otherwise, use the links below.

**Quick install links**

* [Docker](https://docs.docker.com/get-docker/)
* [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
* [kubectl](https://kubernetes.io/docs/tasks/tools/)
* [Git](https://git-scm.com/downloads)
* **Flux CLI**:

  * macOS / Linux → `curl -s https://fluxcd.io/install.sh | sudo bash`
  * Windows → `choco install fluxcd`

(Optional) verify your versions:

```bash
docker --version
kind --version
kubectl version --client --short
flux --version   # only after installing Flux CLI
```

> You don’t need to install Flux *inside* the cluster yet—we’ll do that in a later step.

## 🗃️ Set Up Your GitOps Repository

Everything in GitOps begins with **Git**.

Your repository is more than a place to park YAML—it declares the **desired state** of your system, and the GitOps controller you’ll install later will keep the cluster aligned to that state.

For this lab you’ll fork a **pre‑built** repository *(so you can focus on GitOps, not authoring YAML from scratch).* It already contains:

* a minimal demo application
* the Kubernetes manifests that deploy it
* a folder layout that mirrors real‑world GitOps repos

Forking lets you commit changes and watch the automation respond.

### Stage A – Fork in the browser

1. Open [https://github.com/ahmedmuhi/GitOps-Days](https://github.com/ahmedmuhi/GitOps-Days).
2. In the top‑right corner, click **Fork** (the button may read **Create fork**).
3. Accept the defaults—especially the branch name **main**—and click **Create fork**. You now have your own copy under *YOUR‑USERNAME/GitOps-Days*.

### Stage B – Clone your fork locally

Open your terminal—**PowerShell** on Windows, Terminal on macOS/Linux—and run:

```bash
git clone https://github.com/YOUR-USERNAME/GitOps-Days.git   # clones the main branch
cd GitOps-Days
```

> ⚠️ **Important:** Clone *after* you fork so you’re working against **your** repository, not the original.

Your fork now holds the desired state that the cluster will follow.

### What’s inside

* **Manifests** – Deployment and Service YAML for the demo app
* **Directory layout** – Matches patterns you’ll use when the project grows

We’ll explore the structure shortly; next you’ll create the Kubernetes control plane that keeps everything in sync.

## 🧱 Spin Up a Local Kubernetes Cluster

Before we can see GitOps in action, we need a Kubernetes cluster where changes can happen. Instead of using a cloud provider, you’ll create a small local lab cluster **on your own machine**—perfect for learning, safe to experiment with, and easy to reset. *(kind uses about 3 GB of RAM while it’s running.)*

We’ll use **[kind](https://kind.sigs.k8s.io/)** (*Kubernetes in Docker*) to do this. It runs a full Kubernetes cluster inside Docker containers and takes just a few seconds to start.

## Step 1 · Create the Cluster

Open your terminal—**PowerShell** on Windows, Terminal on macOS/Linux—and run the following command to create a cluster named `gitops-loop-demo`:

```bash
kind create cluster \
  --name gitops-loop-demo \
  --image kindest/node:v1.32.2   # kind’s current default image (Kubernetes 1.32)
```

> 💡 We pin the image so everyone runs the same Kubernetes version—copy it as-is for now.

## Step 2 · Verify the Cluster is Ready

Once the cluster is created, check that it’s running:

```bash
kubectl get nodes
```

You should see output similar to:

```
NAME                            STATUS   ROLES           AGE   VERSION
gitops-loop-demo-control-plane  Ready    control-plane   1m    v1.32.x
```

> ℹ️ **Why only one node?** kind creates a single control-plane node that can also run your pods. That keeps resource usage low and is perfect for local experiments. In production, you’d add separate worker nodes, but for this lab, one node is plenty.

> ⚠️ **Cleanup tip:** When you’re finished—or if you want to start fresh—delete the cluster with:
>
> ```bash
> kind delete cluster --name gitops-loop-demo
> ```

With your local cluster up and running, you're ready to explore the application that GitOps will deploy—and the folder structure that defines what “should” be running.

## 📦 Explore the App You’ll Deploy

Now that your cluster is running, let’s take a quick look at the app that Flux will manage.

We use a simple, no‑surprises container: [`nginxdemos/hello:plain-text`](https://hub.docker.com/r/nginxdemos/hello).

This image:

* Runs a tiny NGINX web server
* Serves a static **Hello World** page
* Starts quickly, with no custom configuration

> We chose this minimal app so you can concentrate on GitOps rather than writing or troubleshooting application code.

### 🗂️ Repository Folder Structure

Inside the repository you forked earlier, you’ll find the manifests that provision our **hello** container here:

```text
examples/day2-gitops-loop-demo/clusters/local/apps/hello/
```

Directory layout:

```
clusters/
└── local/
    └── apps/
        └── hello/
            ├── deployment.yaml
            └── service.yaml
```

| Folder        | Purpose                                                                              |
| ------------- | ------------------------------------------------------------------------------------ |
| `clusters/`   | Top‑level folder for cluster configuration, organised by environment                 |
| `local/`      | The **environment name** (in a real repo you might have `dev`, `staging`, or `prod`) |
| `apps/hello/` | Manifests for this app only (other apps would get their own sub‑folders)             |

> 🧠 Notice how the repo separates **environments** (`local`) from **applications** (`hello`). This clear hierarchy keeps larger GitOps repositories maintainable as they grow.

### 📄 What the Manifests Declare

Inside `apps/hello/` you’ll find two YAML files. Here’s what they do.

#### 🛠️ `deployment.yaml`

A **Deployment** is a Kubernetes controller that keeps a specified number of identical **pods** running.  A pod is the smallest schedulable unit in Kubernetes – think of it as one instance of our container.  The Deployment watches these pods and recreates them if they crash, ensuring the cluster always matches the declared state.

Key fields:

* `replicas: 1` — run a single copy of the app
* `image: nginxdemos/hello:plain-text` — the container image
* `containerPort: 80` — the app listens on port 80
* `labels.app: hello` — tags the pod for the Service selector

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

> 💡 Once Flux is connected to the cluster, it will watch this **Deployment** (and the accompanying **Service**) for drift. Later in the lab we’ll deliberately set `replicas` to 0; Flux will detect the mismatch and restore it to 1 automatically.

#### 🛠️ `service.yaml`

A **Service** gives the pods behind our app a stable network name and virtual IP.  Instead of talking to pods directly (they come and go), other components talk to the Service.  The Service finds matching pods by their **labels** – in this case `app: hello` – and load‑balances traffic across them.

Important bits:

* `name: hello` — the Service name
* `selector.app: hello` — matches pods with that label
* `port: 80` — exposes port 80 inside the cluster

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

### 🧭 Where We Stand

> **✅ Done so far**
> • Forked the GitHub repository (source of truth)
> • Created a kind Kubernetes cluster
> • Reviewed the manifests for the demo app
>
> **🟡 Still to come**
> • Nothing applied to the cluster yet (`kubectl apply` not used)
> • GitOps automation not running yet

The GitOps loop is still quiet for the moment—exactly where we want it before wiring Flux into the cluster.

## 🔁 How Flux Runs the GitOps Loop

So far, **we’ve** forked a repo, spun up a cluster, and reviewed the application manifests. In a typical workflow this is where you’d run:

```bash
kubectl apply -f path/to/your/yaml
```

Today we’ll do something different.  Instead of pushing YAML into the cluster, we’ll hand control to **Flux** and let it keep the cluster in sync with Git—think of it as switching the cluster to *autopilot*.

### 🤖 Meet Flux – the GitOps Agent

Flux is an open‑source CNCF project—sitting alongside tools like Argo CD—that implements GitOps for Kubernetes. It installs as a bundle of controllers (`source-controller`, `kustomize-controller`, plus a few others) running as pods in the **flux-system** namespace. Once installed they:

1. **Watch** a Git repository for changes
2. **Pull & apply** the desired state
3. **Continuously reconcile** the cluster so its live state always matches Git

> This is the same **watch → pull → reconcile** loop we explored earlier—Flux turns that model into running software.

Flux checks Git every minute by default, but you can add a webhook so changes apply almost immediately.

### 🧩 How Does Flux Know What to Do?

To bring that loop to life, Flux relies on two small **custom resources** you’ll create in the next step.  Think of them as its **instructions**:

#### 1. `GitRepository` – *“Where should I look?”*

This object gives Flux the **URL**, **branch**, and **interval** (or webhook) for a Git repo.  The source‑controller fetches that repo on a schedule and keeps a cached copy inside the cluster.  At this point Flux is only *observing*—no YAML is applied yet.

#### 2. `Kustomization` – *“Okay, now what should I apply?”*

A Kustomization points to a **folder** inside that repo, decides **how often** to reconcile it, and sets options such as **`prune`**:

```yaml
prune: true
```

Setting `prune: true` means resources removed from Git are also removed from the cluster—ideal for production. Turning it off (common in dev) limits Flux to create or update actions only.

These two resources work together:

```
GitRepository  ──► pull repo & keep cache
Kustomization ──► apply folder & reconcile drift
```

#### Why two resources instead of one?

* **Clear responsibilities** – one controller focuses on *fetching*, another on *applying*.
* **Fine‑grained scopes** – you can track many repos or paths without auto‑applying all of them.
* **Different cadences** – staging apps might reconcile every minute, while cluster‑wide policy syncs hourly.
* **Delegated ownership** – each team can own its own `Kustomization` without touching the global repo settings.

This separation keeps Flux flexible and maintainable as your repositories and teams grow.

## 🛠️ Install Flux and Connect the Loop

You now understand the core idea: Flux will watch your GitHub repo, pull changes, and continuously keep your cluster in sync.

In this section you will:

✅ Install the **Flux CLI tool**
✅ Install Flux controllers inside your cluster
✅ Wire the controllers to your GitHub repo and start the loop

### 🧰 Step 1 – Install the Flux CLI tool

The easiest way to install Flux is with the Flux CLI—a small tool that runs on your workstation and makes it simple to install Flux in your cluster and connect it to your GitHub repo.

Below is some guidance to help you install the Flux CLI on your machine. Choose the install method that matches your OS—Homebrew on macOS, the script or a package on Linux, and Chocolatey on Windows.

#### macOS

**Homebrew (recommended)** – quick to update and easy to remove

```bash
brew install fluxcd/tap/flux
```

#### Linux

**Official install script** – works on any distribution

```bash
curl -s https://fluxcd.io/install.sh | sudo bash
```

**Ubuntu/Debian package example**

```bash
sudo apt-get update && \
  sudo apt-get install -y fluxcd
```

*(See the **********************[Flux docs](https://fluxcd.io/docs/)********************** for instructions on other Linux distributions.)*

#### Windows

**Chocolatey (recommended)**

```powershell
choco install fluxcd
```

If you prefer, download a standalone binary from the [Flux releases page](https://github.com/fluxcd/flux2/releases) and add it to your PATH.

Then verify the installation:

```bash
flux --version   # any version ≥ 2.5 is fine
```

### ✅ Step 2 – Pre‑flight check

Before Flux touches your Kubernetes cluster, double‑check you’re pointed at the right context. If you run multiple clusters, confirm `kubectl` is set to the **kind‑gitops‑loop‑demo** lab cluster:

```bash
kubectl config current-context
```

If you see **kind-gitops-loop-demo**, you’re on the right cluster.

Now run:

```bash
flux check --pre
```

`flux check --pre` confirms that the cluster is reachable and that your CLI version is compatible with the Kubernetes version running in kind.

### 🚀 Step 3 – Install Flux controllers

Now that your workstation is ready—and `kubectl` is pointing at **kind‑gitops‑loop‑demo**—use the **Flux CLI** to install the controllers:

```bash
flux install
```

The command above creates the namespace **flux-system** and starts the controllers we met earlier:

| Controller                                   | Purpose                                                     |
| -------------------------------------------- | ----------------------------------------------------------- |
| `source-controller`                          | Fetches and caches content from Git                         |
| `kustomize-controller`                       | Applies manifests & reconciles drift                        |
| `notification-controller`, `helm-controller` | Extra features (alerts, Helm); installed but not used today |

✅ The Flux controllers are now running in your cluster, but they aren’t watching Git yet.

> ℹ️ **Heads‑up:** deleting the **flux‑system** namespace removes Flux entirely; you would need to reinstall Flux if that happens.

### 🔗 Step 4 – Wire Flux to your GitHub repo

With the controllers in place, the next step is to point them at your Git repo and tell them which folder to apply.

#### 4.1 Create a **GitRepository**

First we’ll tell **source‑controller** which repo to watch. The command below creates a `GitRepository` object named `gitops-loop-demo`, points it at your fork on the `main` branch, and sets a 30‑second fetch interval so the cache stays fresh.

```bash
flux create source git gitops-loop-demo \
  --url=https://github.com/YOUR-USERNAME/GitOps-Days.git \
  --branch=main \
  --interval=30s          # 30 s fetch keeps cache fresh
```

You might notice we haven’t passed any credentials. That’s because the repo is public. If your repo is private, you’ll need to add `--username` and `--password` or use SSH flags. See the [Flux authentication docs](https://fluxcd.io/flux/components/source/gitrepositories/#authentication) for details.

Now let’s verify that Flux is connected to the correct Git repo. Run:

```bash
flux get sources git
```

You should see output like:

```
NAME                READY   STATUS    AGE
gitops-loop-demo    True    Fetched   30s
```

#### 4.2 Create a **Kustomization**

Now that Flux is watching your Git repo and has fetched the contents, we need to tell it which folder to apply to the cluster. This step connects the `Kustomization` controller to the `GitRepository` we created earlier (`gitops-loop-demo`). That controller keeps a cached copy of the repo, and the `Kustomization` now defines *what* to apply. In our case, we’re pointing it to the manifests under `clusters/local/apps/hello`, which map to the `local` environment and the `hello` app. We’ve also set the interval to 1 minute so that Flux will check and reconcile the cluster frequently.

##### macOS / Linux

```bash
flux create kustomization hello-app \
  --source=GitRepository/gitops-loop-demo \
  --path="./examples/day2-gitops-loop-demo/clusters/local/apps/hello" \
  --prune=true \
  --interval=1m          # 1 min reconcile shows changes quickly
```

##### Windows (PowerShell)

The Windows version of this command uses PowerShell syntax, which differs from macOS/Linux because PowerShell doesn’t support backslash line continuations. Instead, it uses backticks (\`) to split long commands across lines.

```powershell
flux create kustomization hello-app `
  --source=GitRepository/gitops-loop-demo `
  --path "./examples/day2-gitops-loop-demo/clusters/local/apps/hello" `
  --prune true `
  --interval 1m
```

> ⚠️ \*\*Be careful with \*\***`prune: true`** — if you remove a resource from your manifests and commit the change, Flux will delete the live resource from your cluster to match Git. This is intentional and ideal for production, where Git should reflect the true desired state. In development, it’s common to turn prune **off** to avoid accidental deletions while iterating.

### ✅ Quick recap

At this stage:

* The Flux controllers are up and running in the **flux-system** namespace
* Flux is watching your GitHub repository and caching updates every 30 seconds
* The `hello-app` Kustomization is monitoring the correct folder and reconciling every minute

Everything is now in place. We’re ready to bring the GitOps loop to life—which we’ll do next.

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