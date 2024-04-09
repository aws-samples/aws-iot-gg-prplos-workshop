
# Lab One: Intro to prplOS LCM

## Step 1 - Setup image

1.1 Download prplOS image from the official releases

```bash
wget \
    https://gitlab.com/api/v4/projects/20776235/packages/generic/x86-64/3.0.1/openwrt-x86-64-generic-squashfs-combined-efi.img
```

1.2 Increase base image size to 4G:

```bash
qemu-img resize -f raw openwrt-x86-64-generic-squashfs-combined-efi.img 4G
# Image resized.
```

1.3 Loop mount the base image so it can be modified

```bash

loop_device=$(losetup -f)
#
sudo losetup $loop_device openwrt-x86-64-generic-squashfs-combined-efi.img
#
```

1.4 Fix the GPT partition and increase the root partition size to 100% (1024 MiB):

```bash
echo -e "OK\nFix" | sudo parted ---pretend-input-tty "$loop_device" print
# Error: The backup GPT table is corrupt, but the primary appears OK, so that will be used.
# OK/Cancel? OK                                                             
# Warning: Not all of the space available to /dev/loop6 appears to be used, you can fix the GPT to use all of the space (an extra 1797599 blocks) or continue with the current setting? 
# Fix/Ignore? Fix                                                           
# Model: Loopback device (loopback)
# Disk /dev/loop6: 1074MB
# Sector size (logical/physical): 512B/512B
# Partition Table: gpt
# Disk Flags: 

# Number  Start   End     Size    File system  Name  Flags
# 128     17.4kB  262kB   245kB                      bios_grub
#  1      262kB   17.0MB  16.8MB  fat16              legacy_boot
#  2      17.0MB  153MB   136MB


sudo parted "$loop_device" resizepart 2 100%
# Information: You may need to update /etc/fstab.
sudo parted "$loop_device" print
# #Model: Loopback device (loopback)
# Disk /dev/loop6: 1074MB
# Sector size (logical/physical): 512B/512B
# Partition Table: gpt
# Disk Flags: 

# Number  Start   End     Size    File system  Name  Flags
# 128     17.4kB  262kB   245kB                      bios_grub
#  1      262kB   17.0MB  16.8MB  fat16              legacy_boot
#  2      17.0MB  1074MB  1057MB

```

1.5 Remove the loop mount device
```bash
sudo losetup -d $loop_device
#
```

## Step 2 -  Spin image using Qemu

2.1 Start image using `qemu-system-x86_64`

```bash
qemu-system-x86_64 \
    -accel kvm \
    -nographic \
    -smp 8 \
    -M q35 \
    -m 2048 \
    -drive file=openwrt-x86-64-generic-squashfs-combined-efi.img,id=d0,if=none,format=raw \
    -device ide-hd,drive=d0,bus=ide.0 \
    -netdev type=user,id=hn0,net=10.0.2.0/24,dhcpstart=10.0.2.20,hostfwd=tcp:127.0.0.1:2222-10.0.2.20:22 \
    -device e1000,netdev=hn0,id=nic1 \
    -netdev type=user,id=hn1,net=10.0.1.0/24,dhcpstart=10.0.1.10 \
    -device e1000,netdev=hn1,id=nic2
```
## Step 3 -  Configure network

3.1 Change ip address for lan interface to 10.0.2.20

```bash
ubus-cli IP.Interface.lan.IPv4Address.lan.IPAddress="10.0.2.20"

# IP.Interface.3.IPv4Address.1.
# IP.Interface.3.IPv4Address.1.IPAddress="10.0.2.20"
```

3.2 - Check change

