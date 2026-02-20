# Plan: Baremetal ingress/frontend alignment

## Scope
- Switch to baremetal target setup.
- Keep 1 IP for ingress-controller.
- Keep frontend as 2 replicas with affinity and related policies.
- Sync docs and runbooks.

## Steps
1. Audit baremetal manifests and docs for current topology and scheduling rules.
2. Set ingress LB topology to single node/IP (`frontlb count: 1`, one `StaticInstance`).
3. Verify frontend deployment remains at 2 replicas with anti-affinity/topology spread.
4. Update docs to match final target state and commands.
5. Add safe apply runbook with platform check gate.
6. Rename stage-5 files with explicit order prefixes (`5_1 ... 5_5`).
7. Replace all old references to renamed files across workspace.
8. Validate docs consistency and no stale links/references.

## Acceptance checks
- Ingress controller scheduled only on `tdh-front-lb-01`.
- Frontend pods scheduled only on `tdh-front-01` and `tdh-front-02`.
- No references to deprecated `5-create-*` filenames.
- Docs contain safe apply flow and updated stage-5 naming.
