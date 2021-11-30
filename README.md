# Traefik GitOps with Flux

## GitOps Principals
 - **Declartive** - the desired stated of the infrastrucutre is expressed declartively.
 - **Versioned** and immutable - the desired state is stored in a source of truth the enforces immutability.
 - **Pulled Automatically** from a source of truth *a git repo* - the changes are pulled by Controllers and applied on a cluster.
 - **Continuously Reconcilled** - the agents are continuously observe the desired state and attempt to apply the desired state.

## Prerequsities
- Two Kubernetes clusters acting as e.g. staging and production environments
- An empty Github Repo
- Exported environment variables GITHUB_TOKEN, GITHUB_USER, GITHUB_REPO

- the latest Flux CLI installed on a workstation

## Create the inrastructure repository

Create the Git repository where all configuration files will be stored. In our example we will use GITHUB repo but other alternative solution are also supported. See Flux documentation to learn on how to use it with Gitlab or Bitbucket.

The following command can be used to create Github repository:

```sh
gh repo create flux-traefik-demo --public --description "Flux and Traefik - demo"  --confirm
```

This command assumes you are in an empty directory with no git repository and will have the same effect as running `git init`. If your home directory is a git repository, you might want to run `git init` in an empty directory first, before running the above command. This will create the remote repository and take care of setting up the `git remote`.

## Create repository structure

The following command will create the top directory structures to keep manifests that describes the entire infrastructure.

- **apps** - contains a custom manifests per cluster and and Helm Releases
- **infrastructure** - contains a common infrastrucutre tools such as Helm repository definitions or common infrastructure comoponents
- **clusters** - contains a Flux configuration per cluster

```sh
mkdir -pv ./apps/{base,staging,production}/traefik  ./clusters/{production,staging} ./infrastructure/{sources,crds}
```

```sh
├── apps
│   ├── base
│   ├── production
│   └── staging
├── infrastructure
│   ├── crds
│   └── sources
└── clusters
    ├── production
    └── staging
```

## Create Traefik Base configuration

1. Create RBAC resources:

```yaml
cat > ./apps/base/traefik/rbac.yaml <<EOF
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: traefik-ingress-controller
rules:
  - apiGroups:
      - ""
    resources:
      - services
      - endpoints
      - secrets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
      - networking.k8s.io
    resources:
      - ingresses
      - ingressclasses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses/status
    verbs:
      - update
  - apiGroups:
      - traefik.containo.us
    resources:
      - middlewares
      - middlewaretcps
      - ingressroutes
      - traefikservices
      - ingressroutetcps
      - ingressrouteudps
      - tlsoptions
      - tlsstores
      - serverstransports
    verbs:
      - get
      - list
      - watch

---
kind: ServiceAccount
apiVersion: v1
metadata:
  name: traefik-ingress-controller
  namespace: traefik

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: traefik-ingress-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-ingress-controller
subjects:
  - kind: ServiceAccount
    name: traefik-ingress-controller
    namespace: traefik
EOF
```

2. Create Traefik deployment resource.

```yaml
cat > ./apps/base/traefik/traefik.yaml << EOF
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: traefik
  labels:
    app.kubernetes.io/instance: traefik
    app.kubernetes.io/name: traefik
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: traefik
      app.kubernetes.io/instance: traefik
  template:
    metadata:
      labels:
        app.kubernetes.io/name: traefik
        app.kubernetes.io/instance: traefik
    spec:
      serviceAccountName: traefik-ingress-controller
      terminationGracePeriodSeconds: 60
      containers:
        - name: traefik
          image: traefik:2.5.4
          args:
            - "--entryPoints.web.address=:8000/tcp"
            - "--entryPoints.websecure.address=:8443/tcp"
            - "--entryPoints.traefik.address=:9000/tcp"
            - "--api=true"
            - "--api.dashboard=true"
            - "--ping=true"
            - "--providers.kubernetescrd"
            - "--providers.kubernetescrd.allowCrossNamespace=true"
          readinessProbe:
            httpGet:
              path: /ping
              port: 9000
            failureThreshold: 1
            initialDelaySeconds: 5
            periodSeconds: 5
            successThreshold: 1
            timeoutSeconds: 2

          livenessProbe:
            httpGet:
              path: /ping
              port: 9000
            failureThreshold: 3
            initialDelaySeconds: 5
            periodSeconds: 5
            successThreshold: 1
            timeoutSeconds: 2

          resources:
            limits:
              cpu: 1000m
              memory: 1000Mi
            requests:
              cpu: 100m
              memory: 50Mi

          ports:
            - name: web
              containerPort: 8000
              protocol: TCP

            - name: websecure
              containerPort: 8443
              protocol: TCP

            - name: traefik
              containerPort: 9000
              protocol: TCP

          volumeMounts:
            - mountPath: /data
              name: storage-volume
      volumes:
        - name: storage-volume
          emptyDir: {}
EOF
```

