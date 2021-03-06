---
layout: "vcd"
page_title: "vCloudDirector: vcd_network_isolated"
sidebar_current: "docs-vcd-resource-network-isolated"
description: |-
  Provides a vCloud Director Org VDC isolated Network. This can be used to create, modify, and delete internal networks for vApps to connect.
---

# vcd\_network\_isolated

Provides a vCloud Director Org VDC isolated Network. This can be used to create,
modify, and delete internal networks for vApps to connect. This network is not attached to external networks or routers.

Supported in provider *v2.0+*

## Example Usage

```hcl
resource "vcd_network_isolated" "net" {
  org = "my-org" # Optional
  vdc = "my-vdc" # Optional

  name    = "my-net"
  gateway = "192.168.2.1"
  dns1    = "192.168.2.1"

  dhcp_pool {
    start_address = "192.168.2.2"
    end_address   = "192.168.2.50"
  }

  static_ip_pool {
    start_address = "192.168.2.51"
    end_address   = "192.168.2.100"
  }
}
```

## Argument Reference

The following arguments are supported:

* `org` - (Optional; *v2.0+*) The name of organization to use, optional if defined at provider level. Useful when 
  connected as sysadmin working across different organisations
* `vdc` - (Optional; *v2.0+*) The name of VDC to use, optional if defined at provider level
* `name` - (Required) A unique name for the network
* `netmask` - (Optional) The netmask for the new network. Defaults to `255.255.255.0`
* `gateway` (Required) The gateway for this network
* `dns1` - (Optional) First DNS server to use. Defaults to `8.8.8.8`
* `dns2` - (Optional) Second DNS server to use. Defaults to `8.8.4.4`
* `dns_suffix` - (Optional) A FQDN for the virtual machines on this network
* `shared` - (Optional) Defines if this network is shared between multiple vDCs
  in the vOrg.  Defaults to `false`.
* `dhcp_pool` - (Optional) A range of IPs to issue to virtual machines that don't
  have a static IP; see [IP Pools](#ip-pools) below for details.
* `static_ip_pool` - (Optional) A range of IPs permitted to be used as static IPs for
  virtual machines; see [IP Pools](#ip-pools) below for details.

<a id="ip-pools"></a>
## IP Pools

Static IP Pools and DHCP Pools support the following attributes:

* `start_address` - (Required) The first address in the IP Range
* `end_address` - (Required) The final address in the IP Range

DHCP Pools additionally support the following attributes:

* `default_lease_time` - (Optional) The default DHCP lease time to use. Defaults to `3600`.
* `max_lease_time` - (Optional) The maximum DHCP lease time to use. Defaults to `7200`.
