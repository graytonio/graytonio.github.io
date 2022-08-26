---
title: Automating CloudInit Templates in Proxmox
date: 2022-08-26
categories: [Homelab, Ansible, HomelabBible]
tags: [homelab, ansible, automation]
---

# Automating CloudInit Templates in Proxmox

Continuing from my [last post](https://graytonward.com/posts/starting-the-homelab-bible/) on setting up my homelab bible the next thing I want to do is make it easy for me to spin up new vms in my environment from a known good state.  CloudInit is a technology that we can use to accomplish just this and is supported by proxmox.  With the theme of this homelab bible series being automating everything I am going to modify my proxmox setup playbook to make sure I have a cloud init template installed in the cluster and ready to go.

The way I want to accomplish setting up my CloudInit template is through a role.  If you are not familiar with ansible roles you can read more about them [here](https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse_roles.html). By putting this into a role instead of discrete steps in my playbook I can use this role in other playbooks against other types of hypervisors that support cloud-init.

To start my role I need to make a few folders and files to setup.
```
roles/
    cloudinit-template/
        defaults/
            main.yml
        tasks/
            main.yml
```

First I will define all of my default variable values and I will explain what we will use each of them for when they become relevant in the role.

`defaults/main.yml`
```yaml
vm_id: 9000
image_name: ubuntu-22.04-template
default_network: vmbr0
default_memory: 2048
default_cores: 2
default_storage: local-lvm
default_user: admin
```

Now in my tasks the first thing I want to check for is if the template vm I am trying to create already exists

`tasks/main.yml`
```yaml
- name: Check if Template VM Exists
  shell: "qm status {{ vm_id }}"
  ignore_errors: true
  register: template_exists
```

Next I am going to define a block that will create and configure my template with the variables defined earlier.  I am using some clever ansible tricks that I want to explain first before moving on to how I am configuring the template.

`tasks/main.yml`
```yaml
- name: Create VM Template if it Does not Exist
  run_once: true
  block:
    # Create and configure the template
  resuce:
    - name: Remove Malformed VM Template
      shell: "qm destroy {{ vm_id }}"
  when: play_hosts | map('extract', hostvars, 'template_exists') | selectattr('failed', 'defined') | list | count == 0
```

The first thing you might notice is the long when clause attached to the block.  I am using instead of a basic when clause based on `template_exists` because proxmox only allows one vm with a particular id across a cluster so attempting to create a vm with the same id even on a different node will fail.  The when statement makes use of jinja2 filters which you can read in detail [here](https://jinja.palletsprojects.com/en/3.1.x/templates/#builtin-filters) but I will break down the basics of what it is doing. 
    
- `play_hosts` is a special ansible variable that contains an array of every host the play is running against
- `map('extract', hostvars, 'template_exists)` is pulling the fact we set in the previous task for every host in `play_hosts`
- `selectattr('failed', 'defined')` takes each of those `template_exists` facts from the previous step and only passes it through if the task failed
- `list` takes those results and complies them into a list
- `count == 0` asks if the length of the resulting list is 0 which means no host was able to find the template we were looking for

Some more details about the block that are important are the `run_once: true` statement that will ensure that if a template needs to be created it will only be created once this does not allow us to define which host the template is on but for my purposes that is not a big issue and if I need a template to exists on a specific host I can always migrate it there later.

The last important part is the rescue block.  If any task while we are configuring the template fails we want to make sure we don't have a malformed vm template preventing us from running the playbook again so if a task fails in the block we rescue by making sure we delete it before failing the play.

Finally we can look at creating and configuring our CloudInit template starting with downloading our CloudInit image from a repository. For my lab I am using the Ubuntu Server 22.04 LTS image but any CloudInit image will work.

`tasks/main.yml`
```yaml
- name: Download CloudInit Image
    get_url:
    url: https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img
    dest: /root/jammy-server-cloudimg-amd64.img
```

Next we want to create a vm to configure as our template

`tasks/main.yml`
```yaml
- name: Create VM
      shell: "qm create {{ vm_id }} --memory {{ default_memory }} --core {{ default_cores }} --name {{ image_name }} --net0 virtio,bridge={{ default_network }}"
```
This is where most of our default variables come in to create a standard amount of memory and cores we want our template to have.  I typically set these very low and then set them higher if I need them to be.

The next steps mount our cloud init image on the vm as a bootable drive which will allow it to actually be used as our base.

`tasks/main.yml`
```yaml
- name: Attach CloudInit Drive
    shell: "qm set {{ vm_id }} --ide2 {{ default_storage }}:cloudinit"

- name: Make CloudInit Bootable
    shell: "qm set {{ vm_id }} --boot c --bootdisk scsi0"
```

Finally we need to configure some cloud init options and set our display outputs property before finally converting our vm into a template.

`tasks/main.yml`
```yaml
- name: Add Serial Console
    shell: "qm set {{ vm_id }} --serial0 socket --vga serial0"

- name: Configure CloudInit
    shell: "qm set {{ vm_id }} --ciuser \"{{ default_user }}\""

- name: Convert VM to Template
    shell: "qm template {{ vm_id }}"
```

And with those final steps we are done creating our role the last thing to do is add it as a role in our `setup_proxmox.yml` playbook.

`playbooks/setup_proxmox.yml`
```yaml
- name: Configure Proxmox
  hosts: proxmox
  roles:
    - cloudinit-template
```

There are still a few thing we want to do to our proxmox hosts in this playbook but I will leave that for the next post.