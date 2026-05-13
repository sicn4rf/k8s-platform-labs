# Kubernetes Platform Labs

A handful of self-contained labs I worked through to get fluent with the
core stack of modern platform engineering: Kubernetes primitives, Helm,
Envoy Gateway, Kyverno, and writing custom TypeScript GitHub Actions.

Each lab folder has its own README and runs on a local
[kind](https://kind.sigs.k8s.io/) cluster. The YAML manifests are
deliberately written from scratch (not generated) so the structure
mirrors what you'd actually maintain in a real cluster, including
the small ergonomic decisions like where to use a StatefulSet vs
a Deployment, when a Service should be headless, and how to layer
policies on top of the Gateway API.

## What's here

### [`labs/01-kubernetes-multitier-helm/`](./labs/01-kubernetes-multitier-helm)

Standing up Forgejo (a self-hosted git server) backed by MariaDB, on
a fresh cluster, with no charts. Then installing cert-manager via Helm
to feel the difference between writing manifests by hand and consuming
a packaged chart. The reason this matters: until you've manually
wired up a ConfigMap, a Secret, a couple of Services, two StatefulSets,
and two PVCs to talk to each other, the value of Helm doesn't really
click. After this lab it does.

### [`labs/02-envoy-gateway/`](./labs/02-envoy-gateway)

[Envoy Gateway](https://gateway.envoyproxy.io/) installed via Helm,
with the actual Gateway API resources (Gateway, HTTPRoute) routing
traffic through Envoy to a backend, plus the three policy CRDs that
Envoy Gateway adds on top:

- `ClientTrafficPolicy` for enforcing TLS minVersion on the listener
- `BackendTrafficPolicy` for retrying on 5xx responses from upstream
- `SecurityPolicy` for JWT validation against a public JWKS

There's also some xDS spelunking with `egctl` to see what the Envoy
Gateway controller actually pushes to the Envoy proxy pods. That's
the part that ties everything together: K8s resources go in, xDS
config comes out, traffic flows.

### [`labs/03-kyverno-policies/`](./labs/03-kyverno-policies)

Four custom [Kyverno](https://kyverno.io/) `ClusterPolicy` resources
guardrailing the kinds of misuse you'd expect when you let dev teams
write their own `HTTPRoute` and `SecurityPolicy` YAML. Each policy is
co-located with its `kyverno test` fixtures so you can validate the
policy logic offline before applying anything to a live cluster.

The four policies cover different validation styles (`deny.conditions`
with a context apiCall, `validate.anyPattern`, `match.any` + `exclude`
with namespace selectors, and `mutate` with conditional add), which
was the point. Project-style guardrail policies usually need at least
one of each in the toolkit.

### TypeScript GitHub Action (separate repo)

**[`envoy-route-policy-guard`](https://github.com/sicn4rf/envoy-route-policy-guard)**

A custom GitHub Action that fails a PR if it adds an `HTTPRoute` YAML
without a paired `SecurityPolicy` targeting it. Same idea as Kyverno's
`01-match-httproute` policy, just enforced earlier in the pipeline.
Catching this at PR time is cheap; catching it at admission time is
the backstop for anyone who skips the PR (cluster-admin, automation,
emergency hotfix). Both layers are useful.

## What I picked up across all four

- Why Kubernetes built admission controllers as a separate layer
  (auth and authz don't read the payload; somebody has to)
- Why federated gateways need both a router (Envoy Gateway) and a
  policy engine (Kyverno) instead of just one
- How Helm's release lifecycle (install/upgrade/rollback) maps to
  real ops work
- How to use `kyverno test` as your dev loop instead of poking
  policies into a live cluster
- How to write GitHub Actions in TypeScript that read PR file
  contents without burning the API rate limit

## Stack

- Kubernetes (Deployments, StatefulSets, Services, ConfigMaps, Secrets)
- Helm (charts, releases, values overrides, rollback)
- cert-manager
- Envoy Gateway + Kubernetes Gateway API
- Kyverno (validate, mutate; `kyverno test` CLI)
- TypeScript GitHub Actions (Octokit, `@actions/core`, ncc bundling)

## Running labs

Each lab assumes a fresh kind cluster and the usual CLI tools:

```
brew install kind kubectl helm kyverno kyverno-cli
kind create cluster --name lab-name
```

See each lab's README for the specific apply order and verification steps.
