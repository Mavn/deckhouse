# Frontend + Ingress layout (bare metal)

## Карта файлов (быстрые ссылки)

### Документация

1. [INDEX.md](INDEX.md) - локальный индекс baremetal-документов
2. [BAREMETAL.md](BAREMETAL.md) - основной runbook по baremetal
3. [FRONTEND_INGRESS_BAREMETAL.md](FRONTEND_INGRESS_BAREMETAL.md) - этот документ (детали frontend + ingress)
4. [ingress.md](ingress.md) - размещение ingress/frontend и проверки
5. [BAREMETAL-SCHEMA.md](BAREMETAL-SCHEMA.md) - платформенная схема
6. [project-schema.md](project-schema.md) - схема NodeGroup/нод/ролей
7. [TROUBLESHOOTING.md](TROUBLESHOOTING.md) - FAQ и диагностика

### Конфигурация (платформенный слой)

- [../config/2-create-ssh-cred.yml](../config/2-create-ssh-cred.yml) - SSH credentials
- [../config/3-create-worker01.yml](../config/3-create-worker01.yml) - StaticInstance `tdh-worker-01`
- [../config/4-create-no-system.yml](../config/4-create-no-system.yml) - StaticInstance `tdh-system-01`
- [../config/5_2-create-no-front-01.yml](../config/5_2-create-no-front-01.yml) - StaticInstance `tdh-front-01`
- [../config/5_3-create-no-front-02.yml](../config/5_3-create-no-front-02.yml) - StaticInstance `tdh-front-02`
- [../config/5_5-create-no-front-lb-01.yml](../config/5_5-create-no-front-lb-01.yml) - StaticInstance `tdh-front-lb-01`
- [../config/5_6-create-no-monitoring-01.yml](../config/5_6-create-no-monitoring-01.yml) - StaticInstance `tdh-monitoring-01`
- [../config/1-create-worker-ng.yml](../config/1-create-worker-ng.yml) - NodeGroup `worker`
- [../config/4-create-ng-system.yml](../config/4-create-ng-system.yml) - NodeGroup `system`
- [../config/5_1-create-ng-front.yml](../config/5_1-create-ng-front.yml) - NodeGroup `front`
- [../config/5_4-create-ng-front-lb.yml](../config/5_4-create-ng-front-lb.yml) - NodeGroup `frontlb`
- [../config/5_7-create-ng-monitoring.yml](../config/5_7-create-ng-monitoring.yml) - NodeGroup `monitoring`
- [../config/6-create-ingress-nginx.yml](../config/6-create-ingress-nginx.yml) - `IngressNginxController`

### Конфигурация (workload слой)

- [../config/7_1-frontend-deployment.yml](../config/7_1-frontend-deployment.yml) - Deployment `frontend`
- [../config/7_2-frontend-service.yml](../config/7_2-frontend-service.yml) - Service `frontend`
- [../config/7_3-frontend-ingress.yml](../config/7_3-frontend-ingress.yml) - Ingress `frontend`
- [../config/7_4-frontend-pdb.yml](../config/7_4-frontend-pdb.yml) - PodDisruptionBudget
- [../config/7_5-frontend-hpa.yml](../config/7_5-frontend-hpa.yml) - HorizontalPodAutoscaler
- [../config/7_6-frontend-networkpolicy.yml](../config/7_6-frontend-networkpolicy.yml) - NetworkPolicy

### Конфигурация (storage слой, external NFS)

- [../config/8_1-enable-csi-nfs.yml](../config/8_1-enable-csi-nfs.yml) - включает модуль `csi-nfs`
- [../config/8_2-storageclass-nfs-external.yml](../config/8_2-storageclass-nfs-external.yml) - `StorageClass nfs-external` (`192.168.142.99`)
- [../config/8_4-csi-nfs-tls-optional.yml](../config/8_4-csi-nfs-tls-optional.yml) - **опционально**: TLS для `csi-nfs`
- [../config/8_5-pvc-nfs-external-example.yml](../config/8_5-pvc-nfs-external-example.yml) - **опционально**: тестовый PVC на `nfs-external`
- [../config/8_3-monitoring-pvc-mon-repl-compat.yml](../config/8_3-monitoring-pvc-mon-repl-compat.yml) - совместимый PVC `mon-repl-pvc` для monitoring

