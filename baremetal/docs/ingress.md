# Ingress и Frontend: размещение по нодам

## Цель

- `ingress-controller` работает только на ingress-нодах:
  - `tdh-front-lb-01` (1 IP)
- `frontend` поды работают только на:
  - `tdh-front-01`
  - `tdh-front-02`

## Ссылки на Deckhouse по теме Ingress/Frontend

- Ingress NGINX модуль: https://deckhouse.ru/modules/ingress-nginx/
- CRD IngressNginxController: https://deckhouse.ru/modules/ingress-nginx/cr.html
- Node Manager модуль: https://deckhouse.ru/modules/node-manager/
- CRD NodeGroup/StaticInstance: https://deckhouse.ru/modules/node-manager/cr.html

Дополнительно: FAQ и диагностика в [TROUBLESHOOTING.md](TROUBLESHOOTING.md).

> Проверьте имя ноды: иногда встречается опечатка `tdh-fron-02`.

---

## 1) Подготовка нод (Deckhouse platform layer)

Для Deckhouse static bare metal **не рекомендуется** вручную выставлять labels/taints на нодах.
Используйте `StaticInstance` + `NodeGroup`:
- `config/5_2-create-no-front-01.yml` + `config/5_3-create-no-front-02.yml` + `config/5_5-create-no-front-lb-01.yml`
- `config/5_1-create-ng-front.yml` + `config/5_4-create-ng-front-lb.yml`

Префиксы `5_1 ... 5_5` заданы специально для фиксированного порядка применения этапа 5.

Применение:

```bash
d8 k apply -f baremetal/config/5_1-create-ng-front.yml
d8 k apply -f baremetal/config/5_2-create-no-front-01.yml
d8 k apply -f baremetal/config/5_3-create-no-front-02.yml
d8 k apply -f baremetal/config/5_4-create-ng-front-lb.yml
d8 k apply -f baremetal/config/5_5-create-no-front-lb-01.yml
```

Операционный порядок для platform stage: сначала `StaticInstance`, затем `NodeGroup`.

### Safe apply (короткий runbook)

```bash
# 1) platform layer (stage 5)
d8 k apply -f baremetal/config/5_2-create-no-front-01.yml
d8 k apply -f baremetal/config/5_3-create-no-front-02.yml
d8 k apply -f baremetal/config/5_5-create-no-front-lb-01.yml
d8 k apply -f baremetal/config/5_1-create-ng-front.yml
d8 k apply -f baremetal/config/5_4-create-ng-front-lb.yml

# 2) platform check (ожидаемо: front=2, frontlb=1)
d8 k get nodegroup front frontlb
d8 k get staticinstances | grep -E 'front|front-lb'

# 3) ingress + frontend checks
d8 k get pods -A -o wide | grep -i ingress
d8 k -n default get pods -l app=frontend -o wide
```

Если шаг 2 не прошёл, не переходите к workload-изменениям до исправления состояния NodeGroup/StaticInstance.

---

## Как работает LB в baremetal и взаимодействие нод

В текущей схеме используется один внешний ingress IP на ноде `tdh-front-lb-01`.

Поток запроса:

1. Клиент приходит на внешний IP ingress-ноды `tdh-front-lb-01`.
2. `IngressNginxController` на этой ноде принимает HTTP(S)-трафик и выбирает backend по правилам `Ingress`.
3. Трафик направляется в `Service` (`ClusterIP`) приложения `frontend`.
4. `Service` через kube-proxy/CNI распределяет запрос на один из frontend pod'ов на `tdh-front-01` или `tdh-front-02`.

Роли нод:

- `frontlb` (`tdh-front-lb-01`): только входной трафик и ingress-controller.
- `front` (`tdh-front-01`, `tdh-front-02`): только frontend workload.
- control-plane/system: управление кластером, сервис-дискавери и сетевые правила.

Почему поды распределяются между двумя front-нодами:

- `nodeSelector` и `tolerations` фиксируют размещение только на `front`.
- `podAntiAffinity` и `topologySpreadConstraints` уменьшают риск, что обе реплики окажутся на одной ноде.

---

## 2) IngressNginxController (пример привязки)

```yaml
apiVersion: deckhouse.io/v1
kind: IngressNginxController
metadata:
  name: front
spec:
  ingressClass: nginx
  inlet: HostWithFailover
  nodeSelector:
    node-role.deckhouse.io/frontlb: ""
  tolerations:
    - key: dedicated.deckhouse.io
      operator: Equal
      value: frontlb
      effect: NoExecute
```

---

## 3) Frontend Deployment (пример привязки)

```yaml
spec:
  template:
    spec:
      nodeSelector:
        node-role.deckhouse.io/front: ""
      tolerations:
        - key: dedicated.deckhouse.io
          operator: Equal
          value: front
          effect: NoExecute
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                topologyKey: kubernetes.io/hostname
                labelSelector:
                  matchLabels:
                    app: frontend
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: frontend
```

---

## 4) Проверка

### Проверка labels/taints

```bash
d8 k get node tdh-front-lb-01 --show-labels
d8 k describe node tdh-front-lb-01 | grep -A3 Taints
d8 k get node tdh-front-01 --show-labels
d8 k get node tdh-front-02 --show-labels
```

### Проверка ingress-controller

```bash
d8 k get pods -A -o wide | grep -i ingress
```

Ожидаемо: ingress-поды в колонке `NODE` только на `tdh-front-lb-01`.

### Проверка frontend-подов

```bash
d8 k -n NAMESPACE get pods -l app=frontend -o wide
d8 k -n NAMESPACE get pods -l app=frontend \
  -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort -u
```

Ожидаемо: только `tdh-front-01` и `tdh-front-02`.

---

## 5) Оценка текущей структуры

### Что хорошо

- Разделение ролей корректное: ingress вынесен на отдельную ноду, frontend — на отдельные ноды.
- Схема `label + taint + nodeSelector + tolerations` реализует предсказуемое размещение подов.

### Риски

- При ошибках в имени ноды (`tdh-front-02` vs `tdh-fron-02`) размещение может работать не так, как ожидается.

### Рекомендации для production

1. Для текущей схемы используется 1 ingress-нода (`tdh-front-lb-01`, 1 IP).
2. Для frontend добавить `topologySpreadConstraints` или `podAntiAffinity`, чтобы реплики гарантированно распределялись между двумя нодами.
3. Добавить `PodDisruptionBudget` для frontend (и при необходимости для ingress), чтобы плановые операции не уронили сервис.
4. Задать `resources.requests/limits` для ingress-controller и frontend-подов.

### Когда текущая схема подходит

- Для dev/test/stage окружения текущая схема (1 ingress + 2 frontend) обычно достаточна.
