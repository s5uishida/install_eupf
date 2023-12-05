# Install eUPF(eBPF/XDP UPF) on Host
This briefly describes the steps and configuration to build and install [eUPF](https://github.com/edgecomllc/eupf).
There are installation instructions in the eUPF repository, but I would like to write down the steps for actually installing it.
**It is intended to be prepared for use with [Open5GS](https://github.com/open5gs/open5gs) and [free5GC](https://github.com/free5gc/free5gc).**

---

### [Sample Configurations and Miscellaneous for Mobile Network](https://github.com/s5uishida/sample_config_misc_for_mobile_network)

---

<a id="toc"></a>

## Table of Contents

- [Simple Overview of eUPF and Data Network Gateway](#overview)
- [Build eUPF on VM-UP](#build)
  - [Install required packages](#install_packages)
  - [Install Golang and Setting](#install_golang)
  - [Install the Swag command line tool for Golang](#install_swag)
  - [Clone eUPF](#clone)
  - [Run the code generators](#generate_codes)
  - [Build eUPF](#build_1)
- [Setup eUPF on VM-UP](#setup_up)
  - [Create configuration file](#conf)
- [Run eUPF on VM-UP](#run)
- [Setup Data Network Gateway on VM-DN](#setup_dn)
- [Sample Configurations](#sample_conf)
  - [For 5G](#5g_conf)
  - [For 4G](#4g_conf)
- [Changelog (summary)](#changelog)

---

<a id="overview"></a>

## Simple Overview of eUPF and Data Network Gateway

This describes a simple configuration of eUPF and Data Network Gateway, focusing on U-Plane.
**Note that this configuration is implemented with Virtualbox VMs.**

The following minimum configuration was set as a condition.
- One UPF and Data Network Gateway

The built simulation environment is as follows.

<img src="./images/network-overview.png" title="./images/network-overview.png" width=800px></img>

The eBPF/XDP UPF used is as follows.
- eBPF/XDP UPF - eUPF v0.6.0 (2023.12.04) - https://github.com/edgecomllc/eupf

Each VMs are as follows.  
| VM | SW & Role | IP address | OS | CPU<br>(Min) | Memory<br>(Min) | HDD<br>(Min) |
| --- | --- | --- | --- | --- | --- | --- |
| VM-UP | eUPF U-Plane | 192.168.0.151/24 | Ubuntu 22.04 | 1 | 2GB | 20GB |
| VM-DN | Data Network Gateway  | 192.168.0.152/24 | Ubuntu 22.04 | 1 | 1GB | 10GB |

The network interfaces of each VM are as follows.
| VM | Device | Network Adapter | IP address | Interface | XDP |
| --- | --- | --- | --- | --- | --- |
| VM-UP | ~~enp0s3~~ | ~~NAT(default)~~ | ~~10.0.2.15/24~~ | ~~(VM default NW)~~ ***down*** | -- |
| | enp0s8 | Bridged Adapter | 192.168.0.151/24 | (Mgmt NW) | -- |
| | enp0s9 | NAT Network | 192.168.13.151/24 | N3 | x |
| | enp0s10 | NAT Network | 192.168.14.151/24 | N4 | -- |
| | enp0s16 | NAT Network | 192.168.16.151/24 | N6 | x |
| VM-DN | enp0s3 | NAT(default) | 10.0.2.15/24 | (VM default NW) | -- |
| | enp0s8 | Bridged Adapter | 192.168.0.152/24 | (Mgmt NW) | -- |
| | enp0s9 | NAT Network | 192.168.16.152/24 | N6, ***default GW for VM-UP*** | -- |

NAT networks of Virtualbox  are as follows.
| Network Name | Network CIDR |
| --- | --- |
| N3 | 192.168.13.0/24 |
| N4 | 192.168.14.0/24 |
| N6 | 192.168.16.0/24 |

**Note. Virtualbox GUI tool can only register up to 4 Network Adapters in one VM.
Since 5 Network Adapters are registered in VM-UP, one cannot be registered with the GUI tool.
In this case, directly edit the vbox file as follows and register the remaining Network Adapter.**

**For example)**
```diff
--- eupf10.vbox.orig    2023-10-28 21:23:38.808738418 +0900
+++ eupf10.vbox 2023-10-29 06:41:01.157520495 +0900
@@ -68,7 +68,12 @@
           </DisabledModes>
           <NATNetwork name="N4"/>
         </Adapter>
-        <Adapter slot="8" MACAddress="080027FE0AA2" cable="false"/>
+        <Adapter slot="8" enabled="true" MACAddress="080027FE0AA2" type="82540EM">
+          <DisabledModes>
+            <InternalNetwork name="intnet"/>
+          </DisabledModes>
+          <NATNetwork name="N6"/>
+        </Adapter>
         <Adapter slot="9" MACAddress="080027990637" cable="false"/>
         <Adapter slot="10" MACAddress="080027AC33FC" cable="false"/>
         <Adapter slot="11" MACAddress="08002748BC3D" cable="false"/>
```

<a id="build"></a>

## Build eUPF on VM-UP

Please refer to the following for building eUPF.
- eUPF v0.6.0 (2023.12.04) - https://github.com/edgecomllc/eupf

<a id="install_packages"></a>

### Install required packages

```
# apt install git clang llvm gcc-multilib libbpf-dev
```

<a id="install_golang"></a>

### Install Golang and Setting

```
# wget https://go.dev/dl/go1.20.10.linux-amd64.tar.gz
# tar -C /usr/local -zxvf go1.20.10.linux-amd64.tar.gz
# mkdir -p ~/go/{bin,pkg,src}
# echo 'export GOPATH=$HOME/go' >> ~/.bashrc
# echo 'export GOROOT=/usr/local/go' >> ~/.bashrc
# echo 'export PATH=$PATH:$GOPATH/bin:$GOROOT/bin' >> ~/.bashrc
# echo 'export GO111MODULE=auto' >> ~/.bashrc
# source ~/.bashrc
```

<a id="install_swag"></a>

### Install the Swag command line tool for Golang

```
# go install github.com/swaggo/swag/cmd/swag@v1.8.12
```

<a id="clone"></a>

### Clone eUPF

```
# git clone https://github.com/edgecomllc/eupf.git
# cd eupf
```

<a id="generate_codes"></a>

### Run the code generators

```
# go generate -v ./cmd/...
```
If you want to output kernel logs for debugging, add the following option.
```
# BPF_CFLAGS="-DENABLE_LOG" go generate -v ./cmd/...
```
In that case, to see debug log from eBPF programs:
```
# cat /sys/kernel/debug/tracing/trace_pipe
```

<a id="build_1"></a>

### Build eUPF

```
# go build -v -o bin/eupf ./cmd/
```

<a id="setup_up"></a>

## Setup eUPF on VM-UP

Please refer to the following for setup eUPF.
- eUPF v0.6.0 (2023.12.04) - https://github.com/edgecomllc/eupf/blob/main/docs/Configuration.md

First, uncomment the next line in the `/etc/sysctl.conf` file and reflect it in the OS.
```
net.ipv4.ip_forward=1
```
```
# sysctl -p
```
Next, down the default interface`enp0s3` of the VM-UP and set the VM-DN IP address to default GW on the N6 interface`enp0s16`.
```
# ip link set dev enp0s3 down
# ip route add default via 192.168.16.152 dev enp0s16
```

<a id="conf"></a>

### Create configuration file

Create `/root/eupf` directory and put the configuration file there.

- `/root/eupf/config.yml`

```yaml
interface_name: [enp0s9, enp0s16]
xdp_attach_mode: generic
api_address: :8080
pfcp_address: 192.168.14.151:8805
pfcp_node_id: 192.168.14.151
metrics_address: :9090
n3_address: 192.168.13.151
gtp_peer:
echo_interval: 10
qer_map_size: 1024
far_map_size: 1024
pdr_map_size: 1024
resize_ebpf_maps: false
heartbeat_retries: 3
heartbeat_interval: 5
heartbeat_timeout: 5
logging_level: info
feature_ftup: true
teid_pool: 65536
```
If `xdp_attach_mode` is `generic`(Kernel-level implementation), the performance will not be improved.
`native`(Driver-level implementation) or `offload`(NIC-level implementation) will be better.
For reference, a list of drivers that support XDP can be found [here](https://github.com/iovisor/bcc/blob/master/docs/kernel-versions.md#xdp).

<a id="run"></a>

## Run eUPF on VM-UP

```
# cd /root/eupf
# bin/eupf --config config.yml
2023/12/04 20:51:49 Get raw config: map[api_address::8080 echo_interval:10 far_map_size:1024 feature_ftup:true gtp_peer:[] heartbeat_interval:5 heartbeat_retries:3 heartbeat_timeout:5 interface_name:[enp0s9 enp0s16] logging_level:info metrics_address::9090 n3_address:192.168.13.151 pdr_map_size:1024 pfcp_address:192.168.14.151:8805 pfcp_node_id:192.168.14.151 qer_map_size:1024 resize_ebpf_maps:false teid_pool:65536 xdp_attach_mode:generic]
2023/12/04 20:51:49 Apply eUPF config: {InterfaceName:[enp0s9 enp0s16] XDPAttachMode:generic ApiAddress::8080 PfcpAddress:192.168.14.151:8805 PfcpNodeId:192.168.14.151 MetricsAddress::9090 N3Address:192.168.13.151 GtpPeer:[] EchoInterval:10 QerMapSize:1024 FarMapSize:1024 PdrMapSize:1024 EbpfMapResize:false HeartbeatRetries:3 HeartbeatInterval:5 HeartbeatTimeout:5 LoggingLevel:info FTEIDPool:65536 FeatureFTUP:true}
2023/12/04 20:51:50 INF Attached XDP program to iface "enp0s9" (index 2)
2023/12/04 20:51:50 INF Attached XDP program to iface "enp0s16" (index 3)
2023/12/04 20:51:50 INF Starting PFCP connection: 192.168.14.151:8805 with Node ID: 192.168.14.151 and N3 address: 192.168.13.151
[GIN-debug] [WARNING] Creating an Engine instance with the Logger and Recovery middleware already attached.

[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
 - using env:   export GIN_MODE=release
 - using code:  gin.SetMode(gin.ReleaseMode)

[GIN-debug] GET    /api/v1/health            --> github.com/edgecomllc/eupf/cmd/api/rest.(*ApiHandler).InitRoutes.func1 (4 handlers)
[GIN-debug] GET    /api/v1/upf_pipeline      --> github.com/edgecomllc/eupf/cmd/api/rest.(*ApiHandler).listUpfPipeline-fm (4 handlers)
[GIN-debug] GET    /api/v1/xdp_stats         --> github.com/edgecomllc/eupf/cmd/api/rest.(*ApiHandler).displayXdpStatistics-fm (4 handlers)
[GIN-debug] GET    /api/v1/packet_stats      --> github.com/edgecomllc/eupf/cmd/api/rest.(*ApiHandler).displayPacketStats-fm (4 handlers)
[GIN-debug] GET    /api/v1/config            --> github.com/edgecomllc/eupf/cmd/api/rest.(*ApiHandler).displayConfig-fm (4 handlers)
[GIN-debug] POST   /api/v1/config            --> github.com/edgecomllc/eupf/cmd/api/rest.(*ApiHandler).editConfig-fm (4 handlers)
[GIN-debug] GET    /api/v1/uplink_pdr_map/:id --> github.com/edgecomllc/eupf/cmd/api/rest.(*ApiHandler).getUplinkPdrValue-fm (4 handlers)
[GIN-debug] PUT    /api/v1/uplink_pdr_map/:id --> github.com/edgecomllc/eupf/cmd/api/rest.(*ApiHandler).setUplinkPdrValue-fm (4 handlers)
[GIN-debug] GET    /api/v1/qer_map           --> github.com/edgecomllc/eupf/cmd/api/rest.(*ApiHandler).listQerMapContent-fm (4 handlers)
[GIN-debug] GET    /api/v1/qer_map/:id       --> github.com/edgecomllc/eupf/cmd/api/rest.(*ApiHandler).getQerValue-fm (4 handlers)
[GIN-debug] PUT    /api/v1/qer_map/:id       --> github.com/edgecomllc/eupf/cmd/api/rest.(*ApiHandler).setQerValue-fm (4 handlers)
[GIN-debug] GET    /api/v1/far_map/:id       --> github.com/edgecomllc/eupf/cmd/api/rest.(*ApiHandler).getFarValue-fm (4 handlers)
[GIN-debug] PUT    /api/v1/far_map/:id       --> github.com/edgecomllc/eupf/cmd/api/rest.(*ApiHandler).setFarValue-fm (4 handlers)
[GIN-debug] GET    /api/v1/pfcp_associations --> github.com/edgecomllc/eupf/cmd/api/rest.(*ApiHandler).listPfcpAssociations-fm (4 handlers)
[GIN-debug] GET    /api/v1/pfcp_associations/full --> github.com/edgecomllc/eupf/cmd/api/rest.(*ApiHandler).listPfcpAssociationsFull-fm (4 handlers)
[GIN-debug] GET    /api/v1/pfcp_sessions     --> github.com/edgecomllc/eupf/cmd/api/rest.(*ApiHandler).listPfcpSessionsFiltered-fm (4 handlers)
[GIN-debug] GET    /swagger/*any             --> github.com/swaggo/gin-swagger.CustomWrapHandler.func1 (4 handlers)
[GIN-debug] [WARNING] Creating an Engine instance with the Logger and Recovery middleware already attached.

[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
 - using env:   export GIN_MODE=release
 - using code:  gin.SetMode(gin.ReleaseMode)

[GIN-debug] GET    /metrics                  --> github.com/edgecomllc/eupf/cmd/api/rest.(*ApiHandler).InitMetricsRoute.func1.1 (4 handlers)
2023/12/04 20:51:50 INF running on :8080
2023/12/04 20:51:50 INF running on :9090
```

<a id="setup_dn"></a>

## Setup Data Network Gateway on VM-DN

First, uncomment the next line in the `/etc/sysctl.conf` file and reflect it in the OS.
```
net.ipv4.ip_forward=1
```
```
# sysctl -p
```
Next, configure NAPT and routing to N6 IP address of eUPF.
```
# iptables -t nat -A POSTROUTING -s <DN> -j MASQUERADE
# ip route add <DN> via 192.168.16.151 dev enp0s9
```
**Note. Set `<DN>` according to the core network.  
ex) `10.45.0.0/16`**

---
With the above steps, eUPF(eBPF/XDP UPF) has been constructed.
You will be able to work eUPF with Open5GS and free5GC.
I would like to thank the excellent developers and all the contributors of eUPF.

<a id="sample_conf"></a>

## Sample Configurations

<a id="5g_conf"></a>

### For 5G

- [Open5GS 5GC & UERANSIM UE / RAN Sample Configuration - eUPF(eBPF/XDP UPF)](https://github.com/s5uishida/open5gs_5gc_ueransim_eupf_sample_config)
- [free5GC 5GC & UERANSIM UE / RAN Sample Configuration - eUPF(eBPF/XDP UPF)](https://github.com/s5uishida/free5gc_ueransim_eupf_sample_config)

<a id="4g_conf"></a>

### For 4G

- [Open5GS EPC & srsRAN 4G with ZeroMQ UE / RAN Sample Configuration - eUPF(eBPF/XDP UPF(PGW-U))](https://github.com/s5uishida/open5gs_epc_srsran_eupf_sample_config)

<a id="changelog"></a>

## Changelog (summary)

- [2023.12.05] The version confirmed to work in the changelog of 2023.12.04 has been tagged as `v0.6.0`.
- [2023.12.04] Updated as FTUP feature has been merged into `main` branch.
- [2023.11.24] Updated to `120-upf-ftup-fteid` branch that supports FTUP.
- [2023.10.29] Initial release.