3. Create a load balancer type service that expose Traefik.

```yaml
cat > ./apps/base/traefik/svc.yaml << EOF
---
apiVersion: v1
kind: Service
metadata:
  name: traefik
  labels:
    app.kubernetes.io/instance: traefik
    app.kubernetes.io/name: traefik
spec:
  selector:
    app.kubernetes.io/instance: traefik
    app.kubernetes.io/name: traefik
  type: LoadBalancer
  externalTrafficPolicy: Local
  ports:
    - port: 80
      name: web
      targetPort: web
      protocol: TCP
    - port: 443
      name: websecure
      targetPort: websecure
      protocol: TCP
EOF
```

5. Create kustomization files with listed all created resources.

```yaml
cat > ./apps/base/traefik/kustomization.yaml << EOF
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - rbac.yaml
  - traefik.yaml
  - svc.yaml
EOF
```
## Customize Traefik configuration per cluster

### Production cluster

1. Create namespace for Traefik resources.

```yaml
cat > ./apps/production/traefik/namespace.yaml << EOF
---
apiVersion: v1
kind: Namespace
metadata:
  name: traefik-production
EOF
```

2. Create a patch for Traefik deployment for production cluster.

```yaml
cat > ./apps/production/traefik/traefik-patch.yaml << EOF
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: traefik
spec:
  template:
    spec:
      containers:
        - name: traefik
          args:
            - "--entryPoints.web.address=:8000/tcp"
            - "--entryPoints.websecure.address=:8443/tcp"
            - "--entryPoints.traefik.address=:9000/tcp"
            - "--api=true"
            - "--api.dashboard=true"
            - "--ping=true"
            - "--providers.kubernetescrd"
            - "--providers.kubernetescrd.allowCrossNamespace=true"
            - "--certificatesresolvers.myresolver.acme.storage=/data/acme.json"
            - "--certificatesresolvers.myresolver.acme.tlschallenge=true"
            - "--certificatesresolvers.myresolver.acme.email=jakub.hajek+webinar@traefik.io"
EOF
```
3. Create Kustomization to add resources.

```yaml
cat > ./apps/production/traefik/kustomization.yaml << EOF
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: traefik-production
resources:
  - namespace.yaml
  - ../../base/traefik

patchesStrategicMerge:
  - traefik-patch.yaml
EOF
```

### Staging cluster

1.  Create a namespace for Traefik deployment on a staging cluster.

```yaml
cat > ./apps/staging/traefik/namespace.yaml << EOF
---
apiVersion: v1
kind: Namespace
metadata:
  name: traefik-staging
EOF
```

2. Create Traefik patch for staging cluster.

```yaml
cat > ./apps/staging/traefik/traefik-patch.yaml << EOF
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: traefik
spec:
  template:
    spec:
      containers:
        - name: traefik
          args:
            - "--entryPoints.web.address=:8000/tcp"
            - "--entryPoints.websecure.address=:8443/tcp"
            - "--entryPoints.traefik.address=:9000/tcp"
            - "--api=true"
            - "--api.dashboard=true"
            - "--ping=true"
            - "--providers.kubernetescrd"
            - "--providers.kubernetescrd.allowCrossNamespace=true"
            - "--certificatesresolvers.myresolver.acme.storage=/data/acme.json"
            - "--certificatesresolvers.myresolver.acme.tlschallenge=true"
            - "--certificatesresolvers.myresolver.acme.email=jakub.hajek+webinar@traefik.io"
            - "--certificatesresolvers.myresolver.acme.caserver=https://acme-staging-v02.api.letsencrypt.org/directory"
EOF
```

3. Create Kustomization resource for deploying Traefik related resources.

```yaml
cat > ./apps/staging/traefik/kustomization.yaml << EOF
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: traefik-staging
resources:
  - namespace.yaml
  - ../../base/traefik

patchesStrategicMerge:
  - traefik-patch.yaml
EOF
```

## Create Traefik CRD

Traefik requires to have Custom Resources deployed on each of the cluster. The following command will create a common resources including Treafik's CRD that will be deployed on each of the cluster. In that example on two clusters: `staging` and `production`

