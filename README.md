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
  
- Need IAM role that can get authorization tokens for the ECR registry

- At some point we opened up all TCP ports inbound from the node
  security group and inbound to the node security group; for something
  but don't remember what

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

- An IAM role has to be made with special permissions

### Install Istio

- Need to make sure ingress TCP port 15017 from the cluster api
  security group is open on the all node security group otherwise the
  sidecar injection webhook endpoint can't be hit from the cluster api

- Need to add an istio.io/rev label to all sidecar injected namespaces

- Set a revision in istio-base and istio-istiod and revision tags in
  istio-istiod in order to canary upgrade istio in the future
  
- Don't forget to install a gateway

- Outbound calls (and likely inbound?) will not work while istio-proxy
  is not up. Make sure you wrap the start command with a sleep loop on
  the proxy healthcheck endpoint at localhost:15021/healthz/ready
  
- The istio-proxy will not automatically end when the main container
  exits. The proxy has a /quitquitquit endpoint on port 15020 that can
  be called on exit; make sure to configure a call to this endpoint
  via wget/curl after the main command exits.

### Install Kiali

- Easy with the operatir

### Intall external-dns
