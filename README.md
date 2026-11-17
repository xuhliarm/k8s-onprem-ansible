# k8s-onprem-ansible

Production-tested Ansible playbook for upgrading kubeadm-managed Kubernetes clusters on on-premises Rocky Linux servers, designed to minimize service downtime.

The upgrade is performed in two stages: control-plane nodes are upgraded first, followed by worker nodes sequentially (serial: 1). Each node is upgraded individually using cordon, drain, and uncordon operations, allowing workloads to be rescheduled to other available nodes while minimizing disruption to running applications.

## What it does

The original playbook has two Ansible plays that run sequentially:

- **Control plane:** targets the `control-plane` group, one host at a time (`serial: 1`); configures the Kubernetes RPM repository, updates `kubeadm`, executes `kubeadm upgrade apply`, cordons/drains a node, updates `kubelet`/`kubectl`, modifies kubelet systemd parameters, restarts kubelet and uncordons.
- **Workers:** targets the `worker` group, one host at a time (`serial: 1`); configures the RPM repository, updates `kubeadm`, runs `kubeadm upgrade node`, uses delegated `kubectl` commands for cordon/drain/uncordon, updates `kubelet`/`kubectl`, applies kubelet tuning and restarts kubelet.

The existing code includes environment-specific kubelet settings, fixed waits and `kubectl drain` flags intentionally retained from the original implementation.

## Repository layout

```text
k8s-onprem-ansible/
├── README.md
├── ansible.cfg
├── .gitignore
├── inventories/
│   └── example/
│       └── hosts.ini
├── playbooks/
│   ├── upgrade_full_cluster_k8s.yml
│   ├── increase_file_descriptors.yml
│   └── configure_inotify_limits.yml
└── docs/
    ├── upgrade-procedure.md
    └── worker-node-tuning.md
```

## Environment and prerequisites

- Ansible control host with SSH access and privilege escalation (`become`) to target machines.
- On-prem Linux servers using an RPM package manager (`yum`/`dnf`) and `systemd`.
- Kubernetes cluster installed and maintained using `kubeadm`; target Kubernetes packages available at `pkgs.k8s.io`.
- An inventory defining groups **`control-plane`** and **`worker`**. A sample inventory is available at inventories/example/hosts.ini. Update it to match your environment.
- At least the first inventory `control-plane` host must be reachable for worker tasks delegated to `groups['control-plane'][0]` and must have `/etc/kubernetes/admin.conf`.
- Inventory names used in `kubectl cordon`/`drain`/`uncordon` should match Kubernetes Node names. The original control-plane logic references `groups['control-plane'][0]` rather than every current node.
- A verified supported Kubernetes version upgrade path; appropriate etcd/cluster backups and a maintenance window.

**Important:** The playbook is environment-specific. Confirm cluster topology, configuration file paths, resource requirements and maintenance policies before running against a different cluster.

## Usage

The playbook upgrades an on-premises Kubernetes cluster running on Rocky Linux servers. It processes control-plane nodes first, followed by worker nodes sequentially (serial: 1) to minimize service disruption.

Example inventory is provided at `inventories/example/hosts.ini`. Replace its placeholder hosts with real inventory data before use.

### Syntax validation (does not execute an upgrade):

```bash
ansible-playbook -i inventories/example/hosts.ini \
  playbooks/upgrade_full_cluster_k8s.yml --syntax-check
```

### Full Cluster Upgrade.
Run the full-cluster upgrade **interactively** (the control-plane play prompts for `full_k8s_version`):

```bash
ansible-playbook -i inventories/example/hosts.ini \
  playbooks/upgrade_full_cluster_k8s.yml
```

To run non-interactively or override the prompt, pass an explicit target version:

```bash
ansible-playbook -i inventories/example/hosts.ini \
  playbooks/upgrade_full_cluster_k8s.yml \
  -e 'full_k8s_version=X.Y.Z'
```

### Upgrade Specific Node Groups
**Worker-only upgrade:** The version prompt lives in the control-plane play; with `--limit worker` that play is skipped. You **must** supply `full_k8s_version` explicitly:

```bash
ansible-playbook -i inventories/example/hosts.ini \
  playbooks/upgrade_full_cluster_k8s.yml \
  --limit worker -e 'full_k8s_version=X.Y.Z'
```

**Control-plane-only upgrade:** Its interactive prompt is active:

```bash
ansible-playbook -i inventories/example/hosts.ini \
  playbooks/upgrade_full_cluster_k8s.yml --limit control-plane
```

Version `1.36.6` in the prompt is an input-format example, not a recommendation to upgrade to that release. Follow a supported Kubernetes upgrade path.

If you use the originally documented system-wide inventory `/etc/ansible/hosts`, omit the explicit `-i` argument (or point it there). When using `--limit worker`, confirm that delegated control-plane access still works in your environment.

## Important operational notes

- Kubernetes version compatibility: Ensure the target version follows the supported Kubernetes upgrade path and version-skew policy.
- Verify sufficient cluster capacity and workload redundancy to minimize service disruption during sequential node upgrades.
- The playbook uses cordon, drain, and uncordon operations. Worker node draining includes --disable-eviction --force, which bypasses PodDisruptionBudget protections and should be considered before execution.
- The playbook applies environment-specific kubelet parameters, including reserved resources, maximum Pod count, and image-pull behavior. Review these settings before running it in a different environment.
- This playbook was used in a specific production environment. Its compatibility with other cluster configurations has not been independently verified.
- Never commit credentials, private keys, kubeconfigs, sensitive inventory information, or production backups to a public repository.

See [Upgrade procedure](docs/upgrade-procedure.md) for a concise operational checklist.

# Kubernetes Node Tuning
This repository also includes Ansible playbooks for tuning Linux kernel and process limits on on-premises Kubernetes worker nodes.

These playbooks are based on real-world Kubernetes troubleshooting and infrastructure maintenance.

## File Descriptor Limits

playbooks/increase_file_descriptors.yml

Configures file descriptor limits for Linux users and critical Kubernetes components, including containerd and kubelet.

- Updates PAM limits configuration.
- Configures soft and hard nofile limits.
- Applies systemd service overrides.
- Restarts containerd and kubelet to apply the changes.

## Inotify Limits

playbooks/configure_inotify_limits.yml

Configures Linux inotify limits for workloads requiring a high number of filesystem watches.

- Sets fs.inotify.max_user_watches.
- Sets fs.inotify.max_user_instances.
- Applies the updated sysctl configuration.

## Execution

File descriptor tuning:

ansible-playbook playbooks/increase_file_descriptors.yml

Inotify tuning:

ansible-playbook playbooks/configure_inotify_limits.yml

Note: Both playbooks target the kube-empty inventory group. Review the target hosts and service restart impact before execution.