```bash
ifconfig br-lan

# br-lan    Link encap:Ethernet  HWaddr A6:E1:06:24:AB:E5
#           inet addr:10.0.2.20  Bcast:0.0.0.0  Mask:255.255.255.0
#           inet6 addr: fe80::a4e1:6ff:fe24:abe5/64 Scope:Link
#           UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
#           RX packets:350 errors:0 dropped:0 overruns:0 frame:0
#           TX packets:565 errors:0 dropped:0 overruns:0 carrier:0
#           collisions:0 txqueuelen:1000
#           RX bytes:23448 (22.8 KiB)  TX bytes:170242 (166.2 KiB)

```

3.3 - Verify internet connectivity

```bash

netstat -arn

Kernel IP routing table
Destination     Gateway         Genmask         Flags   MSS Window  irtt Iface
0.0.0.0         10.0.1.2        0.0.0.0         UG        0 0          0 eth1
10.0.1.0        0.0.0.0         255.255.255.0   U         0 0          0 eth1
10.0.2.0        0.0.0.0         255.255.255.0   U         0 0          0 br-lan
192.168.2.0     0.0.0.0         255.255.255.0   U         0 0          0 br-guest
192.168.5.0     0.0.0.0         255.255.255.0   U         0 0          0 br-lcm

nslookup www.amazon.com

# Server:        127.0.0.1
# Address:    127.0.0.1:53

# Non-authoritative answer:
# www.amazon.com    canonical name = tp.47cf2c8c9-frontier.amazon.com
# tp.47cf2c8c9-frontier.amazon.com    canonical name = d3ag4hukkh62yn.cloudfront.net
# Name:    d3ag4hukkh62yn.cloudfront.net
# Address: 2600:9000:21ca:e400:7:49a5:5fd3:b641
# ...
# Name:    d3ag4hukkh62yn.cloudfront.net
# Address: 99.86.125.166

```
3.4 - Verify SSH daemon is running and listening on all interfaces. Restart daemon if the service is not online yet.

```bash
ps -efa | grep -i ssh 
# root      5813     1  0 08:11 ?        00:00:00 ssh_server -D /etc/amx/ssh_server/ssh_server.odl

netstat -aln | grep -i listen
# tcp        0      0 192.168.2.1:53          0.0.0.0:*               LISTEN      
# tcp        0      0 10.0.2.10:53            0.0.0.0:*               LISTEN      
# tcp        0      0 127.0.0.1:9999          0.0.0.0:*               LISTEN      
# tcp        0      0 192.168.5.1:53          0.0.0.0:*               LISTEN      
# tcp        0      0 127.0.0.1:53            0.0.0.0:*               LISTEN      
# tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN  <<------      

service ssh-server stop
#
service ssh-server start
#
```

3.5 - Login via SSH from host

>
> NOTE: Use a different terminal session for the next step
>

```bash
ssh root@127.0.0.1 -p 2222

# BusyBox v1.35.0 (2024-02-22 16:42:03 UTC) built-in shell (ash)
#                    _  ___  ____
#   _ __  _ __ _ __ | |/ _ \/ ___|
#  | '_ \| '__| '_ \| | | | \___ \
#  | |_) | |  | |_) | | |_| |___) |
#  | .__/|_|  | .__/|_|\___/|____/
#  |_|        |_| based on OpenWrt
#  --------------------------------
#       prplOS 3.0.1
#  --------------------------------

# === WARNING! =====================================
# There is no root password defined on this device!
# Use the "passwd" command to set up a new password
# in order to prevent unauthorized SSH logins.
# --------------------------------------------------
# root@prplOS:~# 

```

## Step 4 - Test container deployments

4.1 - Open ubus-cli and subscribe to SoftwareModules using `SoftwareModules.?&`

```bash

ubus-cli

# Copyright (c) 2020 - 2024 SoftAtHome
# amxcli version : 0.4.2

# !amx silent true

#                    _  ___  ____
#   _ __  _ __ _ __ | |/ _ \/ ___|
#  | '_ \| '__| '_ \| | | | \___ \
#  | |_) | |  | |_) | | |_| |___) |
#  | .__/|_|  | .__/|_|\___/|____/
#  |_|        |_| based on OpenWrt
#  -----------------------------------------------------
#  ubus - cli
#  -----------------------------------------------------

# root - ubus: - [ubus-cli] (0)
#  > SoftwareModules.?&
# Added subscription for SoftwareModules.
# >
```

