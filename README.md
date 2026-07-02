# sluetze.roxagent

Ansible role to install and configure the [RHACS roxagent](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_security_for_kubernetes/) for VM vulnerability scanning on RHEL 9 and RHEL 10.

The role deploys Podman Quadlet units (`roxagent.image`, `roxagent.container`) and a systemd timer for periodic scans. It supports:

- Registry authentication via Ansible variables, pull secrets, or AAP Podman Image Registry credentials
- SSH to KubeVirt VMs on the pod network through `virtctl port-forward` (for Ansible Automation Platform job templates)
- Local installation on the target host (no virtctl transport)

## Requirements

- Red Hat Enterprise Linux 9 or 10 with an active subscription
- Podman (installed by the role when missing)
- At least one DNF package transaction on the host (`/var/lib/dnf/history.sqlite`) for repo-to-package mapping; the role runs `dnf update` via handler when no package history exists yet (common on RHEL 10 image-based VMs)
- For KubeVirt/AAP deployments: `virtctl` on the controller/execution environment and an OpenShift or Kubernetes API credential

## Role Variables

See [defaults/main.yml](defaults/main.yml) for the full list. Common variables:

| Variable | Default | Description |
|----------|---------|-------------|
| `roxagent_rhacs_image` | `registry.redhat.io/advanced-cluster-security/rhacs-main-rhel9:4.11.0` | RHACS container image |
| `roxagent_registry_username` / `roxagent_registry_password` | `""` | Registry credentials |
| `roxagent_registry_auth` | `{}` | Full pull-secret auth dict (alternative to username/password) |
| `roxagent_virtctl_ssh` | `true` | Route SSH through virtctl for pod-network VMs, or VMs on another Cluster then AAP |
| `roxagent_run_initial_scan` | `true` | Start an initial scan after configuration |
| `roxagent_dnf_seed_history` | `true` | Run `dnf update` when DNF package transaction history is empty |

## Example Playbook

```yaml
- name: Configure roxagent on KubeVirt VMs
  hosts: rhel_vms
  become: true
  roles:
    - role: sluetze.roxagent

- name: Configure roxagent on localhost
  hosts: localhost
  connection: local
  become: true
  vars:
    roxagent_virtctl_ssh: false
  roles:
    - role: sluetze.roxagent
```

When using virtctl SSH transport, include the connection task file before gathering facts:

```yaml
  pre_tasks:
    - name: Configure KubeVirt SSH connection
      ansible.builtin.include_role:
        name: sluetze.roxagent
        tasks_from: set-kubevirt-connection.yml
      when: roxagent_virtctl_ssh | default(true) | bool
```

## Installation

From Ansible Galaxy:

```bash
ansible-galaxy role install sluetze.roxagent
```

From Git:

```bash
ansible-galaxy role install git+https://github.com/sluetze/ansible-role-roxagent.git
```

Or add to `requirements.yml`:

```yaml
roles:
  - name: sluetze.roxagent
    scm: git
    src: https://github.com/sluetze/ansible-role-roxagent
```

## License

MIT
