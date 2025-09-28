+++
date = '2025-09-27T22:04:12+02:00'
draft = true
title = '[Homelab] Talos Linux'
tags = ['opentofu', 'talos', 'kubernetes']
categories = ['homelab']
+++

This is part of the [Homelab](/categories/homelab/) series. Please read disclaimer in the [Intro](/posts/homelab/).

Previous part is [Network](/posts/network/)

## Contents

{{< toc >}}

## Intro

Homelab cluster will run Talos linux. Initial configuration and some core modules will be configured from the same terraform project as networking. I've found another post about [talos configuration on proxmox](https://olav.ninja/talos-cluster-on-proxmox-with-terraform) which was very helpful.

## Initialization

First add required provider to main.tf
```terraform
talos = {
  source = "siderolabs/talos"
}
```

And add new module 
```terraform
module "talos" {
  source = "./talos"
  nodes = local.cluster-nodes
  cluster_name = "homelab"
}
```

## Talos module

Talos module is relatively simple accepting few inputs
**inputs.tf**
```terraform
variable "nodes" {
  type = map(object({
    mac = string
    ip = string
    control = bool
  }))
}

variable "cluster_name" {
  type = string
}
```

And with core implementation

**talos.tf**
```terraform
terraform {
  required_providers {
    talos = {
      source = "siderolabs/talos"
    }
  }
}

resource "talos_machine_secrets" "machine_secrets" {}

data "talos_client_configuration" "talosconfig" {
  cluster_name         = var.cluster_name
  client_configuration = talos_machine_secrets.machine_secrets.client_configuration
  endpoints            = [for k,node in var.nodes: node.ip]
}

data "talos_machine_disks" "system-disk" {
  for_each = var.nodes
  client_configuration = talos_machine_secrets.machine_secrets.client_configuration
  node                 = each.value.ip
  selector             = "disk.transport == 'nvme' && disk.size < 1500u * GB"
}

data "talos_machine_configuration" "machineconfig_cp" {
  for_each = {for k, v in var.nodes : k => v if v.control}
  cluster_name     = var.cluster_name
  cluster_endpoint = "https://${each.value.ip}:6443"
  machine_type     = "controlplane"
  machine_secrets  = talos_machine_secrets.machine_secrets.machine_secrets
  config_patches = [
    templatefile("${path.module}/templates/install-disk-and-hostname.yaml.tmpl", {
      hostname     = each.key
      install_disk = data.talos_machine_disks.system-disk[each.key].disks[0].dev_path
      ip = each.value.ip
      name = each.key
    }),
    file("${path.module}/files/cluster-ips.yaml"),
    file("${path.module}/files/cluster-name.yaml"),
    file("${path.module}/files/cp-scheduling.yaml"),
    file("${path.module}/files/max_map_count.yaml"),
    file("${path.module}/files/disable-kube-proxy-and-cni.yaml"),
    file("${path.module}/files/multihome.yaml"),
  ]
}

resource "talos_machine_configuration_apply" "cp_config_apply" {
  for_each = {for k, v in var.nodes : k => v if v.control}

  client_configuration        = talos_machine_secrets.machine_secrets.client_configuration
  machine_configuration_input = data.talos_machine_configuration.machineconfig_cp[each.key].machine_configuration
  node                        = each.value.ip

  timeouts = {
    create = "1m"
  }
}

resource "talos_machine_bootstrap" "bootstrap" {
  depends_on           = [ talos_machine_configuration_apply.cp_config_apply ]
  client_configuration = talos_machine_secrets.machine_secrets.client_configuration
  node                 = [for k, v in var.nodes : v.ip if v.control][0]

  timeouts = {
    create = "1m"
  }
}

data "talos_cluster_health" "health" {
  depends_on           = [ talos_machine_configuration_apply.cp_config_apply ]
  client_configuration = data.talos_client_configuration.talosconfig.client_configuration
  control_plane_nodes  = [ for k,node in var.nodes: node.ip if node.control ]
  worker_nodes         = [ for k,node in var.nodes: node.ip if !node.control ]
  endpoints            = data.talos_client_configuration.talosconfig.endpoints
}

resource "talos_cluster_kubeconfig" "kubeconfig" {
  depends_on           = [ talos_machine_bootstrap.bootstrap, data.talos_cluster_health.health ]
  client_configuration = talos_machine_secrets.machine_secrets.client_configuration
  node                 = [for k, v in var.nodes : v.ip if v.control][0]
}

output "talosconfig" {
  value = data.talos_client_configuration.talosconfig.talos_config
  sensitive = true
}

output "kubeconfig" {
  value = talos_cluster_kubeconfig.kubeconfig.kubeconfig_raw
  sensitive = true
}
```