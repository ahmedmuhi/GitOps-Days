## 🚀 Day 2 – Your First GitOps Loop (Running Flux in a Local Lab)

Welcome back to **GitOps-Days**.

If you're just joining us, [Day 1 – What Really Is GitOps?](https://github.com/ahmedmuhi/GitOps-Days/blob/main/Day-1-What-really-is-GitOps.md) introduced the problem of **drift** in Kubernetes clusters and showed how GitOps addresses it with four clear principles. If terms like *declarative*, *reconciliation*, or *source of truth* still feel unfamiliar, we recommend reviewing Day 1 first.

Today, you’ll move from concept to practice. You’ll create your first working GitOps loop—right on your laptop—using **Flux**, an open-source tool that brings GitOps to Kubernetes. Flux runs inside your cluster and continuously ensures that what’s running matches what’s declared in Git. When a difference is detected—such as a manual scale-down or deleted resource—Flux automatically corrects it.

In under an hour, you will:

* Set up a local Kubernetes cluster with `kind`
* Install Flux and connect it to your GitHub repository
* Watch Flux apply your application’s configuration automatically
* Break the cluster on purpose and see Flux fix it
* Deploy new versions simply by committing to Git

By the end of this lab, you’ll have a self-healing system powered by Git. You’ll stop applying YAML by hand—and start relying on Git to manage what runs in your cluster.

Let’s begin.

## 🧰 Environment Setup – Tools for the GitOps Lab

This GitOps lab runs entirely on your local workstation using Docker and kind. Before we continue, make sure you have the following tools installed.

