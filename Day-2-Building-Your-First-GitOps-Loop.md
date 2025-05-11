# 🚀 Day 2 – Building Your First Self-Healing Kubernetes System with Flux

**Ever wished your Kubernetes cluster could fix itself?** Today, that becomes your reality.

Welcome back to **GitOps-Days**. On Day 1, we explored why Kubernetes clusters drift away from their intended state and how GitOps principles—such as a single source of truth, declarative configurations, and continuous reconciliation—solve this critical problem. If any of these ideas still feel unfamiliar, you might find it helpful to quickly revisit [Day 1 – What Really Is GitOps?](https://github.com/ahmedmuhi/GitOps-Days/blob/main/Day-1-What-really-is-GitOps.md)

Today, we turn these abstract concepts into concrete practical skills. You'll build your first complete **GitOps loop** using Flux, a powerful open-source GitOps tool that continuously aligns your cluster with your Git repository. Imagine no longer needing to manually push YAML manifests into your Kubernetes cluster. Instead, your cluster actively pulls its configuration from Git, detects when something deviates from your declared state, and then automatically corrects itself—almost like magic, but actually just good engineering.

### 🗺️ Your GitOps Journey Today

**Today's hands-on lab includes:**
* Setting up our lab environment
* Creating our Git-based source of truth
* Spinning up a local Kubernetes cluster
* Deploying Flux as our GitOps engine
* Connecting our Git repo to our cluster
* Watching the system heal itself automatically

This practical experience takes approximately 60-70 minutes to complete and works equally well for GitOps newcomers and Kubernetes veterans looking to eliminate configuration drift challenges.

### 🎯 What You'll Accomplish

By the end of this session, you'll:
* Have a working self-healing Kubernetes system running on your laptop
* Understand how to implement GitOps principles with real tools
* Be able to leverage Git commits for automated deployments
* Experience firsthand how a system can automatically recover from failures
* Build infrastructure that maintains itself, reducing operational overhead

Ready to create a system that literally cannot stay broken for long? Let's begin!

## 🧰 Preparing Your Workspace – Essential Tools for GitOps

Before we dive deeper, let's quickly ensure your workstation is ready. We'll be running Flux inside our Kubernetes cluster. To do that effectively, you'll first need to create a lightweight local environment using Docker and kind. This setup allows you to safely experiment locally, so you can confidently explore GitOps concepts.

> **Tip:** Already have all the tools below installed?
> [Jump straight to Fork and Clone the GitOps Repository](#fork-and-clone-the-gitops-repository).

> **Note:** This lab requires approximately 4GB of available RAM and 2GB of disk space.

---

### Required Tools

The following tools work across Windows, macOS, and Linux (installation steps may vary slightly, but the links provided include platform-specific instructions):

| Tool    | Min. Version (newer versions are generally compatible) | What it does                       | Installation                                                                 |
| ------- | ----------------------------------------------------- | ---------------------------------- | ---------------------------------------------------------------------------- |
| Docker  | ≥24.0                                                 | Runs containers for your cluster   | [Install Docker](https://docs.docker.com/get-docker/)                        |
| kind    | ≥0.25.0                                               | Creates your local Kubernetes lab  | [Install kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) |
| kubectl | ≥1.32                                                 | Interacts with Kubernetes clusters | [Install kubectl](https://kubernetes.io/docs/tasks/tools/)                   |
| Git     | ≥2.40                                                 | Clones and tracks your manifests   | [Install Git](https://git-scm.com/downloads)                                 |

> **Tip:** Docker Desktop v4.29+ for macOS or Windows includes Docker Engine 24+ out of the box—fully compatible with kind.

---

### ✅ Verify Your Setup

Once you've installed the tools, confirm your setup is ready by running these commands in your terminal:

```bash
docker --version
kind --version
kubectl version --client --short
git --version
```

If everything is installed correctly, you should see version numbers matching or exceeding these examples:

```bash
Docker version 24.0.0, build abc1234
kind version 0.25.0
Client Version: v1.32.0
git version 2.40.0
```

> **Note:** If any tool returns a version below the minimum requirement, revisit its installation guide or update it using your package manager.

With your environment checked and ready, you'll next fork the repository that will serve as your GitOps source of truth.

## 📂 Creating Your Source of Truth – Fork and Clone the Repository

In GitOps, our Git repository is the **single source of truth**. It defines the desired state that your Kubernetes cluster continuously aligns with.

For this lab, you'll begin by forking a pre-built GitHub repository. It contains a minimal demo application and ready-to-use Kubernetes manifests. This allows you to focus entirely on GitOps concepts, without spending time writing YAML from scratch.

The repository structure mirrors real-world GitOps patterns, helping you easily apply what you learn here to larger environments later. Once forked and cloned, it becomes the authoritative configuration that your local cluster will follow.

---

### 1️⃣ Fork the Repository on GitHub

1. Open the GitHub repository at [`https://github.com/ahmedmuhi/GitOps-Days`](https://github.com/ahmedmuhi/GitOps-Days).
2. Click the **Fork** (or **Create fork**) button at the top right.
3. Ensure the selected branch is set to `main`, then click **Create fork**.

You should now have a fork at:

```
https://github.com/YOUR-USERNAME/GitOps-Days
```

---

### 2️⃣ Clone Your Fork to Your Local Machine

Now, let's clone your personal fork onto your local machine:

Open your terminal and run:

> **For Windows:** Use PowerShell or Windows Terminal
> **For macOS/Linux:** Use Terminal

```bash
git clone https://github.com/YOUR-USERNAME/GitOps-Days.git
cd GitOps-Days
git remote -v
```

The output of the last command (`git remote -v`) should confirm that `origin` points to your fork (containing `YOUR-USERNAME`), not the original `ahmedmuhi` repository.

> ⚠️ **Important:** Always fork first and then clone your fork, never the original repository. This ensures you have write access to push changes later.

Your local copy now contains the configurations and manifests that your cluster will later reconcile against. Next, we'll create a Kubernetes cluster where these configurations will come to life.

## 🔄 Creating Your Kubernetes Environment – A Local Cluster for GitOps

Before we can see GitOps in action, we need a Kubernetes cluster where our configurations from Git will be applied. Instead of using a cloud provider, you'll create a small local lab cluster on your own machine—perfect for learning, safe to experiment with, and easy to reset.

> 💻 **System Requirements:** `kind` typically uses around **3-4 GB of RAM** during operation. If you're working on a machine with limited memory, close other resource-intensive applications before continuing.

We'll use [**kind**](https://kind.sigs.k8s.io/) (short for *Kubernetes in Docker*) to create our cluster. It runs a full Kubernetes environment inside Docker containers, typically starting in under a minute and requiring minimal configuration.

---

### 🏗️ Creating the Cluster

Open your terminal and run the following command to create a cluster named `gitops-loop-demo`:

> **For Windows:** Use PowerShell or Windows Terminal  
> **For macOS/Linux:** Use Terminal

```bash
kind create cluster \
  --name gitops-loop-demo \
  --image kindest/node:v1.32.2
```

> 💡 We specify the image version to ensure your cluster uses a consistent Kubernetes release (`v1.32.2`), guaranteeing compatible behavior across different setups.

---

### 🔍 Verifying Cluster Readiness

Once the cluster creation completes, let's confirm it's properly running and ready to accept our GitOps configurations:

```bash
kubectl get nodes
```

You should see output similar to:

```
NAME                             STATUS   ROLES           AGE   VERSION
gitops-loop-demo-control-plane   Ready    control-plane   1m    v1.32.x
```

> **What "Ready" means:** The STATUS column shows "Ready" when the node has successfully registered with the cluster, has all required components running, and is available to schedule workloads.

> **Why only one node?** For local testing, `kind` creates a single control-plane node that also runs your application pods. This configuration keeps resource usage low while still providing a fully functional Kubernetes environment. In production clusters, you'd typically have dedicated worker nodes separate from the control plane.

> ⚠️ **Cleanup tip:** When you're done with the lab or need to start fresh, you can delete the cluster by running:
>
> ```bash
> kind delete cluster --name gitops-loop-demo
> ```

Your Kubernetes environment is now ready and waiting for configuration. Next, we'll examine the application manifests that Flux will soon deploy to this cluster.

## 📄 Exploring Your Declarative Configuration – Understanding the Application Manifests

Now that your cluster is running, let's examine the application configuration that Flux will deploy. These manifests in your Git repository define the **desired state** that Flux will soon begin reconciling with your cluster.

For this lab, we're using a minimal and predictable container image: [`nginxdemos/hello:plain-text`](https://hub.docker.com/r/nginxdemos/hello).

This image provides:
* A lightweight NGINX web server
* A simple "Hello World" static page
* Quick startup with zero configuration needed

> 💡 **Why this image?** This straightforward application lets you focus entirely on the GitOps workflow rather than debugging application code. It's perfect for demonstrating how the GitOps loop works without adding unnecessary complexity.

---

### 🗂️ Repository Organization

Inside the repository you forked earlier, the application manifests live in this directory structure:

```text
examples/day2-gitops-loop-demo/clusters/local/apps/hello/
```

Here's how the application is organized:

```
clusters/
└── local/
    └── apps/
        └── hello/
            ├── deployment.yaml
            └── service.yaml
```

This structure follows GitOps best practices:

| Folder        | Purpose                                                                                                                                  |
| ------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `clusters/`   | Top-level folder for all cluster-specific configuration                                                                      |
| `local/`      | The **environment name** (in real-world setups, you'd have folders like `dev`, `staging`, and `prod`)                                                                 |
| `apps/hello/` | Contains manifests for a single application (in larger setups, you'd have multiple application folders like `apps/cart/`, `apps/auth/`, etc.) |

> 🔑 **Key pattern:** Separating **environments** (`local`) from **applications** (`hello`) creates a structure that scales well as your GitOps implementation grows to handle more services and environments.

---

### 📃 What's Inside the Manifest Files

Let's explore what's defined in the `apps/hello/` directory:

#### 🔷 `deployment.yaml` - Defining Our Application Pods

A **Deployment** is a Kubernetes resource that:
- Manages a set of identical **Pods** (the smallest deployable units in Kubernetes)
- Ensures a specified number of replicas are always running
- Handles updates and rollbacks gracefully

Key configuration highlights:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
spec:
  replicas: 1                  # Run one instance of our app
  selector:
    matchLabels:
      app: hello               # Select pods with this label
  template:
    metadata:
      labels:
        app: hello             # Label for our pods
    spec:
      containers:
        - name: hello
          image: nginxdemos/hello:plain-text   # The container image to run
          ports:
            - containerPort: 80                # The port our app listens on
```

> 🔄 **GitOps in action:** Later in the lab, we'll deliberately change the `replicas: 1` value by manually scaling down the deployment. Flux will detect this drift and automatically restore it to match what's declared in Git.

---

#### 🔷 `service.yaml` - Making Our Application Accessible

A **Service** in Kubernetes:
- Provides a stable network endpoint for accessing pods
- Load-balances traffic across all matching pods
- Uses labels to find the right pods, regardless of where they're running

Key configuration highlights:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello
spec:
  selector:
    app: hello               # Target pods with this label
  ports:
    - port: 80               # Port exposed by the Service
      targetPort: 80         # Port to forward to on the Pod
```

This Service will find any pods with the label `app: hello` and make them accessible within the cluster.

---

## 🧭 Ready for GitOps

So far you've:

✅ Forked the GitHub repository (your source of truth)  
✅ Created a local Kubernetes cluster  
✅ Reviewed the manifests that define the application's desired state  

At this point, you have all the components needed for GitOps, but they're not connected yet:
- Your GitHub repository contains configurations
- Your Kubernetes cluster is running but empty
- The GitOps automation isn't yet in place

Next, we'll explore how Flux works and how it will bring these manifests to life, turning our static YAML files into a self-healing system.

## 🔄 Understanding the GitOps Engine – How Flux Creates Self-Healing Systems

So far, you've forked a repository, created a cluster, and reviewed the application manifests. In a traditional Kubernetes workflow, your next step would be:

```bash
kubectl apply -f path/to/your/yaml
```

**But GitOps takes a different approach.**

Instead of manually pushing YAML into the cluster, you'll delegate that responsibility to **Flux**—a GitOps engine that continuously ensures your cluster matches what's defined in Git. It's like putting your infrastructure on **autopilot**, with a self-correcting system that watches for changes and automatically realigns your environment.

---

### 🤖 Introducing Flux – The GitOps Controller

Flux is an open-source Cloud Native Computing Foundation (CNCF) project that implements GitOps principles for Kubernetes. When Flux is installed in your cluster, it creates a continuous reconciliation loop:

1. **Watch** your Git repository for changes
2. **Pull** the latest configuration from Git
3. **Apply** those changes to your cluster
4. **Detect** any drift between the desired state (Git) and the actual state (cluster)
5. **Correct** automatically by reapplying the Git configuration

This means once you commit a change to your application manifests, Flux automatically detects and applies it—no manual intervention required. Even more impressive, if someone makes direct changes to the cluster that don't match Git, Flux will revert those changes to maintain consistency.

> 💡 **Key insight:** In GitOps, Git becomes the single source of truth. Any changes made directly to the cluster will be overwritten during reconciliation.

---

### 🧩 Flux's Two-Part Architecture

To implement this continuous loop, Flux uses two main components:

**1. Source Controller**  
Responsible for monitoring Git repositories and detecting changes. Think of this as the "watcher" that keeps an eye on your source code.

**2. Kustomize Controller**  
Responsible for applying configurations to the cluster and ensuring it matches the desired state. This is the "reconciler" that keeps your cluster aligned with Git.

> 📝 **Note:** This "Kustomize Controller" shouldn't be confused with the Kustomize tool (which is used for templating Kubernetes manifests). They share a name but serve different purposes.

To deploy these controllers, we'll be using two custom resources that you'll create later:

* **GitRepository** tells Flux where to find your configuration
* **Kustomization** tells Flux what to apply and how often to reconcile

Don't worry about the details yet—when you install Flux, we'll revisit these concepts with practical examples.

---

### 💡 Why This Architecture Matters

You might wonder why Flux uses two separate components instead of a single controller. This separation provides several practical benefits:

* **Team-focused access control:** Different teams can manage different parts of your infrastructure by using separate Kustomization resources pointing to their folders
* **Folder-level targeting:** You can apply specific folders from a repository, allowing you to organize environments (dev/staging/prod) or applications in the same repository
* **Multi-source configuration:** Your cluster can pull configurations from multiple repositories, giving you flexibility in how you organize your GitOps workflow

These capabilities make Flux suitable for both individual projects and larger team environments—the same principles scale from personal experiments to enterprise deployments.

---

### 🧭 Where We Are in the Journey

You now understand the core concepts of how Flux implements GitOps. In the next two sections, we'll:

1. Install the Flux CLI (command-line interface) to manage Flux components
2. Deploy Flux controllers to your cluster and connect them to your Git repository

Let's begin by setting up the Flux CLI on your local machine.

## 💻 Setting Up Your Control Tools – Installing the Flux CLI

Before we can deploy Flux to your cluster, you'll need the **Flux Command Line Interface (CLI)** on your local machine. This tool simplifies the process of installing Flux controllers into your Kubernetes cluster and connecting them to your Git repository.

The Flux CLI provides commands to:
- Deploy the Flux controllers to your cluster
- Create and manage GitRepository and Kustomization resources
- Check the status of your GitOps deployments
- Troubleshoot reconciliation issues

Let's install it now using the instructions for your operating system:

---

### 🍏 macOS Installation

**Via Homebrew (recommended):**

```bash
brew install fluxcd/tap/flux
```

---

### 🐧 Linux Installation

**Via universal install script:**

```bash
curl -s https://fluxcd.io/install.sh | sudo bash
```

> 📚 For other Linux installation options or shell-specific completions, see the [official Flux installation docs](https://fluxcd.io/docs/installation/).

---

### 🪟 Windows Installation

**Via Chocolatey:**

```powershell
choco install fluxcd
```

Alternatively, download the appropriate binary for your system from the [Flux GitHub Releases page](https://github.com/fluxcd/flux2/releases), and add it to your system's `PATH`.

> 💡 **Tip:** If the `flux` command isn't recognized after installation, restart your terminal or ensure the binary is in your system's `PATH`.

---

### ✅ Verify Your Installation

Run the following command to confirm the Flux CLI is installed correctly:

```bash
flux --version
```

You should see output similar to:

```
flux version 2.5.1
```

If you receive a "command not found" error or see an unexpected version, double-check your installation steps and ensure your PATH includes the installation directory.

With the Flux CLI successfully installed, you're now ready to deploy the Flux controllers to your Kubernetes cluster and start building your GitOps automation pipeline.

## 🔌 Deploying the GitOps Engine – Installing Flux in Your Cluster

With the Flux CLI installed, it's time to deploy the Flux controllers into your Kubernetes cluster. This step transforms your regular Kubernetes environment into a GitOps-powered, self-healing system.

---

### 🔍 Ensuring Your Cluster is Ready

Before installing anything, let's verify you're connected to the right cluster:

```bash
kubectl config current-context
```

You should see:

```
kind-gitops-loop-demo
```

Next, let's run a pre-flight check to ensure your environment is ready for Flux:

```bash
flux check --pre
```

This validates that your Kubernetes API is accessible and that you have the necessary permissions to install Flux.

---

### ⚙️ Installing the Flux Controllers

Once the pre-flight check passes, deploy the Flux controllers with:

```bash
flux install
```

This command:
- Creates the `flux-system` namespace
- Deploys the core controllers that power the GitOps loop
- Sets up the Custom Resource Definitions (CRDs) that Flux uses

You'll see output showing each component being installed. When complete, verify the installation:

```bash
kubectl get pods -n flux-system
```

You should see all controllers running:

```
NAME                                     READY   STATUS    RESTARTS   AGE
helm-controller-xxxx                     1/1     Running   0          10s
kustomize-controller-xxxx                1/1     Running   0          10s
notification-controller-xxxx             1/1     Running   0          10s
source-controller-xxxx                   1/1     Running   0          10s
```

> 💡 **What each controller does:**
> - **source-controller:** Manages Git repositories and other sources
> - **kustomize-controller:** Applies configurations and reconciles the cluster
> - **notification-controller:** Handles events and alerts (not used in this lab)
> - **helm-controller:** Manages Helm releases (not used in this lab)

---

### 🔗 Connecting Flux to Your Git Repository

Now that Flux is running, let's connect it to your GitHub repository. This creates the first half of the GitOps loop—telling Flux where to look for your configurations.

Create a `GitRepository` resource pointing to your forked repository:

```bash
flux create source git gitops-loop-demo \
  --url=https://github.com/YOUR-USERNAME/GitOps-Days.git \
  --branch=main \
  --interval=30s
```

> 📝 **Remember to replace** `YOUR-USERNAME` with your actual GitHub username!

This command:
- Creates a `GitRepository` resource named `gitops-loop-demo` 
- Points it to your forked repository
- Sets it to check for changes every 30 seconds

Verify that Flux has successfully connected to your repository:

```bash
flux get sources git
```

You should see:

```
NAME                READY   STATUS    AGE
gitops-loop-demo    True    Fetched   30s
```

---

### 📋 Defining What Flux Should Apply

The final step is telling Flux which parts of your repository to apply to the cluster. Create a `Kustomization` resource pointing to your application folder:

```bash
flux create kustomization hello-app \
  --source=GitRepository/gitops-loop-demo \
  --path="./examples/day2-gitops-loop-demo/clusters/local/apps/hello" \
  --prune=true \
  --interval=1m
```

> 🪟 **Windows users:** Replace the backslashes with backticks (`) for PowerShell.

This command:
- Creates a `Kustomization` named `hello-app`
- Links it to your `GitRepository` resource
- Targets the specific folder containing your application manifests
- Enables pruning (resources deleted from Git will be removed from the cluster)
- Sets reconciliation to run every minute

Once created, Flux will immediately begin reconciling your cluster to match the state defined in your Git repository.

---

### ✅ Verifying the GitOps Loop

Check that your `Kustomization` is working:

```bash
flux get kustomizations
```

You should see:

```
NAME         READY   STATUS    AGE
hello-app    True    Applied   1m
```

The status "Applied" confirms that Flux has successfully synchronized your cluster with your Git repository. **Your GitOps loop is now complete!**

Next, we'll verify that your application is running and test the self-healing capabilities of your new GitOps system.

## 🔎 Witnessing GitOps in Action – Verifying Your Automated Deployment

You've done all the setup work:
- Created your Kubernetes cluster
- Installed Flux controllers
- Connected Flux to your Git repository
- Defined which folder to apply

Now comes the exciting part—seeing it all work together! Unlike traditional deployments where you run `kubectl apply` manually, Flux has **already applied** your manifests automatically. Let's verify what happened behind the scenes.

---

### ✅ Confirming the Reconciliation

First, check that Flux successfully reconciled your cluster with Git:

```bash
flux get kustomizations
```

You should see:

```
NAME         READY   STATUS    AGE
hello-app    True    Applied   1m
```

The key word here is "**Applied**"—this confirms that Flux detected your manifests in Git and applied them to the cluster. Without you typing a single `kubectl apply` command, your application is already running!

---

### 👀 Checking Your Deployed Resources

Let's verify that the resources defined in your manifest files now exist in the cluster:

```bash
kubectl get pods,svc
```

You should see output similar to:

```
NAME                         READY   STATUS    RESTARTS   AGE
pod/hello-65d4c4d5c9-xz7vp   1/1     Running   0          1m

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/hello        ClusterIP   10.96.x.x       <none>        80/TCP    1m
service/kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP   30m
```

This confirms that:
- The Deployment created a running Pod (with a randomly generated name)
- The Service is exposing your application on port 80

> 💡 **This is GitOps in action!** The state of your cluster now matches what's defined in Git—all handled automatically by Flux.

---

### 🌐 Accessing Your Application

To see your application in action, you'll need to forward a local port to your Service:

```bash
kubectl port-forward svc/hello 8080:80
```

> 🔍 **What this does:** This command creates a tunnel from port 8080 on your local machine to port 80 on the Service inside the cluster.

With the port forwarding active, open [http://localhost:8080](http://localhost:8080) in your browser. You should see:

> **Welcome to nginx!**

This confirms your application is running correctly! Keep the port-forwarding terminal window open if you want to continue accessing the app.

---

### 🧠 Understanding What Just Happened

Let's appreciate what occurred here:

1. You never ran `kubectl apply` on your manifests
2. You never manually created any Deployments or Services
3. You only defined a Git repository and told Flux which folder to watch

Flux did the rest—pulling your manifests from Git and applying them automatically. This is the core power of GitOps: **your Git repository driving your cluster state**.

Now that we've verified everything is working correctly, let's put Flux's self-healing capabilities to the test. In the next section, we'll intentionally "break" our cluster and watch how Flux automatically restores it to match the state defined in Git.

## 🛠️ Testing Resilience – Breaking and Watching Automatic Recovery

Your GitOps loop is running smoothly—your application is deployed and Flux is keeping everything in sync with Git. Now let's put Flux's self-healing capabilities to the test by intentionally creating "drift" between your cluster state and your Git repository.

This exercise demonstrates one of GitOps' most powerful features: **automatic recovery from unexpected changes**—a capability that becomes invaluable in production environments where accidental modifications or failures are inevitable.

---

### 🧪 Test 1: Manually Scaling the Deployment to Zero

Your Git repository declares `replicas: 1` for the `hello` deployment, ensuring one instance of your application is always running. Let's override this manually to simulate an operational mistake:

```bash
kubectl scale deployment hello --replicas=0
```

This command removes all running instances of your application—it's now completely unavailable. Verify this by checking the deployment:

```bash
kubectl get deployment hello -w
```

> 📋 The `-w` flag "watches" the resource, showing updates in real-time. Press Ctrl+C when you're done watching.

Initially, you'll see:

```
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
hello   0/0     0            0           5m
```

Your application is down! But within about a minute, you'll see something interesting happen:

```
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
hello   0/0     0            0           5m
hello   0/1     0            0           5m30s
hello   0/1     1            0           5m30s
hello   1/1     1            1           5m45s
```

**What happened?** Flux detected that the actual state (0 replicas) differed from the desired state in Git (1 replica), and automatically restored the deployment to match Git. No manual intervention required!

---

### 🧪 Test 2: Completely Deleting the Application

Let's try something more drastic—deleting the entire namespace containing our application:

```bash
kubectl delete namespace hello
```

This removes everything: the Deployment, Service, Pod, and even the namespace itself. Your application has been completely erased from the cluster.

To watch Flux rebuild everything from scratch, monitor the cluster events:

```bash
# Wait a moment for recreation to begin, then run:
kubectl get events --watch
```

Within about a minute, you'll see a series of events as Flux reconstructs your application environment:

1. The namespace is recreated
2. The Deployment is created in the new namespace
3. The ReplicaSet is created
4. A new Pod is scheduled and started
5. The Service is recreated

When complete, verify everything is back by running:

```bash
kubectl get deployment,service,pod
```

All your resources should be present again, just as they were before deletion.

---

### 💡 The Power of Declarative Recovery

What you've just witnessed is fundamentally different from traditional recovery methods:

| Traditional Operations | GitOps Recovery |
|------------------------|----------------|
| Manual intervention required | Completely automatic |
| Recovery scripts needed | Git is the recovery system |
| Human errors during recovery | Consistent, repeatable process |
| Recovery time varies | Predictable recovery time |

This automatic recovery works because:
1. Git remains the source of truth
2. Flux continuously reconciles actual state with desired state
3. When drift is detected, reconciliation restores the intended configuration

In production environments, this capability becomes invaluable—protecting against accidental changes, failed updates, or even complete service outages.

---

### 🎯 What You've Accomplished

In these tests, you've seen how Flux maintains your application state against two types of disruptions:
- Manual overrides (scaling to zero)
- Complete deletion (removing the namespace)

In both cases, Flux detected the drift and automatically restored your application to match the state declared in Git—without any action on your part.

Next, we'll reflect on what you've learned and discuss how these GitOps patterns scale to real-world environments.

## 🏆 Mission Accomplished – Your GitOps Journey So Far

Congratulations! You've successfully implemented a complete GitOps workflow with Flux. Take a moment to appreciate what you've built—it's no small achievement.

In just a few hours, you've transformed a standard Kubernetes cluster into a self-healing, Git-driven system that automatically maintains its desired state. This represents a fundamental shift in how infrastructure is managed:

| Traditional Kubernetes | Your New GitOps Approach |
|------------------------|--------------------------|
| Manual `kubectl apply` commands | Git commits drive deployments |
| Manual intervention for drift | Automatic reconciliation |
| Imperative operations | Declarative configurations |
| No audit trail for changes | Full history in Git |
| Recovery requires human action | Self-healing built in |

---

### 🔑 Key Skills You've Acquired

Through this hands-on lab, you've gained practical experience with:

1. **Declarative Configuration** - Defining your entire application stack in YAML
2. **GitOps Automation** - Using Flux to implement continuous reconciliation
3. **Self-Healing Infrastructure** - Testing resilience against manual changes
4. **Repository Structure** - Organizing manifests in a scalable pattern

These skills form the foundation of modern Kubernetes operations and are directly applicable to production environments.

---

### 💼 Taking GitOps to Production

What you've built today is not just a learning exercise—it's a miniature version of production-grade GitOps. Organizations around the world are using these exact patterns to manage critical infrastructure:

- **Multiple environments** (dev/test/prod) managed through folder structure
- **Automated deployments** triggered by Git commits
- **Recovery capabilities** that minimize downtime
- **Audit trails** that track every change through Git history

The lab you've completed follows the same principles used in large-scale deployments, just in a simplified form.

---

### 🔭 Where We're Heading Next

In Day 3, we'll build on this foundation by taking GitOps to the cloud:

- Deploying your GitOps workflow on Azure Kubernetes Service (AKS)
- Implementing GitOps in a production-like cloud environment
- Adapting the same patterns for larger-scale deployments
- Adding more sophisticated components to the GitOps pipeline

The best part? You'll use the same core concepts and workflows you've mastered today. The manifests will remain similar—you'll just be deploying them to a cloud-native environment.

---

### 🚀 Before You Go

If you want to experiment further with your local GitOps setup:

1. Try changing the `replicas` value in your Git repository and watch Flux automatically scale your application
2. Add a new Kubernetes resource (like a ConfigMap) to your manifests and commit it
3. Explore the Flux CLI's other commands with `flux --help`

When you're ready to clean up:

```bash
kind delete cluster --name gitops-loop-demo
```

**Day 2 complete!** You've built a working GitOps system and seen firsthand how it creates self-healing infrastructure. See you in Day 3!