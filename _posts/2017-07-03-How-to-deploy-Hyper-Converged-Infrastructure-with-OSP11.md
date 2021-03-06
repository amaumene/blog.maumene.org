---
layout: post
title: How to deploy Hyper Converged Infrastructure with OSP11
---

[Composable services](https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/11/html/advanced_overcloud_customization/roles) is a new feature that was released with OSP10. It allows you to add more roles than the pre-defined one. In this example I will deploy a new role named hci ([Hyper Converged Infrastructure](https://en.wikipedia.org/wiki/Hyper-converged_infrastructure), ie Ceph OSD will be running on compute nodes).

This example is for OSP11 but appart from the nic-configs files that need to be adapt, the rest should be working with 10 without an issue.

<!-- TOC -->

- [Network Interface Template configuration](#network-interface-template-configuration)
- [Network Interface Template definition](#network-interface-template-definition)
- [Ports isolation definition](#ports-isolation-definition)
- [Define the new role](#define-the-new-role)
- [Deployment](#deployment)
- [Download my templates](#download-my-templates)

<!-- /TOC -->

## Network Interface Template configuration

First you will need to create a nic-config file describing the new role. I just took the compute.yaml one and added the StorageMgmtIpSubnet. This is needed because the new role will have to replicate the data between OSD:
```
heat_template_version: ocata
description: >
  Software Config to drive os-net-config to configure multiple interfaces for the controller role.
parameters:
  ControlPlaneIp:
    default: ''
    description: IP address/subnet on the ctlplane network
    type: string
  ExternalIpSubnet:
    default: ''
    description: IP address/subnet on the external network
    type: string
  InternalApiIpSubnet:
    default: ''
    description: IP address/subnet on the internal API network
    type: string
  StorageIpSubnet:
    default: ''
    description: IP address/subnet on the storage network
    type: string
  StorageMgmtIpSubnet:
    default: ''
    description: IP address/subnet on the storage mgmt network
    type: string
  TenantIpSubnet:
    default: ''
    description: IP address/subnet on the tenant network
    type: string
  ManagementIpSubnet: # Only populated when including environments/network-management.yaml
    default: ''
    description: IP address/subnet on the management network
    type: string
  ExternalNetworkVlanID:
    default: 10
    description: Vlan ID for the external network traffic.
    type: number
  InternalApiNetworkVlanID:
    default: 20
    description: Vlan ID for the internal_api network traffic.
    type: number
  StorageNetworkVlanID:
    default: 30
    description: Vlan ID for the storage network traffic.
    type: number
  StorageMgmtNetworkVlanID:
    default: 40
    description: Vlan ID for the storage mgmt network traffic.
    type: number
  TenantNetworkVlanID:
    default: 50
    description: Vlan ID for the tenant network traffic.
    type: number
  ManagementNetworkVlanID:
    default: 60
    description: Vlan ID for the management network traffic.
    type: number
  ControlPlaneSubnetCidr: # Override this via parameter_defaults
    default: '24'
    description: The subnet CIDR of the control plane network.
    type: string
  ControlPlaneDefaultRoute: # Override this via parameter_defaults
    description: The default route of the control plane network.
    type: string
  ExternalInterfaceDefaultRoute:
    default: 10.0.0.1
    description: default route for the external network
    type: string
  ManagementInterfaceDefaultRoute: # Commented out by default in this template
    default: unset
    description: The default route of the management network.
    type: string
  DnsServers: # Override this via parameter_defaults
    default: []
    description: A list of DNS servers (2 max for some implementations) that will be added to resolv.conf.
    type: comma_delimited_list
  EC2MetadataIp: # Override this via parameter_defaults
    description: The IP address of the EC2 metadata server.
    type: string
resources:
  OsNetConfigImpl:
    type: OS::Heat::SoftwareConfig
    properties:
      group: script
      config:
        str_replace:
          template:
            get_file: /home/stack/templates/network/scripts/run-os-net-config.sh
          params:
            $network_config:
              network_config:
              - type: interface
                name: nic1
                use_dhcp: false
                dns_servers:
                  get_param: DnsServers
                addresses:
                - ip_netmask:
                    list_join:
                    - /
                    - - get_param: ControlPlaneIp
                      - get_param: ControlPlaneSubnetCidr
                routes:
                - ip_netmask: 169.254.169.254/32
                  next_hop:
                    get_param: EC2MetadataIp
                - default: true
                  next_hop:
                    get_param: ControlPlaneDefaultRoute
              - type: interface
                name: nic2
                use_dhcp: false
                addresses:
                - ip_netmask:
                    get_param: StorageIpSubnet
              - type: interface
                name: nic3
                use_dhcp: false
                addresses:
                - ip_netmask:
                    get_param: StorageMgmtIpSubnet
              - type: interface
                name: nic4
                use_dhcp: false
                addresses:
                - ip_netmask:
                    get_param: InternalApiIpSubnet
              - type: ovs_bridge
                name: br-tenant
                use_dhcp: false
                addresses:
                - ip_netmask:
                    get_param: TenantIpSubnet
                members:
                - type: interface
                  name: nic5
                  use_dhcp: false
                  primary: true
            # Uncomment when including environments/network-management.yaml
            # If setting default route on the Management interface, comment
            # out the default route on the External interface. This will
            # make the External API unreachable from remote subnets.
            #-
            #  type: interface
            #  name: nic7
            #  use_dhcp: false
            #  addresses:
            #    -
            #      ip_netmask: {get_param: ManagementIpSubnet}
            #  routes:
            #    -
            #      default: true
            #      next_hop: {get_param: ManagementInterfaceDefaultRoute}
outputs:
  OS::stack_id:
    description: The OsNetConfigImpl resource.
    value:
      get_resource: OsNetConfigImpl
```

## Network Interface Template definition

The second thing is to configure your environment to use the nic-config file:
```
resource_registry:
  # Network Interface templates to use (these files must exist)
  OS::TripleO::Hci::Net::SoftwareConfig:
    /home/stack/deployment/nic-configs/hci.yaml
```

## Ports isolation definition

The last thing to do is to define the ports for this new role. Again I've just replaced compute by hci and added a port for the ceph replication network (StorageMgmtPort).
```
# Enable the creation of Neutron networks for isolated Overcloud
# traffic and configure each role to assign ports (related
# to that role) on these networks.
resource_registry:
  OS::TripleO::Network::External: ../templates/network/external.yaml
  OS::TripleO::Network::InternalApi: ../templates/network/internal_api.yaml
  OS::TripleO::Network::StorageMgmt: ../templates/network/storage_mgmt.yaml
  OS::TripleO::Network::Storage: ../templates/network/storage.yaml
  OS::TripleO::Network::Tenant: ../templates/network/tenant.yaml

  # Port assignments for the VIPs
  OS::TripleO::Network::Ports::ExternalVipPort: ../templates/network/ports/external.yaml
  OS::TripleO::Network::Ports::InternalApiVipPort: ../templates/network/ports/internal_api.yaml
  OS::TripleO::Network::Ports::StorageVipPort: ../templates/network/ports/storage.yaml
  OS::TripleO::Network::Ports::StorageMgmtVipPort: ../templates/network/ports/storage_mgmt.yaml
  OS::TripleO::Network::Ports::RedisVipPort: ../templates/network/ports/vip.yaml

  # Port assignments for the controller role
  OS::TripleO::Controller::Ports::ExternalPort: ../templates/network/ports/external.yaml
  OS::TripleO::Controller::Ports::InternalApiPort: ../templates/network/ports/internal_api.yaml
  OS::TripleO::Controller::Ports::StoragePort: ../templates/network/ports/storage.yaml
  OS::TripleO::Controller::Ports::StorageMgmtPort: ../templates/network/ports/storage_mgmt.yaml
  OS::TripleO::Controller::Ports::TenantPort: ../templates/network/ports/tenant.yaml

  # Port assignments for the hci role
  OS::TripleO::Hci::Ports::ExternalPort: ../templates/network/ports/noop.yaml
  OS::TripleO::Hci::Ports::InternalApiPort: ../templates/network/ports/internal_api.yaml
  OS::TripleO::Hci::Ports::StoragePort: ../templates/network/ports/storage.yaml
  OS::TripleO::Hci::Ports::StorageMgmtPort: ../templates/network/ports/storage_mgmt.yaml
  OS::TripleO::Hci::Ports::TenantPort: ../templates/network/ports/tenant.yaml
```

## Define the new role

So far we have only created the base files for a new role to be deployed with a correct network configuration. The next thing is to actually define what this new role will do:

Once again, I've just added CephOSD service to the compute role (and rename it). The diff with the default roles_data.yaml will look like that:
```
< - name: Compute
---
> - name: Hci
135d134
<   HostnameFormatDefault: '%stackname%-novacompute-%index%'
140a140
>     - OS::TripleO::Services::CephOSD
```

## Deployment
Just run you openstack overcloud deploy command with -r to use your new_roles_data.yaml file:
```
openstack overcloud deploy --templates /home/stack/templates \
  -r $(pwd)/roles_data_hci.yaml \
  -e $(pwd)/network-isolation.yaml \
  -e $(pwd)/storage-environment.yaml \
  -e $(pwd)/network-environment.yaml \
  --ntp-server pool.ntp.org \
  --timeout 180
```
  
## Download my templates
As always you can download the templates of my example [here](https://blog.maumene.org/files/hci-osp11.tar.gz).
