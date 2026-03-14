# Day 3 – GitOps on AKS: Same Loop, Bigger Stage

> **What you'll need:** Everything from Day 2, plus an Azure account and the Azure CLI (2.76.0+).
> **Time:** ~1 hour (plus ~10 minutes waiting for Azure to provision resources).
> **Cost:** Running a one-node AKS cluster for this session costs about **$0.70**. We'll delete everything at the end. If you're new to Azure, you get [$200 in free credits](https://azure.microsoft.com/free).

Yesterday you built a self-healing system on your laptop and proved the loop works. Today we take the same loop to **Azure Kubernetes Service** — a real, cloud-hosted Kubernetes cluster with a public IP and a load balancer.

The principles don't change. Git declares, Flux enforces, drift gets corrected. The only difference is what's running underneath.

In this session, you'll provision an AKS cluster, bootstrap Flux the production way (one command instead of three), deploy the same Hello app — this time accessible on the internet — and break things again to watch Flux heal cloud infrastructure, not just local pods.

> **One thing to keep in mind:** every command in this guide uses `YOUR-USERNAME` as a placeholder. Replace it with your actual GitHub username wherever you see it.

## Set up your cloud workspace

Same idea as Day 2 — get the logistics done in one go so the rest of the session is pure building. This time there are a few extra steps because we're working with Azure and Flux needs write access to your repo.

### Log into Azure
```shell
az login
```

A browser window will open for you to authenticate. If you have multiple subscriptions, you'll be prompted to pick one — choose whichever you're comfortable creating temporary test resources in.

Verify the right subscription is active:
```shell
az account show --output table
```

### Sync your fork

If you completed Day 2, you already have a fork of GitOps-Days. Make sure it's up to date:

1. Open your fork on GitHub (`https://github.com/YOUR-USERNAME/GitOps-Days`).
2. If you see a **"This branch is behind..."** banner, click **Sync fork → Update branch**.
3. Pull the latest changes locally:
```shell
cd GitOps-Days
git pull origin main
```

> [!TIP]
> Starting fresh without Day 2? Fork the repo from [`https://github.com/ahmedmuhi/GitOps-Days`](https://github.com/ahmedmuhi/GitOps-Days), then clone your fork:
> ```shell
> git clone https://github.com/YOUR-USERNAME/GitOps-Days.git
> cd GitOps-Days
> ```

### Copy Day 3 files into your workspace
```shell
mkdir -p student-work/YOUR-USERNAME/day3
cp -r examples/day3/clusters student-work/YOUR-USERNAME/day3/
```

> [!TIP]
> On Windows PowerShell:
> ```shell
> New-Item -ItemType Directory -Path "student-work\\YOUR-USERNAME\\day3" -Force
> Copy-Item -Recurse examples\\day3\\clusters student-work\\YOUR-USERNAME\\day3\\
> ```

Your Day 3 workspace should now look like this:
```
student-work/YOUR-USERNAME/day3/clusters/aks/apps/hello/
├── deployment.yaml
├── namespace.yaml
├── service.yaml
└── kustomization.yaml
```

The manifests are almost identical to Day 2. The one difference: `service.yaml` now uses `type: LoadBalancer` instead of `ClusterIP`, which tells AKS to provision a public IP. That single line is what makes your app internet-accessible.

### Set up GitHub credentials

Flux's bootstrap command will commit files back to your repo — so it needs permission to push. You'll create a GitHub Personal Access Token (PAT) and export it as an environment variable.

**Create a PAT:**

1. Go to **GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens**.
2. Scope it to **your fork only**, with `contents: read/write` permission.
3. Copy the token immediately — GitHub only shows it once. Save it somewhere secure; you'll use it again in Day 4.

**Export your credentials:**
```shell
export GITHUB_USER=YOUR-USERNAME
export GITHUB_TOKEN=your-token-here
```

> [!TIP]
> On Windows PowerShell:
> ```shell
> $env:GITHUB_USER="YOUR-USERNAME"
> $env:GITHUB_TOKEN="your-token-here"
> ```

Flux will pick these up automatically when it talks to GitHub.

### Checkpoint: workspace ready

Commit your Day 3 workspace so it's in Git before we bootstrap:
```shell
git add student-work/
git commit -m "Create Day 3 workspace"
git push
```

Verify:
```shell
ls student-work/YOUR-USERNAME/day3/clusters/aks/apps/hello/
```

You should see `deployment.yaml`, `kustomization.yaml`, `namespace.yaml`, and `service.yaml`.

You're logged into Azure, your fork is synced, your workspace is ready, and Flux has credentials to push. Let's create the cluster.

## Create your AKS cluster

Time to provision real cloud infrastructure. We'll create a small AKS cluster — one node is enough for this tutorial.

### Set your region

Pick a region close to you to minimise latency:

```shell
LOCATION="australiaeast"
```

Common alternatives: `eastus2` or `westus2` (Americas), `westeurope` (Europe). Use whatever works best for you.

### Create a resource group

Azure uses resource groups to keep related resources together. Everything we create today goes into one group — which makes cleanup a single command later.

```shell
RESOURCE_GROUP="gitops-prod-rg"

az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION
```

### Create the cluster

```shell
az aks create \
  --resource-group $RESOURCE_GROUP \
  --name gitops-prod-aks \
  --location $LOCATION \
  --node-count 1 \
  --node-vm-size Standard_B2s \
  --tier free \
  --enable-managed-identity \
  --generate-ssh-keys
```

This creates a single-node AKS cluster on the Free tier. The `Standard_B2s` VM is the smallest practical size — enough for our Hello app and Flux, and costs about $0.04/hour. The `--generate-ssh-keys` flag handles key management automatically.

> [!TIP]
> This takes about 7–10 minutes. Good time to stretch or refill your coffee.

### Connect kubectl to your cluster

Once provisioning completes, download the credentials:

```shell
az aks get-credentials \
  --resource-group $RESOURCE_GROUP \
  --name gitops-prod-aks \
  --overwrite-existing
```

This merges the AKS context into your `~/.kube/config`. From now on, `kubectl` talks to AKS instead of your local kind cluster.

### Checkpoint: cluster running

```shell
kubectl get nodes
```

You should see one node with `STATUS: Ready`:

```
NAME                                STATUS   ROLES    AGE   VERSION
aks-nodepool1-12345678-vmss000000   Ready    <none>   2m    v1.32.x
```

> [!IMPORTANT]
> If the node shows `NotReady`, give it a minute — AKS nodes sometimes need a moment after provisioning. If it persists, check that the `az aks create` command completed without errors.

You've got a running AKS cluster. Now let's give it a controller — the production way.

## Bootstrap Flux

In Day 2, you installed Flux step by step — `flux install`, then `flux create source git`, then `flux create kustomization`. That was useful for seeing the moving parts. In production, teams use a single command that does all of it at once: `flux bootstrap`.
```shell
flux bootstrap github \\
  --owner=$GITHUB_USER \\
  --repository=GitOps-Days \\
  --branch=main \\
  --path=student-work/$GITHUB_USER/day3/clusters/aks \\
  --personal
```

A quick note on the flags:

- `--owner` → your GitHub username
- `--repository` → the name of your forked repo
- `--path` → the folder Flux will watch for manifests
- `--personal` → tells Flux this is a personal fork, not an organisation repo

Flux picks up `GITHUB_USER` and `GITHUB_TOKEN` from the environment variables you exported earlier — no need to pass credentials on the command line.

> [!TIP]
> This takes 2–3 minutes. You'll know it's working when you see output scrolling about installing components and pushing to your repository.

### Something new happened

When this finishes, go to your fork on GitHub and look inside `student-work/YOUR-USERNAME/day3/clusters/aks/`. You'll see a new `flux-system/` folder that wasn't there before.

In Day 2, Flux only *read* from Git. Bootstrap goes a step further — it commits Flux's own configuration back into your repo so that Flux manages itself through Git too. If someone were to delete Flux from the cluster and re-run bootstrap, it would rebuild itself from those committed files.

You don't need to touch anything inside `flux-system/`. Just know it's there, and it's how Flux keeps itself running.

### Checkpoint: Flux is healthy
```shell
flux check
```

All controllers should show as ready. Then confirm Flux is watching your repo:
```shell
flux get sources git
```
```
NAME          URL                                                 READY   STATUS
flux-system   https://github.com/YOUR-USERNAME/GitOps-Days.git    True    stored artifact for revision 'main@sha1:...'
```

`READY: True` means Flux is installed, connected to your fork, and watching for changes. The loop is running on AKS.

Now let's give it something to deploy.

## Deploy your app to the cloud

This is where Day 3 delivers on its promise. You're about to deploy the same Hello app to AKS — this time with a real load balancer and a public IP — using nothing but a Git commit.

### Edit the root kustomization.yaml

Open the file at:
```
student-work/YOUR-USERNAME/day3/clusters/aks/kustomization.yaml
```

You'll see:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ./flux-system
```

That `./flux-system` entry is how Flux keeps itself running — bootstrap created it. Add your app alongside it:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ./flux-system
  - ./apps/hello       # ← add this line
```

Each line points to a folder of manifests that Flux will apply. You've just told Flux: "manage yourself *and* my app."

### Commit and push
```shell
git add .
git commit -m "Add Hello app to GitOps loop"
git push origin main
```

### Watch Flux reconcile

Within about a minute, Flux will detect your commit and apply the Hello app manifests:
```shell
flux logs --follow --tail 20
```

You'll see messages about the GitRepository updating and resources being applied. Press `Ctrl+C` once you see the reconciliation complete.

### Verify your app is running
```shell
kubectl get pods,svc -n hello
```
```
NAME                         READY   STATUS    RESTARTS   AGE
pod/hello-65d4c4d5c9-xz7vp   1/1     Running   0          2m

NAME            TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)        AGE
service/hello   LoadBalancer   10.0.123.45   20.248.xxx.xxx   80:30123/TCP   2m
```

Notice the `TYPE: LoadBalancer` and the `EXTERNAL-IP`. That's Azure provisioning real cloud infrastructure — a load balancer and a public IP — because your `service.yaml` declared `type: LoadBalancer`. One line in Git, and Azure responded.

> [!TIP]
> If `EXTERNAL-IP` shows `<pending>`, wait 30–60 seconds and try again. Azure is creating the load balancer.

### Open it in your browser
```shell
echo "http://$(kubectl get svc hello -n hello -o jsonpath='{.status.loadBalancer.ingress[0].ip}')"
```

Copy the URL and open it. Your Hello app is live on the internet — deployed entirely through Git.

### What just happened — the loop at cloud scale

The same three phases, running on real infrastructure:

**Watch** — Flux's Source Controller detected your new commit and pulled the updated repo.

**Compare** — The Kustomize Controller found two entries in the root file: `flux-system` (already applied) and `apps/hello` (new). It read the four manifests inside your hello folder and compared them against the cluster — nothing existed yet, so everything was a difference.

**Reconcile** — The controller applied the namespace, Deployment, and Service. Kubernetes created the pod. AKS saw `type: LoadBalancer` and provisioned an Azure load balancer with a public IP.

That's the same loop as Day 2 — watch, compare, reconcile. The only thing that changed is what's underneath. Instead of a kind container on your laptop, it's a managed Kubernetes cluster backed by Azure compute, networking, and load balancing. The loop doesn't care. It just makes the cluster match Git.

You've deployed to the cloud through Git. Now let's change something.

## Make a change through Git

Same workflow as Day 2 — but this time, the cluster responding is running in Azure.

Open `student-work/YOUR-USERNAME/day3/clusters/aks/apps/hello/deployment.yaml` and change:
```yaml
spec:
  replicas: 1
```

to:
```yaml
spec:
  replicas: 3
```

Commit and push:
```shell
git add student-work/YOUR-USERNAME/day3/clusters/aks/apps/hello/deployment.yaml
git commit -m "Scale Hello app to 3 replicas on AKS"
git push origin main
```

Watch the cluster respond:
```shell
kubectl get deployment hello -n hello -w
```

Within about a minute:
```
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
hello   1/1     1            1           10m
hello   1/3     1            1           10m
hello   2/3     2            2           10m
hello   3/3     3            3           11m
```

Press `Ctrl+C` to stop watching.

Same file, same edit, same loop. The only difference is that those three pods are now running on Azure compute, behind a real load balancer, accessible on the internet. The workflow didn't change at all — and that's the point.

Let's push it further. What happens when someone changes the cluster without going through Git?

## Break things on purpose

You've proved Git drives the cluster on AKS. Now let's prove the second half — drift correction at cloud scale.

Before we start, let's shorten Flux's reconciliation interval so you're not waiting long between experiments. Open:
```
student-work/YOUR-USERNAME/day3/clusters/aks/flux-system/gotk-sync.yaml
```

Find the `Kustomization` spec and change the interval:
```yaml
spec:
  interval: 1m0s   # ← change from the old 10m0s
```

Commit and push:
```shell
git add student-work/YOUR-USERNAME/day3/clusters/aks/flux-system/gotk-sync.yaml
git commit -m "Speed up Flux reconciliation to 1 minute"
git push origin main
```

Now Flux will compare and correct every 60 seconds. Let's give it something to correct.

### Experiment 1: The emergency scale

A teammate bypasses Git and scales the app directly:
```shell
kubectl scale deployment hello -n hello --replicas=5
```

Watch what happens:
```shell
kubectl get deployment hello -n hello -w
```
```
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
hello   3/3     3            3           20m
hello   5/5     5            5           20m       # manual change takes effect
hello   3/3     3            3           21m       # Flux corrects it
```

Same behaviour as Day 2. Manual change lands, then the loop undoes it. Press `Ctrl+C`.

### Experiment 2: The catastrophic delete

This is where cloud scale makes the experiment more dramatic. Delete the entire namespace:
```shell
kubectl delete namespace hello
```

This doesn't just remove pods — it tears down the Azure load balancer and releases the public IP. Everything your app needed to be internet-accessible is gone.

Watch Flux rebuild it:
```shell
watch kubectl get all -n hello
```

Over the next 1–3 minutes, you'll see the namespace reappear, the Deployment recreate, pods spin up, the Service come back — and then Azure will provision a **new** load balancer and assign a **new** public IP.

Press `Ctrl+C` when you see everything running again.

Get the new URL:
```shell
echo "http://$(kubectl get svc hello -n hello -o jsonpath='{.status.loadBalancer.ingress[0].ip}')"
```

Open it in your browser. Your app is back online — with a fresh public IP, fully restored from Git.

> [!NOTE]
> The IP address will be different from before. Azure creates a new load balancer each time the Service is recreated. The app is the same; only the address changed.

### What made this different from Day 2

On your laptop, the catastrophic delete restored pods and a ClusterIP service. That was impressive. But here, Flux's reconciliation triggered Azure to provision real cloud infrastructure — a load balancer, a public IP, network routing. The loop didn't just fix Kubernetes resources. It caused the cloud provider to rebuild infrastructure that costs real money and serves real traffic.

Self-healing isn't a local demo trick. It works the same way on infrastructure that matters.

That's both experiments done. Let's clean up before Azure keeps billing.

## Clean up

Cloud resources cost money when they're running. Let's delete everything in one command.

### Delete the resource group
```shell
az group delete --name gitops-prod-rg --yes --no-wait
```

This removes your AKS cluster, worker nodes, load balancers, public IPs, disks — everything. The `--no-wait` flag starts the deletion in the background so you don't have to sit and watch.

### Clean up your local kubeconfig (optional)

Your kubeconfig still has a context pointing at the deleted cluster. Remove it to avoid confusion later:
```shell
kubectl config delete-context gitops-prod-aks || true
kubectl config delete-cluster gitops-prod-aks || true
```

### Verify everything is gone
```shell
az group list --output table
```

You shouldn't see `gitops-prod-rg` in the list. If it still shows, the deletion is in progress — give it a few minutes.

Nothing left running. No ongoing charges.

## What's next — on to Day 4

You started this series with a mental model. You proved it locally. And today, you proved it at cloud scale.

The reconciliation loop — watch, compare, reconcile — worked identically on your laptop and on AKS. The workflow didn't change. The commands didn't change. Even the experiments were the same. The only thing that changed was the infrastructure underneath — and the loop didn't care.

That's the core promise of GitOps, verified across three days: **declare it in Git, and the system makes it real — wherever it runs.**

In Day 4, the question shifts from "does it work?" to "how do teams run this in production?" — the CI pipeline that feeds the loop, and the repo patterns that make GitOps work across multiple environments.

**Ready to go production-ready?** [Continue to Day 4 →](./Day-4-Production-GitOps-Patterns.md)

<details><summary>Things to try before Day 4</summary>

The AKS cluster is deleted, but your fork still has everything. Before moving on:

- Read through the `flux-system/` folder in your repo. Now that you've seen bootstrap in action, the files will make more sense. `gotk-components.yaml` defines what Flux installs. `gotk-sync.yaml` wires Flux to your repo. Together, they're how Flux manages itself through Git.
- Compare your Day 2 and Day 3 workspace folders side by side. The app manifests are nearly identical — the only difference is `type: LoadBalancer` in the service. Everything else was the same loop on a different stage.

</details>
