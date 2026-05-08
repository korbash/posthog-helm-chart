# PostHog behind external Caddy

Kustomize layer for running the HelmRelease behind the existing Docker Compose
Caddy on `posthog`.

Traffic shape:

```text
Internet -> Docker Caddy :443 -> k8s Traefik NodePort :32080 -> PostHog
```

TLS is terminated by the existing Caddy container. Kubernetes Traefik listens on
the plain `web` entryPoint, while PostHog still receives the public HTTPS URL in
`SITE_URL`.

Apply after Flux and the infra operators are ready:

```bash
kubectl apply -k manifests/posthog-caddy
```
