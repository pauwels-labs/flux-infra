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
  
### Install oauth2-proxy

- Needs redis

- Need to store client ID and secret in vault

- When setting things up, add the following:

  - Add envoyExtAuthzHttp filter in istiod helm chart under
    meshConfig.extensionProviders

  - URL of the external authz service should be the CLUSTER-LOCAL
    oauth2-proxy url, i.e. oauth2-proxy.oauth2-proxy.svc.cluster.local
    along with the service port, i.e. 80

  - AuthorizationPolicy can be in the workload namespace selecting
    just the workload, doesn't have to be in one of the istio
    namespaces or selecting the istio gateway exclusively

  - Use skip_provider_button = true to eliminate the oauth2-proxy UI

- You'll need to be very specific about which headers envoy passes
  around in order for everything to be up to spec
  
  - x-forwarded-for header MUST be in includeRequestHeadersInCheck in
    order for the redirect back to the original domain to have the
    proper IP

- oauth2-proxy has a bug that took me days to figure out where x-forwarded-*
  headers are not sent in response, see issue #1818

### Install secrets-store-csi-driver

- Begin considering if we should be splitting all the "infrastructure"
  packages into separate kustomizations independently attached to the
  cluster
  
  - Testing this with secrets-store-csi-driver which is why it isn't
    part of infrastructure
    
  - The requirement for being bundled together for deployment should
    probably be either timing or dependency based

- Use kustomization instead of helm when possible for better CRD
  management
  
- When setting up a policy/role for service account access to vault
  secrets, you can use metadata from the SA token to define the access
  path, but you need to know the kubernetes auth accessor ID e.g.
  secrets/data/{{identity.entity.aliases.auth_kubernetes_fe3d9e77.metadata.service_account_namespace}}/{{identity.entity.aliases.auth_kubernetes_fe3d9e77.metadata.service_account_name}}

### Install kube-prometheus

- Mandatory to setup jsonnet and compile manifests using a custom
  jsonnet file
  
  - This is the best way to get repeatable builds and is quite
    ergonomic, allowing for full customization of the deployed
    resources
    
- Some modifications to the base jsonnet:

  - Add values.common.platform: 'aws' or whatever platform is being
    used for important platform-specific optimizations
    
  - Use values.comon.images to replace all image bases in a single
    convenient place
    
  - Use values.common.grafana.config to add arbitrary grafana config
  
  - Use auth.jwt to authenticate from SSO, and use role_attribute_path
    to map keycloak roles/claims to grafana roles
    
  - Add externalUrls (root_url for Grafana) to indicate the hostname
    where a particular resource will be accessed externally
    
  - You can add an arbitrary authorizationpolicy+:: field to add
    authorization policies and leverage generic auth on top of the
    various components (works very well with grafana jwt auth)
    
- You need to open port 6443 inbound from the cluster security group
  in order for the cluster apiservice to be able to hit the prometheus
  adapter and leverage the prometheus metrics

## Secrets to automate

Bootstraping the cluster is a bit of a tricky matter. Some components,
like istio and vault, are required to manage networking and secrets,
respectively, but flux, which depends on those, needs a secret to be
able to connect to the repository that then configures istio and vault
in the cluster.

- flux-system/flux-system          (done)
- flux-system/github-webhook-hmac  (done)
- keycloak/keycloak-db-credentials (done)
- oauth2-proxy/oauth2-proxy        (done)

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

## On identity

Keycloak has so far proven to be a good and versatile IdP with some
caveats. Its mappers for turning various values into token claims are
sparse and can't achieve the full breadth of things I'd want to
do. Additionally, roles vs groups is a bit wonky and for now I'm
sticking to just using groups.

There's also the topic of AWS identification. Adding the keycloak OIDC
endpoint as an OIDC provider in AWS allows users to authenticate, but
capabilities and documentation of mapping token claims into usable IAM
policy values is sparse and spread about haphazardly.

This issue came up when devising a multi-tenant solution for ECR
authentication. Each tenant namespace needs its own token scoped to
only its ECR namespace (along with infra namespaces) in order for it
to only pull the images it's authorized to pull. I really wanted to
avoid having to make a separate role for each tenant as this adds
infrastructure modifications for tenant onboarding and is a very heavy
additional load for tenant management. Achieving this has proved
incredibly difficult.

There are two policies that need to be written for a singular tenant
role: the ECR policy that allows the role to access some ECR
namespace, and the trust policy that allows a token issued by keycloak
to assume that AWS role. As mentioned earlier, the trust policy can
use the Federated principal to authorize keycloak-signed JWTs to
assume an AWS role. The question then is dynamically authorizing that
JWT to access some namespace in ECR based on values in that JWT.

I'll start with the solution I settled on. Each tenant has a
root-level group in the Keycloak realm and that root-level group has an
attribute that indicates the tenant\_name. This attribute is assigned
to all members of the group, and can therefore be retrieved by a scope
with the User Attribute mapper. By turning on multi-value and
attribute aggregation on that mapper, a user member of multiple tenant
groups will receive all of those tenant_name values as an array-type
claim in its id and access tokens.

The issue becomes where to put this array of tenant names in the
token. AWS has no support for using custom claim values in the IAM
trust relationship policy, but it does have minimal support for some
standard claims. A trust relationship policy federating OIDC users
with AssumeRoleWithWebIdentity can directly retrieve and match against
the azp, aud, amr, and sub claims in the OIDC token used in the
call. Example:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "",
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::274295908850:oidc-provider/identity.pauwelslabs.com/realms/pauwels-labs-main"
            },
            "Action": [
                "sts:AssumeRoleWithWebIdentity"
            ],
            "Condition": {
                "StringEquals": {
                    "identity.pauwelslabs.com/realms/pauwels-labs-main:oaud": "aws",
                    "identity.pauwelslabs.com/realms/pauwels-labs-main:aud": "aws"
                },
                "StringLike": {
                    "identity.pauwelslabs.com/realms/pauwels-labs-main:sub": "*${sts:RoleSessionName}*"
                },
                "ForAnyValue:StringLike": {
                    "identity.pauwelslabs.com/realms/pauwels-labs-main:amr": [
                        "/tenant-${sts:RoleSessionName}/root/admins",
                        "/tenant-${sts:RoleSessionName}/root/writers",
                        "/tenant-${sts:RoleSessionName}/ecr/admins",
                        "/tenant-${sts:RoleSessionName}/ecr/writers"
                    ]
                }
            }
        }
    ]
}
```

In this scenario, our token uses the `azp` claim, so the `:oaud`
suffix maps to `aud` in the token, the `:aud` suffix maps to `azp`,
and the rest are equal to each other. The `amr` claim is the only
claim allowed to be multi-value. Keycloak will break if you try to map
the `azp` claim, the `sub` claim has to be stringified, and the `aud`
claim will break the IAM OIDC provider auth if it's multi-value and
the first value in the array isn't an audience listed in the IAM OIDC
provider. Additionally, specifying a User Attribute mapper that maps
into the `aud` claim breaks keycloak.

Since I wanted to be able to assign ECR permissions based on tenant
name but also authorize RW vs RO users using group membership, I
needed two claims to map data into. I decided to map the user's group
names into the multi-value `amr` claim. This allows me to match in the
trust policy whether or not the user is a member of an admin group or
a read-only group etc. However, `amr` also has one more important
property.

It can't be read outside of a trust relationship policy specifying
some sort of sts:AssumeRole* action. This means it can't be used at
all in the policy outlining ECR permissions, and is therefore useless
for determining whether or not the user has access to a specific ECR
registry. It is only relevant for determining whether a user can
assume a particular RW or RO role.

The only claim accessible outside of the trust relationship policy is
the sub claim (for generic OIDC providers; specific OIDC providers
like facebook have some specific claims they have access to). This is
where a dirty but clever hack comes in. I take advantage of the fact
that keycloak stringifies the sub field even if it's multi-valued,
resulting in a claim that looks a bit like this:

```json
{
    ...
    "sub": "['tenant-1', 'tenant-2']"
    ...
}
```

In the IAM trust policy, I can do a `StringLike` match against this
token, comparing it against the `*${sts:RoleSessionName}*`. The
asterisks on each side allow the session name to match anywhere in the
"array" string value of the sub claim. The role session name is an
arbitrary value provided by the client when performing the
sts:AssumeRoleWithWebIdentity request. It can be a random value or, in
this case, the name of the tenant. What keeps users from getting
tokens for any tenant by simply specifying any value in the role
session name is that the policy matches that requested tenant against
the claim in the JWT token. That claim will only exist if the user is
in the appropriate tenant group and therefore has the appropriate
permissions.

We then wrap up this epic by applying this field to the ECR policy
side of things. Since the `sub` claim is the only one available, we
can again match this, but this time against an attribute attached to
the repository itself. In this case, each repository is tagged with a
TenantName tag that contains the name of the owning tenant. The IAM
policy statement looks like this:

```json
{
    "Action": [
        "ecr:UploadLayerPart",
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:GetDownloadUrlForLayer",
        "ecr:CompleteLayerUpload",
        "ecr:BatchGetImage",
        "ecr:BatchCheckLayerAvailability"
    ],
    "Effect": "Allow",
    "Resource": "arn:aws:ecr:eu-west-1:274295908850:repository/*",
    "Condition": {
        "StringLike": {
            "identity.pauwelslabs.com/realms/pauwels-labs-main:sub": "*${ecr:ResourceTag/TenantName}*"
        }
    },
    "Sid": "AllowTenantRWAccessToECRNamespace"
}
```

Here, the `Resource` field is unlimited and allows access to the
entire registry unhindered. The `Condition` field is where the
tenant-specific magic happens. Here, we do another identical
`StringLike` comparison against the `sub` claim, but we use the
`ecr:ResourceTag` policy variable to fetch the `TenantName` tag from
the target repository. If the repository has been tagged with a tenant
name that is in the user's list of tenant names, then the user is
authorized.

Together, the trust relationship policy and ECR access policy achieve
the following:

1. Only allow users of our Keycloak OIDC realm that are members of
   specific groups to assume our ECR role in AWS using
   sts:AssumeRoleWithWebIdentity
   
2. Authorize those users that have assumed that role to access the ECR
   tenant namespaces that they have access to (i.e. by being members
   of those tenant groups).
   
IAM policy and condition variables were leveraged as much as possible
to make a lightweight set of IAM resources that scale with users
without requiring more roles. The key policy variable was
`identity.pauwelslabs.com/realms/pauwels-labs-main:sub` as it can be
used across both trust relationship policies and normal IAM policies,
and links the OIDC attributes to real locations in the ECR registry.

## On Mimir

This software is a goddamn nightmare. If I can actually get it running
it'll solve a lot of issues, but getting it running is difficult. This
is compounded by:

1. Need to get it running in a multi-tenant way, integrated with
   Grafana, so that orgs/teams can't see across each other and I don't
   have to replicate Grafana instances across multiple people. This is
   with Keycloak-assigned JWTs that are authenticated/authorized by
   the Istio mesh.
   
2. Needs to be cost effective without excessive cross-AZ costs

3. Needs to be scalabale

### IPv6

Mimir is the first piece of software I've had to use with shoddy IPv6
support. I'm waiting on changes to the dskit repo (PR#185) for IPv6
support. I've integrated those changes to Mimir myself and have a
bespoke image for now, it works in IPv6.

The listen addresses are proving problematic. Sometimes the
instance_addr field in the config needs to be wrapped in square
brackets, sometimes it doesn't. So far, the following need to be wrapped in square brackets:

1. frontend
2. ingester

Reference this PR for some others that I haven't determined are
necessary. In this PR, I removed the square brackets, to disastrous
effect.



TRY THIS NEXT:

Use the same AuthorizationPolicy as is used for the echo pod for the
query-frontend pod. It's possible that Grafana is forwarding the
oauth2-proxy cookie and we can leverage that.
