# Cyclops x ArgoCD example

This repository guides you through the setup of Cyclops with ArgoCD on multiple charts.

With this setup, you can manage multiple products (helm charts) where each has multiple tenancies.

The setup is built on the `app-of-apps` pattern with ArgoCD, where the leaf applications are not Argo Applications, but Cyclops Modules. With Cyclops Modules as leaf nodes, you get a more friendly way of deploying applications instead of pure gitops.

In this case, Cyclops does not deploy applications directly to the cluster, but pushes the Module configuration to a git repository where it can be synced into the Kubernetes cluster by ArgoCD.

The repository implements the following hierarchy of apps:

![Screenshot 2025-03-27 at 11.05.05.png](attachment:169dfd88-ade9-4a26-bd29-957c6351e2d3:Screenshot_2025-03-27_at_11.05.05.png)

This repository holds two Helm charts (`/product-one` and `/product-two`) that each represent different applications a company wants to deploy.

- `root-app.yaml` - holds configuration for the root application
- `/products` - folder with Argo Applications that point to folders with Cyclops Module configurations
- `/modules` - will be created once you create your first Module. Folder with Cyclops Modules where Cyclops will push configuration

## Prerequisites

- **fork repository** - since Cyclops needs credentials to push configuration to a repository, you can fork this repository and give it a token to push configuration to your fork
- a running Kubernetes cluster
- `kubectl` installed and set up to manage a cluster
- running ArgoCD - you can check how to install it [here](https://argo-cd.readthedocs.io/en/stable/#getting-started)

## Install Cyclops

**Cyclops** can be installed with a single command:

```jsx
kubectl apply -f https://raw.githubusercontent.com/cyclops-ui/cyclops/v0.19.0-rc.3/install/cyclops-install.yaml
```

To enable Cyclops to push the Module configuration to a git repository, you will need to provide a token for it to authenticate. In case of a GitHub repo, you can follow these [steps](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#creating-a-personal-access-token-classic).

To inject that token, you will need to create a Kubernetes secret and a TemplateAuthRule, which is a Cyclops CRD to define which credentials need to be used for certain repos. More on that [here](https://cyclops-ui.com/docs/templates/private_templates).

Once you have generated your GitHub token, create a secret in your Kubernetes cluster with your username and GH token.

```
kubectl create secret generic my-cyclops-secret \
    -n cyclops \
    --from-literal=username=<github-username> \
    --from-literal=password=<github-token>
```

You will also need a TemplateAuthRule that will reference the newly created secret. You can create a file with its configuration and apply it to the cluster.

Create the file:

```yaml
# template_auth_rule.yaml

apiVersion: cyclops-ui.com/v1alpha1
kind: TemplateAuthRule
metadata:
  name: private-repo-rule
  namespace: cyclops            # has to be in cyclops namespace
spec:
  repo: <github-repo>           # github repo to push configuration to
  password:
    name: my-cyclops-secret     # name of the previously created secret
    key: password
  username:
    name: my-cyclops-secret     # name of the previously created secret
    key: username
```

Apply to the cluster:

```jsx
kubectl apply -f template_auth_rule.yaml
```

You can now expose Cyclops service outside of the cluster to access it. You can do it by port-forwarding it:

```jsx
kubectl port-forward -n cyclops svc/cyclops-ui 3000:3000
```

Cyclops should now be accessible on http://localhost:3000/

## Deploying applications

Before deploying the first application, you can add the templates that reference the Helm chart so they are available in Cyclops. You will need to create templates for `product-one` and `product_two` with the following YAML that you would need to apply to the cluster. **Make sure you enter your repository name in the YAML** (`<your-forked-repo>` in the code below) so Cyclops pushes the configuration to the right repository:

```yaml
apiVersion: cyclops-ui.com/v1alpha1
kind: TemplateStore
metadata:
  name: product-one
  namespace: cyclops
spec:
  repo: <your-forked-repo>
  path: product-one
  version: main
  sourceType: git
  enforceGitOpsWrite:
    repo: <your-forked-repo>
    path: modules/product-one
    version: main
---
apiVersion: cyclops-ui.com/v1alpha1
kind: TemplateStore
metadata:
  name: product-two
  namespace: cyclops
spec:
  repo: <your-forked-repo>
  path: product-two
  version: main
  sourceType: git
  enforceGitOpsWrite:
    repo: <your-forked-repo>
    path: modules/product-two
    version: main

```

The `enforceGitOpsWrite` will make sure that you don't have to manually input where to push the configuration to. Each time a Module is created from the template with the `enforceGitOpsWrite`, it will be pushed to the specified repo and folder.

---

⚠️ **Warning:**
> **Make sure that the Argo applications in the `/products` folder and `root-app.yaml` file reference your repo fork**

You can now install the root app to create the hierarchy shown above:

```bash
kubectl apply -f root-app.yaml
```

You can now sync the root and product Applications in ArgoCD to get all the Modules deployed.

**Argo Applications for products (`product-one` and `product-two`) are in error state since they are pointing to empty folders which will change once you deploy a Module for a certain product.**

### Deploying new Modules

If you want to create a new Module:

1. Open Cyclops on http://localhost:3000/
2. Select `Add module` button
3. Select the product template you want to deploy, and configure it
4. Click `Deploy`
5. Sync the Argo app for a specific product (if auto-sync enabled, it will be synced automatically)

### Batch changes

Cyclops renders UIs based on the `values.schema.json`. If a field is not listed in the schema, the field will not be rendered. In that case, the value from `values.yaml` will be used.

In the example Helm chart, the `general.version` is not included in the schema and can be controlled only from the Helm chart `values.yaml`.

To update the `general.version` on multiple modules at once you can:

1. Update the `values.yaml` with the new `general.version`
2. Commit and push to git
3. Select modules you want to apply the new version to and hit Reconcile

   ![Screenshot 2025-03-27 at 12.25.13.png](attachment:6dd2014b-7f32-4032-9308-001914c0aec6:Screenshot_2025-03-27_at_12.25.13.png)

4. Cyclops will now pull the latest Helm chart and its values and redeploy all the selected Modules
