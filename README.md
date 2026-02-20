1. Deckhouse EE 
2. Yandex cloud
3. Документация:
 - [DOCUMENTATION.md](DOCUMENTATION.md) - индекс документации
 - [yc/docs/YC.md](yc/docs/YC.md) - развертывание в Yandex Cloud
 - [baremetal/docs/BAREMETAL.md](baremetal/docs/BAREMETAL.md) - развертывание bare metal
 - [baremetal/docs/ingress.md](baremetal/docs/ingress.md) - размещение ingress/frontend

4. Baremetal quick note (актуально):
 - Схема: 1 ingress IP (`tdh-front-lb-01`) + 2 frontend-ноды (`tdh-front-01`, `tdh-front-02`).
 - Stage 5 файлы имеют префиксы порядка применения:
   - `baremetal/config/5_1-create-ng-front.yml`
   - `baremetal/config/5_2-create-no-front-01.yml`
   - `baremetal/config/5_3-create-no-front-02.yml`
   - `baremetal/config/5_4-create-ng-front-lb.yml`
   - `baremetal/config/5_5-create-no-front-lb-01.yml`
 - Подробный runbook: [baremetal/docs/BAREMETAL.md](baremetal/docs/BAREMETAL.md)

---

Основной сценарий в этом README ниже относится к Yandex Cloud.

3. Create full infrastructure:
 Load balancer 2 nodes: 2CPU 4G RAM, HDD 20GB
 Master nodes 2 pcs standart
 System node 1 pc standart
 Worker 3 node 8 CPU 8G RAM HDD 30GB
 Monitoring node 1pc 8CPU 8G RAM
4. NFS server 8CPU 8GB RAM, HDD1 40GB, HDD2 300GB NFS RPC - tls

Lets get create yc/config/config.yml for installation

Samples from docs:
---
# Секция, описывающая параметры инстанс-класса для узлов c компонентами, обеспечивающими рабочую нагрузку.
# https://deckhouse.ru/modules/cloud-provider-yandex/cr.html
apiVersion: deckhouse.io/v1
kind: YandexInstanceClass
metadata:
  name: worker
spec:
  # Возможно, захотите изменить.
  cores: 4
  # Возможно, захотите изменить.
  memory: 8192
  # Возможно, захотите изменить.
  diskSizeGB: 30
---
# Секция, описывающая параметры группы узлов c компонентами, обеспечивающими рабочую нагрузку.
# https://deckhouse.ru/modules/node-manager/cr.html#nodegroup
apiVersion: deckhouse.io/v1
kind: NodeGroup
metadata:
  name: worker
spec:
  cloudInstances:
    classReference:
      kind: YandexInstanceClass
      name: worker
    # Максимальное количество инстансов в каждой зоне (используется при масштабировании).
    # Возможно, захотите изменить.
    maxPerZone: 1
    # Минимальное количество инстансов в каждой зоне. Чтобы запустить больше узлов, увеличьте maxPerZone или добавьте зоны.
    minPerZone: 1
    # Список зон, в которых создаются инстансы.
    # Возможно, захотите изменить.
    zones:
    - ru-central1-a
  disruptions:
    approvalMode: Automatic
  nodeType: CloudEphemeral
---
# Секция, описывающая параметры Ingress NGINX Controller.
# https://deckhouse.ru/modules/ingress-nginx/cr.html
apiVersion: deckhouse.io/v1
kind: IngressNginxController
metadata:
  name: nginx
spec:
  ingressClass: nginx
  inlet: LoadBalancer
  # Описывает, на каких узлах будет находиться Ingress-контроллер. Лейбл node.deckhouse.io/group: <NODE_GROUP_NAME> устанавливается автоматически.
  nodeSelector:
    node.deckhouse.io/group: worker