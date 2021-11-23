# Traefik GitOps with Flux 

## Create repository structure

```sh
mkdir -pv ./apps/base/traefik ./apps/staging  ./apps/production  ./clusters/production ./clusters/staging ./infrastructure/crds ./infrastructure/sources
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
---
cat > ./apps/base/traefik/rbac.yaml <<EOF
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
cat > ./apps/base/traefik/traefik.yaml
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
          image: traefik:2.5.3
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
cat > ./apps/base/traefik/svc.yaml
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
cat > ./apps/base/traefik/kustomization.yaml
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
cat > ./apps/production/namespace.yaml << EOF
---
apiVersion: v1
kind: Namespace
metadata:
  name: traefik-production
```

2. Create a patch for Traefik deployment for production cluster. 

```yaml
cat > ./apps/production/traefik-patch.yaml << EOF
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
cat > ./apps/production/kustomization.yaml << EOF
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: traefik-production
resources:
  - namespace.yaml
  - ../../base/traefik

patchesStrategicMerge:
  - traefik-patch.yaml
```

### Staging cluster

1.  Create a namespace for Traefik deployment on a staging cluster. 

```yaml
cat > ./apps/staging/namespace.yaml << EOF
---

apiVersion: v1
kind: Namespace
metadata:
  name: traefik-staging
``` 

2. Create Traefik patch for staging cluster. 

```yaml
cat > ./apps/staging/traefik-patch.yaml << EOF

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
```

3. Create Kustomization resource for deploying Traefik related resources. 

```yaml
cat > ./apps/staging/kustomization.yaml << EOF
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: traefik-staging
resources:
  - namespace.yaml
  - ../../base/traefik

patchesStrategicMerge:
  - traefik-patch.yaml
```

## Create Traefik CRD

Traefik requires to have Custom Resources deployed on each of the cluster. The following command will create a common resources including Treafik's CRD that will be deployed on each of the cluster. In that example on two clusters: `staging` and `production`

```yaml
cat > ./infrastructure/crds/kustomization.yaml << EOF
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
```

Create kustomization file that deploy the file containing in CRDS directory.

```yaml
cat > ./infrastructure/kustomization.yaml << EOF
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - crds
```


## Bootstrap clusters

```
clusters
├── production
│   ├── apps.yaml
│   ├── flux-system
│   │   ├── gotk-components.yaml
│   │   ├── gotk-sync.yaml
│   │   └── kustomization.yaml
│   └── infrastructure.yaml
└── staging
    ├── apps.yaml
    ├── flux-system
    │   ├── gotk-components.yaml
    │   ├── gotk-sync.yaml
    │   └── kustomization.yaml
    └── infrastructure.yaml
```

```
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

## Questions

- how to test pull requests before merging to main branch?

## To do

- [] promotion from staging to production
- [] webhook recievers 
- [] flagger 