```yaml
cat > ./infrastructure/crds/traefik-crds.yaml << EOF
---
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: GitRepository
metadata:
  name: traefik-crds
  namespace: flux-system
spec:
  interval: 30m
  url: https://github.com/traefik/traefik-helm-chart.git
  ref:
    tag: v10.3.0
  ignore: |
    # exclude all
    /*
    # path to crds
    !/traefik/crds/
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: traefik-api-crds
  namespace: flux-system
spec:
  interval: 15m
  prune: false
  sourceRef:
    kind: GitRepository
    name: traefik-crds
    namespace: flux-system
  healthChecks:
  - apiVersion: apiextensions.k8s.io/v1
    kind: CustomResourceDefinition
    name: ingressroutes.traefik.containo.us
  - apiVersion: apiextensions.k8s.io/v1
    kind: CustomResourceDefinition
    name: ingressroutetcps.traefik.containo.us
  - apiVersion: apiextensions.k8s.io/v1
    kind: CustomResourceDefinition
    name: ingressrouteudps.traefik.containo.us
  - apiVersion: apiextensions.k8s.io/v1
    kind: CustomResourceDefinition
    name: middlewares.traefik.containo.us
  - apiVersion: apiextensions.k8s.io/v1
    kind: CustomResourceDefinition
    name: middlewaretcps.traefik.containo.us
  - apiVersion: apiextensions.k8s.io/v1
    kind: CustomResourceDefinition
    name: serverstransports.traefik.containo.us
  - apiVersion: apiextensions.k8s.io/v1
    kind: CustomResourceDefinition
    name: tlsoptions.traefik.containo.us
  - apiVersion: apiextensions.k8s.io/v1
    kind: CustomResourceDefinition
    name: tlsstores.traefik.containo.us
  - apiVersion: apiextensions.k8s.io/v1
    kind: CustomResourceDefinition
    name: traefikservices.traefik.containo.us
EOF
```

Create Kustomization for Traefik CRDS

```yaml
cat > ./infrastructure/crds/kustomization.yaml << EOF
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: flux-system
resources:
  - traefik-crds.yaml
EOF
```

Create kustomization file that deploy the file containing in CRDS directory.

```yaml
cat > ./infrastructure/kustomization.yaml << EOF
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - crds
EOF
```

## Create the initial Flux Configuration:

### Production cluster:

```yaml
cat > ./clusters/production/apps.yaml << EOF
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 10m0s
  dependsOn:
    - name: infrastructure
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./apps/production
  prune: true
  validation: client
EOF
```

```yaml
cat > ./clusters/production/infrastructure.yaml << EOF
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: infrastructure
  namespace: flux-system
spec:
  interval: 10m0s
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./infrastructure
  prune: true
  validation: client
EOF
```

### Staging cluster:

```yaml
cat > ./clusters/staging/apps.yaml << EOF
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 10m0s
  dependsOn:
    - name: infrastructure
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./apps/staging
  prune: true
  validation: client
EOF
```

```yaml
cat > ./clusters/staging/infrastructure.yaml << EOF
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: infrastructure
  namespace: flux-system
spec:
  interval: 10m0s
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./infrastructure
  prune: true
  validation: client
EOF
```


## Creating the initial commit

Once all initial configuration has been created we can commit and push the code to the repository.
The next step is to bootstrap clusters with Flux CLI command.

```sh
├── apps
│   ├── base
│   │   └── traefik
│   │       ├── kustomization.yaml
│   │       ├── rbac.yaml
│   │       ├── svc.yaml
│   │       └── traefik.yaml
│   ├── production
│   │   └── traefik
│   │       ├── kustomization.yaml
│   │       ├── namespace.yaml
│   │       └── traefik-patch.yaml
│   └── staging
│       └── traefik
│           ├── kustomization.yaml
│           ├── namespace.yaml
│           └── traefik-patch.yaml
├── clusters
│   ├── production
│   │   ├── apps.yaml
│   │   └── infrastructure.yaml
│   └── staging
│       ├── apps.yaml
│       └── infrastructure.yaml
└── infrastructure
    ├── crds
    │   └── kustomization.yaml
    ├── kustomization.yaml
    └── sources

```

## Bootstrap clusters

Once the configuration files are created we can boostrap Flux on both clusters. Before running the bootstrap command we have to ensure that the following environments variables are exported.

```sh
export GITHUB_TOKEN
export GITHUB_USER
export GITHUB_REPO
```

### staging

```sh
flux bootstrap github \
--branch=main \
--context=t1.aws.traefiklabs.tech \
--owner=${GITHUB_USER} \
--repository=${GITHUB_REPO} \
--path=clusters/staging \
--components-extra=image-reflector-controller,image-automation-controller  \
--personal
```
### production

