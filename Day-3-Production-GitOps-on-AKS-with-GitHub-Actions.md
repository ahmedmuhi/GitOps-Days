# 🚀 Day 3 – GitOps in the Cloud with AKS: Same Magic, Bigger Stage

Yesterday you built something remarkable on your laptop:
a Kubernetes system that was truely **resilient and self-healing**.
Flux kept every piece in its proper place, automatically repairing whatever chaos you threw at it - scaling changes, deletions, drift. What you saw wasn't just a neat demo; it was the core promise of GitOps delivered: **the cluster always returns to its declared state.**

Now let's ask a bigger question: *what if that same architectural elegance, that precise reliability, translated identically to the cloud - no matter how sprawling or complex the environment?*

That's exactly what we'll prove today.

Welcome back to GitOps Days, where **Git is not just for code anymore**. It's the single source of truth for everything - infrastructure, application configuration, and the glue that holds them together.
Yesterday, Git defined your laptop cluster. Today, Git defines a **production-grade Azure Kubernetes Service (AKS) cluster** in the cloud.

Here's the surprising insight: GitOps doesn't fundamentally care where it runs. Local of cloud, small or enterprise-scale, the rules don't change. Same Git. Same Flux. Same self-healing loop. Only the stage beneath it grows larger.

Today we'll take those principles into the real world:

* Provision an AKS cluster (in the **Free tier** for low-cost learning).
* Bootstrap Flux the production way - one command, end-to-end.
* Deploy the very same Hello app you ran yesterday, now Internet accessible with a single YAML tweak.
* Break things on prupose, and watch Flux restore order - even Azure load balancer and public IPs vanish.

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

👉 Next, we'll install Flux into the cluster using the **bootstrap method** and connect it to your Git repository. That's where the GitOps magic comes alive at cloud scale.

## 🚀 Section 3: Installing Flux on Your Cloud Cluster

**Remember yesterday's manual Flux installation? Today you'll do it the production way—one command instead of three.**

In Day 2, you installed Flux piece by piece to understand how it works. You ran `flux install`, then `flux create source git`, then `flux create kustomization`. That hands-on approach was perfect for learning.

But in production? Teams want simplicity. They want one command that does everything. That's where Flux bootstrap comes in—and that's what you'll use today.

### Setting Up for Success

Let's make sure you have everything ready for a smooth installation:

