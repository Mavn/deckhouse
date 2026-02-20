# Deckhouse EE Bare Metal - Документация

## Назначение

Этот документ описывает bare metal конфигурацию кластера из папки `config/` и правила размещения frontend/ingress.

## Архитектурный подход (Deckhouse-first)

- **Платформенный слой (Deckhouse CRD, состав)**: `StaticInstance`, `NodeGroup`, `IngressNginxController`.
- **Прикладной слой (API workload в Deckhouse-кластере, состав)**: `Deployment`, `Service`, `Ingress`, `PDB`, `HPA`, `NetworkPolicy`.
- Приоритет изменений: сначала платформа Deckhouse (ноды/роль/инлет), затем прикладные объекты. Фактический порядок выполнения приведен в разделе «Порядок применения».

## Навигация

- [Локальный индекс](INDEX.md)
- [FAQ и troubleshooting](TROUBLESHOOTING.md)
- [Схема bare metal](BAREMETAL-SCHEMA.md)
- [Frontend + Ingress (bare metal)](FRONTEND_INGRESS_BAREMETAL.md)
- [Ingress/Frontend проверки](ingress.md)

## Ссылки на Deckhouse по теме bare metal

- Node Manager: https://deckhouse.ru/modules/node-manager/
- CRD NodeGroup/StaticInstance: https://deckhouse.ru/modules/node-manager/cr.html
- Ingress NGINX модуль: https://deckhouse.ru/modules/ingress-nginx/
- CRD IngressNginxController: https://deckhouse.ru/modules/ingress-nginx/cr.html
- Ingress ресурсы Kubernetes в Deckhouse: https://deckhouse.ru/modules/ingress-nginx/faq.html
- CSI NFS модуль: https://deckhouse.ru/modules/csi-nfs/

### Ссылки для HPA / PDB / NetworkPolicy

- Prometheus Metrics Adapter (метрики для HPA в Deckhouse): https://deckhouse.ru/modules/prometheus-metrics-adapter/
- CNI Cilium (поддержка NetworkPolicy в Deckhouse): https://deckhouse.ru/modules/cni-cilium/
- Admission Policy Engine (валидация и ограничения admission stage): https://deckhouse.ru/modules/admission-policy-engine/
- Kubernetes API reference в Deckhouse-кластере (используются нативные ресурсы):
  - HPA: https://kubernetes.io/docs/concepts/workloads/autoscaling/
  - PodDisruptionBudget: https://kubernetes.io/docs/concepts/workloads/pods/disruptions/
  - NetworkPolicy: https://kubernetes.io/docs/concepts/services-networking/network-policies/

Кто за что отвечает:
- `admission-policy-engine` — контроль правил на этапе `create/update` ресурсов.
- `NetworkPolicy` — контроль сетевого трафика между workload-объектами в runtime.
- `PodDisruptionBudget` — защита доступности при добровольных эвикшенах.
- `HorizontalPodAutoscaler` — автоскейлинг реплик по метрикам.

Используется или нет (в рамках `baremetal/config`):
- Node Manager — **используется** через `NodeGroup/StaticInstance` файлы: `1-create-worker-ng.yml`, `3-create-worker01.yml`, `4-create-ng-system.yml`, `4-create-no-system.yml`, `5_1-create-ng-front.yml`, `5_2-create-no-front-01.yml`, `5_3-create-no-front-02.yml`, `5_4-create-ng-front-lb.yml`, `5_5-create-no-front-lb-01.yml`, `5_6-create-no-monitoring-01.yml`, `5_7-create-ng-monitoring.yml`.
- Ingress NGINX — **используется** через `6-create-ingress-nginx.yml` и `7_3-frontend-ingress.yml`.
- CSI NFS — **используется** через `8_1-enable-csi-nfs.yml` и `8_2-storageclass-nfs-external.yml` (внешний NFS, без `StaticInstance/NodeGroup` для NFS).
- Prometheus Metrics Adapter — **используется (косвенно)** через `7_5-frontend-hpa.yml`.
- CNI Cilium — **используется (косвенно)** через `7_6-frontend-networkpolicy.yml`.
- Admission Policy Engine — **не используется** (policy-манифесты admission stage в `baremetal/config` отсутствуют).
- PodDisruptionBudget (Kubernetes) — **используется** через `7_4-frontend-pdb.yml`.
- HorizontalPodAutoscaler (Kubernetes) — **используется** через `7_5-frontend-hpa.yml`.
- NetworkPolicy (Kubernetes) — **используется** через `7_6-frontend-networkpolicy.yml`.

## Конфигурационные файлы bare metal

### Легенда нумерации (этапы и категории)

- `1-*` — базовая worker-группа.
- `2-*` — доступ/credentials.
- `3-*` — базовый worker static instance.
- `4-*` — system группа и system static instance.
- `5_*` — front/frontlb/monitoring platform-слой (NodeGroup + StaticInstance).
- `6-*` — ingress-controller (`IngressNginxController`).
- `7_*` — прикладной workload-слой frontend (`Deployment/Service/Ingress/PDB/HPA/NetworkPolicy`).
- `8_*` — external NFS storage (`ModuleConfig csi-nfs` + `StorageClass`).

Правило: нумерация отражает этап и категорию. Операционный порядок применения (сначала ноды, потом группы нод) задан в разделе ниже.

### Базовые объекты
- [config/2-create-ssh-cred.yml](../config/2-create-ssh-cred.yml) - SSH credentials
- [config/3-create-worker01.yml](../config/3-create-worker01.yml) - StaticInstance `tdh-worker-01`
- [config/4-create-no-system.yml](../config/4-create-no-system.yml) - StaticInstance `tdh-system-01`
- [config/1-create-worker-ng.yml](../config/1-create-worker-ng.yml) - NodeGroup `worker`
- [config/4-create-ng-system.yml](../config/4-create-ng-system.yml) - NodeGroup `system`

### Платформенный слой (Deckhouse)
- [config/5_2-create-no-front-01.yml](../config/5_2-create-no-front-01.yml) - StaticInstance `tdh-front-01`
- [config/5_3-create-no-front-02.yml](../config/5_3-create-no-front-02.yml) - StaticInstance `tdh-front-02`
- [config/5_5-create-no-front-lb-01.yml](../config/5_5-create-no-front-lb-01.yml) - StaticInstance `tdh-front-lb-01`
- [config/5_6-create-no-monitoring-01.yml](../config/5_6-create-no-monitoring-01.yml) - StaticInstance `tdh-monitoring-01`
- [config/5_1-create-ng-front.yml](../config/5_1-create-ng-front.yml) - NodeGroup `front` (frontend)
- [config/5_4-create-ng-front-lb.yml](../config/5_4-create-ng-front-lb.yml) - NodeGroup `frontlb` (ingress node)
- [config/5_7-create-ng-monitoring.yml](../config/5_7-create-ng-monitoring.yml) - NodeGroup `monitoring`
- [config/6-create-ingress-nginx.yml](../config/6-create-ingress-nginx.yml) - `IngressNginxController`

Префиксы `5_1 ... 5_5` и `7_1 ... 7_6` отражают рекомендуемый порядок применения внутри этапов 5 и 7.

### Прикладной слой (API workload в Deckhouse)
- [config/7_1-frontend-deployment.yml](../config/7_1-frontend-deployment.yml) - Frontend Deployment
- [config/7_2-frontend-service.yml](../config/7_2-frontend-service.yml) - Frontend Service (ClusterIP)
- [config/7_3-frontend-ingress.yml](../config/7_3-frontend-ingress.yml) - Frontend Ingress (route to Service)
- [config/7_4-frontend-pdb.yml](../config/7_4-frontend-pdb.yml) - PodDisruptionBudget для frontend
- [config/7_5-frontend-hpa.yml](../config/7_5-frontend-hpa.yml) - HorizontalPodAutoscaler для frontend
- [config/7_6-frontend-networkpolicy.yml](../config/7_6-frontend-networkpolicy.yml) - NetworkPolicy для frontend

### Storage слой (external NFS)
- [config/8_1-enable-csi-nfs.yml](../config/8_1-enable-csi-nfs.yml) - включает модуль `csi-nfs`
- [config/8_2-storageclass-nfs-external.yml](../config/8_2-storageclass-nfs-external.yml) - `StorageClass nfs-external` на сервер `192.168.142.99`
- [config/8_4-csi-nfs-tls-optional.yml](../config/8_4-csi-nfs-tls-optional.yml) - **опционально**: TLS для `csi-nfs` (CA secret + `tlsParameters.ca`)
- [config/8_5-pvc-nfs-external-example.yml](../config/8_5-pvc-nfs-external-example.yml) - **опционально**: тестовый PVC для проверки provisioning
- [config/8_3-monitoring-pvc-mon-repl-compat.yml](../config/8_3-monitoring-pvc-mon-repl-compat.yml) - PVC-совместимость `mon-repl-pvc` в `d8-monitoring` на `nfs-external`

## Целевая схема размещения

- `ingress-controller` работает только на ноде `tdh-front-lb-01` (NodeGroup `frontlb`, 1 IP).
- `frontend` workload работает только на нодах `tdh-front-01` и `tdh-front-02` (NodeGroup `front`).

## Порядок применения

Порядок для Deckhouse-first:
1) применяем платформенный слой (`StaticInstance` → `NodeGroup` → `IngressNginxController`),
2) затем применяем прикладной слой (Deployment/Service/Ingress/PDB/HPA/NetworkPolicy).

Примечание: все команды в этом документе ориентированы на bare metal направление и выполняются через `d8 k ...`.

```bash
d8 k apply -f baremetal/config/2-create-ssh-cred.yml
d8 k apply -f baremetal/config/3-create-worker01.yml
d8 k apply -f baremetal/config/4-create-no-system.yml
d8 k apply -f baremetal/config/5_2-create-no-front-01.yml
d8 k apply -f baremetal/config/5_3-create-no-front-02.yml
d8 k apply -f baremetal/config/5_5-create-no-front-lb-01.yml
d8 k apply -f baremetal/config/5_6-create-no-monitoring-01.yml
d8 k apply -f baremetal/config/1-create-worker-ng.yml
d8 k apply -f baremetal/config/4-create-ng-system.yml
d8 k apply -f baremetal/config/5_1-create-ng-front.yml
d8 k apply -f baremetal/config/5_4-create-ng-front-lb.yml
d8 k apply -f baremetal/config/5_7-create-ng-monitoring.yml
d8 k apply -f baremetal/config/6-create-ingress-nginx.yml
d8 k apply -f baremetal/config/8_1-enable-csi-nfs.yml
d8 k apply -f baremetal/config/8_4-csi-nfs-tls-optional.yml # optional, only if TLS is needed
d8 k apply -f baremetal/config/8_2-storageclass-nfs-external.yml
d8 k apply -f baremetal/config/8_5-pvc-nfs-external-example.yml # optional, verification
d8 k apply -f baremetal/config/8_3-monitoring-pvc-mon-repl-compat.yml
d8 k apply -f baremetal/config/7_1-frontend-deployment.yml
d8 k apply -f baremetal/config/7_2-frontend-service.yml
d8 k apply -f baremetal/config/7_3-frontend-ingress.yml
d8 k apply -f baremetal/config/7_4-frontend-pdb.yml
d8 k apply -f baremetal/config/7_5-frontend-hpa.yml
d8 k apply -f baremetal/config/7_6-frontend-networkpolicy.yml
```

### Быстрый safe apply (1 ingress IP + 2 frontend)

Короткий сценарий с промежуточной проверкой, чтобы не накатывать workload до готовности платформы.

```bash
# 1) platform layer
d8 k apply -f baremetal/config/2-create-ssh-cred.yml
d8 k apply -f baremetal/config/3-create-worker01.yml
d8 k apply -f baremetal/config/4-create-no-system.yml
d8 k apply -f baremetal/config/5_2-create-no-front-01.yml
d8 k apply -f baremetal/config/5_3-create-no-front-02.yml
d8 k apply -f baremetal/config/5_5-create-no-front-lb-01.yml
d8 k apply -f baremetal/config/5_6-create-no-monitoring-01.yml
d8 k apply -f baremetal/config/1-create-worker-ng.yml
d8 k apply -f baremetal/config/4-create-ng-system.yml
d8 k apply -f baremetal/config/5_1-create-ng-front.yml
d8 k apply -f baremetal/config/5_4-create-ng-front-lb.yml
d8 k apply -f baremetal/config/5_7-create-ng-monitoring.yml
d8 k apply -f baremetal/config/6-create-ingress-nginx.yml
d8 k apply -f baremetal/config/8_1-enable-csi-nfs.yml
d8 k apply -f baremetal/config/8_4-csi-nfs-tls-optional.yml # optional, only if TLS is needed
d8 k apply -f baremetal/config/8_2-storageclass-nfs-external.yml
d8 k apply -f baremetal/config/8_5-pvc-nfs-external-example.yml # optional, verification
d8 k apply -f baremetal/config/8_3-monitoring-pvc-mon-repl-compat.yml

# 2) platform check (ожидаемо: front=2, frontlb=1)
d8 k get nodegroup front frontlb
d8 k get staticinstances | grep -E 'front|front-lb'

# 3) workload layer
d8 k apply -f baremetal/config/7_1-frontend-deployment.yml
d8 k apply -f baremetal/config/7_2-frontend-service.yml
d8 k apply -f baremetal/config/7_3-frontend-ingress.yml
d8 k apply -f baremetal/config/7_4-frontend-pdb.yml
d8 k apply -f baremetal/config/7_5-frontend-hpa.yml
d8 k apply -f baremetal/config/7_6-frontend-networkpolicy.yml

# 4) final check
d8 k get pods -A -o wide | grep -i ingress
d8 k -n default get pods -l app=frontend -o wide
d8 k -n default get pvc nfs-external-test-pvc
d8 k -n d8-monitoring get pvc mon-repl-pvc
```