4.2 Use the `InstallDU` method to deploy a sample container hosted at the Prpl Foundation

```bash
SoftwareModules.InstallDU(URL="docker://registry.gitlab.com/prpl-foundation/prplos/prplos/prplos/lcm-test-x86-64:prplos-v1", UUID="bd50974c-0fcf-48f0-82e3-2210d4a7ef70", ExecutionEnvRef="generic", NetworkConfig = { "AccessInterfaces" = [{"Reference" = "Lan"}] })
```

4.3 - Monitor incoming messages. Check that container was deployed successfully.

```bash
[2024-04-05T08:28:52Z] Event dm:object-changed received from SoftwareModules.
<  dm:object-changed> SoftwareModules.DeploymentUnitNumberOfEntries = 0 -> 1

[2024-04-05T08:28:52Z] Event dm:instance-added received from SoftwareModules.DeploymentUnit.
<  dm:instance-added> SoftwareModules.DeploymentUnit.1.VendorConfigList = 
<                   > SoftwareModules.DeploymentUnit.1.Status = Installing
<                   > SoftwareModules.DeploymentUnit.1.VendorLogList = 
<                   > SoftwareModules.DeploymentUnit.1.UUID = 0f032bd7-54bd-5b81-b14e-9441d730092f
<                   > SoftwareModules.DeploymentUnit.1.ExecutionEnvRef = Device.SoftwareModules.ExecEnv.1.
<                   > SoftwareModules.DeploymentUnit.1.Name = prpl-foundation/prplos/prplos/prplos/lcm-test-x86-64
<                   > SoftwareModules.DeploymentUnit.1.ModuleVersion = 
<                   > SoftwareModules.DeploymentUnit.1.ExecutionUnitList = 
<                   > SoftwareModules.DeploymentUnit.1.URL = docker://registry.gitlab.com/prpl-foundation/prplos/prplos/prplos/lcm-test-x86-64:prplos-v1
<                   > SoftwareModules.DeploymentUnit.1.Resolved = 1
<                   > SoftwareModules.DeploymentUnit.1.DUID = c879945e-d002-5775-88a8-e29bc0c641b4
<                   > SoftwareModules.DeploymentUnit.1.Description = Unknown
<                   > SoftwareModules.DeploymentUnit.1.Vendor = Unknown
<                   > SoftwareModules.DeploymentUnit.1.LastUpdate = 0001-01-01T00:00:00Z
<                   > SoftwareModules.DeploymentUnit.1.Alias = cpe-c879945e-d002-5775-88a8-e29bc0c641b4
<                   > SoftwareModules.DeploymentUnit.1.Version = prplos-v1
<                   > SoftwareModules.DeploymentUnit.1.Installed = 0001-01-01T00:00:00Z
SoftwareModules.InstallDU() returned
[
    ""
]

[2024-04-05T08:29:05Z] Event dm:object-changed received from SoftwareModules.DeploymentUnit.1.
<  dm:object-changed> SoftwareModules.DeploymentUnit.1.LastUpdate = 0001-01-01T00:00:00Z -> 1970-01-01T00:00:00Z
<                   > SoftwareModules.DeploymentUnit.1.Status = Installing -> Installed
<                   > SoftwareModules.DeploymentUnit.1.Installed = 0001-01-01T00:00:00Z -> 2024-04-05T08:28:52.497156177Z

[2024-04-05T08:29:05Z] Event DUStateChange! received from SoftwareModules.DeploymentUnit.1.
{
    data = {
        CurrentState = "",
        ExecutionEnvRef = "Device.SoftwareModules.ExecEnv.1.",
        Fault.FaultCode = 0,
        Fault.FaultString = "",
        OperationPerformed = "Install",
        Resolved = 1,
        UUID = "0f032bd7-54bd-5b81-b14e-9441d730092f",
        Version = "prplos-v1"
    },
    eobject = "SoftwareModules.DeploymentUnit.[cpe-c879945e-d002-5775-88a8-e29bc0c641b4].",
    notification = "DUStateChange!",
    object = "SoftwareModules.DeploymentUnit.cpe-c879945e-d002-5775-88a8-e29bc0c641b4.",
    path = "SoftwareModules.DeploymentUnit.1."
}

[2024-04-05T08:29:05Z] Event dm:object-changed received from SoftwareModules.
<  dm:object-changed> SoftwareModules.ExecutionUnitNumberOfEntries = 0 -> 1

[2024-04-05T08:29:05Z] Event dm:instance-added received from SoftwareModules.ExecutionUnit.
<  dm:instance-added> SoftwareModules.ExecutionUnit.1.VendorConfigList = 
<                   > SoftwareModules.ExecutionUnit.1.MemoryInUse = 1976
<                   > SoftwareModules.ExecutionUnit.1.RunLevel = 0
<                   > SoftwareModules.ExecutionUnit.1.AvailableMemory = -1
<                   > SoftwareModules.ExecutionUnit.1.AutoStart = 1
<                   > SoftwareModules.ExecutionUnit.1.ExecEnvLabel = c879945e-d002-5775-88a8-e29bc0c641b4
<                   > SoftwareModules.ExecutionUnit.1.ExecutionFaultMessage = 
<                   > SoftwareModules.ExecutionUnit.1.VendorLogList = 
<                   > SoftwareModules.ExecutionUnit.1.Status = Idle
<                   > SoftwareModules.ExecutionUnit.1.AllocatedDiskSpace = -1
<                   > SoftwareModules.ExecutionUnit.1.ExecutionEnvRef = generic
<                   > SoftwareModules.ExecutionUnit.1.Name = prpl-foundation/prplos/prplos/p
<                   > SoftwareModules.ExecutionUnit.1.AssociatedProcessList = 
<                   > SoftwareModules.ExecutionUnit.1.ExecutionFaultCode = NoFault
<                   > SoftwareModules.ExecutionUnit.1.AvailableDiskSpace = 4755
<                   > SoftwareModules.ExecutionUnit.1.Vendor = 
<                   > SoftwareModules.ExecutionUnit.1.Description = 
<                   > SoftwareModules.ExecutionUnit.1.AllocatedMemory = -1
<                   > SoftwareModules.ExecutionUnit.1.DiskSpaceInUse = 21
<                   > SoftwareModules.ExecutionUnit.1.AllocatedCPUPercent = 100
<                   > SoftwareModules.ExecutionUnit.1.EUID = c879945e-d002-5775-88a8-e29bc0c641b4
<                   > SoftwareModules.ExecutionUnit.1.References = 
<                   > SoftwareModules.ExecutionUnit.1.Alias = cpe-c879945e-d002-5775-88a8-e29bc0c641b4
<                   > SoftwareModules.ExecutionUnit.1.Version = prplos-v1

[2024-04-05T08:29:05Z] Event dm:object-added received from SoftwareModules.ExecutionUnit.1.NetworkConfig.
<    dm:object-added> SoftwareModules.ExecutionUnit.1.NetworkConfig.

[2024-04-05T08:29:05Z] Event dm:object-added received from SoftwareModules.ExecutionUnit.1.NetworkConfig.AccessInterfaces.
<    dm:object-added> SoftwareModules.ExecutionUnit.1.NetworkConfig.AccessInterfaces.

[2024-04-05T08:29:05Z] Event dm:object-added received from SoftwareModules.ExecutionUnit.1.NetworkConfig.PortForwarding.
<    dm:object-added> SoftwareModules.ExecutionUnit.1.NetworkConfig.PortForwarding.

[2024-04-05T08:29:05Z] Event dm:object-added received from SoftwareModules.ExecutionUnit.1.HostObject.
<    dm:object-added> SoftwareModules.ExecutionUnit.1.HostObject.

[2024-04-05T08:29:05Z] Event dm:object-added received from SoftwareModules.ExecutionUnit.1.Extensions.
<    dm:object-added> SoftwareModules.ExecutionUnit.1.Extensions.

[2024-04-05T08:29:06Z] Event dm:instance-added received from SoftwareModules.ExecutionUnit.1.NetworkConfig.AccessInterfaces.
<  dm:instance-added> SoftwareModules.ExecutionUnit.1.NetworkConfig.AccessInterfaces.1.Reference = Lan

[2024-04-05T08:29:06Z] Event dm:object-added received from SoftwareModules.ExecutionUnit.1.Plugins.
<    dm:object-added> SoftwareModules.ExecutionUnit.1.Plugins.

[2024-04-05T08:29:06Z] Event dm:object-changed received from SoftwareModules.ExecutionUnit.1.
<  dm:object-changed> SoftwareModules.ExecutionUnit.1.Status = Idle -> Starting

[2024-04-05T08:29:06Z] Event dm:object-changed received from SoftwareModules.ExecutionUnit.1.
<  dm:object-changed> SoftwareModules.ExecutionUnit.1.Status = Starting -> Active

root - ubus: - [ubus-cli] (0)
```

