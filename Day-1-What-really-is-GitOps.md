# Day-1 What really is GitOps?

## What is **GitOps**?
GitOps is a set of practices, guidelines, and principles for managing cloud-native systems powered by Kubernetes. It defines every aspect of a cloud-native system, such as the underlying infrastructure, application configuration, and policies, as code.
GitOps uses Git as its foundation. Code is stored in Git repositories, version controlled, and collaborated on using Git, hence the name "GitOps".
## What **GitOps** is not?
GitOps is not a single tool or a technology, nor is it a replacement for existing CI/CD tools or Infrastructure as Code (IaC) tools. Instead, GitOps is an approach that uses existing tools such as CI/CD and IaC tools in order to achieve a desired end state of the system.
## Why is **GitOps** gaining popularity between Cloud Native Systems developers?
GitOps enables developers to use the same Git and Git-based tools they use to develop applications to also build and maintain Cloud Native Systems. This enables developers to collaborate on code used to build cloud native systems in a similar way to how they collaborate on application code. Code is version controlled, tested, and maintained in a compliant state with security and audit requirements. Changes to the system are made by pull requests and can be easily rolled back in case of a failure, ensuring the system is always kept in a desired end state.

GitOps enables developers to orchestrate every aspect of cloud native systems, not just one aspect like Infrastructure as Code (IaC), Configuration as Code, or policies. Instead, it combines all of these into one workflow to maintain cloud native systems using Git repositories as a single source of truth.
## The origin of **GitOps**
The term "GitOps" was coined by [Alexis Richardson](https://twitter.com/monadic) from [Weaveworks](https://www.weave.works/) in a blog post titled "[GitOps Operations by Pull Request](https://www.weave.works/blog/gitops-operations-by-pull-request)". It described how Weaveworks maintained their Kubernetes-based application, Weave Cloud, on AWS EC2 using a Git-based workflow they called "GitOps".
He also defined the four principles of GitOps, which we will talk about in Day-2.
