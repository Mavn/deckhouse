# Deckhouse EE на Yandex Cloud - Документация

## Структура проекта

- [Локальный индекс](INDEX.md)
- [README.md](../../README.md) - Обзор инфраструктуры
- [yc/config/config.yml](../config/config.yml) - Конфигурация кластера YC
- [yc/docs/YC-SCHEMA.md](YC-SCHEMA.md) - Схема YC направления
- [yc/docs/JOBS.md](JOBS.md) - Какие Job есть в YC-проекте и что они делают

## Ссылки на Deckhouse по теме YC

- Общая документация: https://deckhouse.ru/
- Каталог модулей: https://deckhouse.ru/modules/
- Cloud Provider Yandex: https://deckhouse.ru/modules/cloud-provider-yandex/
- CRD для Yandex Cloud: https://deckhouse.ru/modules/cloud-provider-yandex/cr.html
- Node Manager (NodeGroup): https://deckhouse.ru/modules/node-manager/cr.html#nodegroup
- Ingress NGINX: https://deckhouse.ru/modules/ingress-nginx/

## Архитектура кластера

Кластер состоит из **14 узлов** в **7 группах** с разными ролями и сервисами:

### Группы узлов и их назначение

| Группа | Количество | CPU | RAM | Диск | Зоны | Роль | Сервисы |
|--------|-----------|-----|-----|------|------|------|---------|
| **load-balancer** | 2 | 2 | 4GB | 20GB | a, b | Ingress | LoadBalancer, Nginx |
| **frontend** | 2 | 4 | 8GB | 30GB | a, b | Frontend | Frontend pods |
| **master** | 2 | 4 | 8GB | 50GB | a, b | Control Plane | API Server, Scheduler, etcd |
| **system** | 1 | 4 | 8GB | 50GB | a | System | Deckhouse, Monitoring Agent |
| **worker** | 3 | 8 | 8GB | 30GB | a, b, c | Приложения | Prometheus, Pod Apps |
| **monitoring** | 3 | 8 | 8GB | 30GB + 100GB | a, b, c | Мониторинг | Prometheus, Grafana, Loki, AlertManager |
| **nfs-server** | 1 | 8 | 8GB | 40GB + 300GB | a | Хранилище | NFS Server (RPC TLS), XFS |

### Детальное описание каждой группы узлов

#### 1. Load Balancer узлы (2 шт)
- **Распределение**: 2 зоны (ru-central1-a, ru-central1-b) для HA
- **Компоненты**:
  - Ingress NGINX Controller
  - Load Balancer (внешняя маршрутизация)
  - Service LoadBalancer endpoints
- **Ограничения**: Только для входящего трафика
- **Автомасштабирование**: Отключено (фиксированное количество)

#### 2. Frontend узлы (2 шт)
- **Распределение**: 2 зоны (ru-central1-a, ru-central1-b) для HA
- **Компоненты**:
  - Frontend workloads (Deployment/Pod)
- **Ограничения**: Размещение только frontend нагрузки
- **Label**: `node.deckhouse.io/group: frontend`

#### 3. Master узлы (2 шт) - Control Plane
- **Распределение**: 2 зоны (ru-central1-a, ru-central1-b) для HA
- **Компоненты**:
  - Kubernetes API Server
  - Scheduler
  - Controller Manager
  - etcd (базу данных кластера)
- **Ограничения**: Только system компоненты
- **Тaint**: `node-role.kubernetes.io/control-plane`
- **Резервное копирование**: Enabled (S3 External)

#### 4. System узел (1 шт)
- **Распределение**: 1 зона (ru-central1-a)
- **Компоненты**:
  - Deckhouse core
  - System daemons
  - kube-proxy
  - kubelet metrics
- **Ограничения**: Только system компоненты Deckhouse
- **Label**: `node.deckhouse.io/group: system`

#### 5. Worker узлы (3 шт) - Приложения
- **Распределение**: 3 зоны (ru-central1-a, ru-central1-b, ru-central1-c)
- **Компоненты**:
  - User Application Pods
  - Prometheus scraper
  - Logging agents
- **Автомасштабирование**: maxPerZone=1, можно увеличить
- **Резервное копирование**: Enabled (S3 External)

#### 6. Monitoring узлы (3 шт)
- **Распределение**: 3 зоны (ru-central1-a, ru-central1-b, ru-central1-c)
- **Компоненты**:
  - Prometheus (сбор метрик)
  - Grafana (визуализация)
  - Loki (логирование)
  - AlertManager (управление алертами)