4.4 - Exit ubus-cli (Ctrl+D) and verify that the container has been deployed using `lxc-ls`

```bash
lxc-ls --fancy
# NAME                                 STATE   AUTOSTART GROUPS IPV4 IPV6 UNPRIVILEGED 
# c879945e-d002-5775-88a8-e29bc0c641b4 RUNNING 0         -      -    -    false    
```

4.5 - Check that container is running using `ubus-cli`

```bash
ubus-cli 'Cthulhu.Container.Instances.*.Status?'
# > Cthulhu.Container.Instances.*.Status?
# Cthulhu.Container.Instances.1.Status="Running"
```

4.6 - Check bundle information

```bash
ubus-cli 'Cthulhu.Container.Instances.*.Bundle?'
# > Cthulhu.Container.Instances.*.Bundle?
# Cthulhu.Container.Instances.1.Bundle="prpl-foundation/prplos/prplos/prplos/lcm-test-x86-64"
```

4.7 Attach to the container and check contents (CTRL+D to exit)

```bash
lxc-attach -n c879945e-d002-5775-88a8-e29bc0c641b4
# BusyBox v1.35.0 (2024-03-27 20:27:30 UTC) built-in shell (ash)

root@c879945e-d002-5775-88a8-e29bc0c641b4:~# cat /etc/os-release
# NAME="prplOS"
# VERSION="3.0.0-a847d6e8"
# ID="prplos"
# ID_LIKE="lede openwrt"
# PRETTY_NAME="prplOS 3.0.0-a847d6e8"
# VERSION_ID="3.0.0-a847d6e8"
# HOME_URL="https://prplfoundation.org"
# BUG_URL="https://jira.prplfoundation.org"
# SUPPORT_URL="https://jira.prplfoundation.org"
# BUILD_ID="r0+20614-a847d6e834"
# OPENWRT_BOARD="x86/64"
# OPENWRT_ARCH="x86_64"
# OPENWRT_TAINTS="no-all"
# OPENWRT_DEVICE_MANUFACTURER="prpl Foundation"
# OPENWRT_DEVICE_MANUFACTURER_URL="https://prplfoundation.org"
# OPENWRT_DEVICE_PRODUCT="Generic"
# OPENWRT_DEVICE_REVISION="v0"
# OPENWRT_RELEASE="prplOS 3.0.0-a847d6e8 r0+20614-a847d6e834"
# root@c879945e-d002-5775-88a8-e29bc0c641b4:~# 

```

