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

- When enabling webhook notifications, make sure to select json as the
  content type of the event in the github settings
  
- When enabling webhook notifications, make sure to send the event to
  the webhook-receiver service, not just the notification-controller
  service

- When making strategic merge patches or RFC6902 patches, the resource
  being patched MUST be defined inside of the resources section of the
  Kustomization file, even if the resource was already made outside of
  Flux

- After installing istio sidecars, you'll find that processing will
  begin to fail with 503 errors from the source controller and
  notification controller; this is because istio fails to handle FQDNs
  with trailing dots (e.g. pod.ns.svc.cluster.local./somepath),
  without an explicit entry in a virtualservice adding that host
  in. The options are:
  
  1. Redefine the controller URLs used by flux to not include trailing
     dots
     
  2. Use a custom EnvoyFilter to enable the strip_trailing_host_dot
     feature in Envoy:
     https://github.com/istio/istio/issues/18904#issuecomment-955702260
     
  3. Use a VirtualService to add a redirect from the host with the dot
     to the host without the dot:
     https://github.com/fluxcd/flux2/issues/598#issuecomment-756999645
     
  Personally I'm a fan of solution #3, as it makes everything explicit
  and visible so that it can be fixed when Istio/Envoy fix the issue.

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

- When setting up gateways, you can have the same port defined with
  multiple hosts with different tls secrets, but the port entries have
  to have different names

### Install Kiali

- Easy with the operator

### Intall external-dns

- Make sure OIDC provider is copied to the DNS account or IRSA won't
  work

- Make sure to add istio-virtualservice and istio-gateway as sources
  in the arguments


### Install aws-ebs-csi-driver

- Make sure to deploy KMS key along with a key policy and IAM role
  that can access it for encrypting the EBS volumes
  
- You should use KMS key aliases instead of direct key ARNs in the
  StorageClass, you can specify an alias ARN in the SC but you cannot
  specify an alias in the "resource" field of an IAM policy; it MUST
  be specified as a kms:ResourceAliases condition

### Install vault

- This is a complicated one, a lot of things come into play to make
  vault resilient and prod-ready
  
- Use integrated storage with RAFT to avoid a consul deployment

- Standup a standard istio sidecar alongside each node, and deploy
  vault with tls_disable/tlsDisable set to true
  
- Normally, this should allow vault to do everything while being
  protected by istio's mutual TLS mesh functionality
  
- Use awskms seal to auto-unseal using a KMS key; this requires an IAM
  role and a kms key built out using terraform
  
- Once vault is running, shell into it and call `vault operator init`
  to kickstart everything
  
- You'll need to find a safe place to put the recovery keys and root
  key
  
- Investigate use of PGP initialization

### Install Keycloak

- This one is difficult

- Need to setup a postgres; opted for an aurora serverless cluster and
  use vpc peering to connect to eks cluster
  
- GRANT ALL PRIVILEGES on the keycloak user to the keycloak db and
  then let keycloak make the tables so it'll be owner and have full
  permissions
  
- Set tlsSecret in the Keycloak CR to INSECURE-DISABLE to let istio
  handle TLS termination; Keycloak operator automatically sets proxy
  config to "edge" so that Keycloak knows its TLS is terminated
  upstream
  
- Add a JAVA_OPTS_APPEND env var to the
  unsupported.podTemplate.spec.containers[0] section in the Keycloak
  CR and give it the following value in order to support
  IPv6/dualstack: "-Djava.net.preferIPv4Stack=false
  -Djava.net.preferIPv6Addresses=true"

- Need to figure out whether Vault will be setup first to provide DB
  creds or whether Keycloak will be setup first to provide auth to
  vault

- The keycloak reverse proxy docs specify the use of certain
  X-Forwarded-* headers:
  
  https://www.keycloak.org/server/reverseproxy
  
  You need to carefully evaluate how you'll provide source IP to
  keycloak. In this context, an NLB is deployed in AWS and an Istio
  gateway handles the request. PROXY protocol is enabled on the NLB
  via an annotation on the LB Service resource, and then PROXY
  protocol is enabled in the Istio gateway via an EnvoyFilter defined
  on the gateway workload. With this setup, Istio will read the source
  IP via the NLB over PROXY protocol and add at L7 via an
  X-Forwarded-For header. It seems to automatically handle setting
  X-Forwarded-Proto to https (which is important since keycloak itself
  is receiving non-HTTPS requests from the sidecar), and it sets the
  Host/Authority headers to be the external domain, not the internal
  one, so no need for X-Forwarded-Host.
  
  With the above setup, there's no need to set the
  externalTrafficPolicy on the Service resource to local, or to enable
  client IP preservation at the NLB level.
  

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