- **Label**: `node.deckhouse.io/group: monitoring`
- **Диски**:
  - Основной: 30GB (OS)
  - Данные: 100GB (DRBD replicated volume для метрик и логов)
- **DRBD (sds-replicated-volume)**:
  - LVMVolumeGroup: `mon-data-on-monitoring-1/2/3`
  - VG на узлах: `mon-data`
  - ReplicatedStoragePool: `mon-repl`
  - ReplicatedStorageClass: `mon-repl`
  - PVC: `mon-repl-pvc` (namespace `d8-monitoring`)

#### 7. NFS Server узел (1 шт) - Хранилище
- **Распределение**: 1 зона (ru-central1-a)
- **Диски**:
  - Основной: 40GB (OS + NFS service)
  - Дополнительный: 300GB (XFS для экспорта)
- **Компоненты**:
  - NFS Server (RPC with TLS/NFSv4.2)
  - XFS файловая система
  - Automatic disk detection and mount
- **Export**: `/mnt/nfs-storage/exports` для Pod Subnet (10.111.0.0/16)
- **Резервное копирование**: Enabled (S3 External)
- **Service**: Доступен через `nfs-server-service.kube-system.svc.cluster.local`

### Матрица сетевой доступности

```
┌─────────────────┬────────────┬────────────┬──────────┬───────────┬─────────────┐
│ Source / Dest   │ LB         │ Master     │ System   │ Worker    │ Monitoring  │
├─────────────────┼────────────┼────────────┼──────────┼───────────┼─────────────┤
│ Load Balancer   │ ClusterIP  │ No         │ No       │ No        │ No          │
│ Master          │ Yes        │ etcd:2379  │ Yes      │ Yes       │ Yes         │
│ System          │ Yes        │ Yes        │ Internal │ Yes       │ Yes         │
│ Worker          │ Yes        │ API:6443   │ Yes      │ Pod Net   │ Scrape:9100 │
│ Monitoring      │ Yes        │ Rules:6443 │ Remote   │ Scrape    │ Web:3000    │
│ NFS Server      │ Yes        │ Yes        │ Yes      │ NFS:2049  │ Yes         │
└─────────────────┴────────────┴────────────┴──────────┴───────────┴─────────────┘
```

### Суммарные ресурсы кластера

| Ресурс | Значение |
|--------|----------|
| **Всего узлов** | 14 |
| **CPU cores** | 80 (2*2+2*4+2*4+4+3*8+3*8+8) |
| **RAM** | 104 GB (2*4+2*8+2*8+8+3*8+3*8+8) |
| **Диск** | 1070 GB (2*20+2*30+2*50+50+3*30+3*(30+100)+40+300) |
| **Зоны доступности** | 3 (a, b, c) |
| **High Availability** | Да (Master 2, LB 2) |

---

## Описание инфраструктуры

Этот проект описывает полную инфраструктуру Kubernetes кластера на основе Deckhouse EE с развертыванием в Yandex Cloud.

### Компоненты инфраструктуры

#### 1. Load Balancer узлы (2 шт)
- **CPU**: 2 cores
- **RAM**: 4 GB
- **Хранилище**: 20 GB HDD
- **Роль**: Ingress контроллер и балансировка нагрузки

#### 2. Frontend узлы (2 шт)
- **CPU**: 4 cores
- **RAM**: 8 GB
- **Хранилище**: 30 GB HDD
- **Роль**: Выделенные узлы для frontend workload

#### 3. Master узлы (2 шт)
- **CPU**: 4 cores
- **RAM**: 8 GB
- **Хранилище**: 50 GB HDD
- **Роль**: Control plane с высокой доступностью (HA)

#### 4. System узел (1 шт)
- **CPU**: 4 cores
- **RAM**: 8 GB
- **Хранилище**: 50 GB HDD
- **Роль**: Системные компоненты Deckhouse

#### 5. Worker узлы (3 шт)
- **CPU**: 8 cores
- **RAM**: 8 GB
- **Хранилище**: 30 GB HDD
- **Роль**: Рабочие узлы для приложений

#### 6. Monitoring узлы (3 шт)
- **CPU**: 8 cores
- **RAM**: 8 GB
- **Хранилище**: 30 GB HDD (OS) + 100 GB HDD (DRBD replicated volume)
- **Роль**: Мониторинг и логирование (Prometheus, Grafana, Loki)