## Step 5 -  Clean-up

5.1 - Stop container

```bash
ubus-cli 'SoftwareModules.ExecutionUnit.1.SetRequestedState(RequestedState = "Idle")'
# > SoftwareModules.ExecutionUnit.1.SetRequestedState(RequestedState = "Idle")
# SoftwareModules.ExecutionUnit.1.SetRequestedState() returned
# [
#     145
# ]
```

5.2 - Check that the container has stopped.

```bash
ubus-cli 'Cthulhu.Container.Instances.*.Status?'
# > Cthulhu.Container.Instances.*.Status?
# Cthulhu.Container.Instances.2.Status="Stopped"

lxc-ls  --fancy 
# NAME                                 STATE   AUTOSTART GROUPS IPV4 IPV6 UNPRIVILEGED 
# c879945e-d002-5775-88a8-e29bc0c641b4 STOPPED 0         -      -    -    false 
```


5.3 - Uninstall container

```bash
ubus-cli 'SoftwareModules.DeploymentUnit.1.Uninstall()'
# SoftwareModules.DeploymentUnit.1.Uninstall() returned
# [
#     ""
# ]

```

5.4 - Check that the container has been deleted
```bash
lxc-ls  --fancy 
#
```

## Step 6 - Setup a test registry service

6.1 - Setup the server

