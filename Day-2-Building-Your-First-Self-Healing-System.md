# Day 2 – Building Your First Self-Healing System

> **What you'll need:** Docker (24.0+), kind (0.25.0+), `kubectl` (1.32+), Git (2.40+), and a GitHub account. Links are in the setup section below if you're missing any.
>
> **Time:** ~1 hour.
>
> **On Windows:** every command in this series is written for bash or zsh. Run them in [WSL](https://learn.microsoft.com/windows/wsl/install) and everything works exactly as printed.

Yesterday you built a mental model. You followed the loop, watched it correct drift, and worked out where the GitOps controller stops and Kubernetes takes over.

Today you build it.

By the end of this session you'll have a cluster running on your laptop, a Git repository you own, and a controller keeping the two in sync. And you'll have broken it a few times to see what happens.

We'll get the environment up first. After that, everything is an experiment. Some changes will come from Git, some from `kubectl`. Some the GitOps controller will correct, and some it will ignore completely.

One question runs through all of them:

**Who should respond to this change?**

By the end of today you'll know which loop acts, why it acts, and — just as importantly — when it does nothing on purpose.

## Set up your workspace

Before we touch Kubernetes or Flux, let's get the logistics done in one pass — a repository you control, a copy of it on your machine, and today's application files. Then we build.

### Fork the repository

Go to [`https://github.com/ahmedmuhi/GitOps-Days`](https://github.com/ahmedmuhi/GitOps-Days) and click **Fork**, then **Create fork**.

Forking gives you a repository you can push to, and that matters more than it sounds. You'll be changing the desired state all day — editing manifests, committing, pushing — and the controller only reconciles what's actually in Git. You can't push to someone else's repository on GitHub, so forking is how you get a copy of these files that's yours to change.

Your fork will live at:

```text
https://github.com/YOUR-USERNAME/GitOps-Days
```

Every command from here on uses `YOUR-USERNAME` as a placeholder. Replace it with your actual GitHub username.

### Clone your fork

```shell
git clone https://github.com/YOUR-USERNAME/GitOps-Days.git
cd GitOps-Days
```

Check that you're pointing at your fork and not the original:

```shell
git remote -v
```

You should see your own username in both URLs:

```text
origin  https://github.com/YOUR-USERNAME/GitOps-Days.git (fetch)
origin  https://github.com/YOUR-USERNAME/GitOps-Days.git (push)
```

> [!IMPORTANT]
> If you see `ahmedmuhi` there instead, you've cloned this repository rather than your fork. Delete the folder and clone again with your own username — you won't be able to push otherwise, and pushing is how you'll drive the cluster today.

### Create your working folder

You'll work in your own folder for the whole series. The examples stay untouched, so when you pull updates later they land cleanly and nothing you've built gets stepped on.

```shell
mkdir -p my-work/day2
cp -r examples/day2/hello my-work/day2/
```

You should now have:

```text
my-work/day2/hello/
├── namespace.yaml
├── deployment.yaml
└── service.yaml
```

### What you're about to declare

Before we hand these files to a controller, open `deployment.yaml` and have a read.

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

One replica of a small NGINX image. The app itself doesn't matter today — that `replicas: 1` does. It's the number you'll change through Git, the number you'll change behind Git's back, and the number the controller keeps putting right.

### Checkpoint: workspace ready

Commit it. As far as the controller is concerned, a file that isn't in Git doesn't exist.

```shell
git add my-work/
git commit -m "Create Day 2 workspace"
git push
```

You've got a repository you control, a clone on your machine, three manifests, and all of it pushed. That's the logistics done — from here on, we build.

## Create your cluster

Now for the cluster the controller will eventually manage.

We'll use [kind](https://kind.sigs.k8s.io/) — Kubernetes in Docker — which runs a full Kubernetes cluster inside Docker containers. It comes up in about a minute, which is what makes it good for experiments like today's.

```shell
kind create cluster --name gitops-loop-demo
```

> [!IMPORTANT]
> The first time you run this, kind downloads the image it runs the cluster from — roughly 435 MB, expanding to about 1.5 GB on disk. It shows up as a single line in the output (`Ensuring node image (kindest/node:v1.36.1)`) with no progress bar, so on a slow or metered connection, budget for it. It's downloaded once and reused by every cluster you create afterwards.

On my laptop the whole thing took 65 seconds, download included.

### Checkpoint: cluster running

Check that Kubernetes is up:

```shell
kubectl get nodes
```

You should see something like:

```text
NAME                             STATUS   ROLES           AGE   VERSION
gitops-loop-demo-control-plane   Ready    control-plane   39s   v1.36.1
```

The part that matters is `STATUS: Ready`. If it says `NotReady`, give it a few seconds and try again — Kubernetes is still starting up.

> [!IMPORTANT]
> If the node stays `NotReady`, first check Docker is running:
>
> ```shell
> docker ps
> ```
>
> Also check Docker has at least 4 GB of memory available. If everything looks healthy and the cluster still won't start, recreate it:
>
> ```shell
> kind delete cluster --name gitops-loop-demo
> kind create cluster --name gitops-loop-demo
> ```

You've got a working Kubernetes cluster. It isn't running anything, it isn't connected to Git, and it has no idea a GitOps controller exists — right now it's just a Kubernetes cluster.

Let's change that.

## Install the controller

Your cluster is running, but nothing in it is watching Git. There's no controller, no repository to compare against, and no way to correct a difference even if one existed.

Flux is one of the two controllers you met yesterday. The other is [Argo CD](https://argo-cd.readthedocs.io/), and either would behave the same way through today's experiments. We're using Flux because it installs quickly and then gets out of the way, which keeps the focus on the loop rather than the tool.

Installing it happens in two places — the CLI on your machine, then the controllers inside your cluster.

### Install the Flux CLI

```shell
curl -s https://fluxcd.io/install.sh | sudo bash
```

> [!TIP]
> For Homebrew, Chocolatey, and other installation methods, see the [Flux installation documentation](https://fluxcd.io/flux/installation/).

Confirm the CLI is available:

```shell
flux --version
```

This is a tool on your laptop, not part of the loop. You'll use it to create and inspect Flux resources, but once those resources exist, everything else happens inside Kubernetes.

### Install Flux into the cluster

Now the part that actually runs in Kubernetes:

```shell
flux install
```

This creates a `flux-system` namespace and deploys Flux's controllers into it. From here on, everything Flux does happens inside your cluster.

Expect about thirty lines of output as it creates custom resource definitions, service accounts, permissions and four Deployments, ending with `✔ install finished`. It took just over a minute.

Confirm the controllers are running:

```shell
kubectl get pods -n flux-system
```

You should see four pods becoming ready:

```text
NAME                                       READY   STATUS    RESTARTS   AGE
helm-controller-77bfb49bb5-vlcwj           1/1     Running   0          63s
kustomize-controller-5ddcb4c6d4-526kq      1/1     Running   0          63s
notification-controller-5f5d67f758-jvrn4   1/1     Running   0          63s
source-controller-666b89b45c-c77zf         1/1     Running   0          63s
```

The random suffixes will differ on your machine.

Yesterday, "the controller" was a single thing. Here it arrives as four pods, and two of them do today's work. The **source controller** fetches from Git, and the **kustomize controller** applies manifests to the cluster — fetching and applying are separate jobs. Argo CD splits them the same way, under different names.

The other two sit out today's lab. The **helm controller** manages Helm releases, and the **notification controller** sends events to systems like Slack.

### Checkpoint: installed and idle

```shell
flux check
```

```text
► checking prerequisites
✔ Kubernetes 1.36.1 >=1.33.0-0
► checking version in cluster
✔ distribution: flux-v2.9.3
✔ bootstrapped: false
► checking controllers
✔ helm-controller: deployment ready
► ghcr.io/fluxcd/helm-controller:v1.6.3
✔ kustomize-controller: deployment ready
► ghcr.io/fluxcd/kustomize-controller:v1.9.4
✔ notification-controller: deployment ready
► ghcr.io/fluxcd/notification-controller:v1.9.2
✔ source-controller: deployment ready
► ghcr.io/fluxcd/source-controller:v1.9.3
► checking crds
✔ alerts.notification.toolkit.fluxcd.io/v1beta3
✔ buckets.source.toolkit.fluxcd.io/v1
✔ externalartifacts.source.toolkit.fluxcd.io/v1
✔ gitrepositories.source.toolkit.fluxcd.io/v1
✔ helmcharts.source.toolkit.fluxcd.io/v1
✔ helmreleases.helm.toolkit.fluxcd.io/v2
✔ helmrepositories.source.toolkit.fluxcd.io/v1
✔ kustomizations.kustomize.toolkit.fluxcd.io/v1
✔ ocirepositories.source.toolkit.fluxcd.io/v1
✔ providers.notification.toolkit.fluxcd.io/v1beta3
✔ receivers.notification.toolkit.fluxcd.io/v1
✔ all checks passed
```

`bootstrapped: false` is expected — you installed Flux directly rather than through `flux bootstrap`, which is the production path we'll come to later in the series.

Every controller is healthy. Now ask each of the two what it's working on.

```shell
flux get sources git
```

```text
✗ no GitRepository objects found in "flux-system" namespace
```

```shell
flux get kustomizations
```

```text
✗ no Kustomization objects found in "flux-system" namespace
```

Nothing, both times. (Both commands also exit with an error code, which is just how `flux get` reports an empty list.)

The source controller has no repository to fetch and the kustomize controller has no manifests to apply. Flux is installed, healthy, and doing absolutely nothing — because a controller with no Git has no desired state, and without a desired state there's nothing to compare the cluster against.

Those two empty results tell you exactly what Flux is waiting for:

* Which repository should it watch?
* Which folder in that repository describes this cluster?

Let's answer both.

## Point Flux at your repository

Two questions, two objects. The first tells the source controller **which repository** to fetch, and the second tells the kustomize controller **which part of it** describes this cluster.

### Create the Git source

```shell
flux create source git gitops-loop-demo \
  --url=https://github.com/YOUR-USERNAME/GitOps-Days.git \
  --branch=main \
  --interval=30s
```

```text
✚ generating GitRepository source
► applying GitRepository source
✔ GitRepository source created
◎ waiting for GitRepository source reconciliation
✔ GitRepository source reconciliation completed
✔ fetched revision: main@sha1:3f22e3ba4e8deaae815fa76d3ddf6cb94cb74d7b
```

That's the **watch** phase from yesterday, made concrete. The source controller now fetches the `main` branch of your fork, stores an artifact of its contents inside the cluster, and refreshes that artifact every thirty seconds.

Thirty seconds is shorter than Flux's default minute. We're shortening it so the experiments move quickly instead of leaving you waiting around.

Confirm it worked:

```shell
flux get sources git
```

```text
NAME               REVISION             SUSPENDED   READY   MESSAGE
gitops-loop-demo   main@sha1:3f22e3ba   False       True    stored artifact for revision 'main@sha1:3f22e3ba'
```

`READY: True` means the source controller reached GitHub, fetched your repository, and stored an artifact from it. Your repository is now inside the cluster — and nothing else has happened. Flux knows **where** to read from, but not **what** in there should become this cluster.

### Create the Kustomization

That's what a Kustomization answers. Your fork holds the whole series — every day's lesson, the example manifests, the images — so the Kustomization tells Flux which folder in it describes this cluster.

```shell
flux create kustomization hello-app \
  --source=GitRepository/gitops-loop-demo \
  --path="./my-work/day2/hello" \
  --prune=true \
  --interval=1m
```

```text
✚ generating Kustomization
► applying Kustomization
✔ Kustomization created
◎ waiting for Kustomization reconciliation
✔ Kustomization hello-app is ready
✔ applied revision main@sha1:3f22e3ba4e8deaae815fa76d3ddf6cb94cb74d7b
```

That completes the loop. The flags mean:

* `--source` selects the Git source you just created.
* `--path` identifies the folder containing the desired state.
* `--prune=true` removes managed resources from the cluster when their declarations disappear from Git.
* `--interval=1m` compares the declared state with the cluster every minute and reconciles any difference.

Notice how the loop is assembled. The source controller fetches your repository every thirty seconds, and the kustomize controller compares that copy with the cluster every minute. Yesterday, watch, compare and reconcile looked like one continuous cycle — in Flux it's separate components, each on its own clock.

### Checkpoint: the loop is running

```shell
flux get kustomizations
```

```text
NAME        REVISION             SUSPENDED   READY   MESSAGE
hello-app   main@sha1:3f22e3ba   False       True    Applied revision: main@sha1:3f22e3ba
```

Look at the message. Not waiting, not pending — **applied**. Flux has already read the manifests at that path and put them on the cluster.

You didn't push a commit and you didn't run `kubectl apply`. You created an object describing what the cluster should contain, and the loop did the rest on its first pass.

Here's what happened. The source controller already had your repository stored, because it fetched it when you created the Git source. So when the Kustomization appeared, the kustomize controller read the three manifests in that folder and compared them with the cluster — where the namespace, the Deployment and the Service were all missing. Every declared resource was a difference, so Flux applied all three.

That's the same comparison you followed yesterday, with the numbers at their most extreme: desired is three resources, actual is none.

Have a look at what you've got:

```shell
kubectl get pods,svc -n hello
```

```text
NAME                         READY   STATUS    RESTARTS   AGE
pod/hello-7788d48f44-j2pbr   1/1     Running   0          19s

NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/hello   ClusterIP   10.96.21.198   <none>        80/TCP    19s
```

One pod and one service, exactly as your manifests declared — and you haven't run `kubectl apply` once.

Finally, prove it's real:

```shell
kubectl port-forward -n hello svc/hello 8080:80
```

Open **[http://localhost:8080](http://localhost:8080)**. You should see a plain-text response:

```text
Server address: 127.0.0.1:80
Server name: hello-7788d48f44-j2pbr
Date: 30/Jul/2026:09:04:03 +0000
URI: /
Request ID: 9dee19112b1e54dd94cf9a884e655ced
```

The server name is the pod name. Your application, running in Kubernetes, deployed entirely through Git and Flux.

Press `Ctrl+C` when you're finished.

The system is running. From here on, every command is an experiment.

## Make a change through Git

Flux deployed your app from existing files. Now let's prove that pushing a change to Git is all it takes to update your cluster.

Open `my-work/day2/hello/deployment.yaml` in your editor and change:

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
git add my-work/day2/hello/deployment.yaml
git commit -m "Scale hello app to 3 replicas"
git push
```

Now watch the cluster respond:

```shell
kubectl get deployment hello -n hello -w
```

Within thirty seconds, you'll see the replica count climb from 1 to 3:

```
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
hello   1/1     1            1           68s
hello   1/3     1            1           74s
hello   1/3     3            1           74s
hello   2/3     3            2           74s
hello   3/3     3            3           75s
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

Within a minute, you'll see this:

```
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
hello   3/3     3            3           102s
hello   3/5     3            3           103s      # manual change takes effect
hello   3/5     5            3           103s
hello   4/5     5            4           104s
hello   5/5     5            5           104s
hello   5/3     5            5           2m19s     # Flux corrects it
hello   3/3     3            3           2m19s
```

The manual scale took effect in about a second. Flux undid it 37 seconds later.

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

Over the next minute you'll see the namespace reappear, the Deployment recreate, pods spin up, and the service come back online. Everything restored — from Git.

Here's how that ran on a kind cluster with three replicas:

```
 1s   namespace Terminating   3 pods running
 6s   namespace Terminating   0 pods running
14s   namespace gone
62s   namespace Active        3 pods running
```

The `kubectl delete` command itself blocked for 11 seconds before returning. Deleting a namespace isn't instant — Kubernetes has to remove everything inside it first, and the namespace sits in `Terminating` until that finishes. Then the namespace was simply absent for another 48 seconds, until Flux's next comparison found three declared resources and none of them present.

Press `Ctrl+C` when you see everything running again.

This is the moment that earns the phrase "self-healing." The namespace and everything in it are declared in Git. When they disappeared from the cluster, the controller treated it the same way it treated the replica mismatch — a difference to be reconciled. The fix isn't special logic. It's just the loop, doing what it always does.

### Why the two cases take different times

You configured two intervals when you set up Flux:

- **Source interval (30s)** — how often the source controller checks Git for new commits.
- **Reconciliation interval (1m)** — how often the kustomize controller compares the cluster against the stored artifact.

It would be reasonable to assume every change waits for both. It doesn't, and the difference is worth understanding.

**A change you push to Git** waits only for the source interval. Once the source controller stores a new artifact, the kustomize controller doesn't sit and wait for its own next slot — it's watching the source, and reconciles as soon as the artifact changes. Three pushes measured on a laptop landed in 5, 7 and 30 seconds: never longer than the 30-second poll.

**Drift you cause with `kubectl`** produces no new artifact and no event, so nothing wakes the kustomize controller early. It waits for its next scheduled comparison. The two manual scales measured above took 37 and 42 seconds, both inside the 1-minute interval.

You can see both patterns in the events log:

```shell
flux events --for Kustomization/hello-app
```

```
3m10s  Normal  NewArtifact              GitRepository/gitops-loop-demo  stored artifact for commit 'Scale hello app to 3 replicas'
3m9s   Normal  ReconciliationSucceeded  Kustomization/hello-app         Reconciliation finished in 331.397798ms, next run in 1m0s
```

One second between the new artifact arriving and the Kustomization acting on it. That's not the interval — that's the controller reacting.

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