#### 7. NFS сервер (1 шт)
- **CPU**: 8 cores
- **RAM**: 8 GB
- **Хранилище**: 40 GB (основной диск) + 300 GB (дополнительный диск) HDD
- **Роль**: Общее сетевое хранилище (Shared Storage)
- **Конфигурация**: Дополнительный диск 300GB автоматически монтируется для NFS

## Общие характеристики кластера

| Показатель | Значение |
|-----------|---------|
| **Всего узлов** | 14 |
| **Всего CPU** | 80 cores |
| **Всего RAM** | 104 GB |
| **Всего хранилища** | 1070 GB |
| **Provider** | Yandex Cloud |
| **High Availability** | Enabled |

## Сетевая конфигурация

- **Pod Subnet**: 10.111.0.0/16
- **Service Subnet**: 10.222.0.0/16
- **DNS Provider**: CoreDNS
- **Network Policy**: Enabled

## Установленные компоненты

### Мониторинг и логирование
- ✅ **Prometheus** - сбор метрик
- ✅ **Grafana** - визуализация
- ✅ **Loki** - централизованное логирование
- ✅ **AlertManager** - управление алертами

### Ingress
- ✅ **Ingress Controller (1 шт)** - маршрутизация внешнего трафика
  - `nginx` (ingressClass: `nginx`)
  - размещение: NodeGroup `load-balancer` (2 ноды, HA)
  - внешний доступ: 1 внешний IP через `inlet: LoadBalancer`

### Frontend
- ✅ **Выделенный frontend NodeGroup (2 шт)**
  - NodeGroup: `frontend` (зоны `ru-central1-a`, `ru-central1-b`)
  - для размещения frontend workload используйте `nodeSelector: node.deckhouse.io/group: frontend`

### Хранилище
- ✅ **NFS Server** с RPC TLS поддержкой (XFS файловая система)
- ✅ **CSI-NFS Driver** - провайдер для ReadWriteMany томов
- ✅ **Yandex Cloud Object Storage (S3)** - облачное S3-совместимое хранилище
- ✅ **Резервное копирование (Backup)** - автоматическое создание снимков в S3

### Отключено
- ❌ **Elasticsearch** - используется Loki вместо него

## Конфигурационные файлы

### yc/config/config.yml
Файл конфигурации содержит Kubernetes CRD объекты для Deckhouse:

#### Основные типы объектов:

1. **YandexInstanceClass** - определяет спецификацию инстансов (CPU, RAM, диск)
2. **NodeGroup** - описывает группу узлов и их параметры масштабирования
3. **IngressNginxController** - настройка входного контроллера (1 контроллер на 2 ingress-ноды)

#### Пример структуры для узла:
```yaml
apiVersion: deckhouse.io/v1
kind: YandexInstanceClass
metadata:
  name: worker
spec:
  cores: 8        # Количество CPU cores
  memory: 8192    # Объем памяти в MB (8192 = 8GB)
  diskSizeGB: 30  # Размер основного диска в GB
```

#### Пример с дополнительными дисками (NFS Server):
```yaml
apiVersion: deckhouse.io/v1
kind: YandexInstanceClass
metadata:
  name: nfs-server
spec:
  cores: 8
  memory: 8192
  diskSizeGB: 40          # Основной диск
  secondaryDisks:
  - sizeGb: 300           # Дополнительный диск
    type: network-hdd     # Тип хранилища
```

Подробнее см. [yc/config/config.yml](../config/config.yml)

## Шаги развертывания

1. Подготовка Yandex Cloud аккаунта
2. Создание инфраструктуры согласно [yc/config/config.yml](../config/config.yml)
3. Установка Deckhouse EE
4. Конфигурация компонентов
5. Проверка статуса кластера

Проверка Ingress контроллеров:

```bash
d8 k get ingressnginxcontroller
d8 k get ingressclass
d8 k -n d8-ingress-nginx get svc -o wide
```

Готовые манифесты frontend:

- [yc/config/frontend-deployment.yml](../config/frontend-deployment.yml)
- [yc/config/frontend-service.yml](../config/frontend-service.yml)
- [yc/config/frontend-ingress.yml](../config/frontend-ingress.yml)

Домен frontend в Ingress: `mavn.biz`.

Применение и проверка:

