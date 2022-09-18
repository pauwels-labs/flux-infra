# flux-infra

## Installation Instructions

### Bootstrap flux

Notes:

- Need external-config-data to share/template variables.

- Need ecr-credentials-sync to exchange an IRSA token for an ECR
  password that can be placed in a Docker secret. CronJob does this
  every 6h to renew the 12h password.

- Need special patches in
  clusters/lefranc-0/flux-system/kustomization.yaml to enable
  multi-tenant lockdown.

### Install cert-manager

- Make sure to have a separate resource for namespace that's installed
  first.
  
- Replace all images with AWS ECR hosted ones.

- The HelmRepository resource can just point to /helm path in ECR
  registry. Individual HelmReleases can then just point to a chart at
  <org>/<chart name>.
  
- Make sure installCRDs is set to true.

- For cluster-wide infra releases, the HelmRelease CR should itself be
  in the flux-system namespace, but its targetNamespace should be the
  namespace it's going to be in (e.g. cert-manager. The
  serviceAccountName should helm-controller if it's a HelmRelease, and
  kustomize-controller if it's a kustomize overlay.

### Install AWS load balancer controller

- Set enableCertManager to true

### Install Istio

