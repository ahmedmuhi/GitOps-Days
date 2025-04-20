## 🚀 Day 2: GitOps in 15 Minutes — Flux CD on Your Laptop  

Welcome back!

[Yesterday](https://github.com/ahmedmuhi/GitOps-Days/blob/main/Day-1-What-really-is-GitOps.md), we unpacked the *what* and *why* of GitOps.  
You saw how GitOps isn’t just “YAML in Git,” but a powerful operational model based on four clear principles: declarative state, version control, pull-based delivery, and continuous reconciliation.

Today, it’s time to make that theory real.

We’re going to break Kubernetes—twice. And we’ll watch GitOps put it back together. Automatically.

No GitHub. No cloud cluster. Just your laptop, a local repo, and a tiny Kubernetes playground. In ~15 minutes, you’ll have Flux CD (our GitOps agent for today) running inside a KIND cluster, syncing to a local Git repo, and healing your changes without you lifting a finger.

You’ll see:
- The GitOps loop come alive—on your screen.
- Drift repaired instantly.
- Operational guardrails you can toggle on and off.

This isn’t just about installing tools—it’s about experiencing how GitOps *feels* when it’s running. Today is your crash-test: manual failure, automatic repair, and a moment of “ohhh… now I get it.”

Let’s build.

---

## 🏗️ Setting Up Your Local Playground

You’ve read the theory. Now it’s time to set the stage for the crash test.

Let’s build a self-contained environment where we can watch GitOps do its thing.  
That means two ingredients:
- A Git repository that defines what the cluster *should* look like  
- A Kubernetes cluster that tries to stay in sync with it

This whole lab runs on your laptop—no cloud accounts, no GitHub, no pipelines.

Let’s get our playground ready.

---

### 🧱 Step 1: Create a Local Git Repository

In a typical GitOps setup, your cluster pulls its configuration from a remote Git repository—like GitHub or GitLab. Tools like Flux or Argo CD connect over HTTPS or SSH, fetch the latest changes, and reconcile them into the cluster.

In this lab, we’re staying 100% local.

So instead, we’ll create a Git repository *on your machine*—and later, we’ll point Flux to it.

Open your terminal and run:

```bash
mkdir ~/gitops-demo
cd ~/gitops-demo
git init -b main
```

> 📝 This creates a folder to hold your Kubernetes manifests, and turns it into a Git repository.  
> The `-b main` flag sets the default branch to `main`, which is what most GitOps tools (including Flux) expect by default.

> 🪟 **Windows users:**  
> If you’re on Windows, replace `~/gitops-demo` with something like:  
> `C:\Users\YourName\gitops-demo`  
> You can create the folder manually or via PowerShell:
> ```powershell
> mkdir "$env:USERPROFILE\gitops-demo"
> cd "$env:USERPROFILE\gitops-demo"
> git init -b main
> ```

No need to push this repo anywhere. It stays local and offline.  
Think of it as your **private GitHub—but just for this cluster**.

---

### 🧱 Step 2: Create a Kubernetes Cluster (and Make It See Your Git Repo)

Next, we’ll spin up a small Kubernetes cluster using [kind](https://kind.sigs.k8s.io/).

Kind stands for **Kubernetes in Docker**. It runs Kubernetes inside a Docker container—ideal for quick, disposable clusters like this one.

But here’s the important part:

Flux (our GitOps agent) runs **inside** the cluster.  
And it normally connects to a **remote Git server** to fetch changes.

But we don’t have one. So we’ll trick it—by mounting our local Git repo into the cluster, and then telling Flux to pull from it using a `file://` path.

To make this work, we’ll:
- Share your Git folder with the cluster using a **mount**
- Point Flux to the mounted path `/repo` inside the cluster

This is how we simulate GitHub, **without needing GitHub**.

---

#### 🛠️ 2.1: Create the Cluster Configuration

Kind lets us customise our cluster using a small YAML file. We’ll use this to:
- Create a single control plane node
- Mount your Git repo into the cluster

Save this file as `kind-cluster-config.yaml`:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraMounts:
    - hostPath: ${HOME}/gitops-demo
      containerPath: /repo
```

> 📁 **What this does:**  
> - `hostPath` is the folder on your machine  
> - `containerPath` is where it will appear *inside* the Kubernetes node  
> - Later, Flux will read from `/repo` just like it would read from GitHub

> 🪟 **Windows users:**  
> Update the path to match your system. For example:
> ```yaml
> hostPath: C:\\Users\\YourName\\gitops-demo
> containerPath: /repo
> ```

---

#### 🚀 2.2: Create the Cluster Using the Config

Now let’s launch the cluster:

```bash
kind create cluster --name gitops-local --config kind-cluster-config.yaml
```

This will take a moment. When it’s ready, confirm everything is working:

```bash
kubectl get nodes
```

You should see something like:

```
NAME                          STATUS   ROLES           AGE   VERSION
gitops-local-control-plane    Ready    control-plane   1m    v1.27.x
```

---

You now have:
- ✅ A local Git repository
- ✅ A Kubernetes cluster running in Docker
- ✅ The repo mounted into the cluster’s file system at `/repo`

In the next step, we’ll install Flux inside the cluster, and point it to this local repo.

Let’s continue.

---

### 🧱 Step 3: Prepare the App and Configuration

So far, you’ve created a Git repository.  
You’ve built a Kubernetes cluster.  
You’ve even mounted that Git repository into the cluster so that tools inside Kubernetes can read it.

But at this point, your cluster is empty—and that’s exactly how it should be.

This step is about preparing what we want the cluster to *eventually* apply:  
A simple application, defined declaratively in YAML, sitting quietly in Git—just waiting to be acted upon.

There’s no GitOps yet.  
We’re still building.

---

#### 📁 3.1: Add a Simple App to Your Git Repo

Every app needs two things:
- A container to run
- And a Kubernetes configuration to declare how it should run

In this case, we’re using a pre-built demo container:  
[`nginxdemos/hello`](https://hub.docker.com/r/nginxdemos/hello)

This container serves a “Hello from NGINX” page, perfect for visual confirmation.  
> 💡 We’re not writing application code here.  
> We’re working with the **infrastructure configuration** that runs it.

To tell Kubernetes how to run this container, we’ll declare two manifests:
1. A **Deployment** – which runs the container in a Pod
2. A **Service** – which exposes it inside the cluster

---

##### 🗂️ Create the App Folder Structure

Inside your Git repository, create the following directory:

```bash
mkdir -p clusters/local/apps/hello
```

Let’s unpack what this folder structure means:

| Folder | Purpose |
|--------|---------|
| `clusters/` | This is where you declare cluster-specific configuration. |
| `local/` | This is the name of your cluster. In our case, we’re running it locally with kind. |
| `apps/hello/` | This is where the configuration for your app lives: its Deployment and Service. |

> 📁 This layout mirrors real GitOps repositories—where teams often separate apps by cluster and environment.  
> We’re not improvising here—we’re modeling production-ready structure.

---

##### 📄 Create the Deployment Manifest

Create a new file inside that folder:  
📄 `clusters/local/apps/hello/deployment.yaml`

Paste the following:

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

> 🧠 This Deployment tells Kubernetes:  
> “Run one copy of the `nginxdemos/hello` container and expose port 80.”

---

##### 📄 Create the Service Manifest

Now create the service definition:  
📄 `clusters/local/apps/hello/service.yaml`

Paste this:

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

> 🧠 This Service gives your app a stable network address **inside the cluster.**  
> It allows other components (or you, via port forwarding) to access it later.

---

#### 🧾 3.2: What’s in Your Git Repo (So Far)

Let’s take a moment to look at what you’ve built inside your Git repository.

Your directory now looks something like this:

```
~/gitops-demo/
├── kind-cluster-config.yaml         # Used to create the cluster (host-only)
└── clusters/
    └── local/
        └── apps/
            └── hello/
                ├── deployment.yaml  # Declares the NGINX demo app
                └── service.yaml     # Exposes the app within the cluster
```

Let’s clarify the purpose of each:

- `kind-cluster-config.yaml`  
  → This was used by `kind` to create the cluster.  
  It doesn’t define what runs in the cluster—it defined **how to launch it**.

- `deployment.yaml`  
  → Tells Kubernetes: *what app to run and how to run it*

- `service.yaml`  
  → Tells Kubernetes: *how to expose that app inside the cluster*

All of these files have been saved inside your Git repository.  
> 🧠 But to be clear: **they haven’t been committed**, and **they haven’t been applied** to the cluster.

They're written. They're staged.  
But for now, they’re just sitting there—waiting.
---

#### 🧭 3.3: Summarise the Manual Build So Far

Let’s pause and take inventory:

✅ You’ve created a **Git repository**  
✅ You’ve spun up a **Kubernetes cluster**  
✅ You’ve **mounted the repo** into that cluster  
✅ You’ve written a **Deployment** and **Service** for a simple app  
🟡 You haven’t applied anything to the cluster yet (`kubectl apply`)
🟡 No GitOps automation is running yet

Everything is still manual.  
And that’s intentional.

You’ve built a clean, declarative foundation—ready for Flux to take over from.

---

#### 🔮 3.4: A Glimpse of What’s Coming

Right now, if you wanted to deploy this app manually, you’d run:

```bash
kubectl apply -f clusters/local/apps/hello
```

But we’re not doing that.

We’re not applying YAML from our terminal.  
We’re going to **commit** YAML into Git—and let the cluster sync itself.

In the next section, we’ll install Flux inside your cluster.  
You’ll connect Flux to your Git repo.  
And you’ll watch Kubernetes apply your app *on its own*—and keep it aligned with Git.

> This is the shift.  
> From `kubectl` to Git.  
> From “push to cluster” to “cluster pulls from Git.”

Welcome to GitOps.

---

## 🔁 From *Manual* to *GitOps*: What Flux Actually Does  

You now have everything a Kubernetes engineer needs:

* **A running cluster**  
* **An `nginxdemos/hello` container** ready to use  
* **Two tiny YAML files** (`Deployment` and `Service`) already saved in your Git repo  

In the “old‑school” workflow you would now type:

```bash
kubectl apply -f clusters/local/apps/hello      # applies every YAML in that folder
```

…but we’re **not** going to do that.  
Instead the **cluster will deploy itself** once Flux is watching your Git repo.  
That inversion of control is the essence of **GitOps**.

---

### In Plain English  
> *GitOps is Commit‑driven operations*: you change **Git**, the cluster changes **itself**.  

No more one‑off `kubectl apply`; every change is versioned, reviewed and continuously reconciled.

---

### Meet Flux — the Agent that Watches Git  

To make that happen we need something **inside the cluster** that can  

1. **Watch** a Git repo  
2. **Apply** whatever it finds  
3. **Keep everything in sync — forever**  

That’s what **Flux** does.

Flux is a small bundle of Kubernetes controllers.  
It doesn’t just install your app once; it **keeps the desired state alive**.  
If something drifts, it fixes it. If something changes in Git, it reacts.

### 🧩 How Flux Knows What to Do

Flux doesn’t just magically apply your app—it works from **two Kubernetes resources** that you define.

Let’s break them down slowly.

---

#### 📦 First: Tell Flux *Where to Look*
That’s the job of a `GitRepository`.

It’s a Kubernetes object you’ll create that says:

> “Hey Flux, clone this Git repo—every 30 seconds—and keep it cached inside the cluster.”

In a real-world setup, that URL might point to GitHub:

```yaml
url: https://github.com/your-team/cluster-configs.git
```

But in our lab, since we mounted our local folder into the cluster, we’ll use this:

```yaml
url: file:///repo
```

That’s pointing Flux to the shared folder from earlier.

Here’s what the full resource looks like:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: local-repo
spec:
  url: file:///repo
  branch: main
  interval: 30s
```

💡 **This file just pulls.**  
It clones the repo and updates it regularly.
It gives the repo a name: local-repo
It does *not* apply anything yet.

---

#### 🧭 Next: Tell Flux *What to Apply*

That’s the job of a `Kustomization`.

It’s a second Kubernetes object you’ll define, and it says:

> “Inside that repo you just cloned, go into this folder, and apply what you find there—on a loop.”

In our case, that folder is `apps/hello`.

Here’s what that looks like:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: hello-sync
spec:
  sourceRef:
    kind: GitRepository
    name: local-repo
  path: ./apps/hello
  prune: true
  interval: 1m
```

Let’s walk through what this means:

- `sourceRef.name: local-repo` — matches the GitRepository you defined earlier  
- `path: ./apps/hello` — tells Flux where your `Deployment` and `Service` YAMLs live  
- `interval: 1m` — re-applies it every minute  
- `prune: true` — removes anything from the cluster that no longer exists in Git

💡 **This file triggers the loop.**  
Once you apply it, Git becomes live.

---

### 🤔 Why Do We Need Two Files?

You might be wondering:
> “Why not just have one big Flux object? Why split it into two?”

That’s a great question.

The answer is **separation of concerns**.

Here’s why Flux separates “pulling” from “applying”:

- You can **pull from one repo** and apply multiple folders (dev / staging / prod)
- You can **watch several repos**, each with its own purpose (infra / apps / secrets)
- You can **sync different parts** on different intervals
- You can **delegate ownership** of folders to different teams

This separation gives you flexibility, composability, and clean Git hygiene.

---

### 📁 So Where Do These Files Go?

Let’s update our directory structure.

You're about to create two new YAML files:

- `local-repo.yaml` — defines the GitRepository
- `hello-sync.yaml` — defines the Kustomization

You'll save them inside your existing Git repo, under a new folder:

```
~/gitops-demo/
└── clusters/
    └── local/
        ├── apps/
        │   └── hello/
        │       ├── deployment.yaml
        │       └── service.yaml
        └── flux/
            ├── local-repo.yaml
            └── hello-sync.yaml
```

So everything—your app config *and* your GitOps wiring—is versioned in one place.

---

### 🔮 What Happens Next

Here’s what you’ll do in the next section:

1. Install the Flux controllers into your cluster
2. Generate the two Flux YAMLs using the Flux CLI
3. Save and commit them into Git  
4. Apply the two Flux resources **once**  
5. Watch the cluster notice the commit, pull the repo, and deploy your app automatically

This will be your first true GitOps moment.  
No `kubectl apply` for your app.  
Just a commit—and the cluster takes care of the rest.

---

## 🚀 Install Flux & Wire It Up

You’ve done the prep.

Your manifests are ready. Your cluster is ready.  
Now it’s time to bring Flux online—and let Git take the wheel.

In this section, you’ll:

- Install the Flux CLI  
- Deploy Flux into your cluster  
- Create two YAML files that define your GitOps loop  
- Commit them into Git  
- Apply them once—and watch Flux do the rest

Let’s go.

---

### 🛠️ 4.1: Install the Flux CLI

The Flux CLI is the tool you’ll use to:

- Install Flux into your cluster  
- Generate GitOps resource YAMLs  
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

---

### ✅ 4.2: Check That Everything Is Ready

Before installing anything, run:

```bash
flux check --pre
```

This checks:

- Is `kubectl` available?
- Is your cluster reachable?
- Is your Flux CLI version compatible?

If all green, you’re good to go.

---

### 📦 4.3: Install Flux Controllers into Your Cluster

Now let’s install the Flux engine into your Kubernetes cluster.

Run:

```bash
flux install
```

This installs several controllers into a dedicated namespace called `flux-system`, including:

- **source-controller** – fetches content from Git  
- **kustomize-controller** – applies manifests to the cluster  
- **notification-controller** – sends alerts (optional, not used yet)  
- **helm-controller** – manages Helm charts (also not used yet)

---

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
It’s running inside your cluster, but it has no Git repo to watch yet.

Let’s give it one.

---

### ✍️ 4.5: Generate the `GitRepository` Resource

Use the Flux CLI to scaffold a GitRepository object.

This will tell Flux:

> “Clone this repo (your local folder), every 30 seconds.”

Run:

```bash
flux create source git local-repo \
  --url=file:///repo \
  --branch=main \
  --interval=30s \
  --export > clusters/local/flux/local-repo.yaml
```

💡 This uses the mounted folder you set up earlier (`/repo` inside the cluster).

> `--export` means “Don’t apply—just print the YAML so I can version it.”

---

### ✍️ 4.6: Generate the `Kustomization` Resource

Now tell Flux what to apply from that repo.

This resource says:

> “Go into `apps/hello` inside the repo, and apply whatever you find—every 1 minute.”

Run:

```bash
flux create kustomization hello-sync \
  --source=GitRepository/local-repo \
  --path="./apps/hello" \
  --prune=true \
  --interval=1m \
  --export > clusters/local/flux/hello-sync.yaml
```

Let’s pause.

At this point, you’ve defined:
- **What repo Flux should pull**
- **Which folder it should apply**
- **How often it should reconcile**

---

### 💾 4.7: Commit Your GitOps Resources

Everything you just generated is sitting in your local repo.

Let’s commit it.

```bash
git add .
git commit -m "wire: declare Flux GitRepository and Kustomization"
```

💡 This is your first real GitOps trigger:  
From now on, commits become deploys.

---

### 🚀 4.8: Apply the Resources (Once)

Now apply the two new YAMLs into your cluster:

```bash
kubectl apply -f clusters/local/flux/
```

This registers the GitRepository and Kustomization as in-cluster resources.

Flux is now officially watching Git.

---

### 👀 4.9: Watch Flux Deploy Your App

To confirm Flux is working:

```bash
flux get kustomizations
```

Look for:

```
NAME         READY   STATUS     AGE
hello-sync   True    Applied    1m
```

Now check if your app is running:

```bash
kubectl get pods
kubectl get svc hello
```

You should see the `hello` pod running, and a Service called `hello`.

Want to verify it works?

```bash
kubectl port-forward svc/hello 8080:80
```

Then open [http://localhost:8080](http://localhost:8080) in your browser.

🎉 Boom. Hello from NGINX—without you running `kubectl apply`.

---

### 🧠 4.10: A Moment of Reflection

Let this sink in:

✅ You didn’t apply the Deployment or Service.  
✅ You didn’t push to a container registry.  
✅ You didn’t trigger a pipeline.

You **committed to Git**—and the cluster did the rest.

That’s GitOps.  
You’ve just installed it, wired it, and watched it go live on your machine.

---

Next up?  
We’ll break the cluster on purpose—and watch Flux heal it automatically.  
That’s the power of **continuous reconciliation**.

---

## 🔄 Flux in Action: Two Quick Self‑Healing Drifts  

So far, you’ve seen GitOps take the wheel:  
You committed a `Deployment` and a `Service` to Git—and Flux made it real.

Now let’s take it one step further.  
Let’s break the cluster.  
Just a little.

And watch Flux put it back.

---

### 🧪 Drift #1 — Scale the App to Zero  

Flux believes `replicas: 1` is the truth—because that’s what’s in Git.  
So what happens if we scale it down to zero?

Let’s try it.

```bash
kubectl scale deploy/hello --replicas=0
watch kubectl get deploy/hello
```

(You’ll see `0/0` pods briefly… then Flux will restore it to `1/1`.)

🧠 **What just happened?**  
Flux noticed that the live state didn’t match what was in Git—and it quietly put things back.

---

### 🧨 Drift #2 — Delete the App Completely  

Let’s go a bit further. What if someone accidentally deleted the whole app?

```bash
kubectl delete deploy/hello svc/hello
watch kubectl get deploy,svc
```

Flux won’t panic. It’ll just re-read the manifests in Git—and reapply them.

Within a minute or less, you’ll see both the Deployment and Service return.

🧠 **Git is still the truth.**  
Even if something vanishes, Flux restores it. That’s reconciliation in action.

---

### 🧠 So What Did You Just Witness?

These two moments—scaling to zero and deleting everything—weren’t just tests.

They were **proof** that Git is no longer just a backup. It’s **active infrastructure**.

You now have a cluster that watches for drift and corrects it.  
No alerts. No panic. Just *quiet restoration*.

Your YAML is no longer just “config.” It’s reality—because Flux enforces it.  
That’s the power of declarative systems. That’s the heart of GitOps.

---

## ✅ Wrapping Up Day 2: What You Just Built (And Why It Matters)

Take a second to Congratulate yourself 🎉.

You didn’t just run a bunch of commands today. You built a system.  
A loop. A contract between Git and your cluster.

Let’s step back and look at what that really means:

- You wrote your application’s desired state—in YAML, in Git  
- You didn’t apply it manually—you let the cluster **pull** and apply it itself  
- You installed **Flux**, a GitOps agent that now lives *inside* your cluster  
- You saw it read Git, sync with the declared state, and launch your app  
- Then you **broke the cluster**—twice—and it healed itself without intervention

That’s not scripting. That’s not a tutorial.  
That’s **infrastructure with a memory**.

You’ve just experienced the difference between:

> *“I hope this is running the way I left it…”*  
> and  
> *“I know exactly what state we should be in—because it’s written in Git.”*

What you saw today is the foundation of modern platform operations:  
Declarative configuration. Continuous reconciliation. Git as the single source of truth.

It wasn’t theoretical. You built it. You watched it work. You even watched it recover.

That’s GitOps, in action.

---

### 🗺️ Coming Up on Day 3  

Next, we’ll take what you’ve built and lift it into the real world:

- Push your repo to GitHub  
- Bootstrap Flux to sync from GitHub instead of local  
- Separate environments (`dev` / `staging` / `prod`) using folder-based structures  
- Walk through real-world promotion flows  
- Introduce drift-tolerant overrides (like `flux suspend`) for safe debugging  
- Compare Flux with Argo CD

By the end of Day 3, you’ll be thinking like a platform engineer—and Git will be thinking for you.

Ready?

**We’ll see you there.**