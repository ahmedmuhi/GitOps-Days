# GitOps Pipeline

GitOps pipeline is a series of automated processes that take your application from source code to production, using Git as the single source of truth for both infrastructure and application configuration. In the previous blog posts I talked about What GitOps is and What are the GitOps Principals, Now let's discuss the pipeline itself.

## The Anatomy of a GitOps Pipeline

A Typical GitOps Pipeline consist of the following components:

1. Git Repository:

At the heart of any GitOps pipeline is the Git repository. All your application code, configuration, and infrastructure-as-code definitions (like Kubernetes manifests or Helm charts) reside here. Every change in the repo can potentially trigger a GitOps-driven action.

2. GitOps Operator (e.g., Flux CD, Argo CD):

A key part of the GitOps pipeline is the GitOps operator software, such as Flux CD or Argo CD. This tool keeps an eye the Git repository. If something changes in the repo, like you update a setting or the application code, this operator springs into action to update the live environment accordingly.

3. Container Registry:

Think of this as a storage space for all your container images. Once you've made changes and built a new version of your application as a container, it gets stored here. Later on, when deploying or updating the app, the GitOps Operator will pull the right container image from here based on the definitions in the Git repo.

4. Container Orchestration System (e.g., Kubernetes):

This is the actual environment where your application actually runs. Kubernetes, takes the rules and settings you've defined (in Kubernetes manifests or Helm charts) and makes sure your application runs that way.

5. CI/CD Pipeline:

This includes:

Continuous Integration (CI): This is about taking code changes, testing them, and then creating a container image.
Continuous Deployment/Delivery (CD): Here, traditionally, we're talking about deploying the containers. But in GitOps, this part usually just updates Kubernetes manifests or Helm charts with new container image information. After that, the GitOps Operator takes over and does the deployment.

## GitOps in Action: Walking Through an Example

Let's use an example of a web app running on Kubernetes to see how this all fits together:

![GitOps Pipeline Diagram](/Assets/gitops_pipeline.png)

* You've got your web app, and it's been turned into a container. This container image is stored in the Container Registry.

* You've also got Kubernetes Manifests & Helm Charts, which are like the instruction manuals for how your app should run on Kubernetes, e.g., how many replicas you want, what network policies to apply, etc.

* All of these instructions, along with your application's code, are in a Git Repository. This captures how you want your app to run or look on Kubernetes. This is essentially what is called the "desired state" of your application.

* Now, if you make a change in this Git Repository, tools like Flux CD or ArgoCD comes into play.

Let's say you update your application's code. Here's the sequence of events:

1. When you update the application code and commit it to your Git repo, typically, a CI pipeline process is triggered. This could be done using tools like Azure DevOps, GitHub Actions, GitLab CI/CD, etc.

2. CI Process:

    * Your CI pipeline first runs tests on your code to ensure everything is functioning as expected.
    * If tests pass, the CI pipeline then builds a new container image containing the updated application code.
    * This new image is then pushed to your container registry with a new tag.
    * Once the new container image is in the registry, the reference to this new image in your Kubernetes Manifest or Helm Charts needs to be updated.

3. Once the Kubernetes manifest or Helm chart in the Git repository is updated with the new image tag, this is considered a change in your Git repo tools like Flux CD detects this change, and will then pulls the new container image from the registry and updates the deployment in Kubernetes to use this new image.

Even though the change originated from a code update and not a direct change to the environment, the updated image tag in the manifest or Helm chart is considered a change to the desired state of your application's environment.

The beauty of this approach is that to deploy or modify your application, all you need to do is change your Git repository. GitOps Operators will ensure that whatever is in the repository gets reflected in your Kubernetes environment. No need for manual commands or scripts; just update your repository, and the rest is handled automatically!