Если шаг 2 не прошёл (NodeGroup/StaticInstance не в ожидаемом состоянии), workload-слой не применять до исправления.

Для external NFS проверки:
- `STATUS=Bound` у `nfs-external-test-pvc` — provisioning успешен.
- `STATUS=Pending` — проверить доступность NFS сервера `192.168.142.99`, export path и настройки TLS (если включен `8_3`).

## Affinity для frontend (рекомендуется)

Ссылки по теме (используется в Deckhouse-кластере):
- Deckhouse Node Manager (размещение нод/ролей): https://deckhouse.ru/modules/node-manager/
- Deckhouse Ingress NGINX: https://deckhouse.ru/modules/ingress-nginx/

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

## Проверка размещения

```bash
d8 k get pods -A -o wide | grep -i ingress
d8 k -n NAMESPACE get pods -l app=frontend \
  -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort -u
d8 k -n default get svc frontend
d8 k -n default get ingress frontend
d8 k -n default get pdb frontend-pdb
d8 k -n default get hpa frontend-hpa
d8 k -n default get networkpolicy frontend-ingress-policy
```

Ожидаемо:
- ingress-поды только на `tdh-front-lb-01`
- frontend-поды только на `tdh-front-01` и `tdh-front-02`

## Удаление (bare metal teardown)

Рекомендуется удалять в обратном порядке: сначала workload слой, затем платформенный слой.

```bash
# workload layer
d8 k delete -f baremetal/config/7_6-frontend-networkpolicy.yml
d8 k delete -f baremetal/config/7_5-frontend-hpa.yml
d8 k delete -f baremetal/config/7_4-frontend-pdb.yml
d8 k delete -f baremetal/config/7_3-frontend-ingress.yml
d8 k delete -f baremetal/config/7_2-frontend-service.yml
d8 k delete -f baremetal/config/7_1-frontend-deployment.yml
d8 k delete -f baremetal/config/8_3-monitoring-pvc-mon-repl-compat.yml
d8 k delete -f baremetal/config/8_5-pvc-nfs-external-example.yml
d8 k delete -f baremetal/config/8_2-storageclass-nfs-external.yml
d8 k delete -f baremetal/config/8_1-enable-csi-nfs.yml

# platform layer
d8 k delete -f baremetal/config/6-create-ingress-nginx.yml
d8 k delete -f baremetal/config/5_7-create-ng-monitoring.yml
d8 k delete -f baremetal/config/5_4-create-ng-front-lb.yml
d8 k delete -f baremetal/config/5_1-create-ng-front.yml
d8 k delete -f baremetal/config/4-create-ng-system.yml
d8 k delete -f baremetal/config/1-create-worker-ng.yml
d8 k delete -f baremetal/config/5_6-create-no-monitoring-01.yml
d8 k delete -f baremetal/config/5_5-create-no-front-lb-01.yml
d8 k delete -f baremetal/config/5_3-create-no-front-02.yml
d8 k delete -f baremetal/config/5_2-create-no-front-01.yml
d8 k delete -f baremetal/config/4-create-no-system.yml
d8 k delete -f baremetal/config/3-create-worker01.yml
d8 k delete -f baremetal/config/2-create-ssh-cred.yml
```

Проверка:

```bash
d8 k get nodegroup
d8 k get staticinstances
d8 k get ingress -A
d8 k get deploy,svc,pdb,hpa,networkpolicy -n default
```