```sh
flux bootstrap github \
--branch=main \
--context=t2.aws.traefiklabs.tech \
--owner=${GITHUB_USER} \
--repository=${GITHUB_REPO} \
--path=clusters/production \
--components-extra=image-reflector-controller,image-automation-controller  \
--personal
```

## Create base deployment for Whoami application

Create a base configuration for Whoami application:

```sh
mkdir -pv ./apps/{base,staging,production}/whoami
```

```yaml
cat > ./apps/base/whoami/deployment.yaml << EOF
---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: whoamiv1
  labels:
    name: whoamiv1
spec:
  replicas: 1
  selector:
    matchLabels:
      task: whoamiv1
  template:
    metadata:
      labels:
        task: whoamiv1
    spec:
      containers:
        - name: whoamiv1
          image: traefik/traefikee-webapp-demo:v2
          args:
            - -ascii
            - -name=FOO
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet:
              path: /ping
              port: 80
            failureThreshold: 1
            initialDelaySeconds: 2
            periodSeconds: 3
            successThreshold: 1
            timeoutSeconds: 2
          resources:
            requests:
              cpu: 10m
              memory: 128Mi
            limits:
              cpu: 200m
              memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: whoamiv1
  namespace: app
spec:
  ports:
    - name: http
      port: 80
  selector:
    task: whoamiv1
EOF
```

Create a IngressRoute object that expose Whoami application:

```yaml
cat > ./apps/base/whoami/ingressroute.yaml << EOF
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: whoami
spec:
  entryPoints:
    - websecure
  routes:
    - kind: Rule
      match: Host(\`fix.me\`)
      services:
        - kind: Service
          name: whoamiv1
          port: 80
  tls:
    certResolver: myresolver
EOF
```

```yaml
cat > ./apps/base/whoami/kustomization.yaml << EOF
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - ingressroute.yaml
EOF
```

### Create custom release for Staging environment.

Create namespace where Whoami will be deployed:

```yaml
cat > ./apps/staging/whoami/namespace.yaml <<EOF
---
apiVersion: v1
kind: Namespace
metadata:
  name: whoami-staging
EOF
```
Create patch for Whoami application:

```yaml
cat > ./apps/staging/whoami/whoami-patch.yaml << EOF
---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: whoamiv1
spec:
  replicas: 4
  template:
    spec:
      containers:
        - name: whoamiv1
          args:
            - -ascii
            - -name=STAGING
EOF
```
Create patch that updates Host rule for the Whoami application:

```yaml
cat > ./apps/staging/whoami/ingressroute-patch.yaml <<EOF
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: whoami
spec:
  routes:
   - kind: Rule
     match: Host(\`whoami.t1.demo.traefiklabs.tech\`)
     services:
        - kind: Service
          name: whoamiv1
          port: 80
EOF
```

Create a Kustomization configuration to deploy the Whoami application on staging cluster

```yaml
cat > ./apps/staging/whoami/kustomization.yaml << EOF
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: whoami-staging
resources:
  - namespace.yaml
  - ../../base/whoami

patchesStrategicMerge:
  - whoami-patch.yaml
  - ingressroute-patch.yaml
EOF
```

### Create custom release for Production environment.

Create a namespace where Whoami on production will be deployed:

```yaml
cat > ./apps/production/whoami/namespace.yaml << EOF
---
apiVersion: v1
kind: Namespace
metadata:
  name: whoami-production
EOF
```

Create patch for Whoami application:

```yaml
cat > ./apps/production/whoami/whoami-patch.yaml << EOF
---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: whoamiv1
spec:
  replicas: 8
  template:
    spec:
      containers:
        - name: whoamiv1
          args:
            - -ascii
            - -name=PRODUCTION
EOF
```

Create patch that updates Host rule for the Whoami application:

```yaml
cat > ./apps/production/whoami/ingressroute-patch.yaml << EOF
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: whoami
spec:
  routes:
   - kind: Rule
     match: Host(\`whoami.t2.demo.traefiklabs.tech\`)
     services:
        - kind: Service
          name: whoamiv1
          port: 80
EOF
```

Create a Kustomization configuration to deploy the Whoami application on staging cluster

```yaml
cat > ./apps/production/whoami/kustomization.yaml << EOF
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: whoami-production
resources:
  - namespace.yaml
  - ../../base/whoami

patchesStrategicMerge:
  - whoami-patch.yaml
  - ingressroute-patch.yaml
EOF
```

## Questions

- how to test pull requests before merging to main branch?

## To do

- [] GITHUB actions that validate the code
- [] GITHUB actions that creates PR if there is a new Flux release.
- [] promotion from staging to production
- [] webhook recievers
- [] flagger
