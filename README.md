# Traefik GitOps with Flux 

## Create repository structure

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

## Customize Traefik configuration per cluster

## Create Traefik CRD

## Create base deployment for Whoami application

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


## Questions

- how to test pull requests before merging to main branch?

## To do

- [] promotion from staging to production
- [] webhook recievers 
- [] flagger 