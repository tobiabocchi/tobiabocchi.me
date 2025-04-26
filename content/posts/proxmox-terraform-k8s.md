---
date: "2025-04-24T18:08:54+02:00"
draft: false
title: "Deploy Kubernetes Clusters Like a Wizard"
description: "Summon your own Kubernetes cluster from thin air using Proxmox, Terraform, Ansible, and the SIGHUP Distribution. A practical guide for DevOps sorcerers ready to automate their homelab (or production envinronment) like magic."
summary: "Summon your own Kubernetes cluster from thin air using Proxmox, Terraform, Ansible, and the SIGHUP Distribution. A practical guide for DevOps sorcerers ready to automate their homelab (or production envinronment) like magic."
cover:
  image: "/img/posts/proxmox-terraform-k8s/devops_magic.jpg"
---

## Introduction

As a DevOps engineer, I've found that the best way to learn something is to get hands-on
with it. So I thought: what better way to dive into Terraform and the SIGHUP Kubernetes
distribution than by creating a Proxmox cluster and deploying a Kubernetes cluster
on it, fully automated?

This article walks you through my experience doing just that, step by step. If you're
ready to conjure up some DevOps magic, let's get started!

> ❤️ Before jumping into it, I'd just like to give a shoutout to a fellow sorcerer
> and colleague of mine, [paranoiasystem](https://paranoiasystem.com/). He did most
> of the heavy lifting in creating the Terraform project and was a huge help throughout
> this learning journey.
>
> **Thank you! 🙌**

By the way, the file for this project can be found on my [GitHub](https://github.com/tobiabocchi/proxmox-sighup-installer)

## Step One - Proxmox

To provision our VMs, we'll use the [Telmate/proxmox](https://registry.terraform.io/providers/Telmate/proxmox/latest/docs)
Terraform provider.

The idea is simple: we'll create a VM template using **cloud-init**, which we'll
use as the base image to clone the VMs that make up our Kubernetes cluster.

> 💡 **cloud-init** is a tool for automating the initial setup of virtual machines
> in cloud environments. Want to dig deeper? Check out the [official documentation](https://cloudinit.readthedocs.io/en/latest/explanation/introduction.html#introduction).

### Template VM

Let's start by creating a Proxmox template that we'll use as the base image to clone
our VMs. For this, we'll use `qm`, the QEMU/KVM virtual machine manager included
with Proxmox.

```sh
# Download the Ubuntu 24.04 cloud-init image
wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img
# Create a VM
qm create 9000 --name ubuntu-2404-cloudinit-template
# Attach a volume using the downloaded image
qm set 9000 \
  --virtio0 fast-storage:0,import-from=/root/cloud-init/noble-server-cloudimg-amd64.img \
  --scsihw virtio-scsi-pci
# Convert the VM into a template
qm template 9000
```

### Provider Setup

According to the provider documentation, the first thing we need to do is create
a Proxmox user and role specifically for Terraform. We'll also generate an authentication
token. To do this, we'll use the Proxmox VE User Manager (`pveum`).

```sh
# Create a custom role with the necessary privileges for Terraform
pveum role add TerraformProv -privs "Datastore.AllocateSpace Datastore.AllocateTemplate Datastore.Audit Pool.Allocate Sys.Audit Sys.Console Sys.Modify VM.Allocate VM.Audit VM.Clone VM.Config.CDROM VM.Config.Cloudinit VM.Config.CPU VM.Config.Disk VM.Config.HWType VM.Config.Memory VM.Config.Network VM.Config.Options VM.Migrate VM.Monitor VM.PowerMgmt SDN.Use"
# Create the Terraform user
pveum user add terraform-prov@pve --password automateAllTheThings
# Assign the role to the user
pveum aclmod / -user terraform-prov@pve -role TerraformProv
# Generate an API token for authentication
pveum user token add terraform-prov@pve mytoken
# 🔐 Save the output of this command! You'll need the token later to authenticate with Proxmox.
```

## Step Two – Terraform

Now let's hop over to our laptop and handcraft some Terraform. We'll start by generating
a simple, dummy VM to make sure everything is working. Once that's confirmed, we'll
move on to spinning up a full fleet of VMs to run our Kubernetes cluster.

### Cloud-Init

Before creating our VMs with Terraform, let's leverage one of cloud-init's powerful
features: **snippets**. Snippets allow us to inject custom configuration into our
virtual machines during provisioning. First, we'll create a location on the Proxmox
server to store these snippets, then add our first one:

```sh
mkdir /var/lib/vz/snippets
tee /var/lib/vz/snippets/qemu-guest-agent.yml <<EOF
#cloud-config
runcmd:
  - apt update
  - apt install -y qemu-guest-agent
  - systemctl enable --now qemu-guest-agent
EOF
```

The `qemu-guest-agent.yml` snippet ensures that the **QEMU Guest Agent** is installed
and running on all our machines. Now that it's in place, let's go ahead and start
creating our VMs.

### Dummy VM

Now we can try provisioning a dummy VM to make sure everything is working as expected:

```terraform
terraform {
  required_providers {
    proxmox = {
      source  = "Telmate/proxmox"
      version = "3.0.1-rc8"
    }
  }
}

provider "proxmox" {
  pm_api_url          = "https://192.168.1.3:8006/api2/json"
  pm_api_token_id     = "terraform-prov@pve!mytoken"
  pm_api_token_secret = "INSERT_SECRET_TOKEN_HERE"
  pm_tls_insecure     = true
}

resource "proxmox_vm_qemu" "cloudinit-example" {
  vmid        = 100
  name        = "test-terraform0"
  target_node = "pve"
  agent       = 1 # Enable qemu-guest-agent

  cores  = 2
  memory = 1024
  boot   = "order=virtio0"      # Must match the OS disk of the template
  clone  = "ubuntu-2404-cloudinit-template" # Name of the template VM

  # Cloud-Init configuration
  cicustom  = "vendor=local:snippets/qemu-guest-agent.yml" # /var/lib/vz/snippets/qemu-guest-agent.yml
  ciupgrade = true
  skip_ipv6 = true
  # Make sure this IP is available in your network and the gateway is correct
  ipconfig0 = "ip=192.168.1.40/24,gw=192.168.1.254"

  serial { # Required by most cloud-init images for proper console output
    id = 0
  }
  disks {
    ide {
      # Some images require a cloud-init disk on the IDE controller, others on
      # the SCSI or SATA controller
      ide0 {
        cloudinit {
          storage = "fast-storage"
        }
      }
    }
    virtio {    # We have to specify the disk from our template, else Terraform
      virtio0 { # will think it's not supposed to be there.
        disk {
          storage = "fast-storage" # Must be at least as large as the template's
          size    = "20G"          # disk, otherwise the disk will be recreated
        }
      }
    }
  }
  network {
    id     = 0
    bridge = "vmbr0"
    model  = "virtio"
  }
}
```

### Fleet of VMs

Alright, let's start structuring our Terraform project a bit more. Here's what the
project layout looks like:

```sh
$ tree -a .
.
├── .auto.tfvars
├── ansible
│   └── update_etc_hosts.yaml
├── cluster.tf
├── out.tf
├── provider.tf
├── templates
│   ├── etc_hosts.tftpl
│   └── hosts.yaml.tftpl
└── variables.tf
```

Let's break it down, file by file.

#### Variables

Let's start with `variables.tf`. This file defines all the input variables used
across the project. All of them are pretty self-explanatory, here's a quick look:

```terraform
# --- Proxmox Server ---
variable "proxmox_host_ip" {
  type        = string
  default     = "192.168.1.3"
  description = "Proxmox host IP address"
}
variable "proxmox_token_id" {
  type        = string
  default     = "terraform-prov@pve!mytoken"
  description = "Proxmox API token ID"
}
variable "proxmox_token_secret" {
  type        = string
  default     = ""
  description = "Proxmox API token secret"
}
variable "proxmox_node_name" {
  type        = string
  default     = "pve"
  description = "Proxmox node name"
}
variable "proxmox_vm_storage" {
  type        = string
  default     = "fast-storage"
  description = "Proxmox VM storage"
}
variable "proxmox_vm_template" {
  type        = string
  default     = "ubuntu-2404-cloudinit-template"
  description = "Proxmox VM template"
}

# --- Kubernetes Cluster ---
variable "k8s_cluster_name" {
  type        = string
  default     = "demo"
  description = "Name of the cluster"
}
variable "k8s_cluster_dns_zone" {
  type        = string
  default     = "example.dev"
  description = "Name of the cluster"
}
variable "k8s_vm_id" {
  type        = number
  default     = 100
  description = "Base ID for VMs in the cluster"
}
variable "k8s_control_plane_count" {
  type        = number
  default     = 1
  description = "Number of control plane nodes"
}
variable "k8s_control_plane_memory" {
  type        = number
  default     = 8192
  description = "Control plane node memory in MB"
}
variable "k8s_control_plane_cores" {
  type        = number
  default     = 2
  description = "Control plane node CPU cores"
}
variable "k8s_control_plane_disk_size" {
  type        = number
  default     = 30
  description = "Control plane disk size in GB"
}
variable "k8s_node_count" {
  type        = number
  default     = 3
  description = "Number of worker nodes"
}
variable "k8s_node_memory" {
  type        = number
  default     = 4096
  description = "Control plane node memory in MB"
}
variable "k8s_node_cores" {
  type        = number
  default     = 4
  description = "Control plane node CPU cores"
}
variable "k8s_node_disk_size" {
  type        = number
  default     = 50
  description = "Node disk size in GB"
}

# --- Misc ---
variable "ssh_public_key" {
  type        = string
  default     = ""
  description = "SSH public key"
}
```

For best practice, sensitive variables, like secrets and tokens, should be stored
in a `.auto.tfvars` file:

```terraform
proxmox_token_secret = "INSERT_SECRET_TOKEN_HERE"
```

#### Providers

The `provider.tf` file defines the Terraform providers used in the project. In our
case, we're only using the `Telmate/proxmox` provider, along with its configuration.

```terraform
terraform {
  required_providers {
    proxmox = {
      source  = "Telmate/proxmox"
      version = "3.0.1-rc8"
    }
  }
}

provider "proxmox" {
  pm_api_url          = "https://${var.proxmox_host_ip}:8006/api2/json"
  pm_api_token_id     = var.proxmox_token_id
  pm_api_token_secret = var.proxmox_token_secret
  pm_tls_insecure     = true
}
```

#### Outputs

The `out.tf` file contains the Terraform code needed to generate a few useful outputs
for configuring the cluster:

- SSH keys to access the VMs — these will be stored in the `out` folder
- A `hosts` file with entries to append to each VM's `/etc/hosts`
- A `hosts.yaml` file used by Ansible to interact with the VMs

```terraform
resource "terraform_data" "create_out_dir" {
  provisioner "local-exec" {
    command = "mkdir -p ${path.module}/out"
  }
}

resource "tls_private_key" "ssh_key" {
  algorithm = "RSA"
  rsa_bits  = 4096
}

resource "local_file" "private_key" {
  depends_on           = [terraform_data.create_out_dir]
  content              = tls_private_key.ssh_key.private_key_pem
  filename             = "${path.module}/out/id_rsa"
  file_permission      = 0600
  directory_permission = 0755
}

resource "local_file" "public_key" {
  depends_on           = [terraform_data.create_out_dir, tls_private_key.ssh_key]
  content              = tls_private_key.ssh_key.public_key_openssh
  filename             = "${path.module}/out/id_rsa.pub"
  file_permission      = 0644
  directory_permission = 0755
}

resource "local_file" "cluster_hosts_yaml" {
  depends_on = [terraform_data.create_out_dir]
  content = templatefile("${path.module}/templates/hosts.yaml.tftpl", {
    control_planes = [
      for vm in proxmox_vm_qemu.control-plane : {
        hostname = vm.name
        ip       = vm.default_ipv4_address
        # index    = vm.count.index
      }
    ],
    nodes = [
      for vm in proxmox_vm_qemu.node : {
        hostname = vm.name
        ip       = vm.default_ipv4_address
        # index = vm.count.index
      }
    ],
    keypath = local_file.private_key.filename,
    user    = var.k8s_cluster_name,
  })
  filename             = "${path.module}/ansible/hosts.yaml"
  file_permission      = 0644
  directory_permission = 0755
}

resource "local_file" "cluster_etc_hosts" {
  depends_on = [terraform_data.create_out_dir]
  content = templatefile("${path.module}/templates/etc_hosts.tftpl", {
    hosts = [
      for vm in concat(proxmox_vm_qemu.control-plane, proxmox_vm_qemu.node) : {
        hostname = "${vm.name}.${var.k8s_cluster_dns_zone}"
        ip       = vm.default_ipv4_address
      }
    ]
  })
  filename             = "${path.module}/ansible/hosts"
  file_permission      = 0644
  directory_permission = 0755
}
```

#### Templates

The two files in the `templates` folder are used by the `out.tf` file to generate
output files.

- `etc_hosts.tftpl` is used to render the `/etc/hosts` entries for each VM
- `hosts.yaml.tftpl` is used to generate the Ansible inventory file

```terraform
# hosts.yaml.tftpl
all:
  vars:
    ansible_ssh_private_key_file: .${keypath}
    ansible_user: ${user}

controlplanes:
  hosts:
%{ for host in control_planes ~}
    ${host.hostname}:
      ansible_host: ${host.ip}
%{ endfor ~}
nodes:
  hosts:
%{ for host in nodes ~}
    ${host.hostname}:
      ansible_host: ${host.ip}
%{ endfor ~}

---
# etc_hosts.tftpl
%{ for host in hosts ~}
${host.ip} ${host.hostname}
%{ endfor ~}
```

#### Ansible

The `update_etc_hosts.yaml` playbook is a simple Ansible script that updates the
`/etc/hosts` file on each VM. This step is necessary because, in my home network,
I don't have a way to configure custom DNS rules. The entries allow each host in
the cluster to resolve the others by name.

```yaml
---
- name: Append custom entries to hosts files on all nodes
  hosts: all
  become: true
  vars:
    cluster_hosts: "{{ lookup('file', 'hosts') }}"
  tasks:
    - name: Append to /etc/hosts
      ansible.builtin.blockinfile:
        path: /etc/hosts
        block: "{{ cluster_hosts }}"
    - name: Append to /etc/cloud/templates/hosts.debian.tmpl
      ansible.builtin.blockinfile:
        path: /etc/cloud/templates/hosts.debian.tmpl
        block: "{{ cluster_hosts }}"
```

#### Cluster

Finally, the star of the show: `cluster.tf`. This is where we define the control
plane and worker nodes that will make up our Kubernetes cluster.

```terraform
resource "proxmox_vm_qemu" "control-plane" {
  # VM Metadata
  count       = var.k8s_control_plane_count
  name        = "${var.k8s_cluster_name}-control-plane-${count.index}"
  tags        = "cluster_${var.k8s_cluster_name}"
  target_node = var.proxmox_node_name
  vmid        = var.k8s_vm_id + count.index

  # VM Source
  clone = var.proxmox_vm_template

  # Virtual Hardware
  boot    = "order=virtio0"
  cores   = var.k8s_control_plane_cores
  memory  = var.k8s_control_plane_memory
  os_type = "cloud-init"
  scsihw  = "virtio-scsi-pci"

  # Cloud-init customization
  agent     = 1 # Enable QEMU guest agent
  ciupgrade = true
  ciuser    = var.k8s_cluster_name
  ipconfig0 = "ip=${cidrhost("192.168.1.0/24", 40 + count.index)}/24,gw=${cidrhost("192.168.1.0/24", 254)}"
  skip_ipv6 = true
  sshkeys   = local_file.public_key.content

  # Disks, Network and Serial Configuration
  disks {
    ide {
      ide0 {
        cloudinit {
          storage = var.proxmox_vm_storage
        }
      }
    }
    virtio {
      virtio0 {
        disk {
          size    = var.k8s_control_plane_disk_size
          storage = var.proxmox_vm_storage
        }
      }
    }
  }
  network {
    id     = 0
    model  = "virtio"
    bridge = "vmbr0"
  }
  serial {
    id = 0
  }
}

resource "proxmox_vm_qemu" "node" {
  # VM Metadata
  count       = var.k8s_node_count
  name        = "${var.k8s_cluster_name}-node-${count.index}"
  tags        = "cluster_${var.k8s_cluster_name}"
  target_node = var.proxmox_node_name
  vmid        = var.k8s_vm_id + 50 + count.index

  # VM Source
  clone = var.proxmox_vm_template

  # Virtual Hardware
  boot    = "order=virtio0"
  cores   = var.k8s_node_cores
  memory  = var.k8s_node_memory
  os_type = "cloud-init"
  scsihw  = "virtio-scsi-pci"

  # Cloud-init customization
  agent     = 1 # Enable QEMU guest agent
  ciupgrade = true
  ciuser    = var.k8s_cluster_name
  ipconfig0 = "ip=${cidrhost("192.168.1.0/24", 45 + count.index)}/24,gw=${cidrhost("192.168.1.0/24", 254)}"
  skip_ipv6 = true
  sshkeys   = local_file.public_key.content

  # Disks, Network and Serial Configuration
  disks {
    ide {
      ide0 {
        cloudinit {
          storage = var.proxmox_vm_storage
        }
      }
    }
    virtio {
      virtio0 {
        disk {
          size    = var.k8s_node_disk_size
          storage = var.proxmox_vm_storage
        }
      }
    }
  }
  network {
    id     = 0
    model  = "virtio"
    bridge = "vmbr0"
  }
  serial {
    id = 0
  }
}
```

#### Deploy

And voilà! Deploying our infrastructure is as simple as:

```sh
terraform init
terraform apply
cd ansible
ANSIBLE_HOST_KEY_CHECKING=false ansible-playbook update_etc_hosts.yaml -i hosts.yaml
# To resolve the hosts on your machine as well:
sudo tee -a /etc/hosts < hosts > /dev/null
```

Now let's move on to installing Kubernetes, leveraging the power of the SIGHUP Distribution.

## Step Three - Kubernetes

At this point, we have a control plane and three worker nodes. To install Kubernetes,
we'll use the [SIGHUP Distribution](https://docs.sighup.io/) with the **OnPremises**
installer.

### Install `furyctl`

In case you need to install `furyctl` you can follow the [documentation](https://docs.sighup.io/furyctl/installation)
and run:

```sh
curl -L "https://github.com/sighupio/furyctl/releases/latest/download/furyctl-$(uname -s)-amd64.tar.gz" \
     -o /tmp/furyctl.tar.gz && tar xfz /tmp/furyctl.tar.gz -C /tmp
chmod +x /tmp/furyctl
sudo mv /tmp/furyctl /usr/local/bin/furyctl
```

### Customize `furyctl.yaml`

To get started quickly, we generate a skeleton configuration with:

```sh
mkdir k8s
cd k8s
furyctl create config --kind OnPremises --name demo-cluster --version v1.31.1
```

Now we can start editing the `furyctl.yaml` manifest that was generated:

- Set `spec.kubernetes.ssh.username` to the one we previously setup using terraform,
  in this case `demo`
- Set `spec.kubernetes.ssh.keyPath` to the key generated by terraform, `../out/id_rsa`
- Set `spec.kubernetes.controlPlaneAddress` to `demo-control-plane-0.example.dev:6443`
- Remove `spec.kubernetes.proxy` block
- Set `spec.kubernetes.loadBalancers.enabled` to `false` and remove the rest of
  the block
- Replace `spec.kubernetes.masters.hosts` with a single entry specifying the correct
  name and ip of the control plane
- Update `spec.kubernetes.nodes.hosts` leaving only the worker section and updating
  the hosts accordingly
- Remove `spec.distribution.common` block
- Set `spec.distribution.modules.ingress.nginx.type` to `single`
- Set `spec.distribution.modules.tracing.type` to `none` and remove the rest of the
  block
- Set `spec.distribution.modules.policy.type` to `none` and remove the rest of the
  block
- Set `spec.distribution.modules.dr.type` to `none` and remove the rest of the block
- Remove `spec.distribution.modules.auth.baseDomain`
- Add a plugin section to install rancher [`local-path-provisioner`](https://github.com/rancher/local-path-provisioner)

You should be left with something looking like this:

```yaml
# yaml-language-server: $schema=https://raw.githubusercontent.com/sighupio/distribution/v1.31.1/schemas/public/onpremises-kfd-v1alpha2.json
---
apiVersion: kfd.sighup.io/v1alpha2
kind: OnPremises
metadata:
  name: demo-cluster
spec:
  distributionVersion: v1.31.1
  kubernetes:
    pkiFolder: ./pki
    ssh:
      username: demo
      keyPath: ../out/id_rsa
    dnsZone: example.dev
    controlPlaneAddress: demo-control-plane-0.example.dev:6443
    podCidr: 172.16.128.0/17
    svcCidr: 172.16.0.0/17
    loadBalancers:
      enabled: false
    masters:
      hosts:
        - name: demo-control-plane-0
          ip: 192.168.1.40
    nodes:
      - name: worker
        hosts:
          - name: demo-node-0
            ip: 192.168.1.45
          - name: demo-node-1
            ip: 192.168.1.46
          - name: demo-node-2
            ip: 192.168.1.47
  distribution:
    modules:
      networking:
        type: "calico"
      ingress:
        baseDomain: internal.example.dev
        nginx:
          type: single
          tls:
            provider: certManager
        certManager:
          clusterIssuer:
            name: letsencrypt-fury
            email: example@sighup.io
            type: http01
      logging:
        type: loki
        loki:
          tsdbStartDate: "2024-11-20"
        minio:
          storageSize: "20Gi"
      monitoring:
        type: "prometheus"
      tracing:
        type: none
      policy:
        type: none
      dr:
        type: none
      auth:
        provider:
          type: none
        baseDomain: example.dev
  plugins:
    kustomize:
      - name: storage
        folder: https://github.com/rancher/local-path-provisioner/deploy?ref=v0.0.31
```

### Summon Kubernetes Cluster: _clusterfacio_! 🪄

Now all that's left to do is apply our configuration and bootstrap a production-grade
Kubernetes cluster!

```sh
# First we need to generate the certificates needed by k8s
$ cd k8s
$ furyctl create pki

# Then we can apply the configuration
$ furyctl apply
INFO Downloading distribution...
INFO Validating configuration file...
INFO Downloading dependencies...
INFO Validating dependencies...
INFO Running preflight checks...
INFO Preflight checks completed successfully
INFO Running preupgrade phase...
INFO Preupgrade phase completed successfully
INFO Configuring SIGHUP Distribution cluster...
INFO Checking that the hosts are reachable...
INFO Applying cluster configuration...
INFO Kubernetes cluster created successfully
INFO Installing SIGHUP Distribution...
INFO Checking that the cluster is reachable...
INFO Checking storage classes...
WARN No storage classes found in the cluster. logging module (if enabled), tracing module (if enabled), dr module (if enabled) and prometheus-operated package installation will be skipped. You need to install a StorageClass and re-run furyctl to install the missing components.
INFO Applying Distribution modules...
INFO SIGHUP Distribution installed successfully
INFO Applying plugins...
INFO Plugins installed successfully
INFO Saving furyctl configuration file in the cluster...
INFO Saving distribution configuration file in the cluster...
```

The cluster should be up and running, except for a few components as shown by the
warning. You'll find a `kubeconfig` in the current directory that we can use to
interact with the cluster:

```sh
$ kubectl --kubeconfig kubeconfig get nodes
NAME                               STATUS   ROLES           AGE   VERSION
demo-control-plane-0.example.dev   Ready    control-plane   17m   v1.31.7
demo-node-0.example.dev            Ready    worker          17m   v1.31.7
demo-node-1.example.dev            Ready    worker          17m   v1.31.7
demo-node-2.example.dev            Ready    worker          17m   v1.31.7
```

How cool is that?! Now let's finish up our setup by making `local-path` the default
storage class and re apply our `furyctl.yaml` so that everything gets setup!

```sh
# Add the annotation to make local-path the default storage class
$ kubectl --kubeconfig annotate kubeconfig storageclass local-path "storageclass.kubernetes.io/is-default-class"="true"

# Re-run, as instructed, because now the storage class will be present, It was installed at the end as plugin
$ furyctl apply --outdir $(pwd) --phase distribution --skip-deps-download --skip-deps-validation
INFO Downloading distribution...
INFO Validating configuration file...
INFO Running preflight checks...
INFO Checking that the cluster is reachable...
INFO Preflight checks completed successfully
INFO Running preupgrade phase...
INFO Preupgrade phase completed successfully
INFO Installing SIGHUP Distribution...
INFO Checking that the cluster is reachable...
INFO Checking storage classes...
INFO Applying Distribution modules...
INFO SIGHUP Distribution installed successfully
INFO Saving furyctl configuration file in the cluster...
INFO Saving distribution configuration file in the cluster...
```

## Conclusion

And that's pretty much all of it! I hope this article was inspiring — because for
me, this experience has been absolutely crazy. I'm genuinely amazed by how much
of this workflow can be automated, and even more excited to dive deeper and get
some real hands-on experience with the SIGHUP Distribution.

If you're a DevOps sorcerer like me, I hope this guide gives you the magic spells
you need to start summoning your own clusters! ✨
