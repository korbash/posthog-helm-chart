# PostHog за внешним Caddy

Kustomize-слой для запуска `HelmRelease` за существующим Docker Compose Caddy
на сервере `posthog`.

Это текущий слой деплоя для `posthog`. Существующий Docker Compose PostHog -
legacy production, его нельзя трогать. Единственное ожидаемое пересечение с ним
- host block в Caddy, который проксирует новый Kubernetes hostname.

Схема трафика:

```text
Internet -> Docker Caddy :443 -> k8s Traefik NodePort :32080 -> PostHog
```

TLS завершается в существующем Caddy-контейнере. Kubernetes Traefik слушает
обычный HTTP `web` entryPoint, при этом PostHog всё равно получает публичный
HTTPS URL в `SITE_URL`.

## Устройство сервера

Фактически наблюдалось на `posthog`:

- OS: Ubuntu 24.04 на одной ноде.
- Kubernetes: single-node k3s `v1.35.4+k3s1`, одна control-plane нода.
- k3s установлен с отключёнными встроенными Traefik и servicelb:
  `--disable traefik --disable servicelb`.
- k3s data dir: `/mnt/k3s-data/rancher`.
- `/mnt/k3s-data` - отдельный ext4-диск с label `k3s-data`; local-path PVC уже
  создаются на нём.
- Legacy Docker Compose лежит в `/opt/containers/posthog`.
- Legacy Caddy контейнер: `posthog-proxy-1`; он занимает публичные порты `80`
  и `443`.
- Legacy compose project называется `posthog`; не перезапускать и не менять app
  services, если отдельно не занимаемся обслуживанием legacy-инстанса.

Сервис k3s Traefik управляется Flux и опубликован как NodePort:

```text
traefik/traefik  NodePort  80:32080/TCP
```

Docker Caddy ходит в этот NodePort через адрес host со стороны Docker bridge.
Временный hostname для smoke-test без правки Cloudflare DNS:

```caddy
k8s-posthog.37.27.124.54.nip.io {
    reverse_proxy http://172.18.0.1:32080
}
```

Сейчас этот блок приходит из `CADDY_EXTRA_CONFIG` в
`/opt/containers/posthog/docker-compose.yml`.

## GitOps-схема

Bootstrap-ресурсы лежат в `manifests/bootstrap/posthog-caddy`:

- `source.yaml` указывает Flux на
  `https://github.com/korbash/posthog-helm-chart.git`, branch `main`.
- `infra.yaml` применяет `./manifests/infra`.
- `posthog-caddy.yaml` применяет `./manifests/posthog-caddy` и зависит от
  `posthog-infra`.

На сервере сейчас reconcile-ятся:

```text
flux-system/posthog-infra   -> ./manifests/infra
flux-system/posthog-caddy   -> ./manifests/posthog-caddy
posthog/posthog HelmRelease -> chart posthog@0.19.3
```

Все обычные изменения Kubernetes должны идти через git, затем через Flux
reconciliation. Прямой `kubectl apply -k` использовать только для bootstrap или
аварийного восстановления.

## Что меняет этот overlay

По сравнению с `../posthog`, этот слой:

- выставляет публичный hostname `k8s-posthog.37.27.124.54.nip.io`;
- включает chart Traefik `IngressRoute`;
- направляет Traefik через plain HTTP entryPoint `web`;
- отключает TLS в Traefik, потому что HTTPS завершается во внешнем Caddy;
- выставляет `SITE_URL` и `OBJECT_STORAGE_PUBLIC_ENDPOINT` в публичный HTTPS
  URL;
- оставляет `SECURE_COOKIES=true`;
- выставляет `DISABLE_SECURE_SSL_REDIRECT=true`, потому что приложение получает
  plain HTTP от Caddy/Traefik после внешнего TLS termination;
- закрепляет persistence для Postgres, ClickHouse, Keeper, Redpanda,
  Elasticsearch, Redis и RustFS на k3s `local-path`.

Не копировать сюда certificate-модель из `posthog-sp` как есть. `posthog-sp`
использует in-cluster cert-manager и Traefik `websecure`; этот слой намеренно
использует внешний Docker Caddy для TLS.

## Текущий blocker по сертификату

Caddy должен быть настроен проксировать текущий hostname, и Kubernetes HTTP path
локально работает:

