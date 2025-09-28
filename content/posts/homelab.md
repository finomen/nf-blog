+++
date = '2025-09-27T22:03:21+02:00'
draft = true
title = '[Homelab] Intro'
categories = ['homelab'] 
+++

## Disclaimer

- Informational use only: This blog documents my personal homelab experiments and configurations. It is provided “as is,” may be incomplete or contain errors, and is not official guidance or a how‑to.
- Not for production: Do not use any information, configurations, or examples from this blog in production environments. Always validate against official vendor documentation, test in a non‑production lab, and apply your organization’s standards and change controls.
- No warranty or liability: All content is provided without any warranties or guarantees of any kind. By using any information here, you accept full responsibility and risk. I am not liable for any loss, damage, downtime, security incidents, compliance issues, data loss, or costs arising from the use or misuse of this content.
- Free use for all: You are free to use, adapt, and share the information in this blog for any purpose. Attribution is appreciated but not required.

## Contents

{{< toc >}}

## Goals 

There are a few main goals:
- Make private and secure data storage accessible from anywhere
- Make a private media server
- Make a place to host pet-projects
- Learn new technologies

## Plan

After seeing the video with [DeskPi RackMate](https://deskpi.com/collections/deskpi-rack-mate) I started thinking about getting one, but found two issues:
- Delivery costs more than the rack itself
- Only Mini-ITX motherboards fit inside

So I've decided to make my own. The first version will be 3D-printed with the following features:
- Enclosed on the sides with acrylic panels leaving space for connections on the side 
- 15U capacity
- 350mm depth to accommodate a small M-ATX motherboard (for example, [this one](https://www.amazon.it/dp/B0CF9XSCLN))

This rack eventually will contain:
- Router ([Mikrotik CRS310-1G-5S-4S+IN](https://mikrotik.com/product/crs310_1g_5s_4s_in))
- Switch ([HORACO 10Gb SFP+ 8 Ports](https://de.aliexpress.com/item/1005006765378093.html))
- Four compute nodes, at least two with GPUs

