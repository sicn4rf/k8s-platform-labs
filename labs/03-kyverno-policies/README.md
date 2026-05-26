# Lab 03: Kyverno custom policies plus the `kyverno test` workflow

Four custom [Kyverno](https://kyverno.io/) `ClusterPolicy` resources
guardrailing the kinds of misuse you'd expect when dev teams have
direct access to apply `HTTPRoute`, `SecurityPolicy`, and
`EnvoyPatchPolicy` resources to a shared cluster. Each policy lives
next to its `kyverno test` fixtures so the policy logic can be
validated offline before touching a live cluster.

## Why these particular policies

Kyverno's policy library has hundreds of examples. The ones in this
lab were chosen because between them they exercise four different
validation patterns, and each pattern shows up regularly in real
guardrail work:

| Folder | Style | What it does |
|---|---|---|
| `01-match-httproute/` | `apiCall` context + `deny.conditions` | Deny `HTTPRoute` creation if no `SecurityPolicy` in the cluster targets it by name |
| `02-tls-min-policy/` | `validate.anyPattern` | Require `ClientTrafficPolicy.spec.tls.minVersion` to be 1.2 or 1.3, deny otherwise |
| `03-namespace-restriction/` | `match.any` + `exclude` w/ `namespaceSelector` | Block `EnvoyPatchPolicy` resources outside namespaces labeled `platform=true` |
| `04-dev-label-mutate/` | `mutate.patchStrategicMerge` + `+(...)` | Inject a default environment label into `HTTPRoute`s in the dev namespace |

Three of these are validation policies (block bad things). One is
a mutation (silently fix a small thing for the user instead of
making them resubmit). Both have their place in a real platform
team's policy set.

## Why policy 01 is the interesting one

Policy 01 is the only one that needs to know about *other* resources
in the cluster. The other three can inspect just the incoming
resource and make a decision. Policy 01 has to look up every
`SecurityPolicy` in the cluster and check whether any of them
references the `HTTPRoute` being admitted.

That's what Kyverno's `context.apiCall` is for. You declare an
`apiCall` block inside the rule, give it a name, point it at a K8s
API path (e.g. `/apis/gateway.envoyproxy.io/v1alpha1/securitypolicies`),
and supply a JMESPath query to shape the response into something
useful. The result becomes available as a variable inside the
`validate` block.

For policy 01, the JMESPath query is:

```
items[*].spec.targetRefs[*].name
```

which flattens the SecurityPolicy list into an array of names like
`["route-a", "route-b", ...]`. Then the `deny.conditions` block
checks whether the incoming `HTTPRoute`'s name is in that array.
If not, deny.

This pattern (cross-resource lookup at admission time) is one of the
main reasons you'd reach for Kyverno over a simpler tool. A
TypeScript GitHub Action can do something similar at PR time, but
it can only see files in the PR. Kyverno sees the whole cluster.

## Why `kyverno test` matters

When I first wrote policy 01, I tested it by applying YAML to a
live cluster and watching what got admitted. That works, but it's
slow, and the cluster state pollutes between runs. The real workflow
is `kyverno test`: you set up fixture resources, declare what should
happen for each (`pass`, `fail`, `skip`), and the CLI runs the
policy against them with no cluster involved.

For policies that use `context.apiCall`, you also supply a
`values.yaml` that mocks what the API call would return. The CLI
substitutes the mock value for the live API response and runs the
policy logic against it. That means you can test policies that
depend on cluster state without ever needing the cluster to be in
that state.

This is the dev loop that makes policy authoring practical at scale.
Without it you're shipping changes and praying.

## Layout of each policy folder

Each numbered folder is self-contained: the `ClusterPolicy` YAML,
the `kyverno-test.yaml` manifest declaring expected results, the
fixture resources, and (for policies that use `apiCall`) a
`values.yaml` that mocks the API call.

```
01-match-httproute/
  match-httproute.yaml         the ClusterPolicy
  kyverno-test.yaml            test manifest with expected pass/fail
  values.yaml                  mocks the SecurityPolicy apiCall response
  httpbin-good.yaml            HTTPRoute paired with a SecurityPolicy
  httpbin-bad.yaml             HTTPRoute alone, should be denied
```

The other three folders follow the same pattern, minus `values.yaml`
where the policy doesn't need an `apiCall`.

## Sample policies (for reference)

The `samples/` folder has two policies pulled from Kyverno's library,
plus the resources used to test them. They're not part of the
custom policy set but they were useful for seeing how the
`validate.pattern` style compares to `deny.conditions`, and for
seeing what a PolicyReport looks like under `Audit` mode.

```
samples/
  restrict-image-registries.yaml    Enforce mode, validate.pattern
  unique-ingress-host.yaml          Audit mode, context.apiCall + deny.conditions
  nginx-pod.yaml                    pod from an untrusted registry
  minimal-ingress.yaml              first ingress for the unique-host test
  minimal-ingress-not-unique.yaml   second ingress, same host — should be flagged
```

## Running tests

```bash
brew install kyverno-cli
cd 01-match-httproute/
kyverno test .
# repeat for the other folders
```

To install everything on a live cluster instead:

```bash
helm repo add kyverno https://kyverno.github.io/kyverno/
helm install kyverno kyverno/kyverno -n kyverno --create-namespace

# Envoy Gateway CRDs are required for HTTPRoute / ClientTrafficPolicy / EnvoyPatchPolicy
helm install eg oci://docker.io/envoyproxy/gateway-helm --version v0.0.0-latest \
  -n envoy-gateway-system --create-namespace
```

## A known issue worth noting

When I tested policy 01 on a live cluster, both fixture HTTPRoutes
were admitted instead of one being rejected. The `kyverno test`
version passes because the `values.yaml` mock returns the expected
array shape, but the live cluster has a subtler problem.

The likely cause: Envoy Gateway's `SecurityPolicy` supports two
field names for the target reference (`spec.targetRef` singular, the
older form, and `spec.targetRefs` plural, the current form). The
JMESPath in the policy assumes the plural form. If the cluster has
SecurityPolicies using the singular form, the path returns an empty
array and the deny condition never fires.

Diagnostic:

```bash
kubectl get securitypolicies -A -o jsonpath='{.items[*].spec}' | jq
```

If you see `targetRef:` (singular), either migrate the fixtures to
the plural form or expand the JMESPath in the policy to try both.

Also worth noting: the policy matches by name only, not by
`(namespace, name, kind)`. Two `HTTPRoute`s in different namespaces
with the same name would both pass as long as at least one
`SecurityPolicy` references that name anywhere. A v2 of the policy
would tighten this.
