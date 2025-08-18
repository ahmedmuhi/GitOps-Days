# 🚀 Day 3 – GitOps in the Cloud with AKS: Same Magic, Bigger Stage

In [GitOps Day 2](./Day-2-Building-Your-First-Self-Healing-System.md) you built something remarkable on your laptop:
a Kubernetes system that was truely **resilient and self-healing**. Flux kept every piece in its proper place, automatically repairing whatever chaos you threw at it - scaling changes, deletions, drift. What you saw wasn't just a neat demo; it was the core promise of GitOps delivered: **the cluster always returns to its declared state.**

Now let's ask a bigger question: *what if that same architectural elegance, that precise reliability, translated identically to the cloud - no matter how sprawling or complex the environment?*

That's exactly what we'll prove today.

Welcome back to GitOps Days, where **Git is not just for code anymore**. It's the single source of truth for everything - infrastructure, application configuration, and the glue that holds them together.
Yesterday, Git defined your laptop cluster. Today, Git defines a **production-grade Azure Kubernetes Service (AKS) cluster** in the cloud.

Here's the surprising insight: GitOps doesn't fundamentally care where it runs. Local of cloud, small or enterprise-scale, the rules don't change. Same Git. Same Flux. Same self-healing loop. Only the stage beneath it grows larger.

Today we'll take those principles into the real world:

