+++
title = 'Installing a Composable Kustomize Source with Flux'
date = 2023-11-05T10:14:16-07:00
draft = false
+++
When going about trying a new app or k8s cluster tool we often reach for a helm chart or perhaps a cli tool... or maybe there's an operator with some CRDs.  There are some other "exotic" distribution formats like jsonnet, tanka or kustomize.

Today I wanted to try out [pomerium](https://www.pomerium.com) for an identity aware ingress controller, for which it so happens [helm is deprecated](https://artifacthub.io/packages/helm/pomerium/pomerium#deprecation) in favor of [kustomize](https://www.pomerium.com/docs/deploy/k8s/quickstart#install-pomerium).

I looked at the source repo and noticed the suggested `default` kustomize target creates CRDs, generates some secrets and kicks off the controller deployment.  Curiously sitting alongside are `prometheus` and `stress-test` resources.  I don't need to stress test my installation but I would like some observability as I monkey around.

In the vein of kustomize's [preference for composability over inheritance](https://github.com/kubernetes/enhancements/blob/master/keps/sig-cli/1802-kustomize-components/README.md) I sought out a way to optionally add a dash of prometheus `ServiceMonitor` along with my bowl of pomerium.

## Tool Selection

I've selected my favorite continuous delivery tool, [fluxcd.io](https://fluxcd.io) for this effort.  I'm excited to see how its native [kustomization](https://fluxcd.io/flux/components/kustomize/) can be leveraged to turn out a clean and stable composed deployment.

## The Stuff

Here's the basic directory structure in pomerium ingress controller [kustomize source](https://github.com/pomerium/ingress-controller/tree/main/config):

```
./config
├── crd
├── default
├── gen_secrets
├── pomerium
├── prometheus
└── stress-test
```

I want to get the `default` + `prometheus` resources.  The `default` is an overlay that uses three bases at the same directory level as `prometheus`:

```
# ./config/default/kustomization.yaml
namespace: pomerium
commonLabels:
  app.kubernetes.io/name: pomerium
bases:
  - ../crd
  - ../pomerium
  - ../gen_secrets
```

And `prometheus` is a simple manifest collector:

```
# ./config/prometheus/kustomization.yaml
resources:
- monitor.yaml

# ./config/prometheus/monitor.yaml
---
# Prometheus Monitor Service (Metrics)
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    control-plane: controller-manager
  name: controller-manager-metrics-monitor
  namespace: system
spec:
  endpoints:
    - path: /metrics
      port: https
      scheme: https
      bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
      tlsConfig:
        insecureSkipVerify: true
  selector:
    matchLabels:
      control-plane: controller-manager
```

## Approaches

I'm looking at two basic approaches:

1.  Use a flux kustomization only
1.  Use a flux kustomization and a local-to-my-gitops-repo k8s kustomization

* * *

### Flux Kustomization Only Approach

#### Double Kustomization ✅

Let's start with two kustomizations since there isn't a single upstream kustomize target that includes the prom resources with the app deployment.

We'll need a git repository source for the upstream repo.

```
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: pomerium-ingress-controller
spec:
  ref:
    tag: v0.32.1
  url: https://github.com/pomerium/ingress-controller.git
```

Which we can then target with a kustomization each for `config/default` and `config/prometheus`.

```
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: pomerium-ingress-controller
spec:
  sourceRef:
    kind: GitRepository
    name: pomerium-ingress-controller
  path: config/default
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: pomerium-ingress-controller-monitoring
spec:
  sourceRef:
    kind: GitRepository
    name: pomerium-ingress-controller
  path: config/prometheus
```

This method works well and is modular.  We could easily enrich the servicemonitor with flux kustomize's support for [`commonMetadata`](https://fluxcd.io/flux/components/kustomize/api/v1/#kustomize.toolkit.fluxcd.io/v1.CommonMetadata) and the blast radius is well contained between the primary and auxiliary components.  The only drawback is that there's two kustomize resources to manage.

#### Git Source Magic

Maybe we could get clever with some composition at the git source level.  We could leverage flux's behind-the-scenes use of [`kustomize create --autodetect --recursive`](https://kubectl.docs.kubernetes.io/references/kustomize/cmd/create/) when presented with a source path missing a `kustomization.yaml`.

*tldr; This pathway is winding with many dead ends.  I've included it as part of my investigation for completeness.  Feel free to skip it if you wish.*

##### Attempt 1 ❌

Let's create a synthetic path composed with what we want

```
./synthetic
├── crd
├── gen_secrets
├── pomerium
└── prometheus
```

by using the [`.spec.include`](https://fluxcd.io/flux/components/source/gitrepositories/#include) from flux's `GitRepository` like so:

```
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: pomerium-ingress-controller
spec:
  include:
    - repository: pomerium-ingress-controller
      fromPath: config/crd
      toPath: synthetic/crd
    - repository: pomerium-ingress-controller
      fromPath: config/gen_secrets
      toPath: synthetic/gen_secrets
    - repository: pomerium-ingress-controller
      fromPath: config/pomerium
      toPath: synthetic/pomerium
    - repository: pomerium-ingress-controller
      fromPath: config/prometheus
      toPath: synthetic/prometheus
  ref:
    tag: v0.32.1
  url: https://github.com/pomerium/ingress-controller.git
```

and target that new directory path

```
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: pomerium-ingress-controller
spec:
  sourceRef:
    kind: GitRepository
    name: pomerium-ingress-controller
  path: synthetic
```

Snap!  We encounter a manifest validation error since the required selector labels in the deployment come from `commonLabels` in the `default` kustomization.

##### Attempt 2 ❌

We can try `.spec.commonMetadata.labels` to weave these back in,

```
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: pomerium-ingress-controller
spec:
  commonMetadata:
    # labels repeated from upstream ./config/default/kustomization.yaml
    labels:
      app.kubernetes.io/name: pomerium
  sourceRef:
    kind: GitRepository
    name: pomerium-ingress-controller
  path: synthetic
```

but validation appears to happen before the labels get patched in by the flux kustomize controller.

Fine-- it would have been wonky anyway to have to rely on this extra post-processing to have a viable deployment.

##### Attempt 3 ❌

Let's compose all the things then:

```
./config
├── crd
├── default
├── gen_secrets
├── pomerium
└── prometheus
```

Bang!  We have duplicate resource definitions since `crd`, `gen_secrets` and `pomerium` are all sourced directly and as bases by `default`.

##### Attempt 4 ❌

How about if we place prometheus under `config/default`?  Maybe the recursive kustomize create will pick this up and we can avoid duplicates while getting the necessary common labels.

```
./config
└── default
    └── prometheus
```

We can map this directory using a git repo include spec

```
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: pomerium-ingress-controller
spec:
  include:
    - repository: pomerium-ingress-controller
      fromPath: config/prometheus
      toPath: config/default/prometheus
  ref:
    tag: v0.32.1
  url: https://github.com/pomerium/ingress-controller.git
```

and source using a very basic flux kustomization

```
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: pomerium-ingress-controller
spec:
  sourceRef:
    kind: GitRepository
    name: pomerium-ingress-controller
  path: config/default
```

This structure is valid and deploys, but where is our service monitor?  Since there is a kustomization.yaml already present in our target, the flux kustomize controller does not do an autodetect create and rightfully so.

##### Give Upi 😭

There just doesn't seem to be a way to have a single flux kustomization or git source arrange these resources in a one-stop-shopping package.  Most of these are a bit confusing anyway.

#### Single Kustomization with Component

Ok time out. There's a neat pattern with kustomize called [components](https://kubectl.docs.kubernetes.io/guides/config_management/components/) that supports just the use case we're going for.  Maybe we can use it here.

##### Attempt 1 ❌

I'll use a simple git repo source and invoke the [`.spec.components`](https://fluxcd.io/flux/components/kustomize/kustomizations/#components) of the flux kustomization to bring in the prometheus resources.

```
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: pomerium-ingress-controller
spec:
  sourceRef:
    kind: GitRepository
    name: pomerium-ingress-controller
  path: config/default
  components:
  - ../prometheus
```

Too good to be true-- components must be `kustomize.config.k8s.io/v1alpha1/Component` but the upstream is the typical `kustomize.config.k8s.io/v1beta1/Kustomization`.

##### Attempt 2 ✅

I want to make this work so I'll see if a slight modification to the upstream would make this into a well composable installation without altering any existing use cases.

I forked the ingress controller repo, made a [new branch](https://github.com/haggishunk/ingress-controller/tree/prometheus-as-kustomize-component) and added a new path in the `./config` directory called `components/prometheus`

```
 ./config
├── components
│   └── prometheus
├── crd
├── default
├── gen_secrets
├── pomerium
├── prometheus
└── stress-test
```

Adding a kustomize component supports referring to the prometheus base

```
# ./config/components/prometheus/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component
resources:
- ../../prometheus
```

We change our git repo source to this fork and branch

```
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: pomerium-ingress-controller
spec:
  ref:
    branch: prometheus-as-kustomize-component
  url: https://github.com/haggishunk/ingress-controller.git
```

And use the same kustomize as in our first attempt but with a slight change of component path

```
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: pomerium-ingress-controller
spec:
  sourceRef:
    kind: GitRepository
    name: pomerium-ingress-controller
  path: config/default
  components:
  - ../components/prometheus
```

Success!  I like this solution the best but it relies on the upstream having a structure that supports it.

We'll come back to that in the wrap up.  Let's move on to the other main approach: a combined flux and k8s kustomization with remote bases.


* * *

### Flux + K8S Kustomization Approach

I'll present a notional directory structure that has a k8s kustomization and a flux kustomization.  No additional git repo source is required as I am going to use remote bases for k8s kustomization and the flux bootstrap repo will hold our flux kustomization.

```
./my-pomerium
├── flux-kustomization.yaml
└── kustomization.yaml
```

#### Remote Source k8s Kustomization ✅

Though kustomize remote bases may present [security concerns](https://fluxcd.io/flux/security/best-practices/#kustomize-controller) and they may not be the most performant, this method is easy to understand, is clearly composable and has a reasonable separation of concerns.

We create our own k8s kustomization to source both of the desired upstream targets:

```
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- https://github.com/pomerium/ingress-controller.git/config/default?ref=v0.23.1
- https://github.com/pomerium/ingress-controller.git/config/prometheus?ref=v0.23.0
```

Sourcing the individual bases works when we supply some common labels but again, this is fragile and deviates unnecessarily from the upstream.

```
# kustomization.yaml (alternate)
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
commonLabels:
  app.kubernetes.io/name: pomerium
resources:
- https://github.com/pomerium/ingress-controller.git/config/crd?ref=v0.23.1
- https://github.com/pomerium/ingress-controller.git/config/pomerium?ref=v0.23.0
- https://github.com/pomerium/ingress-controller.git/config/gen_secrets?ref=v0.23.0
- https://github.com/pomerium/ingress-controller.git/config/prometheus?ref=v0.23.0
```

We set up a flux kustomization to point to this k8s kustomization and voila.

```
# flux-kustomization.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: pomerium-ingress-controller
spec:
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./my-pomerium
```

## Final Words

If you're ok with extra resources then the double kustomization fits the bill with a small bit of fuss.

If you're ok with remote kustomize bases then the flux + k8s kustomization works well.

My favorite is the kustomize component in a single flux kustomization, though not all upstream sources will support this pattern.  I've made a [pull request](https://github.com/pomerium/ingress-controller/pull/800) for pomerium to see what they think.
