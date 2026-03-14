# Day 2 – Building Your First Self-Healing System

> **What you'll need:** Docker (24.0+), kind (0.25.0+), kubectl (1.32+), Git (2.40+), and a GitHub account.
> **Time:** ~1 hour. **Tools not installed?** Links are in the setup section below.

You've got the mental model. You can trace the reconciliation loop — watch, compare, reconcile — and you know what changes when a controller enters the picture. Now let's prove it.

In this session, you'll build a self-healing Kubernetes system on your laptop. You'll install Flux, deploy an app entirely through Git, change it, break it, and watch the loop put it back — all without running `kubectl apply` once.

> **One thing to keep in mind:** every command in this guide uses `YOUR-USERNAME` as a placeholder. Replace it with your actual GitHub username wherever you see it.

## Set up your workspace

Before we touch Kubernetes or Flux, let's get all the logistics out of the way in one go. By the end of this section, you'll have a forked repo, a local clone, and your working files ready. After that, it's pure building.

### Fork the repository

Go to [`https://github.com/ahmedmuhi/GitOps-Days`](https://github.com/ahmedmuhi/GitOps-Days) and click **Fork** in the top-right corner, then **Create fork**.

Why fork? In GitOps, the controller pulls from a Git repository to know what the cluster should look like. For that to work, you need a repo you can push to. A fork gives you your own copy under your GitHub account — one you control completely.

Your fork will live at:

```
https://github.com/YOUR-USERNAME/GitOps-Days
```

### Clone your fork locally

```shell
git clone https://github.com/YOUR-USERNAME/GitOps-Days.git
cd GitOps-Days
```

Verify it's pointing at your fork, not the original:

```shell
git remote -v
```

You should see your username in the URLs:

```
origin  https://github.com/YOUR-USERNAME/GitOps-Days.git (fetch)
origin  https://github.com/YOUR-USERNAME/GitOps-Days.git (push)
```

> [!IMPORTANT]
> If you see `ahmedmuhi` instead of your username, you cloned the original repo by mistake. Delete the folder and clone your fork — Flux won't work otherwise.

### Create your working folder

This repository gets updated as new lessons are added. To make sure upstream syncs never overwrite your work, you'll keep everything in your own folder:

```shell
mkdir -p student-work/YOUR-USERNAME/day2
cp -r examples/day2/clusters/local/apps/hello student-work/YOUR-USERNAME/day2/
```

> [!TIP]
> On Windows PowerShell, use:
> ```shell
> New-Item -ItemType Directory -Path "student-work\YOUR-USERNAME\day2" -Force
> Copy-Item -Recurse examples\day2\clusters\local\apps\hello student-work\YOUR-USERNAME\day2\
> ```

Your working folder should now look like this:

```
student-work/YOUR-USERNAME/day2/hello/
├── namespace.yaml
├── deployment.yaml
└── service.yaml
```

### What you'll be deploying

Take a quick look at the Deployment so you know what Git is declaring:

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
          image: nginxdemos/hello:plain-text
          ports:
            - containerPort: 80
```

One replica of a lightweight nginx demo. Simple on purpose — the app isn't the point today. The loop is. Remember this file — you'll change it later and watch the cluster follow.

### Checkpoint: workspace ready

Run:

```shell
ls student-work/YOUR-USERNAME/day2/hello/
```

You should see `deployment.yaml`, `namespace.yaml`, and `service.yaml`.

Commit your workspace so it's in Git and ready for Flux:

```shell
git add student-work/
git commit -m "Create Day 2 workspace"
git push
```

> [!TIP]
> If you repeat this lab in the future and re-copy files from `examples/`, your previous work will be overwritten. Either rename your old folder first or create a new one (e.g., `day2-v2`).

You now have a fork you control, a local clone, a workspace folder with the Day 2 files, and everything pushed to Git. Logistics are done — from here on, we build.

## Create your cluster

Time to build something. We'll spin up a local Kubernetes cluster using [kind](https://kind.sigs.k8s.io/) (Kubernetes in Docker) — it runs a full cluster inside Docker containers and launches in about a minute.
```shell
kind create cluster --name gitops-loop-demo
```

### Checkpoint: cluster running
```shell
kubectl get nodes
```

You should see:
```
NAME                             STATUS   ROLES           AGE   VERSION
gitops-loop-demo-control-plane   Ready    control-plane   1m    v1.32.x
```

The key thing is `STATUS: Ready`. If you see `NotReady`, give it a few seconds and try again — the node needs a moment to finish starting up.

> [!IMPORTANT]
> If it stays `NotReady`, check that Docker is running (`docker ps`) and that you have at least 4 GB of free RAM. If still stuck, delete and recreate:
> ```shell
> kind delete cluster --name gitops-loop-demo
> kind create cluster --name gitops-loop-demo
> ```

You've got a running cluster. Now let's give it a controller.

## Install Flux

Flux is one of two major GitOps controllers in the CNCF ecosystem — the other being [Argo CD](https://argo-cd.readthedocs.io/). Both implement the same reconciliation loop. We're using Flux here because it's lightweight to install and gets out of your way quickly — which is what you want when the goal is to see the loop in action, not configure a tool.

### Install the Flux CLI

If you don't already have it:

```shell
curl -s https://fluxcd.io/install.sh | sudo bash
```

> [!TIP]
> For other installation methods (Homebrew, Chocolatey, etc.), see the [Flux installation docs](https://fluxcd.io/flux/installation/).

### Install Flux in your cluster

```shell
flux install
```

This deploys Flux's controllers into a `flux-system` namespace in your cluster. You don't need to know what each one does yet — you'll see them in action shortly.

### Connect Flux to your Git repository

Now tell Flux where to watch:

```shell
flux create source git gitops-loop-demo \
  --url=https://github.com/YOUR-USERNAME/GitOps-Days.git \
  --branch=main \
  --interval=30s
```

This sets up the **watch** step of the loop. If you're thinking "that's the first phase from Day 1" — exactly right. Flux will poll your repository every 30 seconds, looking for changes.

### Checkpoint: Flux is healthy and watching

```shell
flux check
```

You should see all controllers marked as ready:

```
► checking controllers
✔ source-controller: deployment ready
✔ kustomize-controller: deployment ready
✔ helm-controller: deployment ready
✔ notification-controller: deployment ready
✔ all checks passed
```

Then confirm your Git source is registered:

```shell
flux get sources git
```

```
NAME               URL                                                 READY   STATUS
gitops-loop-demo   https://github.com/YOUR-USERNAME/GitOps-Days.git    True    stored artifact for revision 'main@sha1:...'
```

`READY: True` means Flux has fetched a copy of your repository and is watching it. Every 30 seconds, it will check for new commits.

Flux is installed and watching your repo. Now let's give it something to deploy.

## Deploy your first app through Git

This is the moment the loop becomes real. You're going to tell Flux what to deploy — and then watch it happen without touching `kubectl apply`.

### Create a Kustomization

This command tells Flux which folder in your repo contains the manifests it should apply to the cluster:

```shell
flux create kustomization hello-app \
  --source=GitRepository/gitops-loop-demo \
  --path="./student-work/YOUR-USERNAME/day2/hello" \
  --prune=true \
  --interval=1m
```

A quick note on the flags:

- `--source` → the Git source you created in the previous step
- `--path` → the folder in your repo where your manifests live
- `--prune=true` → if you delete a file from Git, remove the corresponding resource from the cluster
- `--interval=1m` → compare and reconcile every 60 seconds

This sets up the **compare and reconcile** steps of the loop. The source handles watching Git. The Kustomization handles making the cluster match.

### Confirm the Kustomization is ready

```shell
flux get kustomizations
```

```
NAME        READY   MESSAGE                                       REVISION              SUSPENDED
hello-app   True    Applied revision: main@sha1:123abc456def...   main@sha1:123abc...   False
```

Notice the message: **Applied revision**. That means Flux didn't wait for you to push a new commit. The moment you created the Kustomization, it pulled the manifests from your workspace folder and applied them to the cluster. Your first GitOps deployment has already happened.

### See what Flux deployed

```shell
kubectl get pods,svc -n hello
```

```
NAME                         READY   STATUS    RESTARTS   AGE
pod/hello-65d4c4d5c9-xz7vp   1/1     Running   0          2m

NAME            TYPE        CLUSTER-IP      PORT(S)   AGE
service/hello   ClusterIP   10.96.x.x       80/TCP    2m
```

One pod, one service — exactly what your Deployment manifest declared. The desired state in Git is now the actual state in the cluster.

### Access your running app

Forward the service port to your machine:

```shell
kubectl port-forward -n hello svc/hello 8080:80
```

Open [http://localhost:8080](http://localhost:8080) in your browser. You should see a plain-text response showing the server name and address. That's your app, running in Kubernetes, deployed entirely by Flux.

Press `Ctrl+C` to stop the port-forward when you're done.

### What just happened — the full loop

Let's map what you just experienced back to the mental model from Day 1:

**Watch** — Flux's Source Controller fetched your repository and cached it inside the cluster. It will refresh that cache every 30 seconds.

**Compare** — Flux's Kustomize Controller read the manifests from your workspace folder and compared them against what was running in the cluster. Since the namespace, Deployment, and Service didn't exist yet, everything was a difference.

**Reconcile** — The Kustomize Controller applied all three manifests. Kubernetes created the namespace, spun up the pod, and exposed the service.

That's the loop — running for real, on your laptop. From this point on, Flux will keep checking. If the cluster matches Git, it does nothing. If something changes, it corrects it.

You've just seen Flux deploy. Now let's see it respond to a change.

## Make a change through Git

Flux deployed your app from existing files. Now let's prove that pushing a change to Git is all it takes to update your cluster.

Open `student-work/YOUR-USERNAME/day2/hello/deployment.yaml` in your editor and change:

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
git add student-work/YOUR-USERNAME/day2/hello/deployment.yaml
git commit -m "Scale hello app to 3 replicas"
git push
```

Now watch the cluster respond:

```shell
kubectl get deployment hello -n hello -w
```

Within about a minute, you'll see the replica count climb from 1 to 3:

```
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
hello   1/1     1            1           10m
hello   1/3     1            1           10m
hello   2/3     2            2           10m
hello   3/3     3            3           11m
```

Press `Ctrl+C` to stop watching.

Think about what just happened. You edited a file, pushed to Git, and walked away. No `kubectl apply`. No pipeline to trigger. The Source Controller picked up your new commit, the Kustomize Controller compared it to the cluster, found that 1 ≠ 3, and reconciled. The loop did exactly what Day 1 said it would.

This is the first half of the GitOps promise: **Git drives the cluster.**

The second half is what happens when something changes the cluster *without* going through Git. Let's test that next.

## Break things on purpose

You've proved that Git drives the cluster. Now let's prove the other half: **the cluster resists changes that don't come from Git.**

We'll run two experiments. Both simulate real-world mistakes — and both end the same way.

### Experiment 1: The emergency scale

**The scenario:** A teammate is mid-incident. Under pressure, they bypass Git and scale the app directly:

```shell
kubectl scale deployment hello -n hello --replicas=5
```

Watch what happens:

```shell
kubectl get deployment hello -n hello -w
```

Within about 60 seconds, you'll see this:

```
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
hello   3/3     3            3           20m
hello   5/5     5            5           20m       # manual change takes effect
hello   3/3     3            3           21m       # Flux corrects it
```

The manual change landed — and then it was undone. The Kustomize Controller compared the cluster (5 replicas) against Git (3 replicas), found a mismatch, and reconciled. No alert, no human intervention. The loop handled it.

Press `Ctrl+C` to stop watching.

### Experiment 2: The catastrophic delete

**The scenario:** Someone accidentally deletes the entire namespace — app, service, everything:

```shell
kubectl delete namespace hello
```

Now watch Flux rebuild it:

```shell
watch kubectl get all -n hello
```

Over the next 60 seconds or so, you'll see the namespace reappear, the Deployment recreate, pods spin up, and the service come back online. Everything restored — from Git.

Press `Ctrl+C` when you see everything running again.

This is the moment that earns the phrase "self-healing." The namespace and everything in it are declared in Git. When they disappeared from the cluster, the controller treated it the same way it treated the replica mismatch — a difference to be reconciled. The fix isn't special logic. It's just the loop, doing what it always does.

### Why it takes about 60 seconds

You configured two intervals when you set up Flux:

- **Source interval (30s)** — how often Flux checks Git for new commits.
- **Reconciliation interval (1m)** — how often it compares the cluster against the cached state and corrects drift.

The healing you just saw was the reconciliation interval at work. Flux didn't need a new commit to act — it already knew the desired state. It just needed its next scheduled comparison to notice the drift.

You can see this timing in the events log:

```shell
flux events --for Kustomization/hello-app
```

```
Reconciliation finished in 1.2s, next run in 1m0s
```

Those two intervals are the heartbeat of your system. In production, you'd tune them based on how fast you need drift correction versus how much load you want on the API server. But for this lab, 30 seconds and 1 minute let you see everything happen in real time.

That's both halves of the GitOps promise, verified with your own hands. Git drives the cluster — and the cluster won't stay changed unless Git says so.

## What's next — on to Day 3

You came into today with a mental model. You're leaving with proof.

The reconciliation loop isn't a theory anymore — it's running on your laptop. You've watched Git drive the cluster, and you've watched the cluster refuse to stay changed without Git's say-so. That's GitOps, working.

Everything you built today was local — a kind cluster, a single app, one controller watching one repo. In Day 3, we take the same loop to **Azure Kubernetes Service**. The principles don't change. The scale does.

Here's what you'll do:

- **Provision an AKS cluster** and install Flux with production configuration.
- **Set up GitHub Actions** so CI builds and validates, then hands off to GitOps for deployment.
- **Deploy to the cloud** using the same Git-driven workflow you just proved locally.

Same loop. Bigger stage.

**Ready to take it to the cloud?** [Continue to Day 3 →](./Day-3-GitOps-on-AKS-Self-Healing-Cloud-Scale.md)
