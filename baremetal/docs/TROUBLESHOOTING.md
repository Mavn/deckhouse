# FAQ и Troubleshooting

## Runbook после деплоя (3 команды)

```bash
# 1) ingress-controller на нужных LB нодах
d8 k get pods -A -o wide | grep -i ingress

# 2) frontend поды только на front нодах
d8 k -n default get pods -l app=frontend \
  -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort -u

# 3) сервис и ingress ресурсы созданы
d8 k -n default get svc frontend && d8 k -n default get ingress frontend
```

Ожидаемо:
- ingress-поды на `tdh-front-lb-01`
- frontend-поды на `tdh-front-01` и `tdh-front-02`
- ресурсы `svc/frontend` и `ingress/frontend` в статусе `Running/Ready` (где применимо)

## Быстрые проверки кластера

```bash
d8 k get nodes -o wide
d8 k get pods -A
d8 k get events -A --sort-by=.lastTimestamp | tail -n 50
```

## Ingress не маршрутизирует трафик

Проверки:

```bash
d8 k get ingressclass
d8 k get ingress -A
d8 k get pods -A -o wide | grep -i ingress
d8 k -n default describe ingress frontend
d8 k -n default get svc frontend
d8 k -n default get endpoints frontend
```

Что проверить:
- `ingressClassName` в Ingress совпадает с `ingressClass` в `IngressNginxController`.
- Ingress-controller поды запущены на `tdh-front-lb-01`.
- У сервиса `frontend` есть endpoints.

## Frontend поды не распределяются по двум front-нодам

Проверки:

```bash
d8 k -n default get deploy frontend -o yaml
d8 k -n default get pods -l app=frontend -o wide
d8 k -n default get pods -l app=frontend \
  -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort -u
```

Что проверить:
- В Deployment есть `nodeSelector: node-role.deckhouse.io/front: ""`.
- Есть toleration для `dedicated.deckhouse.io=front:NoExecute`.
- Заданы `podAntiAffinity` и `topologySpreadConstraints`.

## Service не видит backend pod'ы

Проверки:

```bash
d8 k -n default get svc frontend -o yaml
d8 k -n default get pods -l app=frontend --show-labels
d8 k -n default get endpoints frontend -o yaml
```

Что проверить:
- Метки pod'ов (`app: frontend`) совпадают с `spec.selector` сервиса.
- Pod'ы в `Running`/`Ready` состоянии.

## NodeGroup / StaticInstance проблемы (bare metal)

Проверки:

```bash
d8 k get nodegroup
d8 k get staticinstances
d8 k describe nodegroup front
d8 k describe nodegroup frontlb
```

Что проверить:
- `frontlb` имеет `count: 1` и одну static instance.
- `front` имеет `count: 2` и две static instance.

## Полезные ссылки на Deckhouse

- Общая документация: https://deckhouse.ru/
- Каталог модулей: https://deckhouse.ru/modules/
- Node Manager: https://deckhouse.ru/modules/node-manager/
- CRD NodeGroup/StaticInstance: https://deckhouse.ru/modules/node-manager/cr.html
- Ingress NGINX модуль: https://deckhouse.ru/modules/ingress-nginx/
- CRD IngressNginxController: https://deckhouse.ru/modules/ingress-nginx/cr.html
- Cloud Provider Yandex: https://deckhouse.ru/modules/cloud-provider-yandex/
