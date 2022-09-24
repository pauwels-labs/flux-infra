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
  
- Convert patches to patchesStrategicMerge in order to link patches
  made in Terraform to real files in the flux repo
  
- When identifying an item in a list, provide the item's name or some
  unique identifying characteristic, so that the merge can happen

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
  
- Make sure to add --issuer-ambient-credentials=true or
  --cluster-issuer-ambient-credentials=true to enable IRSA

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

- Easy with the operator

- 

### Intall external-dns

- Make sure OIDC provider is copied to the DNS account or IRSA won't
  work

- Make sure to add istio-virtualservice and istio-gateway as sources
  in the arguments


## Secrets to automate

Bootstraping the cluster is a bit of a tricky matter. Some components,
like istio and vault, are required to manage networking and secrets,
respectively, but flux, which depends on those, needs a secret to be
able to connect to the repository that then configures istio and vault
in the cluster.

## Multi-tenant networking

Istio will have to be carefully configured to enable multi-tenant
networking. The gateway deployments and Gateway custom resources will
need to be managed by the ops team, not the tenant, and tenant
namespaces will be authorized to create VirtualService resources that
bind to a particular host in a particular Gateway in the cluster.

Custom domains should be doable; however TLS certificates/secrets have
to reside in the same namespace that the gateway deployment resides
in.  This means TLS certificate resources will have to be defined in
the istio-ingress namespace, and the ops-controlled Gateway resource
will need to be updated to include the custom domain. It will also
need to explicitly authorize the tenant and only the tenant namespace
as capable of binding to it.