```bash

mkdir ~/docker_registry_certs 
cd ~/docker_registry_certs

openssl req \
    -x509 -newkey rsa:4096 \
    -keyout key.pem \
    -out cert.pem \
    -sha256 \
    -days 365 \
    -subj '/CN=myregistry.local' \
    -nodes

```

6.2 Start new container with the registry:2 application

```bash

sudo docker run -d \
    --name test-registry \
    -v ~/docker_registry_certs:/certs \
    -e REGISTRY_HTTP_ADDR=0.0.0.0:443 \
    -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/cert.pem \
    -e REGISTRY_HTTP_TLS_KEY=/certs/key.pem -p 443:443 \
    registry:2

```

## Step 7 - Setup the LCM SDK 

7.1 - Download the image that targets `x86-64` using tag `v3.0.0`

```bash
sudo docker run -t -d  \
    --name lcm_sdk \
    registry.gitlab.com/prpl-foundation/sdk/lcm/lcm_sdk_x86-64:v3.0.0
```

7.2 - Connect to the container and run a shell

```bash
sudo docker exec -it lcm_sdk /bin/bash
# lcmuser@dd2946e6e6a9:/sdkworkdir$ 
```

## Step 8 - Add the demo project to the SDK and build it

8.1 Use `devtool` to add the sample project to your workspace

```bash
devtool add \
    testhellow \
    https://github.com/TravorLZH/autotools-helloworld.git 
# NOTE: Starting bitbake server...
```
8.2 Use `devtool` to build the recipe

```bash
devtool build testhellow
# NOTE: Starting bitbake server...
# NOTE: Tasks Summary: Attempted 553 tasks of which 8 didn't need to be rerun and all succeeded.
```

8.2  Use `devtool` to build the image

```bash
devtool build-image
# NOTE: Starting bitbake server...
# INFO: Successfully built image-lcm-amx-ubus-usp-lcmsampleapp. You can find output files in /sdkworkdir/tmp/deploy/images/container-x86-64
```


## Step 9 - Copy the demo project's image to the container registry

9.1 - Copy the container image to the registry

```bash
skopeo copy \
    oci:/sdkworkdir/tmp/deploy/images/container-x86-64/image-lcm-amx-ubus-usp-lcmsampleapp-container-x86-64.rootfs-oci \
    docker://{_YOUR_EC2_PUBLIC_IP_}/helloworld:latest \
    --dest-tls-verify=false
```


## Step 10 - Deploy container to the prplOS instance

10.1 - Modify Rlyeh to not check for certificate verification `CertificateVerification`. and restart the service.

