# DigitAfrica EDGE-AI BP

---
Basic installation of a K3s instance on a cluster of nodes

## What this deploys (high level)

Current implementation supports the following deployments:
* Tier - 1: K3s implementation, deploying a multi-node K3s cluster

---

## Configuration for the BP

Hosts need to be declared in the ```inventories/prod/hosts.ini``` file.

```
### TIER 1 NODES ###
[tier1_server]
digitafrica-edge-node1 ansible_host=10.64.45.176 ansible_ssh_pass=REDACTED ansible_become_pass=REDACTED

[tier1_agents]
digitafrica-edge-node2 ansible_host=10.64.45.179 ansible_ssh_pass=REDACTED ansible_become_pass=REDACTED
digitafrica-edge-node3 ansible_host=10.64.45.175 ansible_ssh_pass=REDACTED ansible_become_pass=REDACTED

[tier1:children]
tier1_server
tier1_agents

[all:vars]
ansible_user=ubuntu
ansible_become=true
#ansible_ssh_common_args='-o ProxyJump=proxy@bastion1.theblueprintfactory.org'
```

Depending on the type of the deployment (Tier-0/1) only the respective configs need to be present.

## Deploying the BP

To install on the nodes declared at the hosts.ini file, ensure that the deploying machine has ```ansible``` and ```ssh-pass``` installed, and ssh access to all the machines.

```bash
ansible-galaxy collection install -r requirements.yml
```

To install the BP, use the following command:

```bash
ansible-playbook -i inventories/prod/hosts.ini playbooks/site.yml
```

You can uninstall the current version of the BP using the following command:

```bash 
ansible-playbook -i inventories/prod/hosts.ini playbooks/site.yml -e digitafrica_uninstall=true
```