### Легенда нумерации для этого сценария

- `5_*` — platform-объекты для `front/frontlb/monitoring` нод.
- `6-*` — `IngressNginxController`.
- `7_*` — frontend workload-объекты (`Deployment`, `Service`, `Ingress`, `PDB`, `HPA`, `NetworkPolicy`).
- `8_*` — external NFS (`ModuleConfig csi-nfs`, `StorageClass`).

Нумерация отражает этапы/категории, а не всегда буквальный операционный порядок.

### Порядок применения (исполнять именно так)

1. `2-create-ssh-cred.yml`
2. `3-create-worker01.yml`
3. `4-create-no-system.yml`
4. `5_2-create-no-front-01.yml`
5. `5_3-create-no-front-02.yml`
6. `5_5-create-no-front-lb-01.yml`
7. `5_6-create-no-monitoring-01.yml`
8. `1-create-worker-ng.yml`
9. `4-create-ng-system.yml`
10. `5_1-create-ng-front.yml`
11. `5_4-create-ng-front-lb.yml`
12. `5_7-create-ng-monitoring.yml`
13. `6-create-ingress-nginx.yml`
14. `7_1-frontend-deployment.yml`
15. `7_2-frontend-service.yml`
16. `7_3-frontend-ingress.yml`
17. `7_4-frontend-pdb.yml`
18. `7_5-frontend-hpa.yml`
19. `7_6-frontend-networkpolicy.yml`
20. `8_1-enable-csi-nfs.yml`
21. `8_2-storageclass-nfs-external.yml`
22. `8_4-csi-nfs-tls-optional.yml` (опционально, только при TLS)
23. `8_5-pvc-nfs-external-example.yml` (опционально, проверка provisioning)
24. `8_3-monitoring-pvc-mon-repl-compat.yml`

### Где используется каждый файл (связи, НЕ порядок применения)

- `6-create-ingress-nginx.yml`:
  - создает `IngressNginxController`, который обслуживает правила из `7_3-frontend-ingress.yml`.

- `7_1-frontend-deployment.yml`:
  - создает pod'ы `app=frontend`;
  - используется `7_2-frontend-service.yml` (через `spec.selector: app=frontend`);
  - используется `7_5-frontend-hpa.yml` (через `scaleTargetRef: Deployment/frontend`);
  - используется `7_4-frontend-pdb.yml` и `7_6-frontend-networkpolicy.yml` (через `podSelector: app=frontend`).

- `7_2-frontend-service.yml`:
  - публикует frontend внутри кластера (`ClusterIP`, порты `80/443`);
  - используется `7_3-frontend-ingress.yml` как backend (`service: frontend`, порт `80`).

- `7_3-frontend-ingress.yml`:
  - описывает маршрут `host/path -> Service/frontend`;
  - используется ingress-controller из `6-create-ingress-nginx.yml` (класс `nginx`).

- `7_5-frontend-hpa.yml`:
  - масштабирует `Deployment/frontend` по CPU метрикам.

- `7_4-frontend-pdb.yml`:
  - защищает pod'ы `app=frontend` при drain/обновлениях.

- `7_6-frontend-networkpolicy.yml`:
  - ограничивает ingress-трафик к pod'ам `app=frontend`.

## Цель

- Ingress controller запускается только на узлах:
  - `tdh-front-lb-01` (1 IP)
- Frontend workload запускается только на узлах:
  - `tdh-front-01`
  - `tdh-front-02`

## Модель слоя (Deckhouse-first)

- **Deckhouse platform layer (состав)**: `StaticInstance`, `NodeGroup`, `IngressNginxController`.
- **Workload layer (API workload в Deckhouse-кластере, состав)**: `Deployment`, `Service`, `Ingress`, `PDB`, `HPA`, `NetworkPolicy`.
- Операционный порядок для этого документа: сначала `StaticInstance`, затем `NodeGroup`, затем `IngressNginxController`, потом workload.

## Ссылки на Deckhouse по теме

- Ingress NGINX модуль: https://deckhouse.ru/modules/ingress-nginx/
- CRD IngressNginxController: https://deckhouse.ru/modules/ingress-nginx/cr.html
- Node Manager модуль: https://deckhouse.ru/modules/node-manager/
- CRD NodeGroup/StaticInstance: https://deckhouse.ru/modules/node-manager/cr.html

### Ссылки для frontend-hpa / frontend-pdb / frontend-networkpolicy

- Prometheus Metrics Adapter (источник метрик для HPA): https://deckhouse.ru/modules/prometheus-metrics-adapter/
- CNI Cilium (реализация NetworkPolicy в Deckhouse): https://deckhouse.ru/modules/cni-cilium/
- Admission Policy Engine (валидация/ограничения на этапе create/update): https://deckhouse.ru/modules/admission-policy-engine/
- Нативные Kubernetes-ресурсы, используемые в Deckhouse-кластере:
  - HPA: https://kubernetes.io/docs/concepts/workloads/autoscaling/
  - PodDisruptionBudget: https://kubernetes.io/docs/concepts/workloads/pods/disruptions/
  - NetworkPolicy: https://kubernetes.io/docs/concepts/services-networking/network-policies/

### Что за что отвечает

- `admission-policy-engine`: проверяет и ограничивает манифесты при создании/изменении (admission stage).
- `NetworkPolicy`: управляет сетевым доступом между pod/namespace во время работы приложения.
- `PodDisruptionBudget`: ограничивает добровольные прерывания pod (drain/rollout), чтобы сохранить доступность.
- `HorizontalPodAutoscaler`: масштабирует количество pod по метрикам.

Итого: `admission-policy-engine` не заменяет HPA/PDB/NetworkPolicy, а контролирует правила на входе в API, тогда как HPA/PDB/NetworkPolicy работают на runtime-уровне.

### Используется или нет (в рамках `baremetal/config`)

- Node Manager — **используется** через:
  - `1-create-worker-ng.yml`, `4-create-ng-system.yml`, `5_1-create-ng-front.yml`, `5_4-create-ng-front-lb.yml`
  - `5_7-create-ng-monitoring.yml`
  - `3-create-worker01.yml`, `4-create-no-system.yml`, `5_2-create-no-front-01.yml`, `5_3-create-no-front-02.yml`, `5_5-create-no-front-lb-01.yml`, `5_6-create-no-monitoring-01.yml`
- Ingress NGINX — **используется** через:
  - `6-create-ingress-nginx.yml` (`IngressNginxController`)
  - `7_3-frontend-ingress.yml` (`Ingress` ресурс приложения)
- Prometheus Metrics Adapter — **используется (косвенно)** для HPA через `7_5-frontend-hpa.yml`.
- CNI Cilium — **используется (косвенно)** для NetworkPolicy через `7_6-frontend-networkpolicy.yml`.
- Admission Policy Engine — **не используется** (в `baremetal/config` нет policy-манифестов для admission stage).
- PodDisruptionBudget (Kubernetes) — **используется** через `7_4-frontend-pdb.yml`.
- HorizontalPodAutoscaler (Kubernetes) — **используется** через `7_5-frontend-hpa.yml`.
- NetworkPolicy (Kubernetes) — **используется** через `7_6-frontend-networkpolicy.yml`.
- CSI NFS — **используется** через `8_1-enable-csi-nfs.yml` и `8_2-storageclass-nfs-external.yml`.
- TLS для CSI NFS — **опционально** через `8_4-csi-nfs-tls-optional.yml`.
- Проверка CSI NFS provisioning — **опционально** через `8_5-pvc-nfs-external-example.yml`.
- Совместимость с существующими ссылками на `mon-repl-pvc` — **используется** через `8_3-monitoring-pvc-mon-repl-compat.yml`.

