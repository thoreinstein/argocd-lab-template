# ArgoCD Lab Template

Welcome to the ArgoCD Lab! This template will guide you through setting up a local Kubernetes cluster using Kind, deploying ArgoCD, exposing its UI, and implementing the App of Apps pattern. This guide is designed for beginners, so don't worry if you're new to Kubernetes or ArgoCD. Let's get started!

## Prerequisites

Before you begin, ensure you have the following installed:

- [Docker](https://docs.docker.com/get-docker/): Docker is a platform used to develop, ship, and run applications inside containers. Containers are lightweight, portable, and efficient, making them perfect for modern microservices architectures.
- [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/): Kind (Kubernetes IN Docker) is a tool for running local Kubernetes clusters using Docker container nodes. It's great for testing and development purposes.
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/): kubectl is the command-line tool for interacting with Kubernetes clusters. You'll use it to deploy and manage applications on your Kubernetes cluster.

## Setup Steps

### 1. Install Docker

Follow the official [Docker installation guide](https://docs.docker.com/get-docker/) for your operating system. Docker will allow you to create and manage containers, which are essential for running the applications in this lab.

### 2. Install Kind

Install Kind by following the [Kind installation guide](https://kind.sigs.k8s.io/docs/user/quick-start/#installation). Kind will help you set up a local Kubernetes cluster using Docker containers, which is perfect for learning and experimentation.

### 3. Install kubectl

Install kubectl by following the [kubectl installation guide](https://kubernetes.io/docs/tasks/tools/install-kubectl/). kubectl will be your primary tool for managing Kubernetes resources and deploying applications to your cluster.

### 4. Setting up your environment

#### 4.1. Create the repository from a template

In this scenario, the developers manage the application Helm charts in version control. To represent this in the lab, you will create a repository from a template containing the application Helm charts.

1. Click [this link](https://github.com/tensure/argocd-lab-template/generate) or click "Use this template" from the repo main page.
2. Ensure the desired "Owner" is selected (e.g., your account and not an organization).
3. Enter `argocd-lab-template` for the "Repository name".
4. Then click "Create repository from template".

Next, you have a couple of files to update:

1. For the `guestbook.yaml` and `portal.yaml` files in `apps/`:
2. Fix the `spec.source.repoURL` by replacing `<github-username>` with your GitHub username (or the organization name if using one).
3. Fix the `spec.destination.name` by replacing `<environment-name>` with your environment name.
4. Commit the changes to the `main` branch.

```diff
  spec:
    project: default
    source:
--    repoURL: 'https://github.com/<github-username>/argocd-lab-template'
++    repoURL: 'https://github.com/tensure/argocd-lab-template'
      ...
```

### 5. Create a Cluster Using Kind

Create a Kubernetes cluster using Kind with the following command:

```bash
kind create cluster --name argocd-lab-template
```

This command will create a local Kubernetes cluster named `argocd-lab-template`. A cluster is a set of nodes that run containerized applications managed by Kubernetes.

### 6. Configure kubectl to Use the Kind Context

Ensure `kubectl` is set up to use the correct context:

```bash
kubectl cluster-info --context kind-argocd-lab-template
```

Verify access to the cluster:

```bash
kubectl get nodes
```

The `kubectl get nodes` command should list the nodes in your cluster, confirming that kubectl is correctly configured and can communicate with your cluster.

### 7. Deploy ArgoCD on the Cluster

ArgoCD is a declarative, GitOps continuous delivery tool for Kubernetes. It will help you manage your Kubernetes applications with Git as the source of truth.

Deploy ArgoCD using the following steps:

1. **Create a Namespace for ArgoCD:**

    ```bash
    kubectl create namespace argocd
    ```

   Namespaces in Kubernetes are a way to organize and separate resources within a cluster.

2. **Install ArgoCD:**

    ```bash
    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
    ```

   This command applies the ArgoCD installation manifest to your cluster, setting up all the necessary resources.

### 8. Patch ArgoCD to Enable Helm

Helm is a package manager for Kubernetes, which allows you to define, install, and upgrade even the most complex Kubernetes applications.

To enable Helm in ArgoCD, you need to patch the ArgoCD ConfigMap:

1. **Patch the ArgoCD ConfigMap:**

    ```bash
    kubectl patch cm argocd-cm -n argocd --type merge -p '{"data": {"kustomize.buildOptions": "--enable-helm"}}'
    ```

2. **Reload ArgoCD to Apply the Patch:**

    ```bash
    kubectl rollout restart deployment argocd-server -n argocd
    ```

   This step ensures that the changes to the ConfigMap take effect.

### 9. Expose the ArgoCD UI

To expose the ArgoCD UI, use port forwarding:

1. **Port Forward the ArgoCD Server Service:**

    ```bash
    kubectl port-forward svc/argocd-server -n argocd 8080:443
    ```

2. **Access the ArgoCD UI:**

    Open your browser and go to `https://localhost:8080`.

   Port forwarding allows you to access the ArgoCD UI from your local machine.


### 10. Retrieve the Initial Password for ArgoCD 

To log in for the first time, you'll need the initial password, which is stored in a Kubernetes secret:

1. **Get the Initial Admin Password:**

    ```bash
    kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 --decode
    ```
    > If there is `%` at the very end of that, ignore it. It's not part of the password.

2. **Login to the ArgoCD UI:**

    - Username: `admin`
    - Password: (the value you retrieved from the secret)

    If the password doesn't work sometimes a % is added to the end that is not actually part of the password.

## 11. Using Argo CD to Deploy Helm Charts

### 11.1. Create an Application in Argo CD

Before the introduction of Argo CD, the developers were manually deploying any Helm chart changes to the cluster. Now, using an Application, you will declaratively tell Argo CD how to deploy the Helm charts.

Start by creating an Application to deploy the `guestbook` Helm Chart from the repo.

1. Navigate to the Argo CD UI, and click "NEW APP".
2. In the top right, click "EDIT AS YAML".
3. Paste the contents of `apps/guestbook.yaml` from your repo.

   This manifest describes an Application:
    - The name of the Application is `guestbook`.
    - The source is your repo with the Helm charts.
    - The destination is the cluster connected by the agent.
    - The sync policy will automatically create the namespace.

4. Click "SAVE".
   
   At this point, the UI has translated the Application manifest into the corresponding fields in the wizard.

5. In the top left, click "CREATE".

   The new app pane will close and show the card for the Application you created. The status on the card will show "Missing" and "OutOfSync".

6. Click on the Application card titled `argocd/guestbook`.
   
   In this state, the Application resource tree shows the manifests generated from the source repo URL and path defined. You can click "APP DIFF" to see what manifests the Application rendered. Since auto-sync is disabled, the resources do not exist in the destination yet.

7. In the top bar, click "SYNC" then "SYNCHRONIZE" to instruct Argo CD to create the resources defined by the Application.

   The resource tree will expand as the Deployment creates a ReplicaSet that makes a pod, and the Service creates an Endpoint and EndpointSlice. The Application will remain in the "Progressing" state until the pod for the deployment is running.

   Afterwards, all the top-level resources (i.e., those rendered from the Application source) in the tree will show a green checkmark, indicating that they are synced (i.e., present in the cluster).

### 11.2. Syncing changes manually

An Application now manages the deployment of the `guestbook` Helm chart. So what happens when a developer wants to deploy a new image tag?

Well, instead of running `helm upgrade guestbook ./guestbook`, they will trigger a sync of the Application.

1. Navigate to your repo on GitHub, and open the file [guestbook/values.yaml](guestbook/values.yaml).

2. In the top right of the file, click the pencil icon to edit.

3. Update the `image.tag` to the `0.2` list.

4. Click "Commit changes...".

5. Add a commit message. For example `chore(guestbook): bump tag to 0.2`.

6. Click "Commit changes".

7. Switch to the Argo CD UI and go to the `argocd/guestbook` Application.

8. In the top right, click the "REFRESH" button to trigger Argo CD to check for any changes to the Application source and resources.

   The default sync interval is 3 minutes. Any changes made in Git may not apply for up to 3 minutes. You can also click on the diff button again on the bar to see what exactly is not in sync. 

9. In the top bar, click "SYNC" then "SYNCHRONIZE" to instruct Argo CD to deploy the changes.

   Due to the change in the repo, Argo CD will detect that the Application is out-of-sync. It will template the Helm chart (i.e., `helm template`) and patch the `guestbook` deployment with the new image tag, triggering a rolling update.

### 11.3. Enable auto-sync and self-heal for the guestbook Application

Now that you are using an Application to describe how to deploy the Helm chart into the cluster, you can configure the sync policy to automatically apply changes—removing the need for developers to manually trigger a deployment for changes that already made it through the approval processes.

1. In the top menu, click "DETAILS".
2. Under the "SYNC POLICY" section, click "ENABLE AUTO-SYNC" and on the prompt, click "OK".
3. Below that, on the right of "SELF HEAL", click "ENABLE".
4. In the top right of the App Details pane, click the X to close it.

   If the Application were out-of-sync, this would immediately trigger a sync. In this case, your Application is already in sync, so Argo CD made no changes.

### 11.4. Demonstrate Application auto-sync via Git

With auto-sync enabled on the `guestbook` Application, changes made to the `main` branch in the repo will be applied automatically to the cluster. You will demonstrate this by updating the number of replicas for the `guestbook` deployment.

1. Navigate to your repo on GitHub, and open the file [guestbook/values.yaml](guestbook/values.yaml).

2. In the top right of the file, click the pencil icon to edit.
3. Update the `replicaCount` to the `2` list.
4. In the top right, click "Commit changes...".
5. Add a commit message. For example `chore(guestbook): scale to 2 replicas`.
6. In the bottom left, click "Commit changes".
7. Switch to the Argo CD UI and go to the `argocd/guestbook` Application.
8. In the top right, click the "REFRESH" button to trigger Argo CD to check for any changes to the Application source and resources.

   You can view the details of the sync operation by, in the top menu, clicking "SYNC STATUS". Here it will display what "REVISION" it was for, what triggered it (i.e., "INITIATED BY: automated sync policy"), and the result of the sync (i.e., what resources changed).

### 11.5. Demonstrate Application self-heal functionality

In your organization, everyone has direct and privileged access to the cluster. Users may apply changes to the cluster outside of the repo due to the over-provisioned access. For example, applying a Helm chart change to the cluster without getting pushed to the repo first.

With self-heal enabled, Argo CD will reconcile any changes to the Application resources that deviate from the repo.

To demonstrate this:
1. From the `guestbook` Application page in the Argo CD UI:
2. Locate the `guestbook` deploy (i.e., Deployment) resource and click the three vertical dots on the right side of the card.
3. Then click "Delete".
4. Enter the deployment name `guestbook` and click "OK".

   Almost as quickly as you delete it, Argo CD will detect that the deploy resource is missing from the Application. It will briefly display the yellow circle with a white arrow to indicate that the resource is out-of-sync. Then automatically recreate it, bringing the Application back to a healthy status.

## 12. Managing Argo CD Applications declaratively

### 12.1. Create an App of Apps

One of the benefits of using Argo CD is that you are now codifying the deployment process for the Helm charts in the Application spec.

Earlier in the lab, you created the `guestbook` Application imperatively, using the UI. But what if you want to manage the Application manifests declaratively too? This is where [the App of Apps pattern](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/#app-of-apps-pattern) comes in.

1. Navigate to the Applications dashboard in the Argo CD UI, and click "NEW APP".
2. In the top right, click "EDIT AS YAML".
3. Paste the contents of `app-of-apps.yaml` (in the repo's root).

    ```yaml
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: applications
      namespace: argocd
    spec:
      destination:
        name: in-cluster
      project: default
      source:
        path: apps
        repoURL: https://github.com/<github-username>/argocd-lab-template # Update to your repo URL.
        targetRevision: HEAD
    ```

    This Application will watch the `apps/` directory in your repo which contains Application manifests for the `guestbook` and `portal` Helm charts.

4. Update `<github-username>` in `spec.source.repoURL` to match your GitHub username.
5. Click "SAVE".
6. Then, in the top left, click "CREATE".
7. Click on the Application card titled `argocd/applications`.

    At this point, the Application will be out-of-sync. The diff will show the addition of the `argocd.argoproj.io/tracking-id` label to the existing `guestbook` Application, which indicates that the "App of Apps now manages it".
   
   Joe K Note: I do not see this in my diff.

    ```diff
      kind: Application
      metadata:
    ++  annotations:
    ++    argocd.argoproj.io/tracking-id: 'applications:argoproj.io/Application:argocd/guestbook'
        generation: 44
        labels:
          ...
          path: guestbook
          repoURL: 'https://github.com/<github-username>/argocd-lab-template'
        syncPolicy:
          automated: {}
    ```

    Joe K Note: this is what I see maybe this is old functionality or we have something missing?:
    ```diff
      kind: Application
      metadata:
        generation: 44
        labels:
      ++  app.kubernetes.io/instance: applications
          ...
          path: guestbook
          repoURL: 'https://github.com/<github-username>/argocd-lab-template'
        syncPolicy:
          automated: {}
    ```

    Along with a new Application for the `portal` Helm chart.

8. To apply the changes, in the top bar, click "SYNC" then "SYNCHRONIZE".

   From this Application, you can see all of the other Applications managed by it in the resource tree. Each child Application resource has a link to its view on the resource card.

## 13. Review

You have reached the end of the lab. You have an Argo CD instance deploying Helm charts from your repo into your cluster.

## Conclusion

You have successfully set up a Kind cluster, deployed ArgoCD, exposed its UI, retrieved the initial login password, enabled Helm, and implemented the App of Apps pattern. Feel free to explore further and customize the lab to suit your needs.

## Resources

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Kind Documentation](https://kind.sigs.k8s.io/)
- [kubectl Documentation](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

## Inspiration

A large part of this work was taken from [Morey-Tech's Managed ArgoCD Template](https://github.com/morey-tech/managed-argocd-lab-template-template/tree/main) which was designed around doing all this with the [Akuity Platfrom](https://akuity.io/). I've adapted it here for a completely self hosted and managed ArgoCD deployment.

If you encounter any issues or have questions, please open an issue in this repository.

Happy learning!
