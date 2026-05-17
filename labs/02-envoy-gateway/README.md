# Lab 02: Envoy Gateway with TLS, retries, and JWT

This lab takes the Kubernetes Gateway API and the
[Envoy Gateway](https://gateway.envoyproxy.io/) implementation of it,
sets up a working `Gateway` + `HTTPRoute`, and then layers on the
three policy CRDs that Envoy Gateway adds on top of the standard
Gateway API. By the end you've got a gateway that enforces a
minimum TLS version on incoming connections, retries failed upstream
requests, and rejects requests without a valid JWT.

## How Envoy Gateway is actually structured

The thing that surprised me about Envoy Gateway is that it's not a
single component. There's a control plane (a deployment of the
Envoy Gateway controller) running in the `envoy-gateway-system`
namespace, and then there are data plane pods (actual Envoy proxies)
that get created dynamically when you apply a `Gateway` resource.

The flow is:

1. You apply a `Gateway` in some namespace.
2. The Envoy Gateway controller watches the K8s API for changes.
3. It sees the new Gateway, decides "okay, I need a data plane for
   this," and spins up a Deployment + Service for an Envoy proxy in
   `envoy-gateway-system` (not your namespace).
4. As you add `HTTPRoute`, `SecurityPolicy`, etc., the controller
   translates them into Envoy config and pushes the config to the
   proxy pods via xDS.
5. Client traffic only ever hits the proxy pods. The Gateway
   resource itself is just declarative intent; it doesn't serve
   traffic.

That control-plane/data-plane split is what makes the federated
gateway pattern work. Dev teams apply `Gateway` and `HTTPRoute` in
their own namespace; the platform team owns the Envoy proxy pods in
`envoy-gateway-system`. RBAC enforces that boundary.

## What's in this lab

```
ctp.yaml    ClientTrafficPolicy — TLS minVersion 1.2, maxVersion 1.3
btp.yaml    BackendTrafficPolicy — retry 3x on 500/502/503 with backoff
jwt.yaml    SecurityPolicy — JWT validation against a public JWKS
```

The Gateway, HTTPRoute, and the test backend (`httpbin`-style echo
service) come from Envoy Gateway's official quickstart and aren't
checked in here. See "Prerequisites" below.

The TLS cert and key files aren't in the repo either. They're
gitignored because key material should never be committed.
Regenerate locally with:

```bash
openssl req -x509 -newkey rsa:2048 -keyout tls.key -out tls.crt \
  -days 365 -nodes -subj "/CN=www.example.com"
kubectl create secret tls example-cert --cert=tls.crt --key=tls.key
```

The Gateway resource references `example-cert` as the TLS Secret for
its HTTPS listener.

## Prerequisites

```bash
# Install Envoy Gateway via Helm
helm install eg oci://docker.io/envoyproxy/gateway-helm \
  --version v0.0.0-latest \
  -n envoy-gateway-system --create-namespace

# Apply Envoy Gateway's quickstart (GatewayClass + Gateway + HTTPRoute + backend)
kubectl apply -f https://github.com/envoyproxy/gateway/releases/download/latest/quickstart.yaml

# Confirm the data plane got provisioned
kubectl get gateway -A
kubectl get pods -n envoy-gateway-system
```

Then apply the policies:

```bash
kubectl apply -f ctp.yaml
kubectl apply -f btp.yaml
kubectl apply -f jwt.yaml
```

## How to verify (and why negative tests matter)

The lesson I kept coming back to in this lab: it's easy to apply a
policy YAML and assume it works because nothing errored. That's not
verification. Verification means deliberately trying to do the
thing the policy is supposed to block, and watching it get blocked.

### TLS minVersion

Positive test: a normal HTTPS request should work.

```bash
curl -kv --resolve www.example.com:8443:127.0.0.1 \
  https://www.example.com:8443/get
```

Negative test: force an old TLS version and confirm the server
rejects it.

```bash
curl -k --tls-max 1.1 --tlsv1.0 \
  --resolve www.example.com:8443:127.0.0.1 \
  https://www.example.com:8443/get
# expected: tlsv1 alert protocol version
```

If both behaviors match, the minVersion is actually being enforced.
If only the positive test succeeds, you've proven nothing.

### Retries

Hit a path that returns a 500, and look at Envoy's access log instead
of the curl output. The response code on the client side will be 500
either way (Envoy gave up after exhausting retries); the proof is in
the log.

```bash
curl -k --resolve www.example.com:8443:127.0.0.1 \
  https://www.example.com:8443/status/500
kubectl logs -n envoy-gateway-system -l app.kubernetes.io/name=envoy -f
```

Look for `"response_flags": "URX"` in the JSON log. `URX` means
"Upstream Retry Limit eXceeded": Envoy tried, retried up to the
configured count, and gave up. Without retries enabled, you'd see
no `URX` flag and the request duration would be roughly the time of
a single backend call.

### JWT

Without a token, the request should fail.

```bash
curl -k --resolve www.example.com:8443:127.0.0.1 \
  https://www.example.com:8443/get
# expected: 401 with "Jwt is missing"
```

With a valid token from the issuer the SecurityPolicy trusts:

```bash
curl -k -H "Authorization: Bearer $TOKEN" \
  --resolve www.example.com:8443:127.0.0.1 \
  https://www.example.com:8443/get
# expected: 200
```

The JWT validation happens at the Envoy proxy, not the backend. The
backend never sees an unauthenticated request.

## Inspecting xDS config

Once everything's running, you can dump the actual Envoy config
that the controller pushed to the proxy:

```bash
egctl config envoy-proxy listener   # bound port + RDS routeConfigName link
egctl config envoy-proxy route      # virtual hosts, path matches, cluster refs
egctl config envoy-proxy cluster    # backend resolution (CDS)
```

This is the "click moment" of the lab. Every resource in the dump
includes a `metadata.filterMetadata.envoy-gateway.resources` field
that points back to the K8s resource it came from (the Gateway, the
HTTPRoute, etc.). When something's misbehaving in a real cluster,
that field is how you reverse-map "weird Envoy behavior" to "which
YAML to open."