```bash
curl -I http://127.0.0.1:32080 -H 'Host: k8s-posthog.37.27.124.54.nip.io'
```

Ожидаемый результат - ответ PostHog, сейчас это `302` на `/preflight`.

Недостающий сертификат - не проблема Kubernetes `Certificate`. При первой
проверке логи Caddy показывали, что Let's Encrypt падал ещё до HTTP validation,
потому что у hostname не было публичной DNS-записи:

```text
DNS problem: NXDOMAIN looking up A for k8s-posthog.spposthog.gg
DNS problem: NXDOMAIN looking up AAAA for k8s-posthog.spposthog.gg
```

После появления DNS-записи нужно проверить, что она указывает именно на origin
server. Неверный пример, который уже встречался:

```text
k8s-posthog.spposthog.gg.  A  198.18.6.99
```

`198.18.0.0/15` - служебный benchmark/private range, это не публичный IP
сервера `posthog`. Такая запись приводит к browser error вроде
`ERR_EMPTY_RESPONSE`, потому что клиент идёт не на Caddy на `37.27.124.54`.

Правильная запись:

```text
k8s-posthog.spposthog.gg.  A  37.27.124.54
```

В Cloudflare на время выпуска сертификата проще держать запись `DNS only`
(серое облако), чтобы Let's Encrypt HTTP-01 точно попадал прямо в Caddy на
сервере. AAAA-запись нужна только если IPv6 реально маршрутизируется на сервер
и Caddy может на нём ответить. Не добавлять нерабочую AAAA-запись.

После propagation DNS Caddy должен автоматически получить сертификат. Проверки:

```bash
dig @heather.ns.cloudflare.com +short A k8s-posthog.spposthog.gg
dig @trevor.ns.cloudflare.com +short A k8s-posthog.spposthog.gg
ssh -F ~/.ssh/config posthog 'getent ahosts k8s-posthog.spposthog.gg'
ssh -F ~/.ssh/config posthog 'docker logs --tail 120 posthog-proxy-1 2>&1 | grep -E "k8s-posthog|certificate|challenge|error"'
curl -I https://k8s-posthog.spposthog.gg
```

## Временный nip.io hostname

Пока Cloudflare DNS для `k8s-posthog.spposthog.gg` не исправлен, этот overlay
использует:

```text
k8s-posthog.37.27.124.54.nip.io
```

`nip.io` кодирует IP прямо в hostname и резолвит его в `37.27.124.54`, поэтому
для временной проверки не нужна отдельная DNS-запись в зоне `spposthog.gg`.
Проверки:

```bash
dig +short A k8s-posthog.37.27.124.54.nip.io
ssh -F ~/.ssh/config posthog 'getent ahosts k8s-posthog.37.27.124.54.nip.io'
curl -I https://k8s-posthog.37.27.124.54.nip.io
```

Для возврата на постоянный домен нужно:

- исправить Cloudflare DNS `k8s-posthog.spposthog.gg A 37.27.124.54`;
- вернуть hostname в `release.patch.yaml`;
- заменить Caddy host block обратно на `k8s-posthog.spposthog.gg`.

## Безопасные проверки

Read-only проверки, которые не мешают legacy Docker Compose:

```bash
ssh -F ~/.ssh/config posthog 'kubectl get nodes -o wide'
ssh -F ~/.ssh/config posthog 'kubectl get helmrelease,kustomization,gitrepository,ocirepository -A'
ssh -F ~/.ssh/config posthog 'kubectl get pods -n posthog'
ssh -F ~/.ssh/config posthog 'kubectl get svc -n traefik traefik'
ssh -F ~/.ssh/config posthog 'docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Ports}}"'
```

Локально используем `ssh -F ~/.ssh/config`, потому что на этой машине сейчас
есть global OpenSSH config с неправильным ownership:
`/etc/ssh/ssh_config.d/20-systemd-ssh-proxy.conf`. Без `-F` SSH может упасть
ещё до подключения.

## Apply

Применять после готовности Flux и infra operators:

```bash
kubectl apply -k manifests/posthog-caddy
```

Для live-сервера предпочтительно git + Flux:

```bash
git push origin main
ssh -F ~/.ssh/config posthog 'flux reconcile source git posthog-helm-chart -n flux-system'
ssh -F ~/.ssh/config posthog 'flux reconcile kustomization posthog-caddy -n flux-system'
```