* Provision an AKS cluster (using Azure's free credits and low-cost resources for this tutorial).
* Bootstrap Flux the production way - one command, end-to-end.
* Deploy the very same Hello app you ran yesterday, now Internet accessible with a single YAML tweak.
* Break things on prupose, and watch Flux restore order - even Azure load balancer and public IPs.

By the end, you'll see your local success story scale seamlessly into the cloud. The GitOps loop remains unchanged; the only difference is the size of the playground.

## ☁️ Preparing Your Cloud Workspace

Before we build your first AKS cluster, let's pause and get the **essentials** in place. Moving from your laptop to Azure isn't a huge leap - the setup looks almost the same, just one extra wait while Azure spins things up.

But the key difference: this tutorial won't be completely free.
But don't worry - it's very inexpensive. Running a one-node AKS cluster with a load balancer costs **about $1 if you leave it up for a full workday**. If you shut it down after this tutorial, you'll spend less than the price of a coffee. And if you're new to Azure, Microsoft gives you **$200 in free credits** when you sign up.

### 💰 Cost Snapshot

* **Control plane**: free on the *Free* tier
* **Node (Standard_B2s)**: ~$0.04-$0.05 per hour
* **OS disk + Load Balancer + IP**: ~$0.04 per hour
* **Total**: ~$0.08-$0.09 per hour (≈$0.70 for 8 hours)
* **Cleanup**: We'll delete everything at the end so there are no ongoing costs

### ✅ What You'll Need

1. **Azure account**

   * Sign up at [azure.microsoft.com/free](https://azure.microsoft.com/free) if you don't have one
   * Comes with **$200 in credits**

2. **Azure CLI** (version 2.76.0 or later)

   ```bash
   # Check your version
   az --version

   # Install/update if needed:
   # Windows: winget install Microsoft.AzureCLI
   # macOS: brew install azure-cli
   # Linux: curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
   ```
  
3. **Tools from Day2**

   * `kubectl` - for cluster verification
   * `flux` CLI - for bootstrapping GitOps
   * `git` - your single source of truth

### 🔑 Logging Into Azure

1. Open your terminal and run:

   ```bash
   az loging
   ```

   * A browser window will open for you to authenticate with your Azure account.
   * Once signed in, the CLI retrieves your tenants and subscriptions.

2. If you have access to more than one Azure Subscription, you'll see a prompt to **select which subscription** you want to use. The output would look like this:

   ```
   [Tenant and subscription selection]

   No   Subscription name                     Subscription ID                       Tenant
   ---  ------------------------------------  ------------------------------------  -----------------
   [1]  Pay-As-You-Go Dev/Test                xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  Default Directory
   [2]* Visual Studio Enterprise Subscrip...  xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  Default Directory

   The default is marked with an *.
   Select a subscription and tenant (Type a number or Enter for no changes):
   ```

   Pick the subscription where you're comfortable creating temporary test resources.

3. Double-check that the correct subscription is active:

   ```bash
   az account show --output table
   ```

### 📦 A Quick Word on Kubernetes Types

Yesterday, with kind (Kubernetes in Docker), you ran a **local simulation of Kubernetes**. It mimics the control plane and nodes inside containers, perfect for quick experiments.

Today with AKS, Microsoft porvides a **conformant Kubernetes cluster** - meaning it passes the CNCF's official tests and behaves like upstream Kubernetes. The manifests you wrote yesterday will run here too, but now on a real cloud infrastructure managed by Azure.

✅ That's it. You've set up your workspace. logged into Azure. and know what costs to expect.
👉 In the next section, we'll actually **create your AKS cluster** and see your GitOps loop come alive in the cloud.

## ⚙️ Create Your AKS Cluster

With your cloud workspace ready, it's time for the fun part: creating your first **Azure Kubernetes Service (AKS)** cluster.

This is the same GitOps loop you've built yesterday, just running on **real, cloud-hosted infrastructure**. The steps look familiar: we'll create a logical container (called a *resource group*) to hold your cluster, then provision the cluster itself. Once it's up, you'll connect `kubectl` to it so you can talk to it from your laptop.

⏱️ **Time neded:** about 7-10 minutes (most of it waiting for Azure to spin up the nodes).

### Pick a Region

Choose a region that is close to you to minimize latency. For me, that's **Australia East (Sydney)**, but you can use whatever region works best.

Example regions often used for learning:

* **Australia East** (Sydney) - good from New Zealand/Australia
* **East US 2** or **West US 2** - good for the Americas
* **West Europe** - good for Europe

Well set a shell variable for the location for consistency:

```bash
LOCATION="australiaeast"
```

### Create a Resource Group

A **resource group** in Azure is just a logical container to keep related resource together (cluster, disks, load balancer).

```bash
RESOURCE_GROUP="gitops-prod-rg"

az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION
```

### Create Your AKS Cluster

Now let's create a small AKS Cluster - one node is enough for this tutorial.

```bash
az aks create \
  --resource-group $RESOURCE_GROUP \
  --name gitops-prod-aks \
  --location $LOCATION \
  --kubernetes-version 1.33 \
  --node-count 1 \
  --node-vm-size Standard_B2s \
  --tier free \
  --enable-managed-identity \
  --generate-ssh-keys
```

**What this does.**

* Spins up an AKS cluster with **Kubernetes 1.33** (latest long-term supported version)
* Uses **1x Standard_B2s VM** for your node (lowest cost, enough for learning)
* Runs in the **Free tier** (no control-plane cost; only the node, disk, and LB/IP billed)
* Generates SSH keys automatically so Azure can access the node if needed

> [!TIP]
> **Cost reminder:** At this size, running the cluster for 8 hours costs about **$0.70**. We'll delete it at the end, to avoid ongoing cost.

### Connect to Your Cluster

Once creation completes (≈7-10 minutes), download the credentials so `kubectl` can talk to your cluster:

```bash
az aks get-credentials \
  --resource-group $RESOURCE_GROUP \
  --name gitops-prod-aks \
  --overwrite-existing
```

This merges the cluster context into your `~/.kube/config` file. From now on, `kubectl` commands will talk to AKS instead of your local kind cluster.

### Verify Your Cluster

Check that your cluster is ready:

```bash
kubectl get nodes
```

You should see one node with `STATUS = Ready`. For example:

```bash
NAME                                STATUS   ROLES   AGE   VERSION
aks-nodepool1-12345678-vmss000000   Ready    <none>  2m    v1.33.2
```

✅ Congratulations - you now have a real, conformant Kubernetes cluster running in Azure!

👉 Before we install Flux, let's first prepare your **student workspace** so you have a safe place in your fork to work with. This ensures your changes won't be overwritten if the the upstream examples are updated.

## 📂 Preparing Your Student Worspace

Your AKS cluster is live - now it's time to set up the **Git side** of your GitOps loop. The step makes sure you're working in a safe copy of the repo that belongs to you, so nothing you do later gets lost when the examples are updated.

### 1️⃣ Check and Sync Your Fork

All your changes will live in *your fork* of the repository. Before starting Day 3:

* Open your fork on GitHub (`https://github.com/<your-username>/GitOps-Days`)
* If GitHub shows a banner that says **“This branch is behind ...”**, click **Sync fork → Update branch**
* If it says you're already up to date, no action is needed

👉 This ensures your fork contains the latest Day 3 examples before you clone it locally.

### 2️⃣ Clone Your Fork Locally

Always clone **your fork's URL**, not the original repo:

```bash
git clone https://github.com/<your-username>/GitOps-Days.git
cd GitOps-Days
```

This guarantees your local Git repo `origin` points to your fork (the one Flux will use later).

### 3️⃣ Copy Day 3 Examples Into Your Workspace

Never work directly in the sahred `examples/` folder - it may change in future updates. Instead, create your personal workspace:

```bash
mkdir -p student-work/<your-username>/day3
cp -r examples/day3/clusters student-work/<your-username>/day3/
```

You now have:

```
student-work/<your-username>/day3/clusters/aks/apps/hello
├── deployment.yaml
├── namespace.yaml
├── service.yaml
└── kustomization.yaml
```

This is **your safe sandbox**. Any changes you make will live here, untouched by upstream syncs.

### 4️⃣ Set Up GitHub Credentials

Flux will commit its own config into your fork when we set it up later, so it needs your GitHub username and a personal access token (PAT).

#### 1. Create a PAT i GitHub

* Go to **Settings → Developer settings → Personal access tokens**
* Choose **Fine-grained token** (recommended)
* Scope it to **your fork**, with at least `contents: read/write`
* Give it a friendly **name/label** like `gitops-days` so you can recoginize it later in your GitHub account.

> [!IMPORTANT]
> GitHub will show you a **long random secret string** only once when the token is created (it looks like `ghp_abCdEf1234567890XYZ...`).
> Copy this secret immediately - this is the actual token you'll provide to Flux later.
> The name/label you gave the token is just for your GitHub dashboard; Flux never sees that.

#### 2. Export Your Credentials

Paste your GitHub username and the **secret token string** into your shell as environment variables:

```bash
# macOS/Linux
export GITHUB_USER=<your-username>
export GITHUB_TOKEN=<your-token-from-earlier>

# Windows PowerShell
$env:GITHUB_USER="<your-username>"
$env:GITHUB_TOKEN="<your-token-from-earlier>"
```

Later we'll tell `flux` to pick these up automatically when it talks to GitHub.
You don't need to pass `--token` on every command.

✅ At this point you have:

* A GitHub fork up to date with Day 3
* A safe student workspace folder in your fork
* Credentials exported so Flux can authenticate with GitHub

👉 Next, we'll install Flux into the cluster using the **bootstrap method** and connect it to your Git repository. That's where the GitOps magic comes alive at cloud scale.

## 🤖 Installing Flux on Your Cloud Custer

**Yesterday you installed Flux piece by piece. Today, you'll do it the production way - with one command that does everything.**

On Day 2 you ran `flux install`, then create a `GitRepository`, then a `Kustomization`. That was great for learning the moving parts.
In production, though, teams want one clean step. That's what `flux bootstrap` gives you - and that's what you'll use now.

### 1️⃣ Run the Bootstrap Command

With your AKS cluster ready and your student workspace set up, install Flux into the cluster and connect it to your fork:

```bash
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=GitOps-Days \
  --branch=main \
  --path=student-work/$GITHUB_USER/day3/clusters/aks \
  --personal
```

* `--owner` → your GitHub username (the one you exported as `$GITHUB_USER`)
* `--repository` → the name of your forked repo (`GitOps-Days`)
* `--branch` → the branch Flux will track (`main` in our case)
* `--path` → the folder inside your repo that Flux will watch (`student-work/your-username/day3/clusters/aks`)
* `--personal` → tells Flux this is your personal fork, not an org repo

This one command does all the heavy lifting: it installs Flux controllers in your cluster `and` pushes Flux's own configuration into your fork, that's why we provided Flux with your GitHub username/token to allow it to commit to your repo.

⏱️ **How long will this take?** Usually 2-3 minutes. You'll know it's working when you see new commits appear in your fork under `student-work/your-username/day3/clusters/aks/flux-system`.

### 2️⃣ Verify Flux is Healthy

Once the bootstrap finishes, check that everything came up cleanly:

```bash
flux check
flux get sources git
flux get kustomizations
```

You should see:

* Controllers in the `flux-system` namespace are **ready**
* A `GitRepository/flux-system` pointing at your fork
* A `Kustomization/flux-system` applying your root folder

If you see all green checks ✅, congratualtions - Flux is now installed in AKS, connected to your fork, and running GitOps.

That's it. Flux is now alive and watching your repo.

👉 Next, we'll slow down and unpack *what actually happened* during bootstrap - the files Flux created, the `flux-system` namespace, an how Flux now manages itself.

## 🧩 Unpacking Flux's Bootstrap

You just ran `flux bootstrap` and confirmed Flux is alive in your AKS cluster. From the outside it look like one simple step - but behind the scenes, Flux quitly set up quite a bit of scaffolding for you.
Let's slow down and see what actually happened.

### How your repo changed

**Before bootstrap**, your Day 3 workspace was simple - it only contained your Hello app:

```
student-work/<your-username>/day3/clusters/aks/
└── apps/
    └── hello/
        ├── deployment.yaml
        ├── namespace.yaml
        ├── service.yaml
        └── kustomization.yaml
```

**After bootstrap** Flux committed new files into your fork so it could manage itself:

```
student-work/<your-username>/day3/clusters/aks/
├── kustomization.yaml            # Root recipe (entry point for Flux)
├── flux-system/                  # Flux's own configuration
│   ├── gotk-components.yaml
│   ├── gotk-sync.yaml
│   └── kustomization.yaml        # Flux system recipe
└── apps/
    └── hello/
        ├── deployment.yaml
        ├── namespace.yaml
        ├── service.yaml
        └── kustomization.yaml    # Hello app Recipe 
```

#### What This Means

* The **new `flux-system` folder** is Flux adding its own configuration to Git.
* The files inside tell Fulx *what to install* and *where to look*.
* You don't need to edit these files right now - we'll make a small, safe change soon to connect you app.

For now, just remeber: **Flux manages your cluster by following recipes (`kustomization.yaml` files) it finds in your repo.**

👉 But there's one detail that trips most people up at first: *why are there so many `kustomization.yaml` files now, and what's the difference between them?*
We'll clear that up in the next section before you make your first change.

## ❓ Why Flux Has So Many `kustomization.yaml` Files?

If you looked closely at your repo after bootstrap, you may have noticed something surprising: suddenly there are **three different `kustomization.yaml` files.**

They look almost identical at first glance, but each one play a **different role**. Understanding this now ill make the next step - deploying your Hello app much clearer.

### 1️⃣ Root recipe (top level)

* **Where:** `student-work/<your-username>/day3/cluster/aks/kustomization.yaml`
* **Role:** The **entry point** for Flux in this repo.
* Think of it as the master to-do list: “apply this folder, then that folder.”
* Right now its only asking Flux to apply `./flux-system`. Soon, you'll `./apps/hello`.

### 2️⃣ Flux system recipe

* **Where:** `student-work/<your-username>/day3/clusters/aks/flux-system/kustomization.yaml`
* **Role:** Applies Flux's own building blocks.

  * `gotk-components.yaml` → installs the Flux controllers and CRDs.
  * `gotk-sync.yaml` → wires Flux to your repo (GitRepository + Kustomization objects).
* This is how Flux keeps itself running.

### 3️⃣ Hellp app recipe

* **Where:** `student-work/<your-username>/day3/clusters/aks/apps/hello/kustomization.yaml`
* **Role:** Bundles your app's resources together.
* It includes:

  * `namespace.yaml` (the Hello namespace)
  * `deployment.yaml` (the pods)
  * `service.yaml` (the LoadBalancer on AKS)
* When Flux sees this file, it will apply all three resources as one unit.

### ✨ How they connect

* The **root recipe** tells Flux to apply what's inside `./flux-system` folder.
* The **flux-system recipe** tells Flux to apply its own configuration YAML files so it can run.
* Sonn, you'll add `./apps/hello` to the **root recipe**. When you do, Flux will open the Hello app folder, finds the **hello app recipe**, and applies your app's resources as per the recipe.

So the structure is **layered, not duplicated**:

* Root → flux-system → Flux controllers
* Root → hello → your app resources

👉 With that clear, you're ready for the fun part: edit the root recipe to add your Hello app and watch Flux deploy it automatically. That's next.

## ☁️ Your First GitOps Cloud Deployment

**Time to deploy to the cloud using pure GitOps. No `kubectl apply`. Just Git.**

Remember that Hello app from Day 2 - the one that healed itself when Flux noticed drift? You're about to deploy the exact same app to AKS. The only difference is that now it will be reachable on the internet through a real Azure Load Balancer.

### 1️⃣ Verify Your App Folder

You should alreay have the Hello app folder in your repo (created for you during prep):

```
student-work/<your-username>/day3/clusters/aks/apps/hello/
├── deployment.yaml
├── namespace.yaml
├── service.yaml
└── kustomization.yaml
```

This folder contains everything Flux needs to deploy the app, including the `kustomization.yaml` file that lists the three resources above so they are applied together. 

If you don't see this folder go back to the prep step, copy it again from the examples folder or recreate it from your GitHub fork.

### 2️⃣ Edit the Root Recipe

Now we'll tell Flux to include your Hello app in its reconciliation loop.

Open the root `kustomization.yaml`

```
student-work/<your-username>/day3/clusters/aks/kustomization.yaml
```

Add the Hello app folder under `resources:`:

```yaml
apiversion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ./flux-system
  - ./apps/hello       # ← add this line
```

This is the single line that “wires” your hello app into the GitOps loop.

### 3️⃣ Commit and Push

Save you change, then commit and push it to your fork:

```bash
git add .
git commit -m "Add Hello app to root recipe"
git push origin main
```

From now on, Flux will notice this change in Git and apply the Hello app to your cluster automatically.

### 4️⃣ Watch Flux Reconcile

Within about a minute, Flux will detect the commit, pull it down, and reconcile.
You can watch it happen live:

```bash
flux logs --follow --tail 20
```

You'll see messages about GitRepository updating and the root Kustomization applying.

Press `Ctrl+c` once you see the reconciliation complete.

### 5️⃣ Verify Hello App Resources in AKS

Chech that the Hello app is now up and running in AKS:

```bash
kubectl get pods -n hello
kubectl get svc -n hello
```

You'll should see something like:

```
NAME    TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)   AGE
hello   LoadBalancer   10.0.123.45   20.248.xxx.xxx   80/TCP    2m
```

* The `Pod` is running inside the `hello` namespace.
* The `Service` is of type `LoadBalancer`. AKS will provision an Azure Load Balancer and assign it a public IP address.

>[!TIP]
> ⏳ If `EEXTERNAL_IP` shows `<pending>`, wait 30-60 seconds and try again. Azure is creating the load balancer.

### 6️⃣ Access Your App in the Browser

Once the external IP is assigned, get the URL:

```bash
echo "http://$(kubectl get svc hello -n hello -o jsonpath='{.status.loadBalancer.ingress[0].ip}')"
```

Copy the printed URL into your browser.

🎉 **Boom!** Your Hello World app is live on the internet, deployed entirely through GitOps.

### 7️⃣ What Just Happened

Let's recap:

1. You edited Git - not the cluster.
2. Flux noticed the change and reconciled the root recipe.
3. Flux read the Hello app recipe and applied the app resources into the `hello` namespace.
4. AKS provisioned a real Azure Load Balancer with a public IP.
5. You app became internet accessible.

No manual `kubectl apply`. Just Git → Flux → Cloud.

**Same GitOps loop as Day 2 - just running at cloud scale!**

👉 Next, we'll **update the Hello app using GitOps** by changing the replica count in Git and watch Flux reconcile the cluster to match Git.

## 🔄 Updating Your App with GitOps

You've just deployed your Hello app to AKS using Flux. That's powerful on its own - but the real beauty of GitOps is how **every change flows the same way**: edit Git → comit → push → Flux reconciles. Let's try that now.

### 1️⃣ Make a Change in Git

Right now your app is running with a single replica:

```yaml
# student-work/<your-username./day3/clusters/aks/apps/hello/deployment.yaml
spec:
  replicas: 1
```

Open this file and change the replicas from **1** to **3**:

```yaml
spec:
  replicas: 3
```

This tells Kubernetes (via Flux) that you want three pods running instead of one.

### 2️⃣ Commit and Push

Save the change and commit it to your fork:

```bash
git add student-work/<your-username>/day3/clusters/aks/apps/hello/deployment.yaml
git commit -m "Scale Hello app to 3 replicas"
git push origin main
```

### 3️⃣ Watch Flux Reconcile

Flux checks for new commits about every minute. Once it sees your change, it will reconcile the cluster to match Git.

You can watch the rollout live:

```bash
kubectl get deployment hello -n hello -w
```

You'll see the replica count go from **1 → 3**:

```
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
hello   1/1     1            1           5m
hello   2/3     2            2           5m
hello   3/3     3            3           5m
```

Press `Ctrl+c` to stop watching once all 3 replicas are up.

### 4️⃣ What Just Happened

You changed a single line in Git (`replicas: 1 → 3`).
* Flux noticed the new commit and pulled it.
* Flux reconciled your cluster so the deployment matched what Git declared.
* Kubernetes spun up two more pods until the cluster had 3 replicas running.

✅ That's GitOps in action: deployments, updates, and changes all flow through the same loop.

👉 Next, we'll push this one step further: what happens if we “break” things manually in the cluster?
You'll see Flux heal them back into shape automatically.

## 💥 Cloud-Scale Self-Healing

**Now that you've deployed and updated your Hello app the GitOps way, let's see what happens when things drift in the cluster without a Git commit.**

This is where GitOps really shines - Flux continuously reconciles the cluster against what's in Git, reparing anythin that drifts.

>[!NOTE]
>To make the demo feedback fast, we'll first shorten Flux's reconcile internal.

### 1️⃣ Speed Up Self-Healing

By default, Flux reconciles every **10 minutes**. For this demo, we'll set it to **1 minute** so you don't have to wait long.

Open the file:

```
student-work/<your-username>/day3/clusters/aks/flux-system/gotk-sync.yaml
```

Find the `Kustomization` spec and change:

```yaml
spec:
  interval: 1m0s   # ← Change from 10m0s
```

Commit and Push:

```bash
git add student-work/<your-username>/day3/aks/flux-system/gotk-sync.yaml
git commit -m "Speed up Flux reconciliation to 1 minute"
git push origin main
```

Flux now reconciles once a minute.

### 2️⃣ Test 1: Manual Scale Up

Imagine a teammate scaled the deployment directly in the cluster to cope with demand, but forgot to commit the change to Git. Flux will notice and correct it.

```bash
# Scale to 5 replicas manually (bypassing GitOps)
kubectl scale deployment hello -n hello --replicas=5

# Watch the deployment in real-time
kubectl get deployment hello -n hello -w
```

You should see something like this:

```
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
hello   3/3     3            3           12m
hello   5/5     5            5           12m30s   # Manual change applied
hello   3/3     3            3           13m      # Flux detected a drift reconciled it back to Git state
```

Press `Ctrl+c` to stop watching.
✅ Git's declared state (3 replicas) wins. Manual changes don't stick.

### 3️⃣ Test 2: The Nuclear Option

Let's go further: delete the entire namespace. This wipes out the app, service, load balancer, and public IP.

```bash
kubectl delete namespace hello
```

Now watch Flux rebuild it:

```bash
# Watch until the namespace reappears
kubectl get ns -w
```

Within **1-3 minutes** you'll see:

* Namespace recreated
* Deployment and pods spun up
* Service re-created
* Azure provisioned a new load balancer and public IP

Press `Ctrl+c` once everything is back.

### 4️⃣ Verify You App is Back

Get the new public IP:

```bash
kubectl get svc hello -n hello
```

Print the new URL:

```bash
echo "http://$(kubectl get svc hello -n hello -o jsonpath='{.status.loadBalancer.ingress[0].ip}')"
```

Copy the URL and open it in your browser. 🎉 Your app is back online again! Now with a fresh public IP.

>[!NOTE]
>Because Azure created a new Load Balancer, the external IP may change each time the service is re-created.

### 5️⃣ Check Out Flux's Records

See what Flux recorded during this healing:

```bash
flux events -n flux-system --for Kustomization/flux-system
```

You'll see entries like:

```
Reconciliation finished in 1.3s, next run in 1m0s
Applied 3 resources
Namespace hello created
Deployment hello created
Service hello created
```

### 6️⃣ What We LEarned

* **Drift is temporary** - Flux brings the cluster back to what is declared in Git 
* **Manual changes don't persist** - Git is always the source truth
* **Deletions are reparied** - automation prevails.

What we saw on your laptop in Day 2 now holds true at cloud scale. Flux healed not just pods, but also Azure infrastructure like load balancers and public IPs.

🎉 **Congratulations!** You've now seen GitOps enforce and heal workloads in the cloud, end-to-end.

👉 Next, let's clean up your resources and wrap up Day 3.

## 🧹 Cleanup & Next Steps

**Before you log off, let's make sure you delete everything you created today.**
This step is important because cloud resources (like load balancers and node VMs) can continue incurring costs if left running.

### Delete the Resource Group

All your AKS resources were created inside a single Azure resource group (`gitops-prod-rg`). Deleting the group removes **everything inside it** in one go.

```bash
# Delete the entire resource group (and all resources it contains)
az group delete --name gitops-prod-rg --yes --no-wait
```

This will remove:

* Your AKS cluster
* All worker nodes
* Any load balancers
* Public IP addresses
* Disks, NIC, and other linked resources

The `--no-wait` flag tells Azure to start deletion in the background so you don't have to sit around watching it.

### C;ean Up Local Kubeconfig (Optional)

Azure resources are gone, but your kubeconfig may still have a ontext pointing to the deleted cluster.
You can safely remove it:

```bash
kubectl config delete-context gitops-prod-aks || true
kubectl config delete-cluster gitops-prod-aks || true
```

This prevents confusion later if you create a new cluster with the same name.

### Double-Check Azure Resources are Deleted

If you want to confirm the resource group is gone:

```bash
az group list --output table
```

You shouldn't see `gitops-prod-rg` anymore.

✅ That's it! Your cloud environment is fully cleaned up and won't incure any ongoing costs.

👉 Next, we'll **wrap-up** to reflect on what you achieved in Day 3, and then a **Day 4 preview** to tee up production GitOps patterns.

## 🎯 Day 3 Wrap-Up

**Take a step back, you just ran GitOps at cloud scale!**

### What You Did Today

* ✅ **Provisioned AKS** with a single `az` command
* ✅ **Bootsrapped Flux** into the cluster with one command
* ✅ **Understood** how flux wires itself into Git (root + flux-system + app recipes)
* ✅ **Deployed** your Hello app to the internet using GitOps
* ✅ **Updated** your app by changing replicas in Git (no `kubectl apply`)
* ✅ **Stress tested drift** by scaling manually and deleting the namespace, Flux healed everything
* ✅ **Cleaned up** your resources safely

That's huge amount of ground to cover in one day.

### Why It Matters

You proved that:

* GitOps is **infrastructure agnostic**: same workflow, different clusters
* Cloud infrastructure is no different from local, Flux reconciles both
* Self-healing is real, not just a demo trick: even Azure load balancers came back automatically
* Git is now your **single source of truth** for both apps and cluster config

This is the promise of GitOps: consistent, reliable, drift-free deployments, no matter the environment.

👉 Next, in **Day 4**, we'll take the leap from “it works” to **production-ready patterns**: multiple environments, image automation, and secrets. That's when you'll see how real-world teams apply these principles at scale.

## 🚀 Coming Up Next (Day 4)

Tmorrow, you'll see how to take you GitOps workflow from **capable** to **production-ready**:

* **Multi-environment GitOps**
  Manage dev, staging, and production clusters cleanly from Git

* **Image automation with GitHub Actions**
  Build and push new app images automatically, with GitOps handling deployment

* **Azure Container Registry (ACR)**
  Store your container images securely and integrate them into your GitOps loop

* **Secret management**
  Handle sensitive data (like API keys) safely in a GitOps workflow

🎉 Congratulations again on completing Day 3! Take a well-desrved break, you've earned it.

See you in **Day 4**, where we'll add the patterns that make GitOps ready for real-world production.