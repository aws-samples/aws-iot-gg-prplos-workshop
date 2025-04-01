# Lab Two: Deploy Greengrass lite on prplOS

## Step 1 - Connect to the LCM SDK contaienr


1.1 - Attach to the SDK container

```bash
sudo docker exec -it lcm_sdk /bin/bash
# lcmuser@dd2946e6e6a9:/sdkworkdir$
```
## Step 2 - Build container image

3.1 - Enable greengrass
This are all changes necessary to enable systemd in the container and install greengrass lite.
`sdkworkdir/conf/local.conf`

```bash
DISTRO_FEATURES:append = " systemd usrmerge"
DISTRO_FEATURES_BACKFILL_CONSIDERED += "sysvinit"
VIRTUAL-RUNTIME_init_manager = "systemd"
VIRTUAL-RUNTIME_initscripts = "systemd-compat-units"
OCI_IMAGE_ENTRYPOINT_ARGS ?= "unified_cgroup_hierarchy=1"
IMAGE_INSTALL += "greengrass-lite"
```

3.2 - Build image
```bash
devtool build-image
# NOTE: Starting bitbake server...
# INFO: Successfully built image-lcm-amx-ubus-usp-lcmsampleapp. You can find output files in /sdkworkdir/tmp/deploy/images/container-x86-64
```

3.3 - Copy the container image to the registry

```bash
skopeo copy \
    oci:/sdkworkdir/tmp/deploy/images/container-x86-64/image-lcm-amx-ubus-usp-lcmsampleapp-container-x86-64.rootfs-oci \
    docker://{_YOUR_EC2_PUBLIC_IP_}/ggv2-testing:latest \
    --dest-tls-verify=false
```

## Step 4 - Deploy containner

4.1 - Open ubus-cli and subscribe to SoftwareModules using `SoftwareModules.?&`

```bash
ubus-cli
#  > SoftwareModules.?&
# Added subscription for SoftwareModules.
# >
```

4.2 - Create new execution environment and enable it

```bash
SoftwareModules.AddExecEnv(Name="generic-new", Vendor="Cthulhu", Version="3.5.2", ParentExecEnv="", InitialRunLevel=-1, AllocatedMemory=-1, AllocatedDiskSpace=1000000, AllocatedCPUPercent=100,  MaxBandwidthUpstream=-1, MaxBandwidthDownstream=-1)

SoftwareModules.ExecEnv.2.Enable=1


```

4.3 - Use the `InstallDU` method to deploy the custom container using the new execution environment

```bash
SoftwareModules.InstallDU(URL="docker://{_YOUR_EC2_PUBLIC_IP_}/ggv2-testing:latest", UUID="8e2ec7c2-4a8f-5ff1-b241-f2c0e275868b", ExecutionEnvRef="generic-new", NetworkConfig = { "AccessInterfaces" = [{"Reference" = "Wan"}] })
```

4.4 - Check the container status

```bash
lxc-ls --fancy
# NAME                                 STATE   AUTOSTART GROUPS IPV4          IPV6 UNPRIVILEGED
# c9c9f519-2719-544b-82e6-0571578b8154 RUNNING 0         -      192.168.5.100 -    false

```

4.5 - Attach to the container

```bash
lxc-attach  -n c9c9f519-2719-544b-82e6-0571578b8154
# root@a2fbbddf-e374-5235-bbfd-8bc7beb429bb:/#

```

## Step 5 - Configure Nucleus


5.1  - Check the version

```bash
java -jar /greengrass/v2/alts/init/distro/lib/Greengrass.jar --version
# AWS Greengrass v2.12.4


```

5.2 - Create config file

```bash
cat <<EOF > /greengrass/config.yaml
---
system:
  certificateFilePath: "/greengrass/device.pem.crt"
  privateKeyPath: "/greengrass/private.pem.key"
  rootCaPath: "/greengrass/AmazonRootCA1.pem"
  rootpath: "/greengrass/v2"
  thingName: "GGvLitePrplOS"
services:
  aws.greengrass.Nucleus:
    componentType: "NUCLEUS"
    version: "2.12.4"
    configuration:
      awsRegion: "eu-west-1"
      iotRoleAlias: "GreengrassV2TokenExchangeRoleAlias"
      iotDataEndpoint: "{_YOUR_IOT_CORE_ATS_ENDPOINT_}"
      iotCredEndpoint: "{_YOUR_IOT_CORE_CREDENTIALS_PROVIDER_ENDPOINT_}"
EOF

```
5.3 - Use the AWS IoT Console to:

1. Create an AWS IoT Thing called `GGvLitePrplOS`
2. Create an AWS IoT Policy
3. Create a certificate
4. Associate the IoT Policy you just created to the certificate and attach the certificate to the IoT thing you just created


5.4 - Copy the private key

```bash
cat <<EOF > /greengrass/private.pem.key
-----BEGIN RSA PRIVATE KEY-----
EXAMPLE
-----END RSA PRIVATE KEY-----
EOF
```

5.5 - Copy the certificate

```bash
cat <<EOF > /greengrass/device.pem.crt
-----BEGIN CERTIFICATE-----
EXAMPLE
-----END CERTIFICATE-----
EOF
```

5.6 - Copy the root CA

```bash
cat <<EOF > /greengrass/AmazonRootCA1.pem
-----BEGIN CERTIFICATE-----
EXAMPLE
-----END CERTIFICATE-----
EOF
```

5.7 - Start Nucleus

```bash
sudo -E java -Droot="/greengrass/v2" -Dlog.store=FILE \
	-jar /greengrass/v2/alts/init/distro/lib/Greengrass.jar \
	--init-config /greengrass/config.yaml
```