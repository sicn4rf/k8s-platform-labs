# Lab 01: Multi-tier app from raw YAML, then cert-manager via Helm

The goal here was to feel the pain of wiring up a real multi-tier
application from scratch, with no charts, before installing a chart
that does the same kind of thing for you. The application is
[Forgejo](https://codeberg.org/forgejo/forgejo) (a self-hosted git
server, written in Go) backed by a MariaDB database. Both pieces need
persistent storage, and they have to talk to each other inside the
cluster.

## Why a StatefulSet for the database

A Deployment with a single replica and a regular PVC actually works
for a single-instance database, but it's not the right tool. A
StatefulSet gives you two things that matter as soon as you have
more than one replica or care about identity:

1. **Stable pod names**: `mariadb-0`, `mariadb-1`, etc., always in
   that order. A Deployment's pods get random hash suffixes.
2. **Per-replica storage via `volumeClaimTemplates`**. Each replica
   gets its own PVC automatically. If you used a Deployment with one
   shared PVC and tried to scale to 2 replicas, you'd deadlock,
   because `ReadWriteOnce` volumes can't be shared between pods.

For this lab there's only one MariaDB replica, but I used a StatefulSet
anyway because that's the pattern you'd grow into.

## Why a headless Service for the database

A regular ClusterIP Service gives you one virtual IP and load-balances
across all matching pods. That's wrong for a database, because you
usually want to talk to a *specific* pod (the primary, or
`mariadb-0`). Setting `clusterIP: None` on the Service makes it
"headless": DNS resolves directly to pod IPs instead of routing
through a virtual IP, and each StatefulSet pod gets a stable DNS name
like `mariadb-0.forgejo-db.forgejo.svc.cluster.local`.

For Forgejo this doesn't matter much (it just connects to the single
mariadb pod), but it's the same pattern you'd use for any clustered
stateful workload.

## Why `envFrom` instead of `env`

Forgejo reads a bunch of environment variables that map to its INI
config file using a `FORGEJO__<section>__<KEY>` naming convention.
You could list every one of them under `env:` in the pod template,
but that gets tedious fast. `envFrom` pulls every key from a
Secret or ConfigMap into the pod's environment in one block:

```yaml
envFrom:
  - secretRef:
      name: forgejo-secret
  - configMapRef:
      name: forgejo-config
```

The cleaner split is to put sensitive values (the DB password) in
the Secret and everything else in the ConfigMap. Pods pull both.

## File layout

```
namespace.yaml              the forgejo namespace
forgejo-db/
  secret.yaml               DB credentials (placeholders, not real values)
  service.yaml              headless Service on port 3306
  statefulset.yaml          mariadb:lts with a 1Gi PVC
forgejo-app/
  config.yaml               ConfigMap mapping FORGEJO__database__* vars
  secret.yaml               just the DB password
  forgejo-headless.yaml     for in-cluster DNS
  forgejo.yaml              NodePort on 30067 for external access
  statefulset.yaml          Forgejo with a 2Gi PVC for the repo data
helm/
  values.yaml               cert-manager override (replicaCount + crds)
  cert-manager-defaults.yaml   reference dump of the chart's full defaults
```

## Applying the lab

```bash
kind create cluster --name lab1
kubectl apply -f namespace.yaml

# database first; the app's startup probes will fail until this is up
kubectl apply -f forgejo-db/

# now the app
kubectl apply -f forgejo-app/

# verify pods can reach each other
kubectl exec -n forgejo mariadb-0 -- curl forgejo-headless.forgejo.svc.cluster.local:3000
kubectl port-forward -n forgejo forgejo-0 3000:3000

# install cert-manager via Helm with the values override
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  -n cert-manager --create-namespace \
  -f helm/values.yaml
```

## Helm gotcha worth knowing

The cert-manager chart used to expose `installCRDs=true` to install
its CRDs. They renamed it to `crds.enabled=true` in a recent release.
If you copy an example from an old StackOverflow answer with the
old name, Helm doesn't error (unknown values are silently dropped),
the install "succeeds," and the cluster ends up with no CRDs so
nothing works downstream. The `values.yaml` here uses the current
name.

This is the kind of breaking-rename platform teams own at scale: any
chart's major version bump can shuffle value names, and every
consumer has to migrate. Worth getting in the habit of reading
release notes before upgrading a chart.

## Notes

- Both Secret files use `<base64-encoded>` placeholders. Generate
  real values with `echo -n "your-password" | base64` before
  applying.
- The NodePort Service on Forgejo is mostly cosmetic. Inside a kind
  cluster, `kubectl port-forward` is simpler for local access.
- Using a StatefulSet for Forgejo itself is a stylistic choice. A
  Deployment with a single PVC would also work for one replica.
  StatefulSet was used for consistency with the DB layer.
