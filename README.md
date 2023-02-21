# DHCP Operator

This is intended to be a Kubernetes Operator that facilitates using the Claim pattern to acquire/release DHCP addresses on the host network.

**Note that this project is not yet functional! Any documentation below is at this point is a PURELY SPECULATIVE guide for project development.**

The goal is to have a hierarchy of NetworkClass / AddressClaim / Address that mirrors the semantics of StorageClass / PersistentVolumeClaim / PersistentVolume as closely as possible.


One would set up the network with a NetworkAttachmentDefinition and an associated NetworkClass describing DHCP client and options to use to acquire addresses:

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: network
---
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: iface
  namespace: network
spec:
  config: '{
      "cniVersion": "0.3.0",
      "type": "macvlan",
      "mode": "bridge",
      "master": "eno1"
    }'
---
apiVersion: my.dhcp.operator.api/v1alpha1
kind: NetworkClass
metadata:
  name: dhcp
  namespace: network
networkAttachment: iface
provisioner: udhcpc      # spin up a busybox udhcpc pod
# provisioner: cni-dhcp  # use the CNI IPAM DHCP plugin??
reclaimPolicy: Release # or "Retain"
addrBindingMode: Immediate # maybe allow WaitForFirstConsumer?
spec: # Default DHCP options
  provide:
    server-identifier: 192.168.1.1
    client-identifier: FromMacAddress
  request:
  - subnet-mask
  - router
  - domain-name-server
  - host-name
  - domain-name
```

You'd then set up your pod or deployment along the lines of:

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: service
---
apiVersion: my.dhcp.operator.api/v1alpha1
kind: AddressClaim
metadata:
  name: my-addr-claim
  namespace: service
spec:
  addrClassName: network/dhcp
  skipDefaults: false
  provide:
    host-name: example-host
    vendor-class: example-vendor
    81: <fqdn-info> # numeric option: client fully qualified domain name
  request: # request DHCP options by name
  - broadcast-addr
  - 249 # ntp servers (should also support numeric options)
---
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
  annotations:
    my.dhcp.operator.api/address-claim: my-addr-claim
spec:
  containers:
  - image: busybox
    name: example
    command: ["sleep", "infinity"] 
---
```

## How It Will Work Internally

### `NetworkClass`
Just a way to store the information... though, if you set up the network class to use the CNI IPAM DHCP plugin, it would start a plugin daemon instance.

### `AddressClaim`
Creating the claim will trigger the process of acquiring the address.

The basic algorithm will be something like:

- Find the associated `NetworkClass`
- aggregate together the specified provide and request options, etc.
- pick a MAC address for the claimant?
  - TODO: how to let this pod renew the lease under the same mac address as the pod attached with this?
- execute a daemon with the specified DHCP client software configured per spec
  the result might be something a bit like
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: dhcp-client-LONG-UUID-VALUE
    annotations:
      k8s.v1.cni.cncf.io/networks: network/iface
  spec:
    containers:
    - image: udhcpc
      name: udhcpc
      command: [ "udhcpc",
        "-i", "net1", # Interface to use
        "-B", "-R", "-f", "-a", # broadcast replies, release on exit, foreground, validate with ARP
        "-x", "host-name:example-host",
        "-V", "example-vendor",
        "-O", "subnet-mask",
        "-O", "router",
        ...
        "-O", "249",
      ]
  ```
- The pod will execute the client, and on acquiring the lease it'll call into a script to report it up to kubernetes:
  ```bash
  #!/bin/sh
  # /usr/share/udhcpc/default.script
  [ -n "$1" ] || { echo "Error: should be called from udhcpc"; exit 1; }
  case "$1" in
    bound)
      k8s_create_resource_Address
      ;;
    renew)
      k8s_update_resource_Address
      ;;
    deconfig)
      k8s_delete_resource_Address
      ;;
  esac
  exit 0
  ```

- This will create/update/delete the `Address` satisfying the `AddressClaim`, something like:
  ```yaml
  apiVersion: my.dhcp.operator.api/v1alpha1
  kind: Address
  metadata:
    name: adc-LONG-UUID-VALUE
  spec:
    claimRef: my-addr-claim
    daemonRef: dhcp-client-LONG-UUID-VALUE

    mac: "12:34:56:78:9a:bc"
    addr: 192.168.1.45
    gw: 192.168.1.1
    mask: 255.255.255.0
    host: example-host
    domain: example.com
    ntp: pool.ntp.org

    provided: # full list of the DHCP options last sent
      ...
    recieved:  # full list of the DHCP options last recieved
      lease-time: 2m
      renewal-time: 54s
      rebind-time: 1m39s
      ...
    
    cni-network: | # a JSON value insertable into k8s.v1.cni.cncf.io/networks
      {
        "name": "iface",
        "namespace": "network",
        "ips": ["192.168.1.45"],
        "mac": "12:34:56:78:9a:bc",
        "default-route": ["192.168.1.1"],
        "dns": ...
      }
  ```

## Pod Annotation `my.dhcp.operator.api/address`
A pod with this annotation will be processed as follows:

- Find the named `AddressClaim` and the `Address` satisfying it
- If there's a `k8s.v1.cni.cncf.io/networks` annotation in comma-delimited format, convert it to json list format.
- If there's no `k8s.v1.cni.cncf.io/networks`, create an empty json list.
- Add the `cni-network` information to that list.

All we're doing is instructing CNI to use the address we already just allocated.
See the [Kubernetes Network Custom Resource Definition De-facto Standard](https://raw.githubusercontent.com/k8snetworkplumbingwg/multi-net-spec/master/v1.2/%5Bv1.2%5D%20Kubernetes%20Network%20Custom%20Resource%20Definition%20De-facto%20Standard.pdf) for information on the format.