```bash
cat <<EOF > /etc/amx/rlyeh/rlyeh_defaults.odl 
%populate {
    object Rlyeh {
        parameter OnboardingFile = "/usr/share/rlyeh_onboarded";
        parameter ImageLocation = "/usr/share/rlyeh/images";
        parameter ROImageLocation = "/usr/rlyeh/images";
        parameter StorageLocation = "/usr/share/rlyeh/blobs";
        parameter ROStorageLocation = "/usr/rlyeh/blobs";
        parameter SignatureVerification = 1;
        parameter CertificateVerification = 0;
        parameter RemainingDiskSpaceBytes = 1000;
    }
}
EOF

service rlyeh stop
service rlyeh start

```

10.2 - Open ubus-cli and subscribe to SoftwareModules using `SoftwareModules.?&`

```bash
ubus-cli
#  > SoftwareModules.?&
# Added subscription for SoftwareModules.
# >
```

10.3 - Use the `InstallDU` method to deploy the custom container.

```bash
SoftwareModules.InstallDU(URL="docker://{_YOUR_EC2_PUBLIC_IP_}/helloworld:latest", UUID="bd50974c-0fcf-48f0-82e3-2210d4a7ef70", ExecutionEnvRef="generic", NetworkConfig = { "AccessInterfaces" = [{"Reference" = "Lan"}] })
```

10.4 - Exit ubus-cli (CTRL+D) and verify that the container has been deployed using `lxc-ls`

```bash
lxc-ls --fancy
# NAME                                 STATE   AUTOSTART GROUPS IPV4          IPV6 UNPRIVILEGED 
# c879945e-d002-5775-88a8-e29bc0c641b4 RUNNING 0         -      192.168.5.100 -    false    
```

10.5 - Check that container running

```bash
ubus-cli 'Cthulhu.Container.Instances.*.Status?'
# > Cthulhu.Container.Instances.*.Status?
# Cthulhu.Container.Instances.2.Status="Running"
```

10.6 - Check bundle information

```bash
ubus-cli 'Cthulhu.Container.Instances.*.Bundle?'
# > Cthulhu.Container.Instances.*.Bundle?
# Cthulhu.Container.Instances.2.Bundle="helloworld"
```

10.7 - Attach to the container and check contents (CTRL+D to exit)

```bash
lxc-attach -n c879945e-d002-5775-88a8-e29bc0c641b4
# root@c879945e-d002-5775-88a8-e29bc0c641b4:/# helloworld 
# Hello world!
# root@c879945e-d002-5775-88a8-e29bc0c641b4:/# helloconf
# Package Name: helloworld
# Package Version: 1.2
# Version Number: 0.1
# Bug Report: travor_lzh@outlook.com
```

## Step 11 Clean-up

11.1 - Stop container

```bash
ubus-cli 'SoftwareModules.ExecutionUnit.1.SetRequestedState(RequestedState = "Active")'
# > SoftwareModules.ExecutionUnit.1.SetRequestedState(RequestedState = "Idle")
# SoftwareModules.ExecutionUnit.1.SetRequestedState() returned
# [
#     145
# ]
```

11.2 - Check that the container has stopped.

```bash
ubus-cli 'Cthulhu.Container.Instances.*.Status?'
# > Cthulhu.Container.Instances.*.Status?
# Cthulhu.Container.Instances.2.Status="Stopped"

lxc-ls  --fancy 
# NAME                                 STATE   AUTOSTART GROUPS IPV4 IPV6 UNPRIVILEGED 
# c879945e-d002-5775-88a8-e29bc0c641b4 STOPPED 0         -      -    -    false 
```

11.3 - Uninstall container

```bash
ubus-cli 'SoftwareModules.DeploymentUnit.2.Uninstall()'
# > SoftwareModules.DeploymentUnit.2.Uninstall()
# SoftwareModules.DeploymentUnit.2.Uninstall() returned
# [
#     ""
# ]