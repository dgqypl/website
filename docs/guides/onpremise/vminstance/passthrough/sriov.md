---
sidebar_position: 3
---

# 网卡SR-IOV

介绍 SR-IOV 网卡透传与 OVS Offload 硬件卸载使用方式。

## SR-IOV 介绍

SR-IOV 技术是一种基于硬件的虚拟化解决方案，可提高性能和可伸缩性。SR-IOV 标准允许在虚拟机之间高效共享 PCIe（Peripheral Component Interconnect Express，快速外设组件互连）设备，并且它是在硬件中实现的，可以获得能够与本机性能媲美的 I/O 性能。SR-IOV 规范定义了新的标准，根据该标准，创建的新设备可允许将虚拟机直接连接到 I/O 设备。

SR-IOV 规范由 PCI-SIG 在 [http://www.pcisig.com](http://www.pcisig.com) 上进行定义和维护。

SR-IOV 中的两种新功能类型是：

- 物理功能 (Physical Function, PF)
用于支持 SR-IOV 功能的 PCI 功能，如 SR-IOV 规范中定义。PF 包含 SR-IOV 功能结构，用于管理 SR-IOV 功能。PF 是全功能的 PCIe 功能，可以像其他任何 PCIe 设备一样进行发现、管理和处理。PF 拥有完全配置资源，可以用于配置或控制 PCIe 设备。

- 虚拟功能 (Virtual Function, VF)
与物理功能关联的一种功能。VF 是一种轻量级 PCIe 功能，可以与物理功能以及与同一物理功能关联的其他 VF 共享一个或多个物理资源。VF 仅允许拥有用于其自身行为的配置资源。


### 宿主机开启 SR-IOV

除了常规的PCI/PCIe设备透传设置外，还需确保SR-IOV在宿主机BIOS启用，并且需要对设备的VF进行必要的内核设置，具体方法如下：

1. BIOS 中打开设备 SR-IOV 功能(根据机型和设备的型号不同，具体操作方式也可能不一样，具体参考实际机型配置)。

2. 在宿主机节点上打开 Virtual Functions

```bash
# 对于不同的网卡型号打开的方式也不一样，下面以 Intel I350 网卡为例，设置 7 个 Virtual Function(网卡支持的 virtual function 数量也是不一样的，需要根据具体网卡支持的数量来配置)
# 在 grub cmdline 中加上 igb.max_vfs=7
GRUB_CMDLINE_LINUX="...... igb.max_vfs=7"
# 重新生成 grub 引导配置文件
$ grub2-mkconfig -o /boot/grub2/grub.cfg
```

#### Mellanox 网卡配置

Mellanox 网卡需要安装 MLNX_OFED 驱动，参考其官方文档：

- [https://enterprise-support.nvidia.com/s/article/howto-install-mlnx-ofed-driver](https://enterprise-support.nvidia.com/s/article/howto-install-mlnx-ofed-driver)
- [https://docs.nvidia.com/networking/display/MLNXOFEDv461000/Installing+Mellanox+OFED](https://docs.nvidia.com/networking/display/MLNXOFEDv461000/Installing+Mellanox+OFED)
- [https://network.nvidia.com/products/infiniband-drivers/linux/mlnx_ofed/](https://network.nvidia.com/products/infiniband-drivers/linux/mlnx_ofed/)

```bash
# 注意要使用 gcc 高版本
$ yum install centos-release-scl
$ yum install -y devtoolset-9
$ scl enable devtoolset-9 bash
# 下载对应操作系统版本的 iso，注意区分 centos7.x
# 可在 nvidia 提供的驱动网站上下载: https://network.nvidia.com/products/infiniband-drivers/linux/mlnx_ofed/
# 可在 nvidia 提供的驱动网站上下载: https://network.nvidia.com/products/infiniband-drivers/linux/mlnx_ofed/
$ wget https://content.mellanox.com/ofed/MLNX_OFED-5.8-1.1.2.1/MLNX_OFED_LINUX-5.8-1.1.2.1-rhel7.9-x86_64.iso
$ mount -o loop MLNX_OFED_LINUX-5.8-1.1.2.1-rhel7.9-x86_64.iso /mnt
$ cd /mnt
# 等待编译
$ ./mlnxofedinstall --add-kernel-support
Restart needed for updates to take effect.
Log File: /tmp/ZXf5cnZzxo
Real log file: /tmp/MLNX_OFED_LINUX.62574.logs/fw_update.log
You may need to update your initramfs before next boot. To do that, run:
 
   dracut -f
To load the new driver, run:
/etc/init.d/openibd restart
 
$ dracut -f
$ /etc/init.d/openibd restart

$ mst start
# 配置SRIOV VF数量, 重启生效, 具体设备名称查看 mst status -v
$ mlxconfig -d /dev/mst/mt4119_pciconf0 set SRIOV_EN=1 NUM_OF_VFS=64

# mlnx infiniband 网卡开启 sriov 后的注意事项！！
# IB 网卡开启 sriov 后可能会导致 ip 命令如下报错：
Error: Buffer too small for object.
Dump terminated
需要使用 mlnx 的 iproute 包中的 ip 命令替换宿主机原本的 ip 命令，这个包在 ofed 驱动中, 如：
- /mnt/DEBS/mlnx-iproute2_6.4.0-1.2310055_amd64.deb # debian link
- /mnt/RPMS/mlnx-iproute2-5.19.0-1.58203.x86_64.rpm # centos

# 配置 VF 网卡名称 udev 规则，如果不配置会按照系统默认新增网卡的命名规则
cp /usr/share/doc/mlnx-ofa_kernel-5.8/vf-net-link-name.sh /etc/infiniband/
# 如果文件有冲突需要按需合并
cp /usr/share/doc/mlnx-ofa_kernel-5.8/82-net-setup-link.rules /etc/udev/rules.d/

$ reboot
```

5. 设置好后重启宿主机，查看 /proc/cmdline 确认配置生效。

### host-agent 启用 SR-IOV
登陆到对应的宿主机节点
```bash
# SR-IOV 默认是不启用的，需要修改 host-agent 配置
# 配置 SR-IOV 对应的网络 eg:
$ vi /etc/yunion/host.conf
networks:
- eth1/br0/192.168.100.111
- eth2/br1/bcast0
# networks包含了两种配置方式，第一个参数是物理网卡名称，第二个参数是网桥名称，
# 第三个参数有两种，一种是 ip 地址，另外一种是 wire，对于不想配置 ip地址的网卡可以使用 wire 属性，wire 代表的是这个网卡所在的二层网络。
sriov_nics:
- eth2
# 为 eth2 网卡打开 sriov

```

修改完成后重启 host-agent 服务: `kubectl rollout restart ds default-host`, 重启完成等待 host-agent 服务启动成功后可以在控制节点使用 climc 查看配置的 VF 网卡.

e.g.:

```bash
# 例如 I350 Ethernet Controller Virtual Function 就是我们本次配置的 VF 网卡。
$ climc isolated-device-list
+--------------------------------------+----------+-------------------------------------------+---------+------------------+--------------------------------------+--------------------------------------+
|                  ID                  | Dev_type |                   Model                   |  Addr   | Vendor_device_id |               Host_id                |               Guest_id               |
+--------------------------------------+----------+-------------------------------------------+---------+------------------+--------------------------------------+--------------------------------------+
| 37b2fe7b-f202-428e-8c34-35ccdac82852 | NIC      | ConnectX-5 Virtual Function               | 0a:01.1 | 15b3:1018        | c48b1b8b-28a3-453a-863f-ddf7c62bcfe2 |                                      |
| 9a0f2b3a-b372-4e37-840f-60427d026479 | NIC      | ConnectX-5 Virtual Function               | 0a:01.0 | 15b3:1018        | c48b1b8b-28a3-453a-863f-ddf7c62bcfe2 |                                      |
| 37b54017-4da3-44a6-8ffe-056ffc9e853b | NIC      | ConnectX-5 Virtual Function               | 0a:00.7 | 15b3:1018        | c48b1b8b-28a3-453a-863f-ddf7c62bcfe2 |                                      |
| 74558948-f5e9-453b-86d6-89c6bc00c710 | NIC      | ConnectX-5 Virtual Function               | 0a:00.6 | 15b3:1018        | c48b1b8b-28a3-453a-863f-ddf7c62bcfe2 |                                      |
| 75df24b3-c27a-49bb-8cf7-385c9f0d2385 | NIC      | ConnectX-5 Virtual Function               | 0a:00.5 | 15b3:1018        | c48b1b8b-28a3-453a-863f-ddf7c62bcfe2 |                                      |
| 17841618-c3e7-43f6-805a-23d2adbcd70c | NIC      | ConnectX-5 Virtual Function               | 0a:00.4 | 15b3:1018        | c48b1b8b-28a3-453a-863f-ddf7c62bcfe2 |                                      |
| 51627e5a-22af-40ee-8af5-dc1971bf89c9 | NIC      | ConnectX-5 Virtual Function               | 0a:00.3 | 15b3:1018        | c48b1b8b-28a3-453a-863f-ddf7c62bcfe2 |                                      |
| f794eba5-fbe7-4bcd-880a-4082523f5c94 | NIC      | ConnectX-5 Virtual Function               | 0a:00.2 | 15b3:1018        | c48b1b8b-28a3-453a-863f-ddf7c62bcfe2 |                                      |
| 10266e05-8d34-4594-81ca-64afbd365ee7 | NIC      | I350 Ethernet Controller Virtual Function | 03:13.0 | 8086:1520        | c48b1b8b-28a3-453a-863f-ddf7c62bcfe2 |                                      |
| a837b5ca-d3ee-4cd8-8de5-447b3ab274d0 | NIC      | I350 Ethernet Controller Virtual Function | 03:12.4 | 8086:1520        | c48b1b8b-28a3-453a-863f-ddf7c62bcfe2 |                                      |
| fab303c9-5dbb-4b85-8df9-fada299622f4 | NIC      | I350 Ethernet Controller Virtual Function | 03:12.0 | 8086:1520        | c48b1b8b-28a3-453a-863f-ddf7c62bcfe2 |                                      |
| f2cb8fd5-aa26-46a7-8dc2-2eca2b77a887 | NIC      | I350 Ethernet Controller Virtual Function | 03:11.4 | 8086:1520        | c48b1b8b-28a3-453a-863f-ddf7c62bcfe2 |                                      |
| b2933d5c-9a3a-45ff-8c80-15ac3291f5c9 | NIC      | I350 Ethernet Controller Virtual Function | 03:11.0 | 8086:1520        | c48b1b8b-28a3-453a-863f-ddf7c62bcfe2 |                                      |
| b22e6511-c70b-440c-8c05-6d1041ae7661 | NIC      | I350 Ethernet Controller Virtual Function | 03:10.4 | 8086:1520        | c48b1b8b-28a3-453a-863f-ddf7c62bcfe2 |                                      |
| f523f12a-82bf-45a3-80cf-94d5a9649665 | NIC      | I350 Ethernet Controller Virtual Function | 03:10.0 | 8086:1520        | c48b1b8b-28a3-453a-863f-ddf7c62bcfe2 |                                      |
+--------------------------------------+----------+-------------------------------------------+---------+------------------+--------------------------------------+--------------------------------------+
***  Total: 24 Pages: 2 Limit: 15 Offset: 0 Page: 1  ***

# 使用 VF 网卡创建虚机，netdesc中可以指定 sriov-nic-model 或者指定 sriov-nic-id
$ climc server-create --disk ae0de3e9-dd45-458e-83b2-4dbc48d70234 \
                      --net 'sriov-nic-model=I350 Ethernet Controller Virtual Function' \
                      --ncpu 2 --mem-spec 4G wyq-test-sriov-nic --auto-start

# 挂载 VF 网卡
$ climc server-attach-network c15a5a99-75ea-4b8c-8c7a-521e5f980db4 \
                              'sriov-nic-model=I350 Ethernet Controller Virtual Function:8056e77e-dc14-44b5-8ef9-e43aff344c0d'

```


## OVS Offload

OVS Offload 是部分厂商基于 SR-IOV 实现的一种OpenvSwitch数据转发硬件卸载的技术；如果硬件支持 OVS Offload, 则能够让 OVS 数据面卸载到网卡上，OVS 控制面不用做任何更改。

OVS Offload 的基础配置依赖于 SR-IOV 的配置，确保宿主机已经打开了 SR-IOV。然后需要安装开启 OVS Offload 必要的依赖，如 Mellanox 网卡需要安装 MLNX_OFED 驱动。

目前，Mellanox的Connext-6系列网卡支持OVS Offload。

### 安装 OFED驱动，配置 SRIOV VF
```
# 注意要使用 gcc 高版本
$ yum install centos-release-scl
$ yum install -y devtoolset-9
$ scl enable devtoolset-9 bash
# 下载对应操作系统版本的 iso，注意区分 centos7.x
# 可在 nvidia 提供的驱动网站上下载: https://network.nvidia.com/products/infiniband-drivers/linux/mlnx_ofed/
$ wget https://content.mellanox.com/ofed/MLNX_OFED-5.8-1.1.2.1/MLNX_OFED_LINUX-5.8-1.1.2.1-rhel7.9-x86_64.iso
$ mount -o loop MLNX_OFED_LINUX-5.8-1.1.2.1-rhel7.9-x86_64.iso /mnt
$ cd /mnt
# 等待编译
$ ./mlnxofedinstall --add-kernel-support
Restart needed for updates to take effect.
Log File: /tmp/ZXf5cnZzxo
Real log file: /tmp/MLNX_OFED_LINUX.62574.logs/fw_update.log
You may need to update your initramfs before next boot. To do that, run:
 
   dracut -f
To load the new driver, run:
/etc/init.d/openibd restart
 
$ dracut -f
$ /etc/init.d/openibd restart

$ mst start
# 配置SRIOV VF数量, 重启生效, 具体设备名称查看 mst status -v
$ mlxconfig -d /dev/mst/mt4119_pciconf0 set SRIOV_EN=1 NUM_OF_VFS=64
$ reboot
```
Mellanox 网卡安装 MLNX_OFED 驱动参考：
- [https://enterprise-support.nvidia.com/s/article/howto-install-mlnx-ofed-driver](https://enterprise-support.nvidia.com/s/article/howto-install-mlnx-ofed-driver)
- [https://docs.nvidia.com/networking/display/MLNXOFEDv461000/Installing+Mellanox+OFED](https://docs.nvidia.com/networking/display/MLNXOFEDv461000/Installing+Mellanox+OFED)
- [https://network.nvidia.com/products/infiniband-drivers/linux/mlnx_ofed/](https://network.nvidia.com/products/infiniband-drivers/linux/mlnx_ofed/)

Mellanox 配置参考文档：

[https://docs.nvidia.com/networking/display/MLNXOFEDv471001/OVS+Offload+Using+ASAP2+Direct#OVSOffloadUsingASAP2Direct-Overview](https://docs.nvidia.com/networking/display/MLNXOFEDv471001/OVS+Offload+Using+ASAP2+Direct#OVSOffloadUsingASAP2Direct-Overview)

### host-agent 启用 SR-IOV

安装好 OFED驱动后配置平台启用 offload 网卡

#### Mellanox CX6 以下(CX5/CX4) 关闭安全组

Mellanox CX6 以下 ovs offload 对 connection tracking 支持的不好，所以需要关闭安全组：

```bash
$ kubectl -n onecloud edit cm default-host
disable_security_group: true # 改为 true
```

### 配置 host.conf

登陆到对应的宿主机节点

```bash
# SR-IOV 默认是不启用的，需要修改 host-agent 配置
# 配置 SR-IOV 对应的网络 eg:
$ vi /etc/yunion/host.conf
networks:
- eth1/br0/192.168.100.111
- eth2/br1/bcast0
# networks包含了两种配置方式，第一个参数是物理网卡名称，第二个参数是网桥名称，
# 第三个参数有两种，一种是 ip 地址，另外一种是 wire，对于不想配置 ip地址的网卡可以使用 wire 属性，wire 代表的是这个网卡所在的二层网络。
ovs_offload_nics:
- eth2
# 为 eth2 网卡打开 ovs offload
```

修改完成后重启 host-agent 服务: `kubectl rollout restart ds default-host`,在使用上与 SR-IOV 一致。

### offload 网卡配置 bond

Mellanox 网卡支持的 bond 模式为:
- Active-Backup (mode 1)
- XOR           (mode 2)
- LACP          (mode 4)

在 offload 模式下，来自两个 PF 的数据包可以转发到任何一个VF，来自 VF 的流量可以根据 bond 状态转发到两个端口。
这意味着，在 Active-Backup 模式下，只要有一个 PF active，来自任何 VF 的流量都可以通过这个 PF 发送。
在 XOR 或 LACP 模式下，如果两个 PF 都 active，来自 VF 的流量将在这两个 PF 之间分配。

具体Bond配置方法请参考其他文档。

网卡配置好 bond 后修改 host.conf 配置文件：
```bash
$ vi /etc/yunion/host.conf
networks:
- bond0/br1/bcast0 # bond0 为bond口名称
......
ovs_offload_nics:
- bond0
```

## 虚拟机配置

SR-IOV透传到虚拟机后，对虚拟机表现为同型号的物理网卡，对于Mellanox的网卡，也建议安装官方的OFED驱动。

```
$ wget https://content.mellanox.com/ofed/MLNX_OFED-5.8-1.1.2.1/MLNX_OFED_LINUX-5.8-1.1.2.1-rhel7.9-x86_64.iso
$ mount -o loop MLNX_OFED_LINUX-5.8-1.1.2.1-rhel7.9-x86_64.iso /mnt
$ cd /mnt
$ ./mlnxofedinstall
$ dracut -f
$ /etc/init.d/openibd restart
$ reboot
```

## 常见问题

#### 如何验证一个PCI设备支持SR-IOV？

执行命令如下命令，查看设备的 capabilities 中是否包含了 SR-IOV：

```bash
# lspci -v -s 01:00.0
01:00.0 Ethernet controller: Mellanox Technologies MT2894 Family [ConnectX-6 Lx]
	Subsystem: Mellanox Technologies Device 0001
	Physical Slot: 4
	Flags: bus master, fast devsel, latency 0, IRQ 60, NUMA node 0
	Memory at ea000000 (64-bit, prefetchable) [size=32M]
	Expansion ROM at f0300000 [disabled] [size=1M]
	Capabilities: [60] Express Endpoint, MSI 00
	Capabilities: [48] Vital Product Data
	Capabilities: [9c] MSI-X: Enable+ Count=64 Masked-
	Capabilities: [c0] Vendor Specific Information: Len=18 <?>
	Capabilities: [40] Power Management version 3
	Capabilities: [100] Advanced Error Reporting
	Capabilities: [150] Alternative Routing-ID Interpretation (ARI)
	Capabilities: [180] Single Root I/O Virtualization (SR-IOV)
	Capabilities: [1c0] #19
	Capabilities: [230] Access Control Services
	Capabilities: [320] #27
	Capabilities: [370] #26
	Capabilities: [420] #25
	Kernel driver in use: mlx5_core
	Kernel modules: mlx5_core
```

#### 如何验证OVS Offload是否生效

首先，确认OpenvSwitch在控制面已经打开硬件卸载特性：

```bash
$ ovs-vsctl list Open_vSwitch . | grep other_config
## 输出内容需要包含hw-offload="true"
other_config        : {hw-offload="true"}
````

其次，执行  `ovs-appctl dpctl/dump-flows type=offloaded` 查看卸载到硬件转发表的流表，如果硬件卸载未生效，则输出为空。


## 网卡 SR-IOV 常见问题：

如果按照以上配置后重启宿主机 SR-IOV 网卡未正常上报，需要查看 host-agent 日志和宿主机 dmesg 进一步排查。
以下是一些常遇到的问题：

1. 检查宿主机 IOMMU 是否打开
```
$ cat /proc/cmdline
查看是否包含如下选项
intel_iommu=on iommu=pt

运行查看是否有输出，如果打开了 iommu 应该会有输出
$ dmesg | grep -e DMAR -e IOMMU -e AMD-Vi
```


2. dmesg: not enough MMIO resources for SR-IOV
```
$ /etc/default/grub cmdline 中添加 pci=realloc

$ grub2-mkconfig -o /boot/grub2/grub.cfg
$ 重启
```

3. dmesg: ENABLE_HCA(0x104) timeout
BIOS 引导系统中查找这些字样 "ACS", "PCIe Ari Support" and "Iommu" 并打开。