```bash
d8 k apply -f yc/config/frontend-deployment.yml
d8 k apply -f yc/config/frontend-service.yml
d8 k apply -f yc/config/frontend-ingress.yml
d8 k get deploy frontend
d8 k get pods -l app=frontend -o wide
d8 k get svc frontend
d8 k get ingress frontend
```

Проверка DNS и HTTP для `mavn.biz`:

```bash
# Проверка, что домен резолвится во внешний IP ingress
dig +short mavn.biz
nslookup mavn.biz

# Проверка HTTP маршрута через Host header
INGRESS_IP=$(d8 k -n d8-ingress-nginx get svc -o jsonpath='{.items[0].status.loadBalancer.ingress[0].ip}')
curl -I -H "Host: mavn.biz" "http://${INGRESS_IP}/"
```

Debugging frontend/ingress:

```bash
# 1) Ingress и события
d8 k describe ingress frontend
d8 k get events --sort-by=.metadata.creationTimestamp | tail -n 50

# 2) Service и endpoints
d8 k get svc frontend -o wide
d8 k get endpoints frontend -o wide

# 3) Pod'ы frontend
d8 k get pods -l app=frontend -o wide
d8 k describe pods -l app=frontend

# 4) Логи ingress-nginx контроллера
d8 k -n d8-ingress-nginx get pods -o wide
d8 k -n d8-ingress-nginx logs -l app=ingress-nginx --tail=200
```

Если `curl` возвращает `404`, обычно не совпадает `Host` (проверьте `mavn.biz` в Ingress). Если `502/503`, проверьте `endpoints frontend` и готовность pod'ов.

Пример структуры манифеста:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      nodeSelector:
        node.deckhouse.io/group: frontend
      containers:
      - name: frontend
        image: nginx:1.27
        ports:
        - containerPort: 80
```

## Дополнительные ресурсы

- [Deckhouse Official Documentation](https://deckhouse.io/en/)
- [Yandex Cloud Documentation](https://cloud.yandex.ru/docs)

## Настройка NFS сервера и CSI-NFS

### NFS Server RPC-with-TLS (RFC 9289)
В конфиге определен ConfigMap с bash-скриптом для автоматической настройки NFS сервера:
- Установка nfs-kernel-server, nfs-common, rpcbind, openssl, ktls-utils (tlshd)
- **Автоматический поиск дополнительного диска** (проверяются `/dev/vdb`, `/dev/sdb`, `/dev/xvdb`, `/dev/nvme1n1`)
- Форматирование в **XFS** для лучшей производительности
- Монтирование дополнительного диска (300GB) на `/mnt/nfs-storage`
- Экспортирование директории для Kubernetes подов
- **✅ RPC-with-TLS включен по умолчанию** (RFC 9289, NFSv4.2+)
- TLS сертификаты генерируются локально (self-signed) в `/etc/rpc.tls`. При необходимости замените их своими.
- mTLS отключен. Если нужен mTLS, потребуется добавить client cert/key в `csi-nfs` ModuleConfig.

### CSI-NFS Driver с RPC-with-TLS
TLS параметры настраиваются автоматически через Job `csi-nfs-tls-bootstrap`.
Job генерирует CA и серверный сертификат, создает Secret `csi-nfs-tls`,
а затем патчит `ModuleConfig` для `csi-nfs` с base64 CA.

Включение модуля и TLS параметров (base64, заполняется Job-ом):
```yaml
apiVersion: deckhouse.io/v1alpha1
kind: ModuleConfig
metadata:
  name: csi-nfs
spec:
  enabled: true
  version: 1
  settings:
    tlsParameters:
      ca: CHANGE_BASE64_CA_CERT
```

StorageClass создается как `NFSStorageClass`:
```yaml
apiVersion: storage.deckhouse.io/v1alpha1
kind: NFSStorageClass
metadata:
  name: nfs-csi
spec:
  connection:
    host: nfs-server-service.kube-system.svc.cluster.local
    share: /mnt/nfs-storage/exports
    nfsVersion: "4.2"
    tls: true
    mtls: false
  reclaimPolicy: Delete
  volumeBindingMode: WaitForFirstConsumer
```

**DNS имя Service используется автоматически - не требуется ручная настройка IP.**

### Пример использования
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-data
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs-csi
  resources:
    requests:
      storage: 50Gi
```

## DRBD для monitoring (sds-replicated-volume)

Для DRBD используется модуль `sds-replicated-volume` и базовый `sds-node-configurator`.
Они уже включены в [yc/config/config.yml](../config/config.yml) через `ModuleConfig`.

