**[OpenGitOps](https://opengitops.dev/) outlines four fundamental principles of GitOps, as below:**

**1. Declarative**: A system managed by GitOps must have its desired state expressed declaratively.

The first principle emphasizes that infrastructure and applications configuration should be declarative. This means that instead of writing scripts and commands that specify how your system should be set up, you write a description of what you want the end state of your system to look like.

This makes it easy to manage complex systems because you can see exactly what state everything should be in just by looking at your code.

**2. Versioned and Immutable**: Desired state is stored in a way that enforces immutability, versioning and retains a complete version history.

Another key principle of GitOps is that every change to infrastructure and applications configuration is committed to Git as a new version, and previous versions are never modified. This makes it easy to roll back changes if something goes wrong and ensures that everyone is always working from the same version of the truth.

**3. Pulled Automatically**: Software agents automatically pull the desired state declarations from the source.

The third principle of GitOps is that changes should be pulled automatically from Git, rather than pushed to the system being managed. This means that any changes to the Git repository will automatically trigger the deployment of those changes to the system being managed.

This helps ensure that the systems are always up to date with the latest changes and reduce the risk of manual errors.

**4. Continuously Reconciled**: Software agents continuously observe actual system state and attempt to apply the desired state.

The final principle of GitOps is that the system being managed should be continuously reconciled with the desired end state described in Git. This means that any differences between the current state of the system and the desired state in Git should be automatically detected and corrected, to reduce the risk of drifts.
