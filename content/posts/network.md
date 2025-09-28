+++
date = '2025-09-27T22:04:12+02:00'
draft = true
title = '[Homelab] Network'
tags = ['opentofu', 'mikrotik', 'network']
categories = ['homelab']    
+++

This is part of the [Homelab](/categories/homelab/) series. Please read disclaimer in the [Intro](/posts/homelab/). 

## Contents

{{< toc >}}

## Hardware
- Router [Mikrotik CRS310-1G-5S-4S+IN](https://mikrotik.com/product/crs310_1g_5s_4s_in) will be referred as `edge-router`
- Switch [HORACO 10Gb SFP+ 8 Ports](https://de.aliexpress.com/item/1005006765378093.html) will be referred as `lan-wifi`
- WiFi Router [Mikrotik hAP ac](https://mikrotik.com/product/RB962UiGS-5HacT2HnT) will be referred as `lan-wired-sw`
- WiFi Router [Tp-Link Archer BE800](https://www.tp-link.com/us/home-networking/wifi-router/archer-be800/) will be referred as `rack-10G`

TODO: diagram

## Configuration

All configuration will be done using [OpenTofu](https://opentofu.org/). 

State will be stored in [Google Cloud Storage](https://cloud.google.com/storage?hl=en). Free quota is more than enough.

Secrets will be stored in [Google Secret Manager](https://cloud.google.com/security/products/secret-manager?hl=en). Fre quota allows only 6 secrets, but they could contain JSON.


### Initialization

First, [login to google cloud CLI](https://cloud.google.com/docs/authentication/set-up-adc-local-dev-environment):

```bash
gcloud auth application-default login
```

After create `main.tf` file with required providers and backend

```terraform
terraform {
  required_version = ">= 1.9.0"

  backend "gcs" {
    bucket  = var.state_bucket
    prefix  = var.state_bucket_prefix
  }

  required_providers {
    google = {
      source = "hashicorp/google"
    }
  }
}
```

And `variables.tf` with required input variables

```terraform
variable "state_bucket" {
  type = string
  description = "Bucket to store state"
}

variable "state_bucket_prefix" {
  type = string
  description = "Prefix in the bucket to store state"
}

variable "google_project" {
  type = string
  description = "Google project to use"
}
```

Variables could be passed via command line or with `all.auto.tfvars` file:

```terraform
google_project = "<redacted>"
state_bucket = "<redacted>"
state_bucket_prefix = "infra/state"
```

Now call
```bash
tofu init
```

### Prepare hardware

To be able to apply configuration, few preparation steps required:

1. Reset mikrotik routers to factory settings without defconf
2. Add addresses that will not overlap with future networks:
   - `edge-router` `192.168.88.1/24` on `ether1` interface
   - `lan-wired-sw` `192.168.88.2/24` on `sfp1` interface
3. Set admin passwords

### Configuration definition

First, list managed and unmanaged devices

**manged-devices.tf**
```terraform
locals {
  managed-devices = {
    edge = {
      username = "terraform",
      random_password = true,
    },
    lan-wired-sw = {
      username = "terraform",
      random_password = true,
      address     = "192.168.10.2"
      mac_address = "<redacted>"
    },
  }
}
```

**unmanged-devices.tf**
```terraform
locals {
  unmanaged-devices = {
    lan-wifi = {
      address     = "192.168.20.2"
      mac_address = "<redacted>"
    },
    rack-10G = {
      address     = "192.168.10.3"
      mac_address = "<redacted>"
    },
  }
}
```

Next, define desired VLANs

| Id   | Name     | Network         | Purpose                                                   | 
|------|----------|-----------------|-----------------------------------------------------------|
| 10   | mgmt     | 192.168.10.0/24 | Management and interaction between network infrastructure |
| 20   | lan      | 192.168.20.0/24 | Home netowrk                                              |
| 30   | iot      | 192.168.30.0/24 | Network for IoT devices                                   |
| 100  | cluster  | 10.100.0.0/24   | Netowrk for kubernetes clusterr nodes                     |

**vlans.tf**
```terraform
locals {
  vlans = {
    mgmt = {
      vlan_id = 10,
      network = "192.168.10.0/24",
      comment = "VLAN for network infrastructure",
    },
    lan = {
      vlan_id = 20,
      network = "192.168.20.0/24",
      comment = "VLAN for home network",
    }
    iot = {
      vlan_id = 30,
      network = "192.168.30.0/24",
      comment = "VLAN for IoT devices",
    },
    cluster = {
      vlan_id = 100,
      network = "10.100.0.0/24",
      comment = "VLAN for home cluster nodes",
    }
  }
}
```

And cluster nodes

**cluster.tf**
```terraform
locals {
  cluster-nodes = {
    node-1 : {
      mac = "<redacted>"
      ip  = "10.100.0.2"
      control = true
    }
    node-2 = {
      mac = "<redacted>>"
      ip  = "10.100.0.3"
      control = true
    }
    node-3 = {
      mac = "<redacted>",
      ip  = "10.100.0.4",
      control = true
    }
  }
}
```

And wifi for IoT device
**wifi-capsman.tf**
```terraform
locals {
  wifi-capsman = {
    <redacted> = {
      channels = {
        wifi2 = {
          ssid = "<redacted>"
          band = "2ghz-b/g/n"
          frequency = [2442]
          hw_supported_modes = ["b", "g"]
        }
        wifi5 = {
          ssid = "<redacted>"
          band = "5ghz-a/n/ac"
          frequency = []
          hw_supported_modes = ["a", "ac"]
        }
      }
      vlan_id = local.vlans.iot.vlan_id
    }
  }
}
```


### Create secrets

To managed devices, terraform user will be created with password stored in Google secret manager.

To do this we need one more input variable in variables.tf
```terraform
variable "google_project" {
  type = string
  description = "Google project to use"
}
```
And value in `all.auto.tfvars`
```terraform
google_project = "<redacted>"
```

And one more provider in `main.tf` to generate random passwords
```terraform
    random = {
      source  = "hashicorp/random"
    }
```

#### secrets.tf

First, create password for every managed device
```terraform
resource "random_password" "device_password" {
  length = 32
  special = false
  for_each = {for k, v in local.managed-devices: k =>v if v.random_password }
} 
```

Next, save all in google secret manager using json
```terraform
resource "google_secret_manager_secret" "device-accounts" {
  secret_id = "tf_device-accounts"
  project = var.google_project

  replication {
    auto {}
  }
}

resource "google_secret_manager_secret_version" "device-accounts" {
  secret = google_secret_manager_secret.device-accounts.id
  lifecycle {
    create_before_destroy = true
  }
  secret_data = jsonencode({for k, v in local.managed-devices: k => random_password.device_password[k].result if v.random_password })
}
```
This is not the best way to store secrets, but allows to stay in free tier. Unfortunately, opentofu does not support write-only attributes and will preserve these secrets in state.

TODO: use secret_data_wo when possible

For future use in providers define decoded version
```terraform
locals {
  _device-accounts-decoded = jsondecode(google_secret_manager_secret_version.device-accounts.secret_data)
  device-accounts = {for k, v in local.managed-devices: k=> {
    username = v.username,
    password = local._device-accounts-decoded[k]
  } if v.random_password}
}
```

Repeat for Wi-Fi passwords

```terraform

resource "random_password" "wifi_password" {
  length = 32
  special = true
  for_each = {for k, v in local.wifi-capsman: k =>v }
}


resource "google_secret_manager_secret" "wifi_passwords" {
  secret_id = "tf_wifi_passwords"
  project = var.google_project
  replication {
    auto {}
  }
}

resource "google_secret_manager_secret_version" "wifi_passwords" {
  secret = google_secret_manager_secret.wifi_passwords.id
  lifecycle {
    create_before_destroy = true
  }
  secret_data = jsonencode({for k, v in local.wifi-capsman: k => random_password.wifi_password[k].result })
}

locals {
  _wifi_passwords-decoded = jsondecode(google_secret_manager_secret_version.wifi_passwords.secret_data)
  wifi-passwords = {for k, v in local.wifi-capsman: k=> {
    password = local._wifi_passwords-decoded[k]
  }}
}
```

### Edge router

To configure edge router let's create `edge-router` module and include it

**main.tf**
```terraform
module "edge-router" {
  source = "./edge-router"
  hosturl = var.edge_router_url
  account = local.device-accounts["edge"]

  static-leases = merge({
    for k, v in local.managed-devices: k=>{
      address     = v.address,
      mac_address = v.mac_address,
      comment     = k,
    } if contains(keys(v), "mac_address")
    }, {
    for k, v in local.unmanaged-devices : k=>{
      address = v.address,
      mac_address = v.mac_address,
      comment = k,
    } if contains(keys(v), "mac_address")
    }, {
    for k, v in local.cluster-nodes : k=>{
      address = v.ip,
      mac_address = v.mac,
      comment = k,
    }
    })
  vlans = local.vlans
  capsman_wifi = {
    networks = {
      for k,v in local.wifi-capsman: k => {
        channels = v.channels,
        vlan_id = v.vlan_id,
      }
    }
  }
  capsman_psk = {
    for k,v in local.wifi-passwords:
      k => local.wifi-passwords[k].password
  }
}
```

And add provider to main.tf
```terraform
    routeros = {
      source = "terraform-routeros/routeros"
    }
```

#### Edge router module

**edge.tf**
```terraform
terraform {
  required_providers {
    routeros = {
      source = "terraform-routeros/routeros"
    }
  }
}
```

**input.tf**
```terraform
variable "vlans" {
  type = map(object({
    vlan_id = number,
    network = string,
    comment = string,
  }))
}

variable "static-leases" {
  type = map(object({
    address = string,
    mac_address = string,
    comment = string,
  }))
}

variable "capsman_wifi" {
  type = object({
    networks = map(object({
      vlan_id = number
      channels = map(object({
        ssid = string
        band = string
        frequency = list(number)
        hw_supported_modes = list(string)
      }))
    }))
  })
}

variable "capsman_psk" {
  type = map(string)
  sensitive = true
}
```

And one file that will be shared with `lan-wired-sw` (copy-paste)

**tf-account.tf**
```terraform
variable "account" {
  type = object({
    username = string,
    password = string
  })
  sensitive = true
}

resource "routeros_system_user" "user" {
  group = "full"
  name  = var.account.username
  password = var.account.password
}

variable "hosturl" {
  type = string
}

provider "routeros" {
  hosturl  = var.hosturl
  username = var.account.username
  password = var.account.password
}
```

Now we need small hack to initially provision tf account - replace username/password in provider with admin account and run

```bash
tofu init
tofu apply
```

Now we will be able to use our created account for edge-router.

First, let's create VLAN's 

**vlans.tf**
```terraform
locals {
  vlans = {
    mgmt = 10,
    lan = 20,
    iot = 30,
    cluster = 100,
  }
}

resource "routeros_interface_vlan" "vlan" {
  interface = routeros_interface_bridge.bridge.name
  for_each = var.vlans
  name      = each.key
  vlan_id   = each.value.vlan_id
  comment = each.value.comment
}

```

Now we could setup bridge and configure ports
**bridge.tf**
```terraform
locals {
  bridge_ports = {
    ether1 = {
      disabled = false,
      name = "mgmt-eth",
      comment = "Management port",
      untagged = routeros_interface_vlan.vlan["mgmt"].vlan_id,
      tagged = [],
    },
    sfp1 = {
      disabled = true,
    },
    sfp2 = {
      disabled = true,
    },
    sfp3 = {
      disabled = true,
    },
    sfp4 = {
      disabled = true,
    },
    sfp5 = {
      disabled = false,
      name = "lan-wired",
      comment = "LAN router + IoT AP",
      untagged = routeros_interface_vlan.vlan["mgmt"].vlan_id,
      tagged = [
        routeros_interface_vlan.vlan["lan"].vlan_id,
        routeros_interface_vlan.vlan["iot"].vlan_id,
      ],
    },
    sfp-sfpplus1 = {
      disabled = true,
    },
    sfp-sfpplus2 = {
      disabled = false,
      name = "rack-10G",
      comment = "Rack 10G SFP+ Switch",
      untagged = routeros_interface_vlan.vlan["mgmt"].vlan_id,
      tagged = [
        routeros_interface_vlan.vlan["cluster"].vlan_id,
      ],
    },
    sfp-sfpplus3 = {
      disabled = false,
      name = "lan-wifi",
      comment = "WiFi router for LAN",
      untagged = routeros_interface_vlan.vlan["lan"].vlan_id,
      tagged = [],
    }
  }
}

resource "routeros_interface_bridge" "bridge" {
  name = "bridge"
  vlan_filtering = true
}

resource "routeros_interface_ethernet" "port" {
  for_each = local.bridge_ports
  factory_name = each.key
  name         = each.value.disabled ? each.key : each.value.name
  comment = each.value.disabled ? "" : each.value.comment
  disabled = each.value.disabled
}


resource "routeros_interface_bridge_port" "bridge-port" {
  bridge    = routeros_interface_bridge.bridge.name

  for_each =  { for k, v in local.bridge_ports : k => v if !v.disabled }
  interface = routeros_interface_ethernet.port[each.key].name

  comment = each.value.comment

  ingress_filtering = true //true
  frame_types = length(each.value.tagged) == 0 ? "admit-only-untagged-and-priority-tagged" : "admit-all"
  pvid = each.value.untagged
}


resource "routeros_interface_bridge_vlan" "bridge-vlan" {
  bridge = routeros_interface_bridge.bridge.name

  for_each = var.vlans
  vlan_ids = [each.value.vlan_id]

  tagged = concat([for k,v in local.bridge_ports : routeros_interface_bridge_port.bridge-port[k].interface if !v.disabled && contains(v.tagged, each.value.vlan_id)], [routeros_interface_bridge.bridge.name])
}
```

And uplink
**uplink.tf**
```terraform
resource "routeros_interface_ethernet" "wan" {
  factory_name = "sfp-sfpplus4"
  name         = "wan"
  mtu          = 1500
}

resource "routeros_ip_dhcp_client" "wan" {
  interface = routeros_interface_ethernet.wan.name
}

resource "routeros_ipv6_dhcp_client" "wan" {
  interface         = routeros_interface_ethernet.wan.name
  add_default_route = "false"
  request = ["prefix"]
  pool_name         = "uplink-ipv6"
  pool_prefix_length = 64
  use_peer_dns = true
}

resource "routeros_ipv6_address" "wan" {
  from_pool = routeros_ipv6_dhcp_client.wan.pool_name
  interface = routeros_interface_ethernet.wan.name
  //advertise = true
  //eui_64 = true
}

resource "routeros_ipv6_neighbor_discovery" "wan" {
  interface = routeros_interface_ethernet.wan.name
  advertise_mac_address = true
  disabled = false
}

resource "routeros_interface_list" "wan" {
  name = "WAN"
}

resource "routeros_interface_list_member" "wan" {
  interface = routeros_interface_ethernet.wan.name
  list = routeros_interface_list.wan.name
}

resource "routeros_ip_firewall_nat" "wan-srcnat" {
  chain              = "srcnat"
  action             = "masquerade"
  ipsec_policy       = "out,none"
  out_interface_list = routeros_interface_list.wan.name
}
```

Since there is no firewall configured yet, it's better to keep uplink cable unplugged.

Now it's time to configure DHCP and addresses. Because there are many resources to be configured for every network, let's move them into separate module and include for all networks

**dhcp-nets.tf**
```terraform
module "mikrotik-dhcp-net" {
  source = "../mikrotik-dhcp-net"

  for_each = var.vlans
  comment = each.value.comment
  name = each.key
  interface = routeros_interface_vlan.vlan[each.key].name
  ipv6-pool = routeros_ipv6_dhcp_client.wan.pool_name

  network = each.value.network

  providers = {
    routeros = routeros
  }
}

```

Now we could add static leases for known devices
**dhcp-static.tf**
```terraform
resource "routeros_dhcp_server_lease" "static-lease" {
  for_each = var.static-leases
  address     = each.value.address
  mac_address = each.value.mac_address
  comment = each.value.comment
}
```

To keep time in sync we need NTP-client too
**ntp.tf**
```terraform
resource "routeros_system_ntp_client" "ntp" {
  servers = [
    "pool.ntp.org",
    "0.pool.ntp.org",
    "1.pool.ntp.org",
    "2.pool.ntp.org",
    "3.pool.ntp.org"
  ]
  enabled = true
}
```

To configure WiFi with CAPsMAN, we need certificates: 

**cert.tf**
```terraform
resource "routeros_system_certificate" "edge-ca" {
  name        = "edge-ca"
  common_name = "edge-ca"
  trusted = true
  key_usage = ["key-cert-sign","crl-sign"]
  sign {}
}

resource "routeros_system_certificate_scep_server" "edge-ca" {
  ca_cert = routeros_system_certificate.edge-ca.name
  path    = "/scep/edge-ca"
}
```

And now we can setup out CAPsMAN:

**capsman.tf**
```terraform
locals {
  capsman-ip = cidrhost(var.vlans["mgmt"].network, 1)
}

resource "routeros_system_certificate" "capsman" {
  common_name = local.capsman-ip
  name        = "capsman"
  sign {
    ca = routeros_system_certificate.edge-ca.name
  }
}

resource "routeros_capsman_manager" "settings" {
  enabled                  = true
  upgrade_policy           = "none"
  certificate = routeros_system_certificate.capsman.name
  ca_certificate = routeros_system_certificate.edge-ca.name
  require_peer_certificate = true
}

output "capsman-ip" {
  value = local.capsman-ip
}

locals {
  all_channels = {for x in flatten([
    for nk,nv in var.capsman_wifi.networks: [
        for k, v in nv.channels : { nkey = nk, nval = nv, ckey = k, cval = v }
      ]]): format("%s-%s", x.nkey, x.ckey) => { network_key = x.nkey, network = x.nval, channel = x.cval}
    }
}

resource "routeros_capsman_channel" "wifi" {
  for_each = local.all_channels
  name = each.key
  band = each.value.channel.band
  frequency = each.value.channel.frequency
}

resource "routeros_capsman_security" "wifi_password" {
  for_each = var.capsman_wifi.networks
  name                 = each.key
  authentication_types = ["wpa2-psk"]
  passphrase           = var.capsman_psk[each.key]
}

resource "routeros_capsman_datapath" "vlan_tagging" {
  for_each = var.capsman_wifi.networks
  name    = format("%s-tagging", each.key)
  comment = format("WiFi %s VLAN", each.key)
  vlan_id = each.value.vlan_id
  vlan_mode = "use-tag"
  bridge = routeros_interface_bridge.bridge.name
}

resource "routeros_capsman_configuration" "config" {
  for_each = local.all_channels

  country = "switzerland"
  name    = each.value.channel.ssid
  ssid    = each.value.channel.ssid
  comment = ""

  mode = "ap"
  installation = "indoor"

  channel = {
    config = routeros_capsman_channel.wifi[each.key].name
  }
  datapath = {
    config = routeros_capsman_datapath.vlan_tagging[each.value.network_key].name
  }
  security = {
    config = routeros_capsman_security.wifi_password[each.value.network_key].name
  }
}

resource "routeros_capsman_provisioning" "wifi" {
  action               = "create-dynamic-enabled"
  for_each = local.all_channels
  comment              = routeros_capsman_configuration.config[each.key].name
  master_configuration = routeros_capsman_configuration.config[each.key].name
  slave_configurations = []
  hw_supported_modes = each.value.channel.hw_supported_modes
}

resource "routeros_capsman_manager_interface" "bridge" {
  interface = routeros_interface_vlan.vlan["mgmt"].name
  forbid = false
  disabled = false
}
```

##### Firewall

Let's setup very basic firewall to be able to use internet in our network

**firewall.tf**
```terraform
resource "routeros_interface_list" "all-local" {
  name = "ALL_LOCAL"
}

resource "routeros_interface_list_member" "all-local-vlan" {
  list = routeros_interface_list.all-local.name
  for_each = var.vlans
  interface = routeros_interface_vlan.vlan[each.key].name
}

resource "routeros_interface_list" "vlan" {
  for_each = var.vlans
  name = format("vlan-%s", each.key)
}

resource "routeros_interface_list_member" "vlan" {
  list = routeros_interface_list.vlan[each.key].name
  for_each = var.vlans
  interface = routeros_interface_vlan.vlan[each.key].name
}


resource "routeros_ip_firewall_filter" "accept_established_related_untracked" {
  action           = "accept"
  chain            = "input"
  comment          = "accept established, related, untracked"
  connection_state = "established,related,untracked"
  place_before     = routeros_ip_firewall_filter.drop_invalid.id
}
resource "routeros_ip_firewall_filter" "drop_invalid" {
  action           = "drop"
  chain            = "input"
  comment          = "drop invalid"
  connection_state = "invalid"
  place_before     = routeros_ip_firewall_filter.accept_icmp.id
}
resource "routeros_ip_firewall_filter" "accept_icmp" {
  action       = "accept"
  chain        = "input"
  comment      = "accept ICMP"
  protocol     = "icmp"
  place_before = routeros_ip_firewall_filter.capsman_accept_local_loopback.id
}
resource "routeros_ip_firewall_filter" "capsman_accept_local_loopback" {
  action       = "accept"
  chain        = "input"
  comment      = "accept to local loopback for capsman"
  dst_address  = "127.0.0.1"
  place_before = routeros_ip_firewall_filter.drop_all_not_lan.id
}
resource "routeros_ip_firewall_filter" "drop_all_not_lan" {
  action            = "drop"
  chain             = "input"
  comment           = "drop all not coming from LAN"
  in_interface_list = "!ALL_LOCAL"
  place_before      = routeros_ip_firewall_filter.fasttrack_connection.id
}

resource "routeros_ip_firewall_filter" "fasttrack_connection" {
  action           = "fasttrack-connection"
  chain            = "forward"
  comment          = "fasttrack"
  connection_state = "established,related"
  hw_offload       = "true"
  place_before     = routeros_ip_firewall_filter.accept_established_related_untracked_forward.id
}

resource "routeros_ip_firewall_filter" "accept_established_related_untracked_forward" {
  action           = "accept"
  chain            = "forward"
  comment          = "accept established, related, untracked"
  connection_state = "established,related,untracked"
  place_before     = routeros_ip_firewall_filter.drop_invalid_forward.id
}

resource "routeros_ip_firewall_filter" "drop_invalid_forward" {
  action           = "drop"
  chain            = "forward"
  comment          = "drop invalid"
  connection_state = "invalid"
  place_before     = routeros_ip_firewall_filter.drop_all_wan_not_dstnat.id
}

resource "routeros_ip_firewall_filter" "drop_all_wan_not_dstnat" {
  action               = "drop"
  chain                = "forward"
  comment              = "drop all from WAN not DSTNATed"
  connection_nat_state = "!dstnat"
  connection_state     = "new"
  in_interface_list    = "WAN"
}
```

And for ipv6

**firewallv6.tf**
```terraform
resource "routeros_ipv6_firewall_filter" "accept_established_related_untracked" {
  action           = "accept"
  chain            = "input"
  comment          = "accept established, related, untracked"
  connection_state = "established,related,untracked"
  place_before     = routeros_ipv6_firewall_filter.drop_invalid.id
}
resource "routeros_ipv6_firewall_filter" "drop_invalid" {
  action           = "drop"
  chain            = "input"
  comment          = "drop invalid"
  connection_state = "invalid"
  place_before     = routeros_ipv6_firewall_filter.accept_icmp.id
}
resource "routeros_ipv6_firewall_filter" "accept_icmp" {
  action       = "accept"
  chain        = "input"
  comment      = "accept ICMPv6"
  protocol     = "icmpv6"
  place_before     = routeros_ipv6_firewall_filter.accept_dhcpv6.id
}

resource "routeros_ipv6_firewall_filter" "accept_dhcpv6" {
  action       = "accept"
  chain        = "input"
  comment      = "accept DHCPv6"
  protocol     = "udp"
  dst_port = "546"
  place_before = routeros_ipv6_firewall_filter.drop_all_not_lan.id
}

resource "routeros_ipv6_firewall_filter" "drop_all_not_lan" {
  action            = "drop"
  chain             = "input"
  comment           = "drop all not coming from LAN"
  in_interface_list = "!ALL_LOCAL"
  place_before = routeros_ipv6_firewall_filter.drop_invalid_forward.id
}

resource "routeros_ipv6_firewall_filter" "drop_invalid_forward" {
  action           = "drop"
  chain            = "forward"
  comment          = "drop invalid"
  connection_state = "invalid"
}
```

#### mikrotik-dhcp-net module

**input.tf**
```terraform
variable "interface" {
  type = string
  description = "Interface name"
}

variable "name" {
  type = string
  description = "Network name"
}

variable "dns" {
  type = list(string)
  description = "DNS servers"
  nullable = true
  default = null
}

variable "network" {
  type = string
  description = "Network in cidr notation"
}

variable "ipv6-pool" {
  type = string
  description = "IPv6 DHCP Pool name"
}

variable "comment" {
  type = string
  description = "Comment"
}
```

And main body of module
**mikrotik-dhcp-net.tf**
```terraform
terraform {
  required_providers {
    routeros = {
      source  = "terraform-routeros/routeros"
    }
  }
}

locals {
  dhcp-ranges = [format("%s-%s", cidrhost(var.network, 2), cidrhost(var.network, -2))]
  network = cidrhost(var.network, 0)
  router-ip = cidrhost(var.network, 1)
  netmask = tonumber(split("/", var.network)[1])

  router-ip-with-net = format("%s/%d", local.router-ip, local.netmask)
}

resource "routeros_ip_address" "addr" {
  address   = local.router-ip-with-net
  interface = var.interface
  network   = local.network
  comment = var.name
}

resource "routeros_ip_pool" "pool" {
  name   = var.name
  ranges = local.dhcp-ranges
  comment = var.name
}

resource "routeros_ip_dhcp_server_network" "net" {
  address    = var.network
  gateway    = local.router-ip
  dns_server = coalesce(var.dns, [local.router-ip])
  netmask = local.netmask
  comment = var.name
}

resource "routeros_ip_dhcp_server" "srv" {
  name         = var.name
  address_pool = routeros_ip_pool.pool.name
  interface    = var.interface
}

resource "routeros_ipv6_address" "addr-v6" {
  from_pool = var.ipv6-pool
  interface = var.interface
  advertise = true
  eui_64 = true
  comment = var.name
}
```

### lan-wired-sw

Now we could start to configure second device.

**main.tf**
```terraform
module "lan-wired-sw" {
  source = "./lan-wired-sw"
  capsman = module.edge-router.capsman-ip

  hosturl = format("http://%s", local.managed-devices.lan-wired-sw.address)
  account = local.device-accounts["lan-wired-sw"]
  vlans = {for k in ["lan", "mgmt", "iot"]: k => local.vlans[k]}
}
```

### lan-wired-sw module

**lan-wired-sw.tf**
```terraform
terraform {
  required_providers {
    routeros = {
      source = "terraform-routeros/routeros"
    }
  }
}
```

**input.tf**
```terraform
variable "capsman" {
  type = string
  description = "Capsman address"
}
```

First, use the same steps and `tf-account.tf` file.

Next, let's configure basic network

**bridge.tf**
```terraform
locals {
  bridge_ports = {
    ether1 = {
      disabled = false,
      name = "ether1",
      comment = "Port 1",
      untagged = routeros_interface_vlan.vlan["lan"].vlan_id,
      tagged = [],
    },
    ether2 = {
      disabled = false,
      name = "ether2",
      comment = "Port 2",
      untagged = routeros_interface_vlan.vlan["lan"].vlan_id,
      tagged = [],
    },
    ether3 = {
      disabled = false,
      name = "ether3",
      comment = "Port 3",
      untagged = routeros_interface_vlan.vlan["lan"].vlan_id,
      tagged = [],
    },
    ether4 = {
      disabled = false,
      name = "ether4",
      comment = "Port 4",
      untagged = routeros_interface_vlan.vlan["lan"].vlan_id,
      tagged = [],
    },
    ether5 = {
      disabled = false,
      name = "ether5",
      comment = "Port 5",
      untagged = routeros_interface_vlan.vlan["lan"].vlan_id,
      tagged = [],
    },
    sfp1 = {
      disabled = false,
      name = "uplink",
      comment = "To Edge Router",
      untagged = routeros_interface_vlan.vlan["mgmt"].vlan_id,
      tagged = [
        routeros_interface_vlan.vlan["lan"].vlan_id,
        routeros_interface_vlan.vlan["iot"].vlan_id,
      ],
    }
  }
}

resource "routeros_interface_bridge" "bridge" {
  name = "bridge"
  vlan_filtering = true
}

resource "routeros_interface_ethernet" "port" {
  for_each = local.bridge_ports
  factory_name = each.key
  name         = each.value.disabled ? each.key : each.value.name
  comment = each.value.disabled ? "" : each.value.comment
  disabled = each.value.disabled
}

resource "routeros_interface_bridge_port" "bridge-port" {
  bridge    = routeros_interface_bridge.bridge.name

  for_each =  { for k, v in local.bridge_ports : k => v if !v.disabled }
  interface = routeros_interface_ethernet.port[each.key].name

  comment = each.value.comment

  ingress_filtering = true
  frame_types = length(each.value.tagged) == 0 ? "admit-only-untagged-and-priority-tagged" : "admit-all"
  pvid = each.value.untagged
}

resource "routeros_interface_bridge_vlan" "bridge-vlan" {
  bridge = routeros_interface_bridge.bridge.name

  for_each = var.vlans
  vlan_ids = [each.value.vlan_id]

  tagged = concat([for k,v in local.bridge_ports : routeros_interface_bridge_port.bridge-port[k].interface if !v.disabled && contains(v.tagged, each.value)], [routeros_interface_bridge.bridge.name])
}
```

**lan-wired-sw.tf**
```terraform
resource "routeros_ip_dns" "dns" {
  allow_remote_requests = true
}

resource "routeros_ipv6_settings" "disable" {
  disable_ipv6 = "false"
  accept_router_advertisements = "yes"
  forward = true
}

resource "routeros_dhcp_client" "dhcp-client" {
  interface = routeros_interface_vlan.vlan["mgmt"].name
}
```

**vlans.tf**
```terraform
resource "routeros_interface_vlan" "vlan" {
  interface = routeros_interface_bridge.bridge.name
  for_each = var.vlans
  name      = each.key
  vlan_id   = each.value.vlan_id
}
```

Now to configure Wi-Fi we need to add certificates
**cap.tf**
```terraform
data "routeros_ip_addresses" "mgmtip" {

}

resource "routeros_system_certificate" "cap" {
  common_name = split("/", data.routeros_ip_addresses.mgmtip.addresses[0].address)[0]
  name        = "cap"
  key_usage   = ["digital-signature", "key-agreement", "tls-client"]
  sign_via_scep {
    scep_url = format("http://%s/scep/edge-ca", var.capsman)
    refresh = true
  }
}
```

After it we could add CAPs
**cap.tf**
```terraform
locals {
  wlans = {
    wlan1 = {
      disabled = false
    }
    wlan2 = {
      disabled = false
    }
  }
}
resource "routeros_interface_wireless_cap" "cap" {
  bridge = routeros_interface_bridge.bridge.name
  enabled = true
  lock_to_caps_man = true
  caps_man_certificate_common_names = [var.capsman]
  caps_man_addresses = [var.capsman]
  interfaces = [for k,v in local.wlans: k]
  certificate = routeros_system_certificate.cap.name
}
```

And add WLANs to the bridge
**bridge.tf**
```terraform
resource "routeros_interface_bridge_port" "bridge-port-wlan" {
  bridge    = routeros_interface_bridge.bridge.name

  for_each =  { for k, v in local.wlans : k => v if !v.disabled }
  interface = each.key

  ingress_filtering = true
  frame_types = "admit-only-vlan-tagged"
}
```

## Cleanup

Only thing left is to remove temporary IPs added for initial configuration and plug in uplink.



