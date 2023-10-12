---
title: "Gitops Source Problem and Flux Trace"
date: 2023-10-10T16:28:46-07:00
draft: false
toc: false
images:
tags:
  - flux
  - continous-delivery
  - ergonomics
---

## Seeing Clearly

One of the big problems with gitops (and infrastructure as code in a wider sense) is the "source problem" related to [traceability](https://www.bunnyshell.com/blog/how-to-overcome-infrastructure-as-code-iac-challenges/).  It's the question you have when reviewing service components with a mind to change them.  It goes like this:

* Is this resource controlled by any code somewhere?

* If so, where is the source of code that defines it?

In other words, you realize you may be deploying from _a_ source of truth... but _which one_ and _precisely where_ in it?

A [good tagging strategy](https://www.paloaltonetworks.com/blog/prisma-cloud/how-to-adopt-infrastructure-as-code-with-a-secure-by-default-strategy/) can solve this with backreferences to the originating source but not all assets support storing such metadata.

In this post I'd like to look at a feature of one of my favorite tools, [`flux`](https://fluxcd.io), that helps solve this problem for gitops-managed k8s resources.

### Tracing to Source

Let's start with a example investigation.

Say you want to add a new app `foo` to your existing istio service mesh.  One way to effect this is to [annotate the app's namespace](https://istio.io/latest/docs/setup/additional-setup/sidecar-injection/) with `istio-injection=enabled`.  You could manually apply the annotation and get your app up and running with istio.  But if this namespace is managed by IaC that metadata may later be removed by a job -- breaking your service mesh and leaving you wondering why.

You need to find out if it is managed and apply the annotation through its source of truth.

If you're using flux for continuous delivery you can quickly this out using `flux trace`.  For this example, the `foo` app is deployed into the `bar` namespace.

```
❯ flux trace namespace bar

Object:          Namespace/bar
Status:          Managed by Flux
---
Kustomization:   apps
Namespace:       flux-system
Path:            ./flux/apps/staging
Revision:        initial-bootstrap@sha1:9945ec95ce60a483271914bbcf72cd57ff97d45d
Status:          Last reconciled at 2023-10-11 09:55:30 -0700 PDT
Message:         Applied revision: initial-bootstrap@sha1:9945ec95ce60a483271914bbcf72cd57ff97d45d
---
GitRepository:   flux-system
Namespace:       flux-system
URL:             https://github.com/haggishunk/a-flux-cluster.git
Branch:          initial-bootstrap
Revision:        initial-bootstrap@sha1:9945ec95ce60a483271914bbcf72cd57ff97d45d
Status:          Last reconciled at 2023-10-10 10:52:22 -0700 PDT
Message:         stored artifact for revision 'initial-bootstrap@sha1:9945ec95ce60a483271914bbcf72cd57ff97d45d'
```

Ok, it turns out that indeed the `bar` namespace is managed by flux under the `apps` `Kustomization`.  You can also discover the gitops source by looking up the `Kustomization` path and `GitRepository` url and branch.

You could navigate a local clone of the `haggishunk/a-flux-cluster` repo or in the Github UI, checkout the `initial-bootstrap` branch and inspect the path `./initial-bootstrap/flux/apps/clusters/a-flux-cluster`.  Alternatively, the following construction results in a navigable URL.

```
# construction of a github url
<URL-minus-the-.git>/tree/<Branch>/<Path>

https://github.com/haggishunk/a-flux-cluster/tree/initial-bootstrap/flux/apps/clusters/a-flux-cluster
```

Now you've located the source of truth for the `bar` namespace that the app `foo` is deployed in.
