# Kubernetes Worker Node Tuning

## Overview

This document describes Linux system tuning performed on on-premises Kubernetes worker nodes using Ansible.

The configurations are based on practical troubleshooting and maintenance experience in production Kubernetes environments.

The playbooks address two areas:

- File descriptor limits for Linux users, containerd, and kubelet.
- Linux inotify limits for filesystem event monitoring.

## 1. File Descriptor Limits

**Playbook:** `playbooks/increase_file_descriptors.yml`

File descriptors are used by Linux processes to access files, sockets, pipes, and other resources.

Insufficient file descriptor limits can result in errors such as:

```text
Too many open files
```

The playbook configures:

| Component | NOFILE limit |
|---|---|
| Root user | 524288 |
| Other users | 524288 |
| containerd | 524288 |
| kubelet | 1048576 |

Configuration includes:

- PAM limits configuration.
- `/etc/security/limits.conf`.
- systemd service overrides for containerd and kubelet.
- systemd daemon reload and service restarts.

### Verification

Check the effective limits of running processes:

```bash
systemctl show containerd -p LimitNOFILE -p LimitNOFILESoft
systemctl show kubelet -p LimitNOFILE -p LimitNOFILESoft
```

Alternatively:

```bash
cat /proc/$(pidof containerd)/limits
cat /proc/$(pidof kubelet)/limits
```

## 2. Inotify Limits

**Playbook:** `playbooks/configure_inotify_limits.yml`

Inotify is a Linux kernel subsystem used to monitor filesystem changes.

Applications using many filesystem watchers may encounter resource exhaustion when kernel limits are too low.

The playbook configures:

```ini
fs.inotify.max_user_watches=100000
fs.inotify.max_user_instances=100000
```

The settings are persisted in:

```text
/etc/sysctl.d/99-kubernetes-cri.conf
```

### Verification

```bash
sysctl fs.inotify.max_user_watches
sysctl fs.inotify.max_user_instances
```

## 3. Execution

Both playbooks target the `worker` Ansible inventory group.

### File Descriptor Tuning

```bash
ansible-playbook playbooks/increase_file_descriptors.yml
```

### Inotify Tuning

```bash
ansible-playbook playbooks/configure_inotify_limits.yml
```

Before execution, verify that the inventory contains the intended Kubernetes worker nodes.

## 4. Operational Considerations

- File descriptor limits should be selected according to workload requirements and available system resources.
- PAM session limits and systemd service limits are configured separately.
- Changes to systemd service limits require a service restart to take effect.
- Restarting containerd or kubelet can affect Kubernetes node operations.
- Apply tuning during an appropriate maintenance window and verify node health afterward.
- Inotify limits apply at the Linux kernel level and are distinct from per-process file descriptor limits.

These settings were introduced to address operational requirements in a specific environment and should be reviewed before reuse on other clusters.