### Автоматическое создание DRBD

В [yc/config/config.yml](../config/config.yml) добавлен Job `sds-monitoring-bootstrap`, который:

- Находит все monitoring-ноды по label `node.deckhouse.io/group=monitoring`.
- Выбирает **первый consumable BlockDevice** на каждой ноде (обычно это 100GB диск).
- Создает `LVMVolumeGroup` с VG `mon-data`.
- Создает `ReplicatedStoragePool` и `ReplicatedStorageClass` `mon-repl`.

### Проверка авто-настройки

```bash
# Проверить Job
d8 k get job -n kube-system sds-monitoring-bootstrap

# Проверить LVMVolumeGroup
d8 k get lvg

# Проверить ReplicatedStoragePool и StorageClass
d8 k get rsp
d8 k get rsc
```

### Что будет создано

- `LVMVolumeGroup` на каждой monitoring-ноде (VG `mon-data`).
- `ReplicatedStoragePool` `mon-repl`.
- `ReplicatedStorageClass` `mon-repl`.
- PVC `mon-repl-pvc` (namespace `d8-monitoring`).

### Важные требования

- Для DRBD требуется минимум **3 узла** в пуле.
- Репликация синхронная (DRBD/LINSTOR).
- Для monitoring данных используйте PVC `mon-repl-pvc`.

## Yandex Cloud Object Storage (S3)

### Простая конфигурация S3
В конфиге уже предусмотрена StorageClass для S3:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: yandex-s3
provisioner: s3.csi.aws.com
parameters:
  endpoint: "https://storage.yandexcloud.net"
  bucket: "deckhouse-storage"              # Замените на имя вашего bucket
  region: "ru-central1"
```

### Автоматическое создание S3 ключей

Создание bucket и S3 ключей выполняется **OpenTofu**.
Он же создает Kubernetes Secret `csi-s3-secret` **до запуска** `dhctl bootstrap`.
`yc/config/config.yml` уже ссылается на этот Secret для backup и S3 PVC, поэтому **никаких ручных действий не требуется**.

### Использование S3 хранилища
После `dhctl bootstrap` можно сразу создавать PVC:
```bash
d8 k apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-s3-backup
spec:
  storageClassName: yandex-s3
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
EOF
```

### OpenTofu (расширенный пример)
Ниже пример, который полностью автоматизирует S3: создает bucket, сервисный ключ и Secret в Kubernetes **до** запуска `dhctl bootstrap`.

```hcl
variable "yc_cloud_id" { type = string }
variable "yc_folder_id" { type = string }
variable "yc_sa_id" { type = string }
variable "kubeconfig" { type = string }
variable "s3_bucket" { type = string }

provider "yandex" {
  cloud_id  = var.yc_cloud_id
  folder_id = var.yc_folder_id
}

resource "yandex_iam_service_account_key" "s3" {
  service_account_id = var.yc_sa_id
}

resource "yandex_storage_bucket" "s3" {
  bucket = var.s3_bucket
}

provider "kubernetes" {
  config_path = var.kubeconfig
}

resource "kubernetes_secret" "csi_s3" {
  metadata {
    name      = "csi-s3-secret"
    namespace = "kube-system"
  }
  type = "Opaque"
  data = {
    accessKeyID     = yandex_iam_service_account_key.s3.access_key
    secretAccessKey = yandex_iam_service_account_key.s3.secret_key
  }
}
```

Минимальные outputs (по желанию):
```hcl
output "s3_bucket" {
  value = yandex_storage_bucket.s3.bucket
}
```

## Резервное копирование (Backup)

### Конфигурация Backup
Worker и NFS Server узлы автоматически настроены на резервное копирование в S3:

```yaml
backup:
  enabled: true
  s3:
    bucketName: "deckhouse-backup"      # Bucket для резервных копий
    mode: External
    external:
      provider: YCloud                   # Yandex Cloud S3
      accessKey: "csi-s3-secret"        # Secret с accessKeyID
      secretKey: "csi-s3-secret"        # Secret с secretAccessKey
```

### С какого места делаются снимки

- **Worker узлы** - резервные копии данных приложений
- **NFS Server узлы** - резервные копии NFS хранилища (включая 300GB диск)

### Как это работает

1. Yandex Cloud автоматически создает снимки (snapshots) дисков
2. Снимки сохраняются в S3 bucket `deckhouse-backup`
3. Можно восстановить из снимков в любой момент

### Управление backup

```bash
# Просмотреть snapshots в Yandex Cloud
yc compute disk-placement-group list

