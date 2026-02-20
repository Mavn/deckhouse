# Bare Metal Schema

## Scope

Этот файл описывает bare metal схему:
- `baremetal/config/*` (StaticInstance + NodeGroup + IngressNginxController)
- frontend workload слой (`Deployment/Service/Ingress/PDB/HPA/NetworkPolicy`)

## Platform + workload diagram (Mermaid)

```mermaid
flowchart LR
  subgraph Platform[Deckhouse platform layer]
    NGF[NodeGroup front\ncount=2] --> FN1[tdh-front-01]
    NGF --> FN2[tdh-front-02]

    NGLB[NodeGroup frontlb\ncount=1] --> LB1[tdh-front-lb-01]

    IC[IngressNginxController main\nclass=nginx\nnodeSelector frontlb] --> NGLB
  end

  subgraph Workload[API workload layer]
    DEP[Deployment frontend] --> SVC[Service frontend]
    SVC --> ING[Ingress frontend\nhost=frontend.local]
    DEP --> PDB[frontend-pdb]
    DEP --> HPA[frontend-hpa]
    DEP --> NP[frontend-networkpolicy]
  end

  DEP --> NGF
  ING -.served by.-> IC
```

## Related docs

- [BAREMETAL.md](BAREMETAL.md)
- [ingress.md](ingress.md)
- [project-schema.md](project-schema.md)
