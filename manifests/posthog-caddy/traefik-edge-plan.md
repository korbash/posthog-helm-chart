# План: Traefik как внешний edge для web-routing

## Цель

Сделать Traefik единственной публичной точкой входа для HTTP/HTTPS, но пока не
трогать внешний доступ к PostgreSQL. DB routing добавим отдельным следующим
шагом.

Целевая схема сейчас:

```text
Internet :80/:443
  -> Traefik

Traefik:
  spposthog.gg
    -> Docker Caddy legacy backend

  k8s-posthog.37.27.124.54.nip.io
    -> Kubernetes PostHog services
```

## Что не делаем сейчас

- Не открываем PostgreSQL наружу.
- Не добавляем `pg-k8s...` route.
- Не меняем Cloudflare.
- Не переносим сертификат `spposthog.gg` из Caddy в Traefik.
- Не меняем логику legacy PostHog внутри Docker Compose.

## Важное решение по TLS

Для `spposthog.gg` используем TLS passthrough до Docker Caddy:

```text
Internet :443
  -> Traefik TCP SNI HostSNI(spposthog.gg)
  -> Docker Caddy :443
  -> legacy PostHog
```

Это нужно, чтобы:

- Caddy продолжил сам отдавать сертификат `spposthog.gg`;
- Caddy продолжил сам обновлять сертификат;
- не пришлось менять Cloudflare;
- legacy TLS-поведение осталось максимально прежним.

Для `k8s-posthog.37.27.124.54.nip.io` TLS может завершаться в Traefik/cert-manager,
потому что это временный `nip.io` hostname и Cloudflare здесь не участвует.

## Текущее состояние

Сейчас публичные порты заняты Docker Caddy:

```text
posthog-proxy-1
  0.0.0.0:80->80/tcp
  0.0.0.0:443->443/tcp
```

Kubernetes Traefik сейчас не является внешним edge:

```text
traefik/traefik NodePort 80:32080/TCP
```

Текущий web path в Kubernetes:

```text
Internet :443
  -> Docker Caddy
  -> http://172.18.0.1:32080
  -> Traefik NodePort
  -> PostHog IngressRoute
```

## План работ

### 1. Подготовить Traefik как public edge

Для текущего single-node сервера самый простой вариант:

```text
Traefik hostNetwork: true
```

Traefik должен слушать:

```text
:80
:443
```

DB порт `:5432` пока не добавляем.

### 2. Сделать Docker Caddy внутренним backend

Docker Caddy должен перестать занимать публичные `0.0.0.0:80` и `0.0.0.0:443`,
но остаться доступным для Traefik.

Вариант:

```text
127.0.0.1:18080 -> Docker Caddy :80
127.0.0.1:18443 -> Docker Caddy :443
```

Или другой host-only/private адрес, доступный Traefik.

Важно: Caddy должен сохранить возможность принимать HTTP-01 challenge для
`spposthog.gg` через Traefik route на `:80`.

### 3. Добавить Kubernetes Service для legacy Caddy backend

Создать Service без selector и EndpointSlice/Endpoints, указывающий на внутренний
адрес Docker Caddy.

Пример логической цели:

```text
posthog/legacy-caddy-http  -> 127.0.0.1:18080
posthog/legacy-caddy-https -> 127.0.0.1:18443
```

Точную реализацию выбрать с учетом того, какой адрес будет доступен из Traefik
pod при `hostNetwork: true`.

### 4. Route для legacy HTTP

Обычный HTTP route:

```text
Host(`spposthog.gg`)
  -> legacy Caddy :80
```

Назначение:

- HTTP redirect legacy;
- ACME HTTP-01 для Caddy;
- совместимость с текущим поведением.

### 5. Route для legacy HTTPS

TCP route с TLS passthrough:

```text
HostSNI(`spposthog.gg`)
  -> legacy Caddy :443
```

Traefik не завершает TLS для `spposthog.gg`.

### 6. Route для Kubernetes PostHog

Оставить существующую PostHog маршрутизацию через Traefik, но убрать зависимость
от Docker Caddy и NodePort bridge.

Hostname:

```text
k8s-posthog.37.27.124.54.nip.io
```

Routes остаются такими же по смыслу:

```text
/i/v0/ai             -> posthog-capture-ai
/i/v1/logs,traces    -> posthog-capture-logs
/e,/i/v0,/batch,...  -> posthog-capture
/s                   -> posthog-replay-capture
/flags               -> posthog-feature-flags
/surveys,/array      -> posthog-hypercache-server
/livestream          -> posthog-livestream
/public/webhooks     -> posthog-plugins
/posthog             -> posthog-rustfs-svc
fallback             -> posthog-web
```

## Порядок переключения

1. Добавить GitOps-манифесты для Traefik edge-режима и legacy routes.
2. Подготовить Docker Caddy на внутренних портах.
3. Проверить, что Traefik может достучаться до Docker Caddy backend.
4. Освободить публичные `80/443` у Docker Caddy.
5. Включить Traefik на публичных `80/443`.
6. Сразу проверить legacy:

```bash
curl -I http://spposthog.gg
curl -I https://spposthog.gg
```

7. Проверить Kubernetes PostHog:

```bash
curl -I https://k8s-posthog.37.27.124.54.nip.io
```

## Риски

- Самый рискованный момент: передача публичных `80/443` от Docker Caddy к Traefik.
- Если route `spposthog.gg :80` не попадет в Caddy, может сломаться renewal через
  HTTP-01.
- Если `spposthog.gg :443` сделать TLS termination в Traefik вместо passthrough,
  придется переносить сертификат и это уже может потребовать дополнительных
  DNS/Cloudflare решений.

## Критерии готовности

- `spposthog.gg` открывается как раньше.
- Сертификат `spposthog.gg` по-прежнему отдает Docker Caddy.
- `k8s-posthog.37.27.124.54.nip.io` открывается через Traefik напрямую.
- Docker Caddy больше не является публичным first-hop для Kubernetes PostHog.
- Cloudflare не менялся.
- PostgreSQL наружу пока не открыт.
