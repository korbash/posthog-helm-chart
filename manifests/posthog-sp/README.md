# PostHog on `sp`

Kustomize layer for the `sp` k3s server. It follows the same base-plus-patches
pattern as `manifests/posthog-ci`: `../posthog` remains the base, while this
directory carries only server-specific configuration.

## What this layer changes

- Routes PostHog through the existing Traefik ingress controller.
- Uses `posthog.94.103.93.202.nip.io` as the public hostname.
- Requests a Let's Encrypt certificate with cert-manager HTTP-01.
- Enables HTTPS values in PostHog (`SITE_URL`, secure cookies, public object
  storage endpoint).
- Pins stateful storage to the k3s `local-path` StorageClass.
- Keeps the upstream `ghcr.io/blitss/*` images and chart source intact.

## Apply order

Install Flux first if it is not present:

```bash
flux install
```

Apply cluster-wide prerequisites:

```bash
kubectl apply -k manifests/infra
```

Wait for operators and CRDs, then apply this layer:

```bash
kubectl apply -k manifests/posthog-sp
```

Check convergence:

```bash
kubectl get helmrelease -n posthog posthog
kubectl get certificate -n posthog posthog-tls
kubectl get pods -n posthog
```