# Список вещей в S3 bucket
aws s3 ls s3://deckhouse-backup --endpoint-url https://storage.yandexcloud.net
```

## Deckhouse Модули

### Конфигурация Yandex Cloud провайдера
`YandexClusterConfiguration` - основной конфиг для подключения к YC:
```yaml
apiVersion: deckhouse.io/v1
kind: YandexClusterConfiguration
layout: Standard
provider:
  cloudID: "YOUR_CLOUD_ID"
  folderID: "YOUR_FOLDER_ID"
  serviceAccountJSON: |
    {
      "id": "...",
      "private_key": "-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----\n"
    }
```

Ключ получите командой:
```bash
yc iam key create --service-account-id=<SA_ID> --format=json
```

### Включение TLS для NFS RPC (уже включено)
TLS включен по умолчанию в конфиге. Параметры сертификатов будут поддерживаться Kubernetes/Deckhouse автоматически.

## Примечания

- High Availability включена для master узлов
- Autoscaling настроен для worker узлов (максимум 10 узлов)
- NFS Storage использует **XFS файловую систему** для оптимальной производительности
- **NFS over TLS включен по умолчанию** (NFSv4.2+)
- CSI-NFS использует DNS имя Service - **не требуется ручная настройка IP адреса!**
- DRBD для monitoring реализован через `sds-replicated-volume` (3 monitoring-узла)
- **Резервное копирование (Backup)** автоматически настроено для Worker и NFS Server узлов в S3
- **S3 настраивается автоматически OpenTofu**, без ручного ввода ключей
- Все узлы используют HDD хранилище (рекомендуется использовать SSD для production)

## Контакты и поддержка

Для вопросов по конфигурации обратитесь к документации Deckhouse или Yandex Cloud.

## Быстрый старт

### 1. Подготовка параметров Yandex Cloud

```bash
# Получить Cloud ID и Folder ID
yc config list

# Создать (или использовать существующий) Service Account
yc iam service-account create --name deckhouse

# Выдать права на работу с облаком
yc resource-manager folder add-access-binding <FOLDER_ID> \
  --role admin \
  --subject serviceAccount:<SA_ID>

# Создать JSON ключ для авторизации
yc iam key create --service-account-id=<SA_ID> --format=json > key.json
```

### 2. Обновить yc/config/config.yml

В `YandexClusterConfiguration` замените:
```yaml
provider:
  cloudID: "YOUR_CLOUD_ID"              # из yc config list
  folderID: "YOUR_FOLDER_ID"            # из yc config list
  serviceAccountJSON: |
    {
      "id": "...",
      "service_account_id": "...",
      ...скопировать содержимое key.json...
    }
```

### 3. Для S3 хранилища (через OpenTofu)

OpenTofu создает bucket, ключи и Secret `csi-s3-secret` заранее.
`yc/config/config.yml` уже ссылается на этот Secret (backup для Worker/NFS настроен на `csi-s3-secret`).

### 4. Применить конфигурацию

```bash
d8 k apply -f yc/config/config.yml

# Проверить статус узлов
d8 k get nodes -o wide

# Проверить StorageClasses
d8 k get storageclasses
```

### Примеры использования

**NFS (ReadWriteMany):**
```bash
d8 k apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-nfs-data
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs-csi
  resources:
    requests:
      storage: 50Gi
EOF
```

**S3 (облачное хранилище):**
```bash
d8 k apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-s3-backup
spec:
  storageClassName: yandex-s3
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
EOF
```

## Удаление (YC teardown)

### 1) Удалить объекты из yc/config/config.yml

```bash
d8 k delete -f yc/config/config.yml
```

### 2) Проверить, что прикладные объекты удалены

```bash
d8 k get ingress -A
d8 k get svc -A
d8 k get pvc -A
d8 k get job -A
```

### 3) Удалить облачную инфраструктуру (если создавали через IaC)

- Если создавали через OpenTofu/Terraform: выполнить `destroy` в вашем IaC-проекте.
- Если создавали вручную в YC: удалить VM/диски/LB/сети/S3 bucket и ключи доступа.

Примечание: `d8 k delete -f yc/config/config.yml` удаляет Kubernetes/Deckhouse CRD объекты,
но не всегда автоматически удаляет все внешние ресурсы в Yandex Cloud.