> 💡 **Already have these tools installed?** [Skip ahead to Fork and Clone the GitOps Repository](#fork-and-clone-the-gitops-repository)

---

### ✅ Required Tools

These tools form the foundation for running a local Kubernetes cluster and interacting with it. If you're missing any, use the links below to install them.

| Tool                                                                 | Minimum Version | Purpose                                             |
| -------------------------------------------------------------------- | --------------- | --------------------------------------------------- |
| [Docker](https://docs.docker.com/get-docker/)                        | ≥ 24.0          | Runs the containers that power your local cluster   |
| [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) | ≥ 0.25.0        | Creates a local Kubernetes cluster inside Docker    |
| [kubectl](https://kubernetes.io/docs/tasks/tools/)                   | ≥ 1.32          | CLI to interact with the Kubernetes API             |
| [Git](https://git-scm.com/downloads)                                 | ≥ 2.40          | Used to clone and track changes to your GitOps repo |

> 💡 Docker Desktop v4.29+ for macOS/Windows already includes Docker Engine 24+, which works with kind.

---

### 🔍 Verify Installed Versions

After installing the tools above, confirm that your environment is ready by running the following commands:

```bash
docker --version
kind --version
kubectl version --client --short
git --version
```

If all your versions meet or exceed the minimum requirements, you’re ready to move on.

## 🗃️ Fork and Clone the GitOps Repository

In GitOps, your Git repository declares the **desired state** of your system.
For this lab, you’ll fork a pre-built repository that includes a minimal demo app and ready-made Kubernetes manifests. This lets you focus on learning GitOps—not writing YAML from scratch.

The repository also follows a real-world folder structure used in larger GitOps environments. Once forked and cloned, it becomes the source of truth that your cluster will follow.

---

### 🔀 Stage A – Fork in the Browser

1. Open [`https://github.com/ahmedmuhi/GitOps-Days`](https://github.com/ahmedmuhi/GitOps-Days)
2. In the top-right corner, click **Fork** (the button may read **Create fork**).
3. Accept the defaults—especially the branch name `main`—and click **Create fork**.
   You now have your own copy under `YOUR-USERNAME/GitOps-Days`.

---

### 💻 Stage B – Clone Your Fork Locally

Open your terminal (**PowerShell** on Windows or Terminal on macOS/Linux) and run:

```bash
git clone https://github.com/YOUR-USERNAME/GitOps-Days.git
cd GitOps-Days
```

> ⚠️ **Important:** Make sure you fork first, then clone your version—**not** the original repo.

At this point, your local copy holds the configuration that your cluster will later reconcile against.

---

### 📂 What's Inside the Repository

The repo includes:

* **Manifests** – Kubernetes `Deployment` and `Service` YAML for the demo app
* **Directory structure** – Reflects environment-based GitOps patterns (e.g. `clusters/local/...`)

We’ll take a closer look at this layout once your local cluster is running.

## 🧱 Spin Up a Local Kubernetes Cluster

Before we can see GitOps in action, we need a Kubernetes cluster where changes can occur. Instead of using a cloud provider, you’ll create a small local lab cluster on your own machine—perfect for learning, safe to experiment with, and easy to reset.

> 💡 **Heads-up:** `kind` typically uses around **3 GB of RAM**. If you're working on a machine with limited memory, close other heavy applications before continuing.

We’ll use [**kind**](https://kind.sigs.k8s.io/) (short for *Kubernetes in Docker*) to do this. It runs a full Kubernetes cluster inside Docker containers and usually starts in just a few seconds.

---

### ▶️ Step 1 · Create the Cluster

Open your terminal—**PowerShell** on Windows or Terminal on macOS/Linux—and run the following command to create a cluster named `gitops-loop-demo`:

```bash
kind create cluster \
  --name gitops-loop-demo \
  --image kindest/node:v1.32.2
```

> 💡 We specify the image version so your cluster uses a consistent Kubernetes release (`v1.32.2`), ensuring reproducible results across setups.

---

### ✅ Step 2 · Verify the Cluster Is Ready

Once the cluster is created, check that it’s running:

```bash
kubectl get nodes
```

You should see output similar to:

```
NAME                             STATUS   ROLES           AGE   VERSION
gitops-loop-demo-control-plane   Ready    control-plane   1m    v1.32.x
```

> ℹ️ **What does “Ready” mean?** It indicates that the node is available to schedule workloads and that the cluster is functioning properly.

> ℹ️ **Why only one node?** `kind` creates a single control-plane node that also runs your pods. This setup keeps resource usage low—ideal for learning environments. In production, you'd typically add dedicated worker nodes.

> ⚠️ **Cleanup tip:** To delete the cluster later (or reset it), run:
>
> ```bash
> kind delete cluster --name gitops-loop-demo
> ```

---

With the cluster ready, it’s time to connect it to the application declared in your GitOps repository—and see how that folder structure defines what actually runs.

## 📦 Understand the Application and Its Manifests

Now that your cluster is running, let’s examine the application configuration declared in your Git repository. These manifests define the **desired state** that Flux will soon begin reconciling.

We use a minimal and predictable container image: [`nginxdemos/hello:plain-text`](https://hub.docker.com/r/nginxdemos/hello).

This image:

* Runs a lightweight NGINX web server
* Serves a static **Hello World** page
* Starts quickly, with no custom configuration

> This minimal app lets you focus entirely on the GitOps process—without needing to write or debug application code.

---

### 🗂️ Repository Folder Structure

Inside the repository you forked earlier, you’ll find the manifests that define this application here:

```text
examples/day2-gitops-loop-demo/clusters/local/apps/hello/
```

Here’s how the application is structured within the repository:

```
clusters/
└── local/
    └── apps/
        └── hello/
            ├── deployment.yaml
            └── service.yaml
```

| Folder        | Purpose                                                                                                                                  |
| ------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `clusters/`   | Top-level folder for cluster-specific configuration, organised by environment                                                            |
| `local/`      | The **environment name** (in a real repo you might have `dev`, `staging`, or `prod`)                                                                 |
| `apps/hello/` | Contains manifests for a single application (`hello`). Other apps would get their own sub-folders, such as `apps/cart/` or `apps/auth/`. |

> 🧠 Notice how the repo separates **environments** (`local`) from **applications** (`hello`). This hierarchy helps GitOps repositories stay organised and scalable as they grow.

---

### 📄 What the Manifests Declare

Inside `apps/hello/`, you’ll find two YAML files:

#### 🛠️ `deployment.yaml`

A **Deployment** manages a **ReplicaSet**, which ensures that a specified number of identical **pods** are running at all times. A pod is the smallest deployable unit in Kubernetes—essentially one instance of our container. The Deployment ensures that the cluster stays aligned with the declared state.

Key fields:

* `replicas: 1` – run a single copy of the app
* `image: nginxdemos/hello:plain-text` – the container image
* `containerPort: 80` – the app listens on port 80
* `labels.app: hello` – tags the pod for the Service selector

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

> 💡 Once Flux is connected to the cluster, it will watch this **Deployment** (and the accompanying **Service**) for drift. Later in the lab, we’ll deliberately scale `replicas` to `0`; Flux will detect the mismatch and restore it to `1` automatically.

---

#### 🛠️ `service.yaml`

A **Service** provides a stable network identity for a set of pods. Rather than targeting individual pods—which come and go—the Service uses **label selectors** to find and load-balance across matching pods.

Key fields:

* `name: hello` – the Service name
* `selector.app: hello` – matches pods with that label
* `port: 80` – exposes port 80 inside the cluster

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

---

### 🧭 Where We Stand

> **✅ Done so far**
> • Forked the GitHub repository (source of truth)
> • Created a local Kubernetes cluster
> • Reviewed the manifests that define the application’s desired state
>
> **🟡 Still to come**
> • Nothing applied to the cluster yet (`kubectl apply` not used)
> • GitOps automation not running yet

The GitOps loop is still quiet—exactly where we want it before exploring how Flux works and how it will bring these manifests to life.

## 🔁 Inside the GitOps Loop: How Flux Works

So far, you've forked a repository, spun up a cluster, and reviewed the application manifests. In a typical Kubernetes workflow, this is where you might run:

```bash
kubectl apply -f path/to/your/yaml
```

Today, we’ll do something different.

Instead of pushing YAML into the cluster manually, you’ll hand control to **Flux**—a GitOps agent that continuously ensures your cluster matches what’s in Git.
Think of it as putting your infrastructure on **autopilot**—or more precisely, into a self-correcting mode that watches for changes and realigns your system automatically.

---

### 🤖 Meet Flux – the GitOps Agent

Flux is an open-source CNCF project—alongside tools like Argo CD—that implements GitOps for Kubernetes. It installs as a **bundle of controllers** (custom Kubernetes components that watch and act on specific resources), each running as a pod inside the `flux-system` namespace.

Once installed, these controllers:

1. **Watch** a Git repository for changes
2. **Pull & apply** the desired state
3. **Continuously reconcile** the cluster so its live state always matches Git

> This is the same **watch → pull → reconcile** loop introduced earlier—Flux turns that pattern into running infrastructure.

Flux checks Git every minute by default (using the `interval` setting in its configuration). You can also configure a webhook so changes apply almost immediately after a commit.

---

### 🧩 How Does Flux Know What to Do?

To bring the loop to life, Flux relies on two custom resources—think of them as its instructions:

#### 1. `GitRepository` – *“Where should I look?”*

This resource tells Flux the **URL**, **branch**, and **interval** for checking a Git repository.
The `source-controller` watches this resource, fetches the repository on a schedule, and keeps a cached copy inside the cluster.

At this point, Flux is only observing—no YAML is applied yet.
That responsibility belongs to the next resource: the `Kustomization`, which defines what to apply.

---

#### 2. `Kustomization` – *“Okay, now what should I apply?”*

A `Kustomization` points to a **folder** inside the Git repository and tells Flux:

* What to apply (via the path)
* How often to reconcile it
* Whether to **prune** deleted resources

```yaml
prune: true
```

Setting `prune: true` means that if a manifest is deleted from Git, the corresponding resource is also deleted from the cluster. This keeps the cluster tightly in sync with Git—a good practice in production.
For local testing or experimentation, you might turn pruning off.

Together, these two resources work like this:

```
GitRepository  ──► pull repo & keep cache
Kustomization ──► apply folder & reconcile drift
```

---

### 🔄 Why Two Resources Instead of One?

Flux separates **what to fetch** (`GitRepository`) from **what to apply** (`Kustomization`). While this might seem like added complexity at first, it introduces important flexibility—both for scaling GitOps and for managing teams and workloads effectively.

Here’s what that separation enables:

* **Multiple Git sources**
  You can define multiple `GitRepository` resources—each pointing to a different repository, branch, or tag.
  These sources are fetched and cached independently, so the cluster can track code from various locations—even if not all of it is applied immediately.

* **Folder-level targeting**
  A single Git repository may include multiple environments (`dev`, `prod`) or applications (`cart`, `auth`, `checkout`).
  Each `Kustomization` targets a specific folder, letting you apply just the configuration relevant to a given workload.

* **Custom sync intervals**
  Repositories with frequent changes can sync every minute, while more stable ones can reconcile hourly.
  Each `Kustomization` has its own `interval`, letting you fine-tune update frequency to match the app’s needs.

* **Team-specific control**
  You can assign responsibility for each `Kustomization` to a separate team.
  Teams can manage changes, test updates, or revert configurations independently—without requiring access to unrelated workloads.

This model keeps your GitOps setup modular and predictable.
Git sources are fetched once and reused across the cluster; changes are applied only where needed.
It’s a pattern that works equally well for small teams and large organisations managing multiple services and environments.

## 🛠️ Install the Flux CLI

To begin using Flux, you’ll first install the **Flux CLI** on your local machine. This tool helps you install Flux into your Kubernetes cluster and connect it to your Git repository in the steps ahead.

Use the instructions below to install the CLI for your system:

---

### macOS

**Recommended (via Homebrew):**

```bash
brew install fluxcd/tap/flux
```

---

### Linux

**Recommended (via universal install script):**

```bash
curl -s https://fluxcd.io/install.sh | sudo bash
```

> 📚 For other Linux installation options or shell-specific completions, see the [official Flux installation docs](https://fluxcd.io/docs/installation/).

---

### Windows

**Recommended (via Chocolatey):**

```powershell
choco install fluxcd
```

Alternatively, download the appropriate binary for your system from the [Flux GitHub Releases page](https://github.com/fluxcd/flux2/releases), and add it to your system’s `PATH`.

> 💡 **Tip:** If the `flux` command isn’t recognised after install, reload your shell or terminal and ensure the binary is in your system’s `PATH`.

---

### ✅ Verify the Installation

Run the following command:

```bash
flux --version
```

You should see output similar to:

```
flux version 2.5.1
```

If you receive a "command not found" error or see an unexpected version, double-check your installation and environment variables.
Now that the Flux CLI is installed and verified, you're ready to install Flux into your Kubernetes cluster.
In the next section, you’ll deploy the Flux controllers into the cluster and set up the control loop that keeps it in sync with Git.

## 🚀 Install Flux in the Cluster

With the Flux CLI installed, it’s time to deploy the Flux controllers into your Kubernetes cluster. This step sets up the control loop that watches Git and applies changes automatically.

---

### 🔍 Confirm Your Kubernetes Context

Before installing anything, double-check that you're pointing at the right cluster—especially if you run more than one.

Run:

```bash
kubectl config current-context
```

If the result is:

```
kind-gitops-loop-demo
```

—you’re in the right place.

Now check that your environment is ready for Flux:

```bash
flux check --pre
```

This ensures your CLI version is compatible with the Kubernetes API, and that required permissions are available.

---

### 📦 Install the Controllers

Once the pre-flight check passes, install the Flux controllers into the cluster by running:

```bash
flux install
```

This command:

* Creates the `flux-system` namespace
* Deploys the core set of controllers responsible for pulling from Git and applying manifests

You should see confirmation output with status messages as each controller is created.

To verify the installation:

```bash
kubectl get pods -n flux-system
```

You should see output similar to:

```
NAME                                     READY   STATUS    RESTARTS   AGE
helm-controller-xxxx                     1/1     Running   0          10s
kustomize-controller-xxxx                1/1     Running   0          10s
notification-controller-xxxx             1/1     Running   0          10s
source-controller-xxxx                   1/1     Running   0          10s
```

All pods should be in a `Running` state with `READY` set to `1/1`.

---

### 🔎 What Got Installed?

| Controller                | Role                                                            |
| ------------------------- | --------------------------------------------------------------- |
| `source-controller`       | Clones Git repositories and keeps them cached in the cluster    |
| `kustomize-controller`    | Applies Kubernetes manifests and keeps the cluster reconciled   |
| `notification-controller` | Handles eventing and alerts (not used in this lab)              |
| `helm-controller`         | Enables Helm chart releases via Git (also not used in this lab) |

> ⚠️ **Important:** If you delete the `flux-system` namespace, all Flux components will be removed. You’ll need to reinstall them from scratch.

## 🔗 Start the GitOps Loop: Connect Flux to Git

With the Flux controllers now running in your cluster, it's time to activate the GitOps loop by connecting Flux to your GitHub repository and defining what should be applied—and how.

---

### 📥 1. Create a `GitRepository`

Before Flux can apply anything, it needs to know where to fetch your configuration from.

The command below creates a `GitRepository` resource named `gitops-loop-demo`. It tells Flux to track your fork of the GitOps-Days repo on the `main` branch and to check for changes every 30 seconds.

> 📌 **Note:** This example assumes your repository is public. If it's private, you'll need to supply authentication credentials using `--username` and `--password`, or configure SSH. See the [Flux authentication docs](https://fluxcd.io/flux/components/source/gitrepositories/#authentication) for more options.

```bash
flux create source git gitops-loop-demo \
  --url=https://github.com/YOUR-USERNAME/GitOps-Days.git \
  --branch=main \
  --interval=30s
```

Verify that the repository has been registered correctly:

```bash
flux get sources git
```

You should see:

```
NAME                READY   STATUS    AGE
gitops-loop-demo    True    Fetched   30s
```

---

### 🧩 2. Create a `Kustomization`

Now that Flux is fetching your repo, it needs to know **what** to apply and **how**.

The `Kustomization` resource tells Flux which folder in the repo to apply, how often to reconcile it, and whether to prune resources that have been removed from Git.

Configuration summary:

* **Source:** `gitops-loop-demo`
* **Path:** `./examples/day2-gitops-loop-demo/clusters/local/apps/hello`
* **Prune:** `true`
* **Interval:** `1m`

#### macOS / Linux

```bash
flux create kustomization hello-app \
  --source=GitRepository/gitops-loop-demo \
  --path="./examples/day2-gitops-loop-demo/clusters/local/apps/hello" \
  --prune=true \
  --interval=1m
```

#### Windows (PowerShell)

```powershell
flux create kustomization hello-app `
  --source=GitRepository/gitops-loop-demo `
  --path "./examples/day2-gitops-loop-demo/clusters/local/apps/hello" `
  --prune true `
  --interval 1m
```

Expected output:

```
► applying Kustomization reconciliation hello-app
✔ Kustomization reconciliation hello-app created
```

> ⚠️ **Note on `--prune=true`:** This keeps your cluster tightly aligned with Git. If a manifest is deleted from the repo, the resource will also be deleted from the cluster. For dev/test scenarios, you might prefer `--prune=false` to avoid accidental removal.

---

### 🔁 What Just Happened?

As soon as the `Kustomization` was created, Flux began reconciling your cluster to match the state in Git.

No `kubectl apply`. No manual triggers. Just Git → cluster.

```
GitRepo (in GitHub)
   ↓
source-controller (pull)
   ↓
Kustomization (apply)
   ↓
Kubernetes resources (live in cluster)
```

---

### ✅ Quick Recap

At this point:

* Flux is watching your GitHub repository and caching changes every 30 seconds
* The `hello-app` Kustomization applies configuration from a specific folder in your repo
* Reconciliation is running every minute
* Your cluster now reflects the desired state defined in Git

Next, we’ll confirm that the application was actually deployed—and watch Flux recover from manual drift.

## 🎬 Verify the GitOps Loop in Action

You’ve done the work:

* Forked and structured your GitOps repo
* Created a Kubernetes cluster
* Installed Flux and connected it to Git

When you created the `Kustomization`, Flux immediately scanned your Git repository, detected Kubernetes manifests, and applied them to the cluster—**without you touching `kubectl apply`.**

Now let’s confirm what just happened.

---

### ✅ Step 1 – Check the Reconciliation Status

Run:

```bash
flux get kustomizations
```

You should see:

```
NAME         READY   STATUS    AGE
hello-app    True    Applied   1m
```

This confirms Flux:

* Detected that no matching resources were present
* Applied the Deployment and Service from Git
* Updated the cluster to match the declared state

The keyword **Applied** means changes were made. That’s reconciliation in action.

---

### ✅ Step 2 – Confirm the App Is Running

Run:

```bash
kubectl get pods,svc
```

You should see something like:

```
NAME     READY   STATUS    RESTARTS   AGE
hello    1/1     Running   0          1m

NAME     TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
hello    ClusterIP   10.x.x.x       <none>        80/TCP    1m
```

Look for a pod named `hello` and a Service exposing it on port 80—these were created from the manifests in Git.

To access the app:

```bash
kubectl port-forward svc/hello 8080:80
```

Then open [http://localhost:8080](http://localhost:8080) in your browser. You should see:

> **Welcome to nginx!**

That response confirms your app is running in the cluster, and serving traffic from a Git-defined Deployment.

---

### 🔄 What Just Happened?

Here’s what you didn’t do:

* No `kubectl apply`
* No manual rollout
* No script

Here’s what did happen:

* Git declared the state
* Flux detected it
* Your cluster followed it

This wasn’t a one-time deploy.
It’s **continuous, declarative operations.**

## 🔥 Break the Cluster, Watch It Heal

Your app is live.
Your GitOps loop is running.
Now let’s test how well it holds under pressure.

We’re going to simulate real-world drift—manual changes to the cluster that aren't reflected in Git—and watch Flux quietly correct them.

---

### 🧪 Drift Test 1: Scale the App to Zero

Your Git repository declares `replicas: 1` for the `hello` deployment.
Let’s override that manually.

```bash
kubectl scale deployment/hello --replicas=0
```

Then monitor the deployment:

```bash
watch kubectl get deployment hello
```

*(Or run the command manually every few seconds.)*

At first, you’ll see:

```
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
hello   0/0     0            0           2m
```

The pod is gone. The app is unavailable.

But within about a minute...

✅ Flux detects the drift
✅ It re-applies the desired state from Git
✅ The pod is restored to `replicas: 1`

This was a **manual override**—Flux reversed it automatically.

---

### 💥 Drift Test 2: Delete the App Completely

Let’s go further.
You’ll now simulate a full deletion—both the Deployment and Service.

```bash
kubectl delete deployment/hello service/hello
```

Check the current state:

```bash
watch kubectl get deployment,service
```

You’ll briefly see:

```
No resources found in default namespace.
```

Then… Flux acts.

Within about a minute:

✅ The Deployment is recreated
✅ The Service is restored
✅ Your app is live again—no intervention needed

---

### 🧠 A Quiet but Powerful Shift

This wasn’t a redeploy.
This wasn’t a script.
This was **continuous reconciliation**.

> **Git declared it.
> Flux enforced it.
> You didn’t have to intervene.**

You now have a system that defends its own state—automatically.

## 🎯 Reflect and Wrap Up Day 2

Let’s take a moment to pause.

What you built today wasn’t just a demo—it was a **self-correcting system**:

* A Kubernetes cluster that **pulls its desired state** from Git
* A Git repository that **declares** how your app should run
* A GitOps agent (Flux) that **watches**, **applies**, and **heals** automatically

You didn’t push YAML.
You didn’t roll back changes.
You let Git drive the state of your cluster—and Flux kept it honest.

> 🧠 You shifted from imperative to declarative.
> You moved from scripts to reconciliation.
> You now operate through version control—not ad hoc fixes.

This is GitOps in action. And you’ve seen it **working live**—including how it defends itself when drift occurs.

---

### 💡 What You Can Already Do

If you stopped here, you’ve already achieved something valuable:

* A fully working GitOps loop
* A real cluster that syncs to Git
* Declarative deployment, continuous reconciliation, and self-healing

You could take this repo, show your team, and even use it as the foundation for real dev/test environments.

---

## 📈 Where We’re Heading Next

In Day 3, we’ll take this exact setup—and run it **on Azure Kubernetes Service (AKS)**.

You’ll see:

* How to provision AKS and connect it to Flux
* How the same GitOps flow works in a cloud-native context
* A production-grade foundation you can build on

Same repo.
Same manifests.
Now running in the cloud.

**Day 2: complete.
Day 3: we go cloud-native.**
