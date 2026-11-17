# Kubernetes upgrade – operational checklist

This checklist documents how to approach the existing playbook. It does not modify its execution logic.

## Before the maintenance window

1. Confirm the starting and target Kubernetes versions and compatibility with your cluster's runtime, CNI and other add-ons.
2. Verify the supported `kubeadm` upgrade sequence. Plan minor-version upgrades in supported steps.
3. Confirm inventory groups `[control-plane]` and `[worker]`, node-name correspondence, SSH connectivity and privilege escalation.
4. Confirm that the control-plane host delegated to by worker tasks is available and `/etc/kubernetes/admin.conf` works there.
5. Review cluster health (`kubectl get nodes`, `kubectl get pods -A`), capacity, workload availability, and PodDisruptionBudgets.
6. Take and verify suitable backups (including etcd where applicable) and document the recovery procedure.
7. Review the original playbook's drain flags, kubelet tuning, node capacity, paths and fixed waits for the target environment.

## Execution

The target Kubernetes version is now prompted in the control-plane play when no `-e` override is supplied.

```bash
ansible-playbook -i inventories/example/hosts.ini \
  playbooks/upgrade_full_cluster_k8s.yml \
  --syntax-check

ansible-playbook -i inventories/example/hosts.ini \
  playbooks/upgrade_full_cluster_k8s.yml
```

For a worker-only upgrade, the control-plane prompt does not run, so provide the version:

```bash
ansible-playbook -i inventories/example/hosts.ini \
  playbooks/upgrade_full_cluster_k8s.yml \
  --limit worker -e 'full_k8s_version=X.Y.Z'
```

For an unattended full-cluster run, also supply `-e 'full_k8s_version=X.Y.Z'`.

**Review before running:** `vars_prompt` needs to be aligned at play level (the pasted example has incorrect indentation). The worker play also references an undefined `kubeadm_upgrade_plan_output` variable in a debug statement. Both are noted here; the operational upgrade tasks have not been rewritten.

Use a suitable maintenance window. Do not execute in a cluster whose compatibility and recovery plan have not been verified.

## After upgrade

1. Verify that all nodes are `Ready` and their versions are correct: `kubectl get nodes -o wide`.
2. Check system workloads and applications: `kubectl get pods -A`.
3. Check node schedulability, DNS, CNI and workload connectivity.
4. Review relevant service logs and any alerts.
5. Record upgrade results and version information.