1. **Your GitOps repository**
   - Still have your fork from Day 2? Great! But let's make sure it's current.
   - We'll be updating this repository throughout the series, so delete your old fork and create a fresh one:
     1. Go to [github.com/ahmedmuhi/GitOps-Days](https://github.com/ahmedmuhi/GitOps-Days)
     2. Click **Fork** → **Create fork**
   - This ensures you have all the latest Day 3 content

2. **Flux CLI installed**
   ```bash
   flux --version
   # You already have this from Day 2
   ```

3. **kubectl pointing to your AKS cluster**
   ```bash
   kubectl config current-context
   # Should show: gitops-prod-aks (from Section 2)
   ```

4. **GitHub Personal Access Token**
   - Need the same `repo` permissions as Day 2
   - Lost yours? Create a new one:
     1. Go to [github.com/settings/tokens](https://github.com/settings/tokens)
     2. Click **Generate new token (classic)**
     3. Give it a name like "gitops-days"
     4. Check the `repo` checkbox
     5. Click **Generate token** and save it

With these ready, let's see what makes bootstrap special.

### The Production Way: Flux Bootstrap

Remember those three commands from Day 2? Bootstrap combines them into one, plus adds something powerful—it makes Flux manage itself through GitOps.

Here's what bootstrap does:
- Installs all Flux components (like `flux install`)
- Creates a GitRepository to watch your repo (like `flux create source git`)
- Creates a Kustomization to apply configurations (like `flux create kustomization`)
- **Plus**: Commits Flux's own configuration to Git so it can manage itself

One command. Complete setup. That's production efficiency.

### Running Bootstrap

Time to experience the magic. First, let's set up your GitHub credentials:

```bash
# Set your GitHub details
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
```

Now run the bootstrap command:

```bash
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=GitOps-Days \
  --branch=main \
  --path=examples/day3/clusters/aks \
  --personal
```

> [!NOTE]
> The `--personal` flag tells Flux this is a personal account, not an organizational one. Flux will expect authentication via personal access token rather than organization-level credentials.

Once bootstrap completes, let's speed up the reconciliation:

```bash
# Create a Kustomization with 1-minute interval
flux create kustomization flux-system \
  --source=GitRepository/flux-system \
  --path="./examples/day3/clusters/aks" \
  --prune=true \
  --interval=1m \
  --export > flux-system-1m.yaml

# Update the existing Kustomization
kubectl apply -f flux-system-1m.yaml

# Clean up the temporary file
rm flux-system-1m.yaml
```

> [!TIP]
> **Faster feedback**: Bootstrap creates a 10-minute default interval. We just updated it to match Day 2's 1-minute responsiveness. This `kubectl apply` here is for Flux configuration only—your actual deployments will still happen through pure Git!

### Watch Bootstrap Work Its Magic

As bootstrap runs (about 2 minutes), here's the dance happening behind the scenes:

1. **In your cluster**: Flux components deploy to the `flux-system` namespace
2. **In your Git repo**: A new folder structure appears:
   ```
   clusters/
   └── aks/
       └── flux-system/   # Flux's configuration files
   ```
3. **The clever part**: Flux immediately starts using these files to manage itself

You can watch the progress:
```bash
flux check
```

### Understanding What Bootstrap Created

Once complete, let's see what bootstrap built for you:

```bash
# Check the components
flux get sources git
flux get kustomizations
```

You'll see:
- A `flux-system` GitRepository (watching your entire repo)
- A `flux-system` Kustomization (applying everything under `clusters/aks/`)

**Here's the key insight**: That Kustomization is watching the entire `clusters/aks/` path, not just `flux-system`. This means any folder you add under `clusters/aks/` will automatically be applied. No extra commands needed!

Right now, it's only managing Flux itself (the `flux-system` folder). But when you add your app folders in Section 4, they'll be picked up automatically. Beautiful simplicity.

### 🎯 That's It!

Your AKS cluster now has:
- ✅ Flux running and managing itself through GitOps
- ✅ A Kustomization watching `clusters/aks/` for any changes
- ✅ The same self-healing powers you saw in Day 2

But your cluster feels a bit empty, doesn't it? No applications running yet. 

Remember that Hello World app from Day 2? The one that healed itself when you deleted it? In Section 4, you'll deploy it to the cloud using the same image, just changing one line to expose it to the internet.

The best part? You won't need any new Kustomizations. Just add your app under `clusters/aks/apps/hello/` and bootstrap's existing Kustomization will handle it. Why the nested folders? Because under `apps/` you might have `hello/`, `checkout/`, `payment/`—each app in its own folder, all managed by the same GitOps loop.

Ready to see your first cloud application deploy automatically? Let's make it happen...

## 🚀 Section 4: Your First Cloud Deployment

**Time to deploy to the cloud using pure GitOps. No kubectl apply. Just Git.**

Remember that Hello World app from Day 2? The one that magically healed itself? You're about to deploy the exact same app to AKS—with just one tiny change to make it internet-accessible.

### The YAMLs You'll Deploy

Let's create the same three files from Day 2, with that one magical change:

First, create the folder structure:
```bash
# Make sure you're in your GitOps-Days repository
cd ~/GitOps-Days

# Create the nested folders (matching where Flux is watching)
mkdir -p examples/day3/clusters/aks/apps/hello
cd examples/day3/clusters/aks/apps/hello
```

Now create these three files:

**1. Create `namespace.yaml`:**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: hello
```

**2. Create `deployment.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
  namespace: hello
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
        image: wbitt/hello-world:latest
        ports:
        - containerPort: 80
```

**3. Create `service.yaml`:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello
  namespace: hello
spec:
  type: LoadBalancer  # ← THE ONLY CHANGE! (was ClusterIP in Day 2)
  selector:
    app: hello
  ports:
    - port: 80
      targetPort: 80
```

### Spot the Difference?

That's it! One word changed: `ClusterIP` → `LoadBalancer`. This tells AKS to:
- Create an Azure Load Balancer automatically
- Assign a public IP address
- Route internet traffic to your app

Everything else stays exactly the same. Same image. Same deployment. Same GitOps principles.

### Deploy with Git (Not kubectl!)

Now for the GitOps magic. Deploy your app using only Git:

```bash
# Add your new files
git add .

# Commit with a meaningful message
git commit -m "Deploy hello app to AKS with LoadBalancer"

# Push to trigger Flux
git push origin main
```

### Watch Flux Detect Your Changes

This is where it gets exciting. Within 30 seconds, Flux will notice your commit:

```bash
# Watch Flux logs in real-time (this is new!)
flux logs --follow --tail 20
```

You'll see something like:
```
2024-12-02T10:15:30Z info GitRepository/flux-system.flux-system - stored artifact for commit 'Deploy hello app to AKS'
2024-12-02T10:15:31Z info Kustomization/flux-system.flux-system - Reconciliation finished
2024-12-02T10:15:32Z info Kustomization/flux-system.flux-system - applied 3 resources
```

Press `Ctrl+C` to stop following logs once you see the reconciliation complete.

### Verify Your Deployment

Let's check what Flux created:

```bash
# See your pods running
kubectl get pods -n hello

# Check the service (this is where the magic happens!)
kubectl get svc -n hello
```

You'll see something special:
```
NAME    TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)        AGE
hello   LoadBalancer   10.0.123.45   20.248.xxx.xxx   80:30657/TCP   2m
```

That `EXTERNAL-IP` is your app's public address! 

> [!NOTE]
> If EXTERNAL-IP shows `<pending>`, wait 30 seconds and run the command again. Azure is provisioning your Load Balancer.

### 🌍 Access Your Cloud App

No port-forwarding needed! Just open your browser:

```bash
# Get your app's public URL
echo "http://$(kubectl get svc hello -n hello -o jsonpath='{.status.loadBalancer.ingress[0].ip}')"
```

Click that link or copy it to your browser. 

**🎉 BOOM!** Your Hello World app is running on the internet, deployed through pure GitOps!

> [!WARNING]
> **Security Note**: Your app is now publicly accessible on the internet! While this is perfect for testing, remember to clean up these resources when done. We'll delete everything safely in Section 6 to avoid any unexpected costs or security risks.

### What Just Happened?

Let's appreciate this moment:
1. You pushed YAML files to Git
2. Flux detected the change within 30 seconds
3. Flux created the namespace, deployment, and service
4. AKS provisioned a real Load Balancer with public IP
5. Your app became internet-accessible

No `kubectl apply`. No manual steps. Just Git → Flux → Cloud.

**This is the same GitOps loop from Day 2, just running at cloud scale!**

Ready to test if cloud-scale self-healing works too? Let's break some things...

## 💥 Section 5: Cloud-Scale Self-Healing

**Time for the fun part—let's break your cloud deployment and watch GitOps fix it automatically!**

In Day 2, you deleted pods and watched Flux restore them on your laptop. But does this magic work in the cloud? With public IPs? With Azure Load Balancers? 

Spoiler alert: It works even better.

### Test 1: The Emergency Scale-Down

Let's simulate someone "temporarily" scaling your deployment to zero during an incident:

```bash
# Scale to zero replicas (goodbye, app!)
kubectl scale deployment hello -n hello --replicas=0

# Watch the deployment status
kubectl get deployment hello -n hello -w
```

You'll see:
```
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
hello   1/1     1            1           10m
hello   0/1     0            0           10m      # Your manual change
hello   0/1     0            0           10m30s   # Flux notices...
hello   0/1     1            0           10m45s   # Flux says "nope!"
hello   1/1     1            1           11m      # Back to normal!
```

Press `Ctrl+C` to stop watching. Within 60 seconds, Flux restored your deployment!

### Test 2: The Nuclear Option

Now let's go extreme. Delete EVERYTHING:

```bash
# Delete the entire namespace (and everything in it)
kubectl delete namespace hello

# This deletes:
# ✗ Your deployment
# ✗ Your service  
# ✗ Your load balancer
# ✗ Your public IP
# ✗ Everything!
```

Now watch the magic:
```bash
# Watch Flux rebuild from scratch
watch kubectl get all -n hello
```

Every 2 seconds, you'll see the resurrection:
- First: "No resources found" (everything's gone!)
- 30 seconds later: Namespace reappears
- 45 seconds later: Deployment and pods are back
- 60 seconds later: Service and load balancer return
- 90 seconds later: New public IP assigned!

Press `Ctrl+C` when everything's back.

### Verify Your App Still Works

The ultimate test—is your app accessible again?

```bash
# Get the NEW public IP (yes, it might be different!)
kubectl get svc hello -n hello

# Get the URL
echo "http://$(kubectl get svc hello -n hello -o jsonpath='{.status.loadBalancer.ingress[0].ip}')"
```

Open that URL in your browser. Your app is back, with a fresh Azure Load Balancer and new public IP!

### What Makes This Incredible

Think about what just happened:
1. You deleted cloud infrastructure (including an Azure Load Balancer)
2. Flux detected the drift within seconds
3. Flux recreated everything automatically
4. Azure provisioned new cloud resources
5. Your app got a new public IP and kept working

**No tickets. No manual fixes. No panic.**

This is the same self-healing from Day 2, but now it's healing real cloud infrastructure. The GitOps principles scaled perfectly from your laptop to Azure.

### Watch Flux's Thought Process

Want to see Flux's logs during the healing?

```bash
# See recent Flux events
flux events --for Kustomization/flux-system

# You'll see entries like:
# Reconciliation finished in 1.2s, next run in 1m0s
# Applied 3 resources
# Namespace hello created
# Deployment hello/hello created
# Service hello/hello created
```

### The Bottom Line

Whether on your laptop or in the cloud, GitOps behaves identically:
- **Drift is temporary** - Flux always wins
- **Deletions don't stick** - Git is truth
- **Manual changes are futile** - Automation prevails

Your cloud infrastructure is now as self-healing as your local cluster was. Same principles. Same reliability. Just bigger scale.

🎉 **Congratulations!** You've proven GitOps truly works anywhere!

Ready to clean up and celebrate what you've achieved? Let's wrap this up...

## 🧹 Section 6: Cleanup & Next Steps

**Time to clean up your cloud resources and celebrate what you've achieved!**

### Delete Your Azure Resources

Let's remove everything to avoid any charges:

```bash
# Delete the entire resource group (and everything in it)
az group delete --name gitops-prod-rg --yes --no-wait

# This deletes:
# - Your AKS cluster
# - All load balancers
# - All public IPs  
# - Everything!
```

The `--no-wait` flag means you don't have to wait for deletion to complete. Azure will clean up everything in the background.

### 🎉 And With That... Day 3 is Complete!

What an incredible journey from laptop to cloud. Let's take a moment to appreciate what you've accomplished.

**You started with**: A working GitOps setup on your laptop (Day 2)  
**You ended with**: The exact same GitOps running on Azure Kubernetes Service

You proved that GitOps is truly portable:
- ✅ Same Flux tool (just bootstrap vs manual install)
- ✅ Same Git repository structure  
- ✅ Same self-healing behavior
- ✅ Same drift correction
- ✅ Only difference: `LoadBalancer` instead of `ClusterIP`

### The Power of What You Built

Think about this—you deployed a cloud application without:
- ❌ Learning cloud-specific deployment tools
- ❌ Writing cloud-specific configurations
- ❌ Changing your GitOps workflow
- ❌ Using `kubectl apply` even once

Your local knowledge scaled perfectly to the cloud. That's the promise of GitOps delivered.

### Your GitOps Capabilities Now

After three days, you can confidently:
- **Explain** GitOps principles to anyone (Day 1 knowledge)
- **Build** self-healing Kubernetes systems from scratch (Day 2 skills)
- **Deploy** to any Kubernetes cluster—local or cloud (Day 3 proof)
- **Trust** that your infrastructure will never drift
- **Sleep** peacefully knowing Git is your source of truth

You're no longer just GitOps-curious. You're GitOps-capable.

### Your Journey So Far

Let's see how far you've come:

**Day 1**: Understood why GitOps exists and how it solves drift  
**Day 2**: Built self-healing infrastructure on your laptop  
**Day 3**: Proved the same patterns work identically in the cloud

You now have hands-on proof that GitOps principles are universal. Whether you're deploying to a Raspberry Pi or a massive cloud cluster, the workflow remains the same.

### 🚀 Coming Next: Day 4 - Production GitOps Patterns

Now that you've proven GitOps works anywhere, it's time to learn how teams use it in production.

Tomorrow, we'll level up with real-world patterns:
- **Azure Container Registry**: Host your own images securely
- **GitHub Actions CI/CD**: Automate image builds and deployments  
- **Multi-environment workflows**: Dev, staging, and production
- **Secrets management**: Handle sensitive data the GitOps way

You've mastered the fundamentals. Tomorrow, we add the patterns that make GitOps production-ready.

### Before You Go

Did you:
- ✅ See your app running with a public IP?
- ✅ Watch Flux heal your broken deployment?
- ✅ Delete your Azure resources?
- ✅ Smile at how simple cloud deployments can be?

**One more thing**: If you're finding value in this series, please ⭐ star the repository! It helps others discover GitOps-Days and ensures you get notifications for new content.

### 🎊 Congratulations!

You've taken GitOps from concept to cloud reality. Your clusters—local or cloud—will never drift again.

See you tomorrow for Day 4, where we'll add the patterns that make GitOps truly production-ready!

---

*P.S. Still amazed that one LoadBalancer change was all it took? That's the beauty of cloud-native applications with GitOps. Tomorrow gets even better.*