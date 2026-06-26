# Manual test (multiple KubeVirt VMs)

Repeat on **each VM**. Requires RHEL 9/10, active subscription, `/dev/vsock`, and registry.redhat.io pull access.

## Per VM

1. **Connect**
   ```bash
   virtctl ssh -n <namespace> <vm-name>
   ```

2. **Preflight**
   ```bash
   ls -l /dev/vsock
   subscription-manager status
   ```

3. **Install Ansible + role**
   ```bash
   sudo dnf install -y ansible-core
   ansible-galaxy role install sluetze.roxagent
   ```

4. **Create test playbook** (`~/test-roxagent.yml`)
   ```yaml
   ---
   - hosts: localhost
     connection: local
     become: true
     gather_facts: true
     vars:
       roxagent_virtctl_ssh: false
     roles:
       - sluetze.roxagent
   ```

5. **Create vars** (`~/roxagent-vars.yml`) — use your service-account token (https://access.redhat.com/terms-based-registry/accounts):
   ```yaml
   roxagent_registry_username: "<org-id>|service-account-name"
   roxagent_registry_password: "<offline-token>"
   ```

6. **Run**
   ```bash
   ansible-playbook ~/test-roxagent.yml -e @~/roxagent-vars.yml
   ```

7. **Verify**
   ```bash
   systemctl status roxagent.timer
   ls -l /etc/containers/systemd/roxagent.{image,container}
   ```

8. **Second VM** — repeat steps 1–7 on the other VM.

## From your workstation (both VMs at once)

Use virtctl SSH transport instead of running Ansible on each guest:

```ini
# inventory.ini
vm1 ansible_host=vm/<vm1>.<namespace> ansible_user=cloud-user
vm2 ansible_host=vm/<vm2>.<namespace> ansible_user=cloud-user

[all:vars]
ansible_ssh_common_args=-o ProxyCommand="virtctl port-forward --stdio=true %h %p"
```

```bash
ansible-galaxy role install sluetze.roxagent
ansible-playbook -i inventory.ini test-roxagent.yml -e @roxagent-vars.yml \
  -e roxagent_virtctl_ssh=true
```

Use a playbook with `hosts: all`, `roxagent_virtctl_ssh: true`, and the role’s `set-kubevirt-connection.yml` pre_task (see main README).
