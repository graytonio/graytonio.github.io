---
title: Starting My Homelab Bible
date: 2022-08-18
categories: [Homelab, Ansible]
tags: [homelab, ansible, automation]
---

# Starting My Homelab Bible

This will server as the start of my journey to create a set of ansible playbooks and tasks to be able to do all the things in my homelab from ansible.  The other goal is that if my homelab blew up I could use these playbooks to recreate the entire lab with no manual steps.

There are a few assumptions I am making with this set of playbooks if you want to use them for your own homelab.

1. If a server can run proxmox as a hypervisor it will
2. Any vms will run Ubuntu Server 22.04 LTS unless it is an appliance
3. All ssh connections are secured with ssh keys

With that being said I have a few initial goals to get started:

1. Setup proxmox host
2. Create vm template
3. Create vm from template

First things first I need a repo to work with to hold all my ansible things.

```
inventory/
    hosts
playbooks/
roles/
tasks/
ansible.cfg
```

This will be the basic structure I will use to organize everything I need.  I also setup a few configurations to make my life easier.

`ansible.cfg`
```ini
[defaults]
nocows=False
inventory=inventory/
roles_path=~/.ansible/roles:/usr/share/ansible/roles:/etc/ansible/roles:roles/
```

This will setup some defaults so its easier to use our playbooks.  If you want to make this easier to use in your shell you can set the env ANSIBLE_CONFIG to point to this file to make these settings universal.  

We'll create a small inventory file to setup our proxmox hosts and write a basic playbook to make sure our setup works.

`inventory/hosts`
```ini
[proxmox]
192.168.50.10

[proxmox:vars]
ansible_user=root
```

Our first playbook will be a pretty simple one just to make sure all the packages on our proxmox nodes are all up to date.

`playbooks/setup_proxmox.yml`
```yaml
- name: Configure Proxmox
  hosts: proxmox
  tasks:
    - name: Update Package Repositories
      apt:
        update_cache: yes

    - name: Upgrade All Packages
      apt:
        upgrade: yes
```

Unfortunately with a default proxmox install we will get this error

```
TASK [Update Package Repositories] ************************************************************
fatal: [192.168.50.10]: FAILED! => {"changed": false, "msg": "Failed to update apt cache: E:Failed to fetch https://enterprise.proxmox.com/debian/pve/dists/bullseye/InRelease  401  Unauthorized [IP: 144.217.225.162 443], E:The repository 'https://enterprise.proxmox.com/debian/pve bullseye InRelease' is not signed."}
```

because we are trying to update from the enterprise repo without having a valid subscription.  So we need to disable the enterprise repository and enable the no-subscription repo. [More Info](https://pve.proxmox.com/wiki/Package_Repositories)

To do this we'll add a few tasks above our update task to do that

```yaml
- name: Configure Proxmox
  hosts: proxmox
  tasks:
    - name: Ensure Enterprise Repo is Removed
      apt_repository:
        repo: deb https://enterprise.proxmox.com/debian/pve bullseye pve-enterprise
        state: absent

    - name: Ensure No-Subscription Repo is Enabled
      apt_repository:
        repo: deb http://download.proxmox.com/debian/pve bullseye pve-no-subscription
        state: present

    - name: Update Package Repositories
      apt:
        update_cache: yes

    - name: Upgrade All Packages
      apt:
        upgrade: yes
```

Now our playbook can update and upgrade all our packages correctly with the proper repositories. The last thing we will want to check is if the server requires a reboot after the package updates we can do this very easily with two more tasks.

```yaml
- name: Check if reboot is required
    stat:
    path: /var/run/reboot-required
    register: p

- name: Reboot Server if required
    reboot:
    when: p.stat.exists
```

In the future we will add to this playbook to configure things in proxmox just the way we like but this will be all for this post.