Дополнительно: FAQ и диагностика в [docs/TROUBLESHOOTING.md](../docs/TROUBLESHOOTING.md).

## Что изменено в `config/`

1. `5_4-create-ng-front-lb.yml`
  - `staticInstances.count` установлен в `1`.

2. `5_5-create-no-front-lb-01.yml`
  - создан `StaticInstance` для ingress-ноды с label `role: frontlb`.

3. `6-create-ingress-nginx.yml`
   - ingress-controller закреплен за NodeGroup `frontlb`:
     - `nodeSelector: node-role.deckhouse.io/frontlb`
     - toleration: `dedicated.deckhouse.io=frontlb:NoExecute`

4. `7_1-frontend-deployment.yml`
   - добавлен пример frontend `Deployment` с `nodeSelector` на `front`,
     а также `podAntiAffinity` и `topologySpreadConstraints`.

5. `7_2-frontend-service.yml`
  - добавлен frontend `Service` типа `ClusterIP` с портами `80` и `443`
    (`80 -> targetPort 80`, `443 -> targetPort 80`).

6. `7_3-frontend-ingress.yml`
  - добавлен `Ingress` ресурс для маршрутизации внешнего HTTP трафика
    к сервису `frontend`.
  - `host: frontend.local`, backend: `service/frontend:80`
    (внешний вход через ingress-controller: `80/443`).

7. `7_4-frontend-pdb.yml`
  - добавлен `PodDisruptionBudget` (`minAvailable: 1`) для frontend.

8. `7_5-frontend-hpa.yml`
  - добавлен `HorizontalPodAutoscaler` для `Deployment/frontend`
    (`minReplicas: 2`, `maxReplicas: 6`, target CPU 70%).

9. `7_6-frontend-networkpolicy.yml`
  - добавлен `NetworkPolicy` для ограничения ingress-трафика к frontend.

10. `5_6-create-no-monitoring-01.yml`
  - добавлен `StaticInstance` для monitoring-ноды `tdh-monitoring-01` (`192.168.142.98`).

11. `5_7-create-ng-monitoring.yml`
  - добавлен `NodeGroup` `monitoring` с `role: monitoring` (`count: 1`).

12. `8_1-enable-csi-nfs.yml`
  - включен модуль `csi-nfs` для работы с внешним NFS.

13. `8_2-storageclass-nfs-external.yml`
  - добавлен `StorageClass nfs-external` на внешний NFS-сервер `192.168.142.99`.

14. `8_4-csi-nfs-tls-optional.yml`
  - добавлен опциональный TLS-конфиг для `csi-nfs` (CA secret + `tlsParameters.ca`).

15. `8_5-pvc-nfs-external-example.yml`
  - добавлен опциональный тестовый PVC для проверки работы `StorageClass nfs-external`.

16. `8_3-monitoring-pvc-mon-repl-compat.yml`
  - добавлен PVC `mon-repl-pvc` в namespace `d8-monitoring` на `nfs-external` для совместимости.

## Affinity для frontend (пример для Deployment)

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

## Проверка после применения

```bash
d8 k apply -f baremetal/config/

d8 k get pods -A -o wide | grep -i ingress
d8 k -n NAMESPACE get pods -l app=frontend \
  -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort -u
d8 k -n default get svc frontend
d8 k -n default get ingress frontend
d8 k -n default get pdb frontend-pdb
d8 k -n default get hpa frontend-hpa
d8 k -n default get networkpolicy frontend-ingress-policy
d8 k -n default get pvc nfs-external-test-pvc
d8 k -n d8-monitoring get pvc mon-repl-pvc
```

Ожидаемо:
- ingress-поды на `tdh-front-lb-01`
- frontend-поды на `tdh-front-01` и `tdh-front-02`
- `nfs-external-test-pvc` в статусе `Bound` (если применен `8_4`).
- `mon-repl-pvc` в статусе `Bound`.
