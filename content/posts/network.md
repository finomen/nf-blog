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
- Router [Mikrotik CRS310-1G-5S-4S+IN](https://mikrotik.com/product/crs310_1g_5s_4s_in)
- Switch [HORACO 10Gb SFP+ 8 Ports](https://de.aliexpress.com/item/1005006765378093.html)
- WiFi Router [Mikrotik hAP ac](https://mikrotik.com/product/RB962UiGS-5HacT2HnT)
- WiFi Router [Tp-Link Archer BE800](https://www.tp-link.com/us/home-networking/wifi-router/archer-be800/)

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
