# Day 4 – Production GitOps Patterns: Closing the Loop

> **What you'll need:** Everything from Day 3, plus a GitHub account with Actions enabled (free on all plans).
> **Time:** ~1 hour.

You've proved the reconciliation loop works — locally and in the cloud. You can deploy through Git, scale through Git, and watch the cluster heal itself when things drift. But there's one piece of the picture you haven't built yet.

Go back to the workflow diagram from Day 1. A developer pushes code. CI builds and tests it. CI updates the config repo with a new image tag. Flux picks up the change and reconciles. That's the full lifecycle — and so far, you've only done the second half by hand. You edited YAML, pushed it yourself, and watched Flux deploy. The pipeline that's supposed to feed the loop has been missing.

Today we close that gap.

You'll build a GitHub Actions pipeline that takes a code change, builds a container image, pushes it to a registry, and updates your config repo automatically. Then you'll watch Flux pick up that change and deploy it — without you touching a single manifest. That's the Day 1 diagram, end to end, running for real.

After that, we'll look at two patterns that every team hits when they take GitOps beyond a single app on a single cluster: how to structure your repo for multiple environments, and how to handle secrets when everything lives in Git.

One lab. Two patterns. By the end, you'll have seen the full GitOps lifecycle and know how to scale it.

## The missing piece

Let's be specific about what's been missing — and what we're about to build.

In Day 1, we described the full GitOps workflow with two repositories:

- **Application repo** — where developers push code. CI builds it, tests it, creates a container image, and pushes it to a registry. Then CI does one last thing: it updates the config repo with the new image tag.
- **Configuration repo** — where Flux watches. It detects the tag change and reconciles the cluster to match.

The two repos meet at a single point: the image tag in the config repo. CI writes it. Flux reads it.

In Days 2 and 3, you played both roles yourself. You were the developer *and* the pipeline. You edited the Deployment manifest by hand, changed the replica count or the image tag, committed, pushed, and watched Flux reconcile. That was the right way to learn — it kept the focus on Flux and the reconciliation loop.

But in the real world, no one edits image tags in YAML files by hand. A pipeline does it. And until you've seen that pipeline run, the left half of the Day 1 diagram is still theory.

Here's what we're going to build:
```
Developer pushes code
        ↓
GitHub Actions builds container image
        ↓
Pushes image to GitHub Container Registry (ghcr.io)
        ↓
Updates image tag in config repo (your GitOps-Days fork)
        ↓
Flux detects the change
        ↓
Cluster reconciles — new image running
```

Every arrow, automated. By the end of this lab, pushing a code change will result in a new container running in your cluster — and you won't touch a single manifest along the way.

Let's set it up.

## Set up the application repo

The two-repo pattern from Day 1 means the application code and the deployment configuration live in separate repositories. You already have the config repo — your GitOps-Days fork. Now you need an application repo.

We'll keep this deliberately simple. The app is a static page served by nginx — just enough to build a container image and see a visible change when it deploys.

### Create the repo on GitHub

1. Go to [github.com/new](https://github.com/new).
2. Name it `hello-gitops`.
3. Set it to **Public** (so GitHub Container Registry works without extra configuration).
4. Check **Add a README file**.
5. Click **Create repository**.

Clone it locally:
```shell
git clone https://github.com/YOUR-USERNAME/hello-gitops.git
cd hello-gitops
```

### Add the application code

Create a simple HTML file that will be your app's "code":
```shell
mkdir -p src
```

Create `src/index.html`:
```html
<!DOCTYPE html>
<html>
<body>
  <h1>Hello from GitOps!</h1>
  <p>Version: 1.0 — deployed automatically by Flux.</p>
</body>
</html>
```

This is the file you'll change later to prove the full pipeline works. When you update the message and push, CI will build a new image, update the config repo, and Flux will deploy it — all without you touching a manifest.

### Add a Dockerfile

Create `Dockerfile` in the repo root:
```dockerfile
FROM nginx:alpine
COPY src/index.html /usr/share/nginx/html/index.html
```

Two lines. It takes the official nginx image and copies your HTML file into it. That's the entire application.

### Push everything
```shell
git add .
git commit -m "Add app source and Dockerfile"
git push
```

### Checkpoint: application repo ready

You should now have:
```
hello-gitops/
├── README.md
├── Dockerfile
└── src/
    └── index.html
```

This is your **application repo** — the left side of the Day 1 diagram. It has code and a Dockerfile, but no Kubernetes manifests. Those live in the config repo, and the pipeline we're about to build is what connects the two.

Let's build that pipeline.

## Build the CI pipeline

This is the lab. By the end of this section, pushing a code change to your application repo will automatically result in a new container running in your cluster.

### Prepare your environment

You need a cluster with Flux watching your config repo. You did this in Day 2 — let's set it up quickly.

**Spin up a local cluster:**
```shell
kind create cluster --name gitops-pipeline-demo
```

**Create a Day 4 workspace in your config repo:**
```shell
cd ~/GitOps-Days
mkdir -p student-work/YOUR-USERNAME/day4/hello
```

Create `student-work/YOUR-USERNAME/day4/hello/namespace.yaml`:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: hello
```

Create `student-work/YOUR-USERNAME/day4/hello/deployment.yaml`:
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
          image: ghcr.io/YOUR-USERNAME/hello-gitops:placeholder
          ports:
            - containerPort: 80
```

Notice the image line: `ghcr.io/YOUR-USERNAME/hello-gitops:placeholder`. That tag will be updated by the pipeline — it's the handoff point between CI and GitOps. For now, it's a placeholder that Flux will try (and fail) to pull. That's fine — the pipeline will fix it in a moment.

Create `student-work/YOUR-USERNAME/day4/hello/service.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello
  namespace: hello
spec:
  selector:
    app: hello
  ports:
    - port: 80
      targetPort: 80
```

Push your workspace:
```shell
git add student-work/
git commit -m "Create Day 4 workspace with GHCR image reference"
git push
```

**Install Flux and connect it:**
```shell
flux install

flux create source git gitops-loop-demo \\
  --url=https://github.com/YOUR-USERNAME/GitOps-Days.git \\
  --branch=main \\
  --interval=30s

flux create kustomization hello-app \\
  --source=GitRepository/gitops-loop-demo \\
  --path="./student-work/YOUR-USERNAME/day4/hello" \\
  --prune=true \\
  --interval=1m
```

Flux is now watching your Day 4 workspace. The Deployment will show `ImagePullBackOff` because the `placeholder` tag doesn't exist yet — that's expected. The pipeline we're about to build will push a real image and update that tag.

### Add a secret for cross-repo access

Your pipeline runs in the `hello-gitops` repo but needs to push a commit to your `GitOps-Days` repo. That requires a Personal Access Token — you already have one from Day 3.

1. Go to your `hello-gitops` repo on GitHub.
2. Navigate to **Settings → Secrets and variables → Actions → New repository secret**.
3. Name it `CONFIG_REPO_TOKEN`.
4. Paste your GitHub PAT (the same one you used in Day 3).

This is how the pipeline will authenticate when it updates the config repo. The token never appears in code — GitHub Actions injects it securely at runtime.

### Create the workflow

Back in your `hello-gitops` repo:
```shell
cd ~/hello-gitops
mkdir -p .github/workflows
```

Create `.github/workflows/build-and-update.yml`:
```yaml
name: Build and Update GitOps

on:
  push:
    branches: [main]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout application code
        uses: actions/checkout@v4

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push container image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository_owner }}/hello-gitops:${{ github.sha }}

      - name: Update config repo with new image tag
        run: |
          git clone https://x-access-token:${{ secrets.CONFIG_REPO_TOKEN }}@github.com/${{ github.actor }}/GitOps-Days.git config-repo
          cd config-repo
          sed -i "s|image: ghcr.io/.*/hello-gitops:.*|image: ghcr.io/${{ github.repository_owner }}/hello-gitops:${{ github.sha }}|" \\
            student-work/${{ github.actor }}/day4/hello/deployment.yaml
          git config user.name "github-actions"
          git config user.email "github-actions@github.com"
          git add .
          git commit -m "Update hello-gitops image to ${{ github.sha }}"
          git push
```

Let's walk through what this does:

**Trigger** — runs on every push to `main`. That's the developer's action.

**Build and push** — checks out the code, logs into GitHub Container Registry, builds the Docker image, and pushes it tagged with the commit SHA. The SHA makes every image traceable back to the exact code that produced it.

**Update config repo** — clones your GitOps-Days fork, finds the Deployment manifest, replaces the image tag with the new SHA, commits, and pushes. This is the handoff. From this point on, Flux takes over.

### Push the workflow and trigger the pipeline
```shell
git add .
git commit -m "Add CI pipeline for GitOps"
git push
```

This push triggers the workflow immediately — because the workflow runs on pushes to `main`, and you just pushed to `main`.

Go to your `hello-gitops` repo on GitHub and click the **Actions** tab. You should see the workflow running.

> [!TIP]
> The first run takes 1–2 minutes. Subsequent runs are faster because Docker layers get cached.

### Watch the full cycle

Once the Actions run completes, switch to your `GitOps-Days` repo on GitHub. You should see a new commit from `github-actions` with the message "Update hello-gitops image to [SHA]."

That commit is the handoff. CI just wrote to your config repo. Now Flux reads it.

Back in your terminal, watch Flux pick up the change:
```shell
kubectl get deployment hello -n hello -w
```

Within about a minute, you'll see the Deployment update. The `ImagePullBackOff` disappears, replaced by a running pod with your custom image.

Verify it's your app:
```shell
kubectl port-forward -n hello svc/hello 8080:80
```

Open [http://localhost:8080](http://localhost:8080). You should see: **"Hello from GitOps! Version: 1.0 — deployed automatically by Flux."**

That page was served by an image that was built by CI, pushed to a registry, referenced in a config repo commit, detected by Flux, and deployed to your cluster. You didn't touch a manifest. The pipeline connected the two halves.

Press `Ctrl+C` to stop the port-forward.

### Prove it again — change the code

Open `src/index.html` in your `hello-gitops` repo and change:
```html
<p>Version: 1.0 — deployed automatically by Flux.</p>
```

to:
```html
<p>Version: 2.0 — the full loop, automated.</p>
```

Commit and push:
```shell
git add src/index.html
git commit -m "Update to version 2.0"
git push
```

Now watch:

1. **GitHub Actions** — the workflow starts building the new image.
2. **GitOps-Days repo** — a new commit appears updating the image tag.
3. **Your cluster** — Flux detects the tag change and reconciles.

Port-forward again and refresh your browser:
```shell
kubectl port-forward -n hello svc/hello 8080:80
```

You should see: **"Version: 2.0 — the full loop, automated."**

You pushed code. CI built it. The pipeline updated the config repo. Flux deployed it. You never opened a manifest.

Every arrow in the Day 1 diagram — automated.

## Pattern: multi-environment structure

You've seen the full lifecycle work for one app on one cluster. But real teams don't run one environment — they run dev, staging, and production. The app is the same. The configuration varies: fewer replicas in dev, stricter resource limits in production, different image tags as changes graduate through environments.

The question is: how do you structure your config repo so the same app can be deployed differently across environments without duplicating manifests?

### The problem with copying

The naive approach is to create a complete set of manifests for each environment:
```
clusters/
├── dev/hello/
│   ├── namespace.yaml
│   ├── deployment.yaml    ← replicas: 1
│   └── service.yaml
├── staging/hello/
│   ├── namespace.yaml
│   ├── deployment.yaml    ← replicas: 2
│   └── service.yaml
└── prod/hello/
    ├── namespace.yaml
    ├── deployment.yaml    ← replicas: 5
    └── service.yaml
```

This works — until you need to change the container port or add a health check. Now you're making the same edit in three places and hoping you don't miss one. The manifests drift from each other the same way clusters drift from Git. Duplication is just drift in a different form.

### The overlay pattern

Kustomize solves this with a **base plus overlays** structure. You write the common configuration once and override only what changes per environment:
```
hello-app/
├── base/
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── deployment.yaml       ← replicas: 1 (default)
│   └── service.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml    ← uses base as-is
    ├── staging/
    │   ├── kustomization.yaml    ← patches replicas to 2
    │   └── replica-patch.yaml
    └── prod/
        ├── kustomization.yaml    ← patches replicas to 5, adds resource limits
        └── prod-patches.yaml
```

The base contains everything that's shared. Each overlay points back to the base and adds only what's different. Here's what the staging overlay looks like:

`overlays/staging/kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
patches:
  - path: replica-patch.yaml
```

`overlays/staging/replica-patch.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
  namespace: hello
spec:
  replicas: 2
```

That's it. The staging overlay says: "take everything from base, but set replicas to 2." The namespace, the service, the container spec — all inherited. If you change the container port in base, it changes everywhere. One source of truth, environment-specific variations layered on top.

### How Flux uses this

In practice, each environment has its own Flux Kustomization pointing at the right overlay folder:

- The **dev** cluster's Flux watches `overlays/dev/`
- The **staging** cluster's Flux watches `overlays/staging/`
- The **production** cluster's Flux watches `overlays/prod/`

Each cluster reconciles only its own overlay. Changes to base propagate to every environment on the next reconciliation cycle. Changes to an overlay affect only that environment.

This is the same reconciliation loop you've been running all series — just pointed at different folders. Flux doesn't need special multi-environment features. The folder structure *is* the multi-environment strategy.

### Promoting changes across environments

The typical flow: your CI pipeline updates the image tag in `overlays/dev/`. The team validates it in dev. When they're satisfied, they open a PR that copies the new tag into `overlays/staging/`. After staging passes, another PR promotes it to `overlays/prod/`.

Every promotion is a Git commit. Every commit is reviewable, reversible, and auditable. The loop handles the rest.

You don't need to build this today. But when your team is ready to move beyond a single environment, this is the pattern — and you now have enough Flux experience to implement it.

## What's next — your GitOps roadmap

You started this series four days ago not knowing what GitOps was. Now you've built it end to end.

Day 1 gave you the mental model — watch, compare, reconcile. Day 2 proved it on your laptop. Day 3 proved it in the cloud. And today, you closed the last gap: the CI pipeline that feeds the loop, and the repo structure that scales it across environments.

The Day 1 diagram isn't theory anymore. Every arrow is something you've built, run, and verified with your own hands.

**You have everything you need to bring GitOps to your team.**

There's more to explore when you're ready:

- **Secrets in GitOps** — you can't put passwords in Git, but tools like [Sealed Secrets](https://sealed-secrets.netlify.app/) and [SOPS](https://github.com/getsops/sops) solve this by encrypting sensitive data before it reaches the repo. Worth exploring when you bring real workloads into your GitOps workflow.
- **Image automation** — Flux's [Image Reflector and Automation controllers](https://fluxcd.io/flux/guides/image-update/) can watch a container registry and update your config repo automatically when a new image appears, removing the `sed` step from your pipeline entirely.
- **Progressive delivery** — [Flagger](https://flagger.app/) works alongside Flux to do canary deployments and automated rollbacks based on metrics.
- **Policy enforcement** — tools like [OPA Gatekeeper](https://open-policy-agent.github.io/gatekeeper/) let you define rules about what's allowed in your cluster, catching misconfigurations before they're applied.

These are signposts, not assignments. Pick the one that matters most to your team and go from there.

**Thank you for building with GitOps-Days.** If you found this series useful, share it with your team or post about your journey with `#GitOpsDays`. Found something that could be better? [Open an issue](https://github.com/ahmedmuhi/GitOps-Days/issues) or send a pull request — this series is built in the open, and your feedback makes it better.

> When you're done with today's lab, clean up your local cluster:
> ```shell
> kind delete cluster --name gitops-pipeline-demo
> ```