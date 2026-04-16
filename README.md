## Installation

There are three ways to install this Blueprint:

- [1. Helm Chart](#method-1-helm-chart) 
- [2. Krateo Composable Operation](#method-2-krateo-composable-operation)
- [3. Krateo Composable Portal](#method-3-krateo-composable-portal)

---

### Method 1: Helm Chart

**Step 1: Download the default Helm chart values**

```sh
helm repo add marketplace https://marketplace.krateo.io
helm repo update marketplace
helm inspect values marketplace/github-scaffolding-with-composition-page --version 1.2.5 > ~/github-scaffolding-with-composition-page-values.yaml
```

**Step 2: Edit the values file**

Modify `~/github-scaffolding-with-composition-page-values.yaml` as shown below:

```yaml
argocd:
  namespace: krateo-system
  application:
    project: default
    source:
      path: chart/
    destination:
      server: https://kubernetes.default.svc
      namespace: fireworks-app
    syncEnabled: false
    syncPolicy:
      automated:
        prune: true
        selfHeal: true
app:
  service:
    type: NodePort
    port: 30086
git:
  unsupportedCapabilities: true
  insecure: true
  fromRepo:
    scmUrl: https://github.com
    org: krateoplatformops-blueprints
    name: github-scaffolding-with-composition-page
    branch: main
    path: skeleton/
    credentials:
      authMethod: generic
      secretRef:
        namespace: krateo-system
        name: github-repo-creds
        key: token
  toRepo:
    scmUrl: https://github.com
    org: krateoplatformops-test
    name: fireworks-app
    branch: main
    path: /
    credentials:
      authMethod: generic
      secretRef:
        namespace: krateo-system
        name: github-repo-creds
        key: token
    private: false
    initialize: true
    deletionPolicy: Delete
    verbose: false
    configurationRef:
      name: repo-config
      namespace: demo-system
```

**Step 3: Install the blueprint**

```sh
helm install <release-name> github-scaffolding-with-composition-page \
  --repo https://marketplace.krateo.io \
  --namespace <release-namespace> \
  --create-namespace \
  -f ~/github-scaffolding-with-composition-page-values.yaml \
  --version 1.2.5 \
  --wait
```

---

### Method 2: Krateo Composable Operation

**Step 1: Install the `CompositionDefinition`**

```sh
cat <<EOF | kubectl apply -f -
apiVersion: core.krateo.io/v1alpha1
kind: CompositionDefinition
metadata:
  name: github-scaffolding-with-composition-page
  namespace: krateo-system
spec:
  chart:
    repo: github-scaffolding-with-composition-page
    url: https://marketplace.krateo.io
    version: 1.2.5
EOF
```

As a result of this command:
- The core-provider generates a new Custom Resource Definition (CRD) based on the `github-scaffolding-with-composition-page` chart's schema.
- Resources of kind `GithubScaffoldingWithCompositionPage` (with `apiVersion: composition.krateo.io/v1-2-5`) can be created in the cluster.


**Step 2: Install the Blueprint**

```sh
cat <<EOF | kubectl apply -f -
apiVersion: composition.krateo.io/v1-2-5
kind: GithubScaffoldingWithCompositionPage
metadata:
  name: <release-name> 
  namespace: <release-namespace> 
spec:
  argocd:
    namespace: krateo-system
    application:
      project: default
      source:
        path: chart/
      destination:
        server: https://kubernetes.default.svc
        namespace: fireworks-app
      syncEnabled: false
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
  app:
    service:
      type: NodePort
      port: 30086
  git:
    unsupportedCapabilities: true
    insecure: true
    fromRepo:
      scmUrl: https://github.com
      org: krateoplatformops-blueprints
      name: github-scaffolding-with-composition-page
      branch: main
      path: skeleton/
      credentials:
        authMethod: generic
        secretRef:
          namespace: krateo-system
          name: github-repo-creds
          key: token
    toRepo:
      scmUrl: https://github.com
      org: krateoplatformops-test
      name: fireworks-app
      branch: main
      path: /
      credentials:
        authMethod: generic
        secretRef:
          namespace: krateo-system
          name: github-repo-creds
          key: token
      private: false
      initialize: true
      deletionPolicy: Delete
      verbose: false
      configurationRef:
        name: repo-config
        namespace: demo-system
EOF
```

This command:
- Creates a Composition resource, which deploys the `github-scaffolding-with-composition-page` chart (version 1.2.5).
- Uses the configurations provided in the spec block to override the chart's default values.

---

### Method 3: Krateo Composable Portal

**Step 1: Apply the `portal-blueprint-page` `CompositionDefinition`**

```sh
cat <<EOF | kubectl apply -f -
apiVersion: core.krateo.io/v1alpha1 
kind: CompositionDefinition
metadata:
  name: portal-blueprint-page
  namespace: krateo-system
spec:
  chart:
    repo: portal-blueprint-page
    url: https://marketplace.krateo.io
    version: 1.0.6
EOF
```

As a result of this command:
- The core-provider generates a new Custom Resource Definition (CRD) based on the `portal-blueprint-page` chart's schema.
- Resources of kind `PortalBlueprintPage` (with `apiVersion: composition.krateo.io/v1-0-6`) can now be created in the cluster.


**Step 2: Install the Blueprint**

> Note: `metadata.name` and `spec.blueprint.repo` must be the same.

```sh
cat <<EOF | kubectl apply -f -
apiVersion: composition.krateo.io/v1-0-6
kind: PortalBlueprintPage
metadata:
  name: github-scaffolding-with-composition-page
  namespace: demo-system
spec:
  blueprint:
    repo: github-scaffolding-with-composition-page
    url: https://marketplace.krateo.io
    version: 1.2.5 # this is the Blueprint version
    hasPage: true
  form:
    alphabeticalOrder: false
  panel:
    title: GitHub Scaffolding with Composition Page
    icon:
      name: fa-cubes
EOF
```

As a result of this command:
- A new `PortalBlueprintPage` Composition is created, which deploys the portal widgets and generates a new `CompositionDefinition` for the `github-scaffolding-with-composition-page` chart using the values provided in `spec.blueprint`.
- A blueprint card is created in the Krateo Portal's "Blueprints" section, allowing users to configure and deploy this blueprint directly from the UI.


---

## Access the FireworksApp (Optional)

This blueprint scaffolds the Fireworks app, which is a basic web page. To access it, edit the field `spec.argocd.application.syncEnabled` to `true`, either in the _"Values"_ section from the portal, or with the follwing command:

```sh
kubectl patch githubscaffoldingwithcompositionpage my-fireworks-app \
  -n krateo-system \
  --type='merge' \
  -p '{"spec":{"argocd":{"application":{"syncEnabled":true}}}}'
```

Now, the ArgoCD application begins reconciling the state of the repo in the cluster, creating the deployment and service of the FireworksApp.

Port-forward its service:

```sh
kubectl port-forward -n krateo-system svc/my-fireworks-app 31180:31180
```

From the browser, open the FireworksApp at `http://localhost:31180`.
