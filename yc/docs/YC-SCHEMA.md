# YC Schema

## Scope

Этот файл описывает логическую схему YC-направления проекта:
- основной конфиг: `yc/config/config.yml`
- платформенные модули Deckhouse в YC
- storage/bootstrap/jobs в YC

## High-level diagram (Mermaid)

```mermaid
flowchart TB
  A[yc/config/config.yml] --> B[YandexClusterConfiguration]
  A --> C[YandexInstanceClass]
  A --> D[NodeGroup Cloud/Static]
  A --> E[IngressNginxController]
  A --> F[ModuleConfig cloud-provider-yandex]
  A --> G[ModuleConfig csi-nfs]
  A --> H[ModuleConfig sds-node-configurator]
  A --> I[ModuleConfig sds-replicated-volume]

  G --> J[Job csi-nfs-tls-bootstrap]
  J --> K[Secret csi-nfs-tls]
  J --> G

  I --> L[Job sds-monitoring-bootstrap]
  L --> M[LVMVolumeGroup]
  L --> N[ReplicatedStoragePool]
  L --> O[ReplicatedStorageClass]
  O --> P[PVC mon-repl-pvc]
```

## Related docs

- [YC.md](YC.md)
- [Jobs mapping](JOBS.md)
