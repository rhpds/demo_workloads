# CCLM playbook

Post-install chain for the Troshka `ocp-cclm` catalog item (nested source SNO+2 /
destination SNO on KubeVirt).

## Roles (order matters)

1. `troshka_workload_cclm_operators` — OLM subscriptions
2. `troshka_workload_cclm_hco` — decentralized live migration
3. `troshka_workload_cclm_network` — migration NAD, NNCPs, MetalLB, external ODF
4. `troshka_workload_cclm_forklift` — Submariner + MTV maps
5. `troshka_workload_cclm_seed_vms` — namespaces + optional demo VM

## Troshka ad-hoc run (single role)

Pin the feature branch so the runner pod clones this work:

```json
{
  "kind": "ad_hoc",
  "role_fqcn": "rhpds.demo_workloads.troshka_workload_cclm_operators",
  "requirements_content": {
    "collections": [{
      "name": "https://github.com/rhpds/demo_workloads.git",
      "type": "git",
      "version": "feat/cclm-workloads"
    }]
  },
  "target_map": { "mode": "cluster", "cluster_id": "source" }
}
```

Run each role in order. For multi-cluster roles, pass a `clusters` extra-var with
both `source` and `destination` API URLs/tokens (or run once per cluster where
applicable).

## Prerequisites

- Both clusters at control-plane-usable (Troshka ops-pod install complete)
- External Ceph secret `rook-ceph-external-cluster-details` in `openshift-storage`
  (supplied via catalog/workload `extra_vars`, not created by Troshka)
