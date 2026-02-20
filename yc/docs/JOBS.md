# Jobs в проекте: что это и к каким файлам относятся

## Что такое Job в контексте этого проекта

`Job` — одноразовая задача в кластере, которая выполняет bootstrap/инициализацию.
В этом проекте Job используется для автоматической подготовки инфраструктурных объектов Deckhouse.

Важно:
- для операторских действий в документации используется `d8 k ...`;
- внутри контейнера Job может использоваться `kubectl` (runtime-скрипт), потому что `d8` обычно недоступен в образе Job.

## Карта Job -> файл -> назначение

| Job | Файл-источник | Назначение | Что создаёт/изменяет |
|---|---|---|---|
| `csi-nfs-tls-bootstrap` | [yc/config/config.yml](../config/config.yml) | Автонастройка TLS для CSI NFS | Secret `csi-nfs-tls`, патч `ModuleConfig/csi-nfs` (CA) |
| `sds-monitoring-bootstrap` | [yc/config/config.yml](../config/config.yml) | Автосборка DRBD-слоя для monitoring | `LVMVolumeGroup`, `ReplicatedStoragePool`, `ReplicatedStorageClass` |

## Детализация по каждому Job

### 1) `csi-nfs-tls-bootstrap`

**Где объявлен:** [yc/config/config.yml](../config/config.yml)

**Связанные объекты в этом же файле:**
- `ServiceAccount` `csi-nfs-tls-bootstrap`
- `ClusterRole` `csi-nfs-tls-bootstrap`
- `ClusterRoleBinding` `csi-nfs-tls-bootstrap`
- `ModuleConfig` `csi-nfs`
- `NFSStorageClass` `nfs-csi`

**Что делает:**
1. Генерирует CA и серверный сертификат для NFS TLS.
2. Создает/обновляет Secret `csi-nfs-tls` в `kube-system`.
3. Патчит `ModuleConfig/csi-nfs` и записывает CA в `spec.settings.tlsParameters.ca`.

### 2) `sds-monitoring-bootstrap`

**Где объявлен:** [yc/config/config.yml](../config/config.yml)

**Связанные объекты в этом же файле:**
- `ServiceAccount` `sds-monitoring-bootstrap`
- `ClusterRole` `sds-monitoring-bootstrap`
- `ClusterRoleBinding` `sds-monitoring-bootstrap`
- `ModuleConfig` `sds-node-configurator`
- `ModuleConfig` `sds-replicated-volume`
- PVC `mon-repl-pvc`

**Что делает:**
1. Находит monitoring-ноды по лейблу `node.deckhouse.io/group=monitoring`.
2. Выбирает consumable BlockDevice на каждой monitoring-ноде.
3. Создает `LVMVolumeGroup` для каждой ноды.
4. Создает `ReplicatedStoragePool` и `ReplicatedStorageClass` (`mon-repl`).

## Быстрая проверка статуса Job

```bash
d8 k -n kube-system get job csi-nfs-tls-bootstrap
d8 k -n kube-system get job sds-monitoring-bootstrap

d8 k -n kube-system logs job/csi-nfs-tls-bootstrap
d8 k -n kube-system logs job/sds-monitoring-bootstrap
```

## Связанные документы

- [YC документация](YC.md)
- [YC схема](YC-SCHEMA.md